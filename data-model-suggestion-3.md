# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Data Labeling Platform · Created: 2026-05-20

## Philosophy

This model uses relational tables for the structural backbone (projects, tasks, users, workflows) while storing annotation content, ontology definitions, and variable metadata in PostgreSQL JSONB columns. The key insight is that annotation data varies wildly across modalities (bounding boxes, text spans, preference rankings, keypoints, segmentation masks) and across deployment contexts (healthcare metadata vs. autonomous vehicle sensor data vs. NLP corpus attributes), making a fixed relational schema either too rigid or too wide.

This is the approach used by Label Studio, which stores annotation content as raw JSON within PostgreSQL while using relational tables for project structure, task management, and user accounts. Argilla similarly stores annotation payloads as flexible JSON/Arrow structures. The pattern is well-suited to platforms that need to support rapid experimentation with new annotation types, multi-modal projects, and jurisdiction-specific metadata extensions without schema migrations.

JSONB columns are indexed using GIN indexes for containment queries and BTREE indexes on commonly extracted paths. The relational layer provides referential integrity for the structural entities, while JSONB provides schema flexibility for the content layer.

**Best for:** Early-stage platforms that need to ship fast, support multiple annotation modalities from day one, and iterate on the data model without downtime. Also well-suited for multi-tenant platforms where different organisations have different metadata requirements.

**Trade-offs:**
- (+) Single annotation table handles all modalities — no discriminated union, no new tables per type
- (+) New annotation types can be added without schema migrations
- (+) Organisation-specific and jurisdiction-specific metadata stored naturally in JSONB
- (+) Faster development iteration: change the JSON shape in code, not in migration files
- (+) PostgreSQL GIN indexes provide efficient JSONB containment queries
- (-) No database-level type enforcement on annotation content — validation must live in application code
- (-) JSONB queries can be slower than relational queries for complex aggregations
- (-) Harder to generate correct COCO/YOLO exports without intermediate validation of JSON shapes
- (-) GIN indexes consume more storage than BTREE indexes on typed columns
- (-) Schema drift risk: different annotation versions may produce subtly different JSON structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C Web Annotation Data Model | Annotation JSONB payload follows Body/Target structure; `motivation` field aligns with W3C vocabulary |
| COCO Dataset Format | Export function reads from JSONB `content` field, maps to COCO `annotations[]` array |
| JSON Schema (Draft 2020-12) | `ontology.schema_definition` stores JSON Schema for runtime validation of annotation payloads |
| ISO/IEC 5259-2:2024 | Dataset quality metrics stored as relational records; computed from JSONB annotation content |
| EU AI Act Annex IV | `task.provenance` JSONB field captures full annotation context per task |
| ISO/IEC 27001 | `audit_log` table with relational structure for security compliance |
| OpenAPI 3.1 | API request/response schemas reference the JSON Schema in ontology definitions |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "default_storage": "s3",
    --   "allowed_modalities": ["image", "text", "llm_pair"],
    --   "data_residency": "eu-west-1",
    --   "sso_provider": "okta",
    --   "custom_fields": {"department": "string", "cost_center": "string"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_provider_id VARCHAR(255),
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example: {"languages": ["en", "de"], "expertise": ["medical", "cv"], "hourly_rate": 25.00}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'annotator',
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example: {"projects": ["read", "write"], "datasets": ["read"], "admin": false}
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
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
    scopes          JSONB NOT NULL DEFAULT '[]',
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Project & Ontology

```sql
CREATE TABLE project (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    modality        VARCHAR(50) NOT NULL,    -- image, text, audio, video, llm_pair, multi
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "assignment_strategy": "round_robin",
    --   "min_annotations_per_task": 3,
    --   "auto_accept_confidence": 0.95,
    --   "enable_active_learning": true,
    --   "export_formats": ["coco", "yolo"],
    --   "guidelines_md": "## Instructions\n\nLabel all vehicles...",
    --   "guidelines_hash": "sha256:abc123..."
    -- }
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
    schema_definition JSONB NOT NULL,
    -- schema_definition example (JSON Schema for annotation validation):
    -- {
    --   "features": [
    --     {
    --       "id": "feat_vehicle",
    --       "name": "Vehicle",
    --       "type": "object",
    --       "tool": "bounding_box",
    --       "color": "#FF0000",
    --       "attributes": [
    --         {"name": "vehicle_type", "type": "enum", "options": ["car", "truck", "bus", "motorcycle"]},
    --         {"name": "occluded", "type": "boolean"},
    --         {"name": "truncated_pct", "type": "number", "min": 0, "max": 100}
    --       ]
    --     },
    --     {
    --       "id": "feat_sentiment",
    --       "name": "Sentiment",
    --       "type": "classification",
    --       "tool": "radio",
    --       "options": ["positive", "negative", "neutral"]
    --     },
    --     {
    --       "id": "feat_preference",
    --       "name": "Response Preference",
    --       "type": "ranking",
    --       "tool": "pairwise_comparison",
    --       "criteria": ["helpfulness", "harmlessness", "honesty"]
    --     }
    --   ]
    -- }
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE INDEX idx_ontology_project ON ontology(project_id);
```

## Data Management

```sql
CREATE TABLE dataset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    storage_config  JSONB NOT NULL DEFAULT '{}',
    -- storage_config example: {
    --   "backend": "s3",
    --   "bucket": "my-annotations",
    --   "prefix": "datasets/medical-v2/",
    --   "region": "eu-west-1",
    --   "credentials_ref": "secret:aws-prod-key"
    -- }
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
    -- metadata varies by modality and organisation:
    -- Image: {"width": 1920, "height": 1080, "exif": {"camera": "Canon EOS R5"}}
    -- DICOM: {"patient_id_hash": "abc", "modality": "CT", "study_date": "2026-01-15", "series_uid": "1.2.3.4"}
    -- Text: {"language": "en", "source": "customer_support", "word_count": 342}
    -- LLM pair: {"model_a": "gpt-4o", "model_b": "claude-opus-4-6", "prompt_tokens": 150}
    -- Video: {"width": 1920, "height": 1080, "fps": 30, "frame_count": 900, "codec": "h264"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_row_dataset ON data_row(dataset_id);
CREATE INDEX idx_data_row_external ON data_row(external_id);
CREATE INDEX idx_data_row_media_type ON data_row(media_type);
CREATE INDEX idx_data_row_metadata ON data_row USING GIN (metadata jsonb_path_ops);

CREATE TABLE dataset_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES dataset(id) ON DELETE CASCADE,
    version_number  INT NOT NULL,
    description     TEXT,
    row_count       INT NOT NULL DEFAULT 0,
    snapshot        JSONB NOT NULL DEFAULT '{}',
    -- snapshot example: {"row_ids": ["uuid1", "uuid2", ...], "filters_applied": {"media_type": "image/png"}}
    created_by      UUID NOT NULL REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dataset_id, version_number)
);
```

## Tasks & Workflows

```sql
CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    priority        INT NOT NULL DEFAULT 0,
    assigned_to     UUID REFERENCES "user"(id),
    reviewer_id     UUID REFERENCES "user"(id),
    round           INT NOT NULL DEFAULT 1,
    workflow_state  JSONB NOT NULL DEFAULT '{}',
    -- workflow_state example: {
    --   "current_stage": "review",
    --   "stage_history": [
    --     {"stage": "label", "entered_at": "2026-05-01T10:00:00Z", "completed_at": "2026-05-01T10:15:00Z", "actor": "uuid"},
    --     {"stage": "review", "entered_at": "2026-05-01T10:15:00Z"}
    --   ]
    -- }
    provenance      JSONB NOT NULL DEFAULT '{}',
    -- provenance example (EU AI Act Annex IV):
    -- {
    --   "pre_label_model": "yolov8-large",
    --   "pre_label_model_version": "v2.1.0",
    --   "pre_label_confidence": 0.87,
    --   "ontology_version": 3,
    --   "guidelines_hash": "sha256:abc123",
    --   "annotation_duration_ms": 45000,
    --   "annotator_expertise": ["medical", "radiology"]
    -- }
    submitted_at    TIMESTAMPTZ,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_project ON task(project_id);
CREATE INDEX idx_task_status ON task(status);
CREATE INDEX idx_task_assigned ON task(assigned_to);
CREATE INDEX idx_task_priority ON task(project_id, priority DESC);
CREATE INDEX idx_task_provenance ON task USING GIN (provenance jsonb_path_ops);
```

## Annotations (Single Table, All Modalities)

```sql
CREATE TABLE annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    feature_id      VARCHAR(255) NOT NULL,   -- references ontology schema_definition feature id
    feature_name    VARCHAR(255) NOT NULL,    -- denormalised for query convenience
    annotation_type VARCHAR(50) NOT NULL,     -- bounding_box, polygon, keypoint, ner_span, classification, ranking, segmentation
    annotator_id    UUID NOT NULL REFERENCES "user"(id),
    content         JSONB NOT NULL,
    -- content varies by annotation_type:
    --
    -- bounding_box:
    -- {"x": 0.15, "y": 0.22, "w": 0.30, "h": 0.45, "rotation": 0,
    --  "attributes": {"vehicle_type": "car", "occluded": false}}
    --
    -- polygon:
    -- {"points": [[0.1, 0.2], [0.3, 0.4], [0.5, 0.2], [0.1, 0.2]],
    --  "attributes": {"terrain": "road"}}
    --
    -- keypoint:
    -- {"points": [{"x": 0.5, "y": 0.3, "name": "nose", "visible": true},
    --             {"x": 0.45, "y": 0.25, "name": "left_eye", "visible": true}]}
    --
    -- ner_span:
    -- {"start": 42, "end": 58, "text": "Acme Corporation",
    --  "attributes": {"entity_type": "ORG", "confidence": 0.92}}
    --
    -- classification:
    -- {"value": "positive", "free_text": "The customer seems satisfied"}
    --
    -- ranking (RLHF):
    -- {"prompt": "Explain quantum computing...",
    --  "response_a": "Quantum computing uses qubits...",
    --  "response_b": "A quantum computer is...",
    --  "chosen": "A",
    --  "rationale": "Response A is more detailed and accurate",
    --  "criteria_scores": {"helpfulness": 4, "harmlessness": 5, "honesty": 4}}
    --
    -- segmentation:
    -- {"mask_rle": "encoded_rle_string", "area": 15234, "iscrowd": 0}
    --
    is_pre_label    BOOLEAN NOT NULL DEFAULT false,
    confidence      FLOAT,
    version         INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotation_task ON annotation(task_id);
CREATE INDEX idx_annotation_annotator ON annotation(annotator_id);
CREATE INDEX idx_annotation_type ON annotation(annotation_type);
CREATE INDEX idx_annotation_content ON annotation USING GIN (content jsonb_path_ops);

-- Annotation history for versioning
CREATE TABLE annotation_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    annotation_id   UUID NOT NULL REFERENCES annotation(id) ON DELETE CASCADE,
    version         INT NOT NULL,
    content         JSONB NOT NULL,
    changed_by      UUID NOT NULL REFERENCES "user"(id),
    change_reason   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ann_history_annotation ON annotation_history(annotation_id);
```

## Quality Control

```sql
CREATE TABLE review_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES task(id) ON DELETE CASCADE,
    annotation_id   UUID REFERENCES annotation(id),
    reviewer_id     UUID NOT NULL REFERENCES "user"(id),
    decision        VARCHAR(50) NOT NULL,    -- accept, reject, fix, escalate
    comment         TEXT,
    corrections     JSONB,                   -- reviewer's suggested corrections in same format as annotation content
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_task ON review_decision(task_id);

CREATE TABLE quality_metric (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    scope           JSONB NOT NULL,
    -- scope example: {"task_id": "uuid"} or {"feature_name": "Vehicle"} or {"annotator_pair": ["uuid1", "uuid2"]}
    metric_type     VARCHAR(50) NOT NULL,    -- cohen_kappa, fleiss_kappa, krippendorff_alpha, iou, f1
    value           FLOAT NOT NULL,
    sample_size     INT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example: {"confusion_matrix": [[10, 2], [1, 15]], "per_class": {"car": 0.85, "truck": 0.72}}
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quality_project ON quality_metric(project_id);
CREATE INDEX idx_quality_scope ON quality_metric USING GIN (scope jsonb_path_ops);

CREATE TABLE qa_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    rule_definition JSONB NOT NULL,
    -- rule_definition example:
    -- {
    --   "type": "python_script",
    --   "code": "def check(annotation):\n    return annotation['content'].get('area', 0) > 100",
    --   "applies_to": ["bounding_box", "polygon"]
    -- }
    -- or:
    -- {
    --   "type": "threshold",
    --   "field": "content.attributes.occluded",
    --   "condition": "must_exist_when",
    --   "when": "annotation_type == 'bounding_box'"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Active Learning

```sql
CREATE TABLE al_strategy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    config          JSONB NOT NULL,
    -- config example: {
    --   "strategy_type": "uncertainty",
    --   "model_endpoint": "https://ml.internal/predict",
    --   "batch_size": 100,
    --   "selection_criteria": "least_confident",
    --   "diversity_weight": 0.3
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE al_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id     UUID NOT NULL REFERENCES al_strategy(id) ON DELETE CASCADE,
    data_row_id     UUID NOT NULL REFERENCES data_row(id),
    score           FLOAT NOT NULL,
    model_version   VARCHAR(255),
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example: {"class_probabilities": {"car": 0.45, "truck": 0.42, "bus": 0.13}, "entropy": 1.52}
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_al_score_strategy ON al_score(strategy_id);
CREATE INDEX idx_al_score_value ON al_score(score DESC);
```

## Audit & Export

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    actor_id        UUID REFERENCES "user"(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    -- changes example: {"status": {"old": "in_review", "new": "accepted"}}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

CREATE TABLE export_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    format          VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {"filters": {"status": "accepted"}, "include_provenance": true, "dataset_version": 3}
    file_path       TEXT,
    row_count       INT,
    requested_by    UUID NOT NULL REFERENCES "user"(id),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {"retry_count": 3, "timeout_ms": 5000, "custom_headers": {"X-Source": "labeling-platform"}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example JSONB Queries

```sql
-- Find all bounding box annotations for "Vehicle" with occluded=true
SELECT a.id, a.content
FROM annotation a
WHERE a.annotation_type = 'bounding_box'
  AND a.feature_name = 'Vehicle'
  AND a.content @> '{"attributes": {"occluded": true}}';

-- Find all RLHF rankings where Response A was chosen
SELECT a.id, a.content->>'rationale' AS rationale
FROM annotation a
WHERE a.annotation_type = 'ranking'
  AND a.content->>'chosen' = 'A';

-- Find tasks that used a specific pre-label model
SELECT t.id, t.provenance->>'pre_label_model_version' AS model_version
FROM task t
WHERE t.provenance @> '{"pre_label_model": "yolov8-large"}';

-- Compute average annotation area for bounding boxes in a project
SELECT
    a.feature_name,
    AVG((a.content->>'w')::FLOAT * (a.content->>'h')::FLOAT) AS avg_area
FROM annotation a
JOIN task t ON t.id = a.task_id
WHERE t.project_id = '{{project_uuid}}'
  AND a.annotation_type = 'bounding_box'
GROUP BY a.feature_name;

-- Find data rows with specific DICOM metadata
SELECT id, file_path, metadata->>'modality' AS imaging_modality
FROM data_row
WHERE metadata @> '{"modality": "CT"}'
  AND dataset_id = '{{dataset_uuid}}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisation, user, organisation_member, api_key |
| Project & Ontology | 2 | project, ontology (features in JSONB) |
| Data Management | 3 | dataset, data_row, dataset_version |
| Tasks & Workflow | 1 | task (workflow state in JSONB) |
| Annotations | 2 | annotation, annotation_history |
| Quality Control | 3 | review_decision, quality_metric, qa_rule |
| Active Learning | 2 | al_strategy, al_score |
| Audit & Export | 3 | audit_log, export_job, webhook |
| **Total** | **20** | 10 fewer tables than normalised model |

---

## Key Design Decisions

1. **Single annotation table with JSONB content** — All annotation types (bounding boxes, text spans, classifications, rankings, segmentation masks) share one table. The `content` JSONB column holds the type-specific payload. This eliminates the need for type-specific detail tables and simplifies queries that span annotation types. The trade-off is that content validation must happen in application code, not database constraints.

2. **Ontology as JSON Schema** — The `ontology.schema_definition` JSONB column holds the full ontology definition including features, attributes, and validation rules. This allows the ontology to be version-controlled as a single document and validated against JSON Schema Draft 2020-12. It also means the platform can validate annotation `content` against the ontology schema at the application layer.

3. **Workflow state in JSONB** — Rather than separate `workflow_stage` and `task_stage_transition` tables, the task's workflow progress is tracked in a `workflow_state` JSONB column with a stage history array. This is simpler for simple linear workflows and avoids JOINs when rendering the task detail view. For complex conditional workflows, the JSONB structure can be extended without migrations.

4. **Provenance embedded in task** — EU AI Act Annex IV provenance data (pre-label model, ontology version, guidelines hash, annotation duration) is stored directly on the task as JSONB. This keeps provenance co-located with the task for efficient export, though it means provenance updates require a task row update.

5. **GIN indexes on JSONB columns** — The `jsonb_path_ops` GIN index class is used for containment queries (`@>` operator). This provides efficient filtering on JSONB fields without extracting them into relational columns. For high-cardinality queries (e.g., filtering by media type), a conventional BTREE index on the extracted path can be added.

6. **Quality metrics with JSONB scope** — The `quality_metric.scope` JSONB column allows metrics to be scoped flexibly (per task, per feature, per annotator pair, per dataset version) without a rigid foreign key structure. This supports the variety of IAA metric aggregation levels needed in practice.

7. **Annotation history table** — Rather than event sourcing, annotation changes are tracked via a simple history table that stores previous versions of the `content` JSONB. This provides versioning and undo capability without the complexity of a full event store.

8. **Configuration over tables** — Project configuration (assignment strategy, auto-accept threshold, export formats), QA rule definitions, and active learning strategy parameters are all stored as JSONB rather than as separate relational tables. This reduces table count and allows features to be added or modified without migrations. The application code is responsible for interpreting these configurations correctly.
