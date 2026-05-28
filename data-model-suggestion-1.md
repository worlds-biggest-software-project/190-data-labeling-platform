# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Data Labeling Platform · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form relational design, giving every domain concept its own table with strict foreign key enforcement. Every annotation type, quality metric, and workflow state is represented as a discrete relational entity with explicit constraints. The design prioritises data integrity, queryability, and standards alignment over schema flexibility.

Platforms like CVAT (Django/PostgreSQL) and traditional enterprise annotation systems use this approach. The CVAT hierarchy of Organisation > Project > Task > Job > Annotation maps cleanly onto normalised tables. This model is the natural choice when the annotation ontology is well-defined at deployment time and the platform must produce audit-grade provenance chains for EU AI Act Annex IV compliance.

**Best for:** Regulated environments (healthcare, government, automotive) where data integrity, referential consistency, and audit-grade provenance are non-negotiable requirements.

**Trade-offs:**
- (+) Full referential integrity enforced at the database level
- (+) Standard SQL queries for all reporting, no custom query languages needed
- (+) Clear schema documentation; every field has a defined type and constraint
- (+) Straightforward ORM mapping for Django/SQLAlchemy/Prisma
- (-) Adding new annotation modalities requires schema migrations (new tables or columns)
- (-) Higher table count increases JOIN complexity for cross-cutting queries
- (-) Multi-modal annotation data (bounding boxes vs. text spans vs. preference rankings) requires separate annotation detail tables or a discriminated union pattern
- (-) Schema rigidity can slow iteration during early product development

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C Web Annotation Data Model | Annotation, AnnotationBody, and AnnotationTarget tables mirror the Body/Target/Motivation structure |
| COCO Dataset Format | Export views map directly from normalised tables to COCO JSON (images, annotations, categories) |
| ISO/IEC 5259-2:2024 | Dataset-level quality measures stored in `dataset_version` and `quality_score` tables |
| ISO/IEC 27001 | `audit_log` table captures all state changes for security management |
| EU AI Act Annex IV | `annotation_provenance` table records labeling methodology, annotator identity, pre-label model version |
| Cohen's/Fleiss' Kappa | `iaa_score` table stores per-task agreement metrics with method and value columns |
| OAuth 2.0 / OIDC | `user` and `api_key` tables support token-based and key-based authentication |
| COCO/YOLO/VOC | `export_format` enum and normalised annotation structure enable deterministic format conversion |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),                         -- NULL for SSO-only users
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, google, github, oidc
    auth_provider_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'annotator', -- owner, admin, manager, reviewer, annotator
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
CREATE INDEX idx_org_member_user ON organisation_member(user_id);

CREATE TABLE api_key (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',           -- read, write, admin
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Project & Ontology Management

```sql
CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    modality        VARCHAR(50) NOT NULL,  -- image, text, audio, video, llm_pair, multi
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- draft, active, paused, completed, archived
    guidelines_md   TEXT,                  -- Markdown labeling instructions
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_project_org ON project(organisation_id);
CREATE INDEX idx_project_status ON project(status);

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
    parent_id       UUID REFERENCES ontology_feature(id),  -- for nested classifications
    name            VARCHAR(255) NOT NULL,
    feature_type    VARCHAR(50) NOT NULL,   -- object, classification, relationship
    tool            VARCHAR(50),            -- bounding_box, polygon, keypoint, ner_span, text_class, ranking
    color           VARCHAR(7),             -- hex color for UI
    sort_order      INT NOT NULL DEFAULT 0,
    constraints     JSONB NOT NULL DEFAULT '{}',
    -- constraints example: {"min_points": 3, "options": ["positive", "negative", "neutral"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ontology_feature_ontology ON ontology_feature(ontology_id);
```

## Data Management

```sql
CREATE TABLE dataset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    storage_backend VARCHAR(50) NOT NULL DEFAULT 'local', -- local, s3, gcs, azure_blob
    storage_config  JSONB NOT NULL DEFAULT '{}',
    -- storage_config example: {"bucket": "my-bucket", "prefix": "datasets/v1/", "region": "us-east-1"}
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_row (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES dataset(id) ON DELETE CASCADE,
    external_id     VARCHAR(512),           -- user-supplied identifier
    media_type      VARCHAR(50) NOT NULL,   -- image/png, image/jpeg, text/plain, audio/wav, video/mp4, application/dicom
    file_path       TEXT NOT NULL,           -- relative path or object key
    file_size_bytes BIGINT,
    width           INT,                     -- for images/video
    height          INT,                     -- for images/video
    duration_ms     BIGINT,                  -- for audio/video
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"original_filename": "scan_001.dcm", "patient_id_hash": "abc123", "capture_date": "2026-01-15"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_row_dataset ON data_row(dataset_id);
CREATE INDEX idx_data_row_external ON data_row(external_id);

CREATE TABLE dataset_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES dataset(id) ON DELETE CASCADE,
    version_number  INT NOT NULL,
    description     TEXT,
    row_count       INT NOT NULL DEFAULT 0,
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dataset_id, version_number)
);

CREATE TABLE dataset_version_row (
    dataset_version_id UUID NOT NULL REFERENCES dataset_version(id) ON DELETE CASCADE,
    data_row_id        UUID NOT NULL REFERENCES data_row(id) ON DELETE CASCADE,
    PRIMARY KEY (dataset_version_id, data_row_id)
);
```

## Task & Workflow Management

```sql
CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    -- queued, assigned, in_progress, submitted, in_review, accepted, rejected, skipped
    priority        INT NOT NULL DEFAULT 0,         -- higher = more urgent
    assigned_to     UUID REFERENCES "user"(id),
    reviewer_id     UUID REFERENCES "user"(id),
    round           INT NOT NULL DEFAULT 1,         -- re-annotation round
    pre_label_model VARCHAR(255),                   -- model used for pre-labeling, if any
    pre_label_confidence FLOAT,
    submitted_at    TIMESTAMPTZ,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_project ON task(project_id);
CREATE INDEX idx_task_status ON task(status);
CREATE INDEX idx_task_assigned ON task(assigned_to);
CREATE INDEX idx_task_data_row ON task(data_row_id);

CREATE TABLE workflow_stage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    stage_type      VARCHAR(50) NOT NULL,   -- label, review, qa_check, export
    stage_order     INT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {"min_reviewers": 2, "auto_accept_confidence": 0.95}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE task_stage_transition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    from_stage_id   UUID REFERENCES workflow_stage(id),
    to_stage_id     UUID NOT NULL REFERENCES workflow_stage(id),
    triggered_by    UUID NOT NULL REFERENCES "user"(id),
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_stage_task ON task_stage_transition(task_id);
```

## Annotations

```sql
CREATE TABLE annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    ontology_feature_id UUID NOT NULL REFERENCES ontology_feature(id),
    annotator_id    UUID NOT NULL REFERENCES "user"(id),
    annotation_type VARCHAR(50) NOT NULL,   -- bounding_box, polygon, keypoint, ner_span, classification, ranking
    is_pre_label    BOOLEAN NOT NULL DEFAULT false,
    confidence      FLOAT,                  -- model confidence for pre-labels
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotation_task ON annotation(task_id);
CREATE INDEX idx_annotation_annotator ON annotation(annotator_id);
CREATE INDEX idx_annotation_type ON annotation(annotation_type);

-- Spatial annotations (bounding boxes, polygons, keypoints)
CREATE TABLE annotation_spatial (
    annotation_id   UUID PRIMARY KEY REFERENCES annotation(id) ON DELETE CASCADE,
    bbox_x          FLOAT,     -- top-left x (normalised 0-1)
    bbox_y          FLOAT,     -- top-left y (normalised 0-1)
    bbox_w          FLOAT,     -- width (normalised 0-1)
    bbox_h          FLOAT,     -- height (normalised 0-1)
    polygon_points  FLOAT[][],  -- array of [x, y] pairs
    keypoints       FLOAT[][],  -- array of [x, y, visibility] triples
    rotation        FLOAT DEFAULT 0,
    frame_index     INT         -- for video: which frame
);

-- Text span annotations (NER, text classification)
CREATE TABLE annotation_text_span (
    annotation_id   UUID PRIMARY KEY REFERENCES annotation(id) ON DELETE CASCADE,
    start_offset    INT NOT NULL,
    end_offset      INT NOT NULL,
    text_content    TEXT        -- the selected text
);

-- Classification annotations (whole-item labels)
CREATE TABLE annotation_classification (
    annotation_id   UUID PRIMARY KEY REFERENCES annotation(id) ON DELETE CASCADE,
    value           VARCHAR(255) NOT NULL,
    free_text       TEXT        -- for free-form text classifications
);

-- LLM preference / ranking annotations (RLHF)
CREATE TABLE annotation_ranking (
    annotation_id   UUID PRIMARY KEY REFERENCES annotation(id) ON DELETE CASCADE,
    prompt_text     TEXT NOT NULL,
    response_a_id   UUID,       -- reference to a stored LLM response
    response_b_id   UUID,
    chosen          VARCHAR(1) NOT NULL CHECK (chosen IN ('A', 'B', 'tie')),
    rationale       TEXT,
    criteria        TEXT[]      -- e.g., ['helpfulness', 'harmlessness', 'honesty']
);
```

## Quality Control

```sql
CREATE TABLE iaa_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    task_id         UUID REFERENCES task(id),
    ontology_feature_id UUID REFERENCES ontology_feature(id),
    metric          VARCHAR(50) NOT NULL,   -- cohen_kappa, fleiss_kappa, krippendorff_alpha, iou
    value           FLOAT NOT NULL,
    annotator_pair  UUID[],                 -- for pairwise metrics: [user_id_a, user_id_b]
    sample_size     INT NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iaa_project ON iaa_score(project_id);
CREATE INDEX idx_iaa_task ON iaa_score(task_id);

CREATE TABLE review_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    annotation_id   UUID REFERENCES annotation(id),
    reviewer_id     UUID NOT NULL REFERENCES "user"(id),
    decision        VARCHAR(50) NOT NULL,   -- accept, reject, fix, escalate
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_task ON review_decision(task_id);

CREATE TABLE qa_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    rule_type       VARCHAR(50) NOT NULL,   -- python_script, threshold, regex
    definition      TEXT NOT NULL,           -- Python code, threshold expression, or regex
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE qa_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    qa_rule_id      UUID NOT NULL REFERENCES qa_rule(id) ON DELETE CASCADE,
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    passed          BOOLEAN NOT NULL,
    message         TEXT,
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Active Learning

```sql
CREATE TABLE al_query_strategy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    strategy_type   VARCHAR(50) NOT NULL,   -- uncertainty, diversity, influence, random
    model_endpoint  TEXT,                    -- URL of the model used for scoring
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE al_sample_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id     UUID NOT NULL REFERENCES al_query_strategy(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    score           FLOAT NOT NULL,          -- uncertainty or informativeness score
    model_version   VARCHAR(255),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_al_score_strategy ON al_sample_score(strategy_id);
CREATE INDEX idx_al_score_data_row ON al_sample_score(data_row_id);
CREATE INDEX idx_al_score_value ON al_sample_score(score DESC);
```

## Provenance & Audit

```sql
CREATE TABLE annotation_provenance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    annotation_id   UUID NOT NULL REFERENCES annotation(id) ON DELETE CASCADE,
    annotator_id    UUID NOT NULL REFERENCES "user"(id),
    ontology_version INT NOT NULL,
    guidelines_hash VARCHAR(64),             -- SHA-256 of guidelines document at annotation time
    pre_label_model VARCHAR(255),
    pre_label_model_version VARCHAR(128),
    annotation_duration_ms INT,              -- time spent annotating
    client_info     JSONB NOT NULL DEFAULT '{}',
    -- client_info example: {"user_agent": "...", "screen_resolution": "1920x1080"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    actor_id        UUID REFERENCES "user"(id),
    action          VARCHAR(100) NOT NULL,   -- annotation.created, task.assigned, project.updated, etc.
    resource_type   VARCHAR(100) NOT NULL,   -- annotation, task, project, dataset, user
    resource_id     UUID NOT NULL,
    old_value       JSONB,
    new_value       JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Export & Webhooks

```sql
CREATE TABLE export_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    format          VARCHAR(50) NOT NULL,    -- coco, yolo, pascal_voc, json, csv, darwin
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    filters         JSONB NOT NULL DEFAULT '{}',
    file_path       TEXT,                    -- path to generated export file
    row_count       INT,
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,          -- task.completed, annotation.created, export.ready, etc.
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id      UUID NOT NULL REFERENCES webhook(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    response_code   INT,
    response_body   TEXT,
    delivered_at    TIMESTAMPTZ,
    retry_count     INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisation, user, organisation_member, api_key |
| Project & Ontology | 3 | project, ontology, ontology_feature |
| Data Management | 4 | dataset, data_row, dataset_version, dataset_version_row |
| Task & Workflow | 3 | task, workflow_stage, task_stage_transition |
| Annotations | 5 | annotation + 4 type-specific detail tables |
| Quality Control | 4 | iaa_score, review_decision, qa_rule, qa_result |
| Active Learning | 2 | al_query_strategy, al_sample_score |
| Provenance & Audit | 2 | annotation_provenance, audit_log |
| Export & Webhooks | 3 | export_job, webhook, webhook_delivery |
| **Total** | **30** | |

---

## Key Design Decisions

1. **Discriminated union for annotations** — Rather than a single wide annotation table with nullable columns for every modality, the design uses a base `annotation` table joined to type-specific detail tables (`annotation_spatial`, `annotation_text_span`, `annotation_classification`, `annotation_ranking`). This enforces type-correct constraints at the database level while keeping the base table lean for cross-cutting queries.

2. **Ontology versioning** — Ontologies are versioned per project. When the label schema changes, a new ontology version is created rather than mutating existing features. This ensures historical annotations reference the exact ontology they were created under, supporting EU AI Act Annex IV compliance.

3. **Dataset versioning via junction table** — `dataset_version_row` is a many-to-many mapping that allows dataset versions to share data rows without duplication. A version is an immutable snapshot of which rows are included.

4. **Normalised IAA scores** — Inter-annotator agreement metrics are stored per task, per feature, per metric type. This enables fine-grained quality analysis (e.g., "Cohen's Kappa for 'vehicle' bounding boxes in project X is 0.72") without JSONB parsing.

5. **Separate provenance table** — `annotation_provenance` captures the full context of each annotation action (guidelines version, model version, time spent) as a separate record from the annotation itself. This keeps the annotation table fast for rendering while supporting Annex IV-compliant audit export.

6. **RBAC at organisation level** — Roles are assigned per organisation membership, not globally. The `organisation_member.role` column supports a simple role hierarchy. Project-level permissions can be layered on top via a `project_member` table if needed.

7. **Webhook delivery tracking** — Every webhook invocation is logged with response code and retry count, enabling debugging and ensuring pipeline reliability for MLOps loop integration.
