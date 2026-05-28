# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Data Labeling Platform · Created: 2026-05-20

## Philosophy

This model adds a property graph layer on top of a relational foundation to address the relationship-heavy aspects of a data labeling platform that relational models handle poorly: annotation provenance chains, annotator-quality networks, ontology hierarchies, data lineage tracking, and cross-project dataset reuse. The relational layer handles operational CRUD (tasks, users, projects), while graph edges capture the rich, queryable relationships between entities.

The graph layer is implemented as two PostgreSQL tables (`graph_node` and `graph_edge`) using the property graph pattern, avoiding the need for a separate graph database like Neo4j. This keeps the deployment simple (single PostgreSQL instance) while enabling recursive CTE-based graph traversals for provenance chains, impact analysis, and relationship queries. PostgreSQL's `ltree` extension is used for hierarchical ontology paths.

This approach is inspired by knowledge graph systems, supply chain traceability platforms, and biomedical annotation systems where data lineage and provenance are first-class concerns. For a data labeling platform, the graph layer answers questions like: "Which model versions were trained on annotations by this low-performing annotator?", "Trace the full provenance chain from this model prediction back through all labeling decisions", and "Which datasets share data rows with this contaminated batch?"

**Best for:** Platforms focused on data lineage, provenance analysis, annotator quality networks, cross-project dataset traceability, and regulatory environments that require end-to-end auditability from training data to model deployment. Also suited for platforms where conflict-of-interest or bias detection in annotator pools is important.

**Trade-offs:**
- (+) Rich provenance queries via graph traversal (recursive CTEs)
- (+) Cross-entity relationship analysis without complex JOINs across many tables
- (+) Data lineage tracking from raw data through annotation to model training
- (+) Annotator quality networks: detect bias clusters, systematic disagreement patterns
- (+) Single PostgreSQL deployment — no external graph database required
- (+) Graph edges can represent any relationship type; extensible without schema changes
- (-) Graph queries (recursive CTEs) are more complex to write and optimise than flat SQL
- (-) The `graph_edge` table can grow very large in high-volume annotation workflows
- (-) Property graph pattern in PostgreSQL lacks the query ergonomics of a native graph database (Cypher, Gremlin)
- (-) Dual storage (relational + graph) means some data is denormalised across the two layers
- (-) Developers must understand both relational and graph query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C Web Annotation Data Model | Annotation Body/Target relationships modeled as graph edges between annotation and data_row nodes |
| EU AI Act Annex IV | Full provenance graph from data ingestion through annotation to export; traversable via recursive CTEs |
| ISO/IEC 5259-2:2024 | Dataset quality lineage tracked as graph paths connecting quality metrics to annotation batches |
| ISO/IEC 27001 | Graph-based audit trail enables impact analysis: "which downstream systems are affected by this data breach?" |
| Cohen's/Fleiss' Kappa | IAA scores stored relationally; annotator agreement patterns discoverable via graph analysis of annotator-task relationships |
| COCO/YOLO/VOC | Export reads from relational annotation tables; graph layer provides provenance metadata for Annex IV-compliant exports |
| ltree (PostgreSQL) | Ontology feature hierarchies stored as materialised paths for efficient ancestor/descendant queries |

---

## Graph Layer

```sql
-- Enable ltree extension for hierarchical paths
CREATE EXTENSION IF NOT EXISTS ltree;

-- ============================================================
-- PROPERTY GRAPH: Nodes
-- ============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(100) NOT NULL,
    -- Node types: organisation, user, project, dataset, data_row, task,
    --             annotation, ontology, model_version, export_batch,
    --             qa_rule, review_decision, al_strategy
    entity_id       UUID NOT NULL,            -- FK to the relational entity
    label           VARCHAR(255),             -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"created_at": "2026-05-01", "status": "accepted", "confidence": 0.92}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- ============================================================
-- PROPERTY GRAPH: Edges
-- ============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    --   belongs_to          (user -> organisation, project -> organisation)
    --   contains            (dataset -> data_row, project -> task)
    --   assigned_to         (task -> user)
    --   annotated_by        (annotation -> user)
    --   annotation_of       (annotation -> data_row)
    --   reviewed_by         (review_decision -> user)
    --   review_of           (review_decision -> annotation)
    --   pre_labeled_by      (annotation -> model_version)
    --   trained_on          (model_version -> export_batch)
    --   exported_from       (export_batch -> project)
    --   derived_from        (dataset_v2 -> dataset_v1)
    --   uses_ontology       (project -> ontology)
    --   parent_feature      (ontology_feature -> ontology_feature)
    --   triggered_by        (qa_result -> qa_rule)
    --   selected_by         (task -> al_strategy)
    --   shares_data_with    (dataset_a -> dataset_b, via shared data_rows)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"role": "reviewer", "confidence": 0.87, "at": "2026-05-01T10:30:00Z"}
    weight          FLOAT DEFAULT 1.0,        -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING GIN (properties jsonb_path_ops);
```

## Relational Core: Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_provider_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'annotator',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
CREATE INDEX idx_org_member_user ON organisation_member(user_id);
```

## Relational Core: Project & Ontology (with ltree)

```sql
CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    modality        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_project_org ON project(organisation_id);

CREATE TABLE ontology (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    version         INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE TABLE ontology_feature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ontology_id     UUID NOT NULL REFERENCES ontology(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    feature_type    VARCHAR(50) NOT NULL,     -- object, classification, relationship
    tool            VARCHAR(50),              -- bounding_box, polygon, keypoint, etc.
    color           VARCHAR(7),
    path            LTREE NOT NULL,           -- hierarchical path, e.g., 'vehicle.car.sedan'
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_of_ontology ON ontology_feature(ontology_id);
CREATE INDEX idx_of_path ON ontology_feature USING GIST (path);

-- Example ltree queries:
-- All features under 'vehicle': SELECT * FROM ontology_feature WHERE path <@ 'vehicle';
-- Direct children of 'vehicle': SELECT * FROM ontology_feature WHERE path ~ 'vehicle.*{1}';
-- All ancestors of 'vehicle.car.sedan': SELECT * FROM ontology_feature WHERE 'vehicle.car.sedan' <@ path;
```

## Relational Core: Data & Tasks

```sql
CREATE TABLE dataset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    storage_config  JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_row (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES dataset(id) ON DELETE CASCADE,
    external_id     VARCHAR(512),
    media_type      VARCHAR(100) NOT NULL,
    file_path       TEXT NOT NULL,
    file_size_bytes BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_row_dataset ON data_row(dataset_id);
CREATE INDEX idx_data_row_external ON data_row(external_id);

CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    priority        INT NOT NULL DEFAULT 0,
    assigned_to     UUID REFERENCES "user"(id),
    reviewer_id     UUID REFERENCES "user"(id),
    round           INT NOT NULL DEFAULT 1,
    submitted_at    TIMESTAMPTZ,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_project ON task(project_id);
CREATE INDEX idx_task_status ON task(status);
CREATE INDEX idx_task_assigned ON task(assigned_to);

CREATE TABLE annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    feature_id      UUID NOT NULL REFERENCES ontology_feature(id),
    annotator_id    UUID NOT NULL REFERENCES "user"(id),
    annotation_type VARCHAR(50) NOT NULL,
    content         JSONB NOT NULL,
    is_pre_label    BOOLEAN NOT NULL DEFAULT false,
    confidence      FLOAT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotation_task ON annotation(task_id);
CREATE INDEX idx_annotation_annotator ON annotation(annotator_id);
CREATE INDEX idx_annotation_content ON annotation USING GIN (content jsonb_path_ops);

CREATE TABLE review_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    annotation_id   UUID REFERENCES annotation(id),
    reviewer_id     UUID NOT NULL REFERENCES "user"(id),
    decision        VARCHAR(50) NOT NULL,
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE model_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(128) NOT NULL,
    model_type      VARCHAR(100),            -- yolov8, bert, gpt-4o, custom
    endpoint_url    TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE export_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    format          VARCHAR(50) NOT NULL,
    row_count       INT,
    file_path       TEXT,
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Quality Control

```sql
CREATE TABLE iaa_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    task_id         UUID REFERENCES task(id),
    feature_id      UUID REFERENCES ontology_feature(id),
    metric          VARCHAR(50) NOT NULL,
    value           FLOAT NOT NULL,
    annotator_pair  UUID[],
    sample_size     INT NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iaa_project ON iaa_score(project_id);

CREATE TABLE al_strategy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE al_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id     UUID NOT NULL REFERENCES al_strategy(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    score           FLOAT NOT NULL,
    model_version_id UUID REFERENCES model_version(id),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_al_score_strategy ON al_score(strategy_id);
CREATE INDEX idx_al_score_value ON al_score(score DESC);
```

## Audit

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    actor_id        UUID REFERENCES "user"(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Graph Queries (Recursive CTEs)

```sql
-- ============================================================
-- PROVENANCE CHAIN: Trace an annotation back to its data source
-- ============================================================
-- "Show the full provenance of this annotation: who labeled it,
--  what model pre-labeled it, what dataset it came from, and
--  what export batch included it."

WITH RECURSIVE provenance AS (
    -- Start from the annotation node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        gn.properties,
        ge.edge_type,
        0 AS depth
    FROM graph_node gn
    WHERE gn.node_type = 'annotation' AND gn.entity_id = '{{annotation_uuid}}'

    UNION ALL

    -- Traverse outgoing edges
    SELECT
        target.id AS node_id,
        target.node_type,
        target.label,
        target.properties,
        ge.edge_type,
        p.depth + 1
    FROM provenance p
    JOIN graph_edge ge ON ge.source_id = p.node_id
    JOIN graph_node target ON target.id = ge.target_id
    WHERE p.depth < 10  -- prevent infinite recursion
)
SELECT * FROM provenance ORDER BY depth;

-- ============================================================
-- IMPACT ANALYSIS: Which models were trained on data from a
-- specific annotator?
-- ============================================================

WITH RECURSIVE downstream AS (
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.entity_id,
        gn.label,
        ge.edge_type,
        0 AS depth
    FROM graph_node gn
    WHERE gn.node_type = 'user' AND gn.entity_id = '{{annotator_uuid}}'

    UNION ALL

    SELECT
        target.id,
        target.node_type,
        target.entity_id,
        target.label,
        ge.edge_type,
        d.depth + 1
    FROM downstream d
    JOIN graph_edge ge ON ge.target_id = d.node_id  -- reverse: who points TO this node
    JOIN graph_node target ON target.id = ge.source_id
    WHERE d.depth < 10
      AND ge.edge_type IN ('annotated_by', 'exported_from', 'trained_on')
)
SELECT DISTINCT node_type, entity_id, label
FROM downstream
WHERE node_type = 'model_version';

-- ============================================================
-- ANNOTATOR QUALITY NETWORK: Find annotators who frequently
-- disagree with each other
-- ============================================================

SELECT
    u1.display_name AS annotator_a,
    u2.display_name AS annotator_b,
    iaa.metric,
    iaa.value AS agreement_score,
    iaa.sample_size
FROM iaa_score iaa
JOIN "user" u1 ON u1.id = iaa.annotator_pair[1]
JOIN "user" u2 ON u2.id = iaa.annotator_pair[2]
WHERE iaa.project_id = '{{project_uuid}}'
  AND iaa.metric = 'cohen_kappa'
  AND iaa.value < 0.40  -- poor agreement threshold
ORDER BY iaa.value ASC;

-- ============================================================
-- DATASET LINEAGE: Which datasets share data rows?
-- ============================================================

SELECT
    source_node.label AS dataset_a,
    target_node.label AS dataset_b,
    (ge.properties->>'shared_row_count')::INT AS shared_rows
FROM graph_edge ge
JOIN graph_node source_node ON source_node.id = ge.source_id
JOIN graph_node target_node ON target_node.id = ge.target_id
WHERE ge.edge_type = 'shares_data_with'
ORDER BY shared_rows DESC;

-- ============================================================
-- ONTOLOGY HIERARCHY: Find all features under "vehicle"
-- using ltree
-- ============================================================

SELECT id, name, feature_type, tool, path
FROM ontology_feature
WHERE ontology_id = '{{ontology_uuid}}'
  AND path <@ 'vehicle'
ORDER BY path;

-- ============================================================
-- CONTAMINATION CHECK: Find all tasks/annotations downstream
-- of a contaminated data_row
-- ============================================================

WITH RECURSIVE contaminated AS (
    SELECT gn.id AS node_id, gn.node_type, gn.entity_id, 0 AS depth
    FROM graph_node gn
    WHERE gn.node_type = 'data_row' AND gn.entity_id = '{{data_row_uuid}}'

    UNION ALL

    SELECT target.id, target.node_type, target.entity_id, c.depth + 1
    FROM contaminated c
    JOIN graph_edge ge ON ge.target_id = c.node_id
    JOIN graph_node target ON target.id = ge.source_id
    WHERE c.depth < 10
)
SELECT node_type, COUNT(*) AS affected_count
FROM contaminated
WHERE node_type IN ('task', 'annotation', 'export_batch', 'model_version')
GROUP BY node_type;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Identity & Multi-Tenancy | 3 | organisation, user, organisation_member |
| Project & Ontology | 3 | project, ontology, ontology_feature (with ltree) |
| Data Management | 2 | dataset, data_row |
| Tasks & Annotations | 3 | task, annotation, review_decision |
| Model & Export Tracking | 2 | model_version, export_batch |
| Quality & Active Learning | 3 | iaa_score, al_strategy, al_score |
| Audit | 1 | audit_log |
| **Total** | **19** | 2 graph + 17 relational |

---

## Key Design Decisions

1. **Property graph in PostgreSQL** — The `graph_node`/`graph_edge` tables implement the property graph pattern within PostgreSQL, avoiding the operational complexity of deploying and synchronising a separate graph database. Recursive CTEs provide traversal capability. For deployments that outgrow PostgreSQL graph performance, the same node/edge data can be migrated to Neo4j or Apache AGE without changing the relational layer.

2. **Graph edges as relationship first-class citizens** — Every significant relationship (annotated_by, pre_labeled_by, trained_on, exported_from) is captured as a graph edge with typed properties. This enables provenance chains, impact analysis, and contamination checks that would require many-table JOINs in a purely relational model.

3. **ltree for ontology hierarchies** — PostgreSQL's `ltree` extension provides native support for hierarchical paths with efficient ancestor, descendant, and pattern-matching queries. The `path` column (e.g., `vehicle.car.sedan`) replaces the self-referential `parent_id` pattern, which requires recursive CTEs for every hierarchy query. With ltree, "find all features under vehicle" is a single `<@` operator query.

4. **Dual storage with clear separation** — The relational tables are the operational source of truth for CRUD operations (create a task, submit an annotation, assign a reviewer). The graph layer is maintained by application-level event handlers that create edges when relationships form. This means the graph can be rebuilt from relational data if needed, though it would lose edge properties not derivable from the relational state.

5. **Model version tracking** — The `model_version` table and its graph edges (`pre_labeled_by`, `trained_on`) close the loop between annotation and model training. This is critical for the platform's AI-native positioning: you can trace which annotations influenced which model versions and detect when a low-quality annotator's work has contaminated a production model.

6. **Weight column on edges** — The `graph_edge.weight` column supports weighted graph algorithms (PageRank for annotator influence, shortest path for provenance chains, community detection for bias clusters). This enables advanced quality analytics without modifying the schema.

7. **Annotations as JSONB content** — Like the Hybrid model (Suggestion 3), annotations use a single table with JSONB content. This keeps the relational core lean while the graph layer handles the relationship complexity. The annotation content itself doesn't need to be relational — the relationships around it do.

8. **Graph-enabled contamination checks** — When a data row, annotator, or model version is flagged as problematic, a single recursive CTE traversal can identify all downstream entities affected. This is a capability that purely relational models can only approximate with multiple manual JOINs across specific table pairs.
