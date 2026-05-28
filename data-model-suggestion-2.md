# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Data Labeling Platform · Created: 2026-05-20

## Philosophy

This model treats every meaningful action in the annotation workflow as an immutable event in an append-only event store. The current state of any annotation, task, or project is derived by replaying its event history. Materialised read models (projections) are maintained for fast querying, following the CQRS (Command Query Responsibility Segregation) pattern.

Event sourcing is used in financial trading systems, healthcare record-keeping, and compliance-heavy platforms where the ability to reconstruct any historical state is a regulatory requirement. For a data labeling platform subject to EU AI Act Annex IV documentation requirements, this architecture is a natural fit: the event store *is* the audit trail, not a secondary log bolted onto mutable tables. Every annotation creation, modification, review, and quality score is recorded as a first-class event with full context.

The design maintains a small set of mutable projection tables for the read path (rendering the annotation UI, listing tasks, computing dashboards) while the event store serves as the canonical source of truth. This separation allows the read models to be rebuilt, restructured, or extended without losing any historical data.

**Best for:** Platforms where full audit trails, temporal queries ("what was the annotation state on date X?"), and regulatory compliance documentation are primary requirements — especially healthcare AI, autonomous vehicle, and EU AI Act-regulated deployments.

**Trade-offs:**
- (+) Complete, immutable audit trail by default — no separate audit logging needed
- (+) Full temporal query support: reconstruct any entity's state at any point in time
- (+) EU AI Act Annex IV compliance is inherent: annotation provenance is the event stream itself
- (+) Event replay enables debugging, quality investigation, and what-if analysis
- (+) Read models can be rebuilt or added without schema migrations to the event store
- (-) Higher storage cost: events accumulate indefinitely (snapshotting mitigates read performance)
- (-) Eventual consistency between event store and read projections; read models may lag by milliseconds
- (-) More complex application code: every write is an event emission, reads go through projections
- (-) Querying across entities requires well-maintained projections rather than ad-hoc JOINs
- (-) Developers must understand event sourcing patterns; steeper onboarding curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EU AI Act Annex IV | Event stream provides complete training data provenance: who labeled, when, under which guidelines version, with which pre-label model |
| ISO/IEC 5259-2:2024 | Dataset quality measures are derived projections computed from annotation and review events |
| W3C Web Annotation Data Model | Annotation events carry Body/Target/Motivation semantics in their payload |
| ISO/IEC 27001 | The event store itself satisfies audit logging requirements; immutability prevents tampering |
| Cohen's/Fleiss' Kappa | IAA metrics are computed as projection events triggered by annotation submission events |
| COCO/YOLO/VOC | Export projections materialise annotation state into format-ready structures |

---

## Event Store (Source of Truth)

```sql
-- Core event store: append-only, immutable
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID (task, project, dataset, etc.)
    stream_type     VARCHAR(100) NOT NULL,    -- task, project, dataset, annotation, user
    event_type      VARCHAR(200) NOT NULL,    -- e.g., annotation.created, task.submitted, review.accepted
    event_version   INT NOT NULL,             -- schema version for this event type
    sequence_number BIGINT NOT NULL,          -- monotonically increasing within a stream
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"actor_id": "uuid", "ip_address": "1.2.3.4", "client": "web-v2.1", "correlation_id": "uuid"}
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Optimistic concurrency: sequence_number prevents conflicting writes to the same stream
CREATE INDEX idx_event_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_occurred ON event_store(occurred_at);
CREATE INDEX idx_event_stream_type ON event_store(stream_type);

-- Snapshot table for performance: periodically captures aggregate state
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    sequence_number BIGINT NOT NULL,          -- snapshot taken at this sequence
    state           JSONB NOT NULL,           -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

### Event Type Taxonomy

```
-- Project lifecycle
project.created          -- {name, organisation_id, modality, created_by}
project.updated          -- {fields_changed: {name: {old, new}}}
project.status_changed   -- {old_status, new_status}
project.guidelines_updated -- {guidelines_hash, guidelines_md}

-- Ontology lifecycle
ontology.created         -- {project_id, version, features: [...]}
ontology.feature_added   -- {feature_id, name, feature_type, tool}
ontology.feature_removed -- {feature_id}

-- Dataset lifecycle
dataset.created          -- {name, organisation_id, storage_backend}
dataset.version_created  -- {version_number, row_ids: [...]}
data_row.ingested        -- {dataset_id, media_type, file_path, metadata}

-- Task lifecycle
task.created             -- {project_id, data_row_id, priority}
task.assigned            -- {assignee_id, assigned_by}
task.started             -- {annotator_id}
task.submitted           -- {annotator_id, annotation_count}
task.review_started      -- {reviewer_id}
task.accepted            -- {reviewer_id, comment}
task.rejected            -- {reviewer_id, comment, rejection_reason}
task.reassigned          -- {old_assignee, new_assignee, reason}
task.skipped             -- {annotator_id, reason}

-- Annotation lifecycle
annotation.created       -- {task_id, feature_id, annotation_type, geometry/value, is_pre_label}
annotation.updated       -- {fields_changed: {...}}
annotation.deleted       -- {reason}
annotation.pre_label_generated -- {model_name, model_version, confidence, geometry/value}

-- Quality events
review.decision          -- {task_id, annotation_id, decision, comment, reviewer_id}
iaa.computed             -- {project_id, task_id, metric, value, annotator_pair, sample_size}
qa_rule.executed         -- {rule_id, task_id, passed, message}

-- Active learning events
al.scores_computed       -- {strategy_id, model_version, scores: [{data_row_id, score}]}
al.batch_selected        -- {strategy_id, data_row_ids: [...], selection_criteria}

-- Export events
export.requested         -- {project_id, format, filters}
export.completed         -- {file_path, row_count, duration_ms}
export.failed            -- {error_message}

-- Webhook events
webhook.delivered        -- {webhook_id, event_type, response_code}
webhook.failed           -- {webhook_id, event_type, error, retry_count}
```

---

## Read Projections (Mutable, Rebuildable)

These tables are derived from the event store and can be dropped and rebuilt at any time.

```sql
-- ============================================================
-- PROJECTION: Current state of organisations and users
-- ============================================================
CREATE TABLE proj_organisation (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    member_count    INT NOT NULL DEFAULT 0,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_user (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_org_member (
    organisation_id UUID NOT NULL REFERENCES proj_organisation(id),
    user_id         UUID NOT NULL REFERENCES proj_user(id),
    role            VARCHAR(50) NOT NULL,
    PRIMARY KEY (organisation_id, user_id)
);

-- ============================================================
-- PROJECTION: Current project state
-- ============================================================
CREATE TABLE proj_project (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    modality        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    task_count      INT NOT NULL DEFAULT 0,
    completed_count INT NOT NULL DEFAULT 0,
    avg_iaa_score   FLOAT,
    current_ontology_version INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_project_org ON proj_project(organisation_id);
CREATE INDEX idx_proj_project_status ON proj_project(status);

-- ============================================================
-- PROJECTION: Current task state
-- ============================================================
CREATE TABLE proj_task (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    data_row_id     UUID NOT NULL,
    status          VARCHAR(50) NOT NULL,
    assigned_to     UUID,
    reviewer_id     UUID,
    annotation_count INT NOT NULL DEFAULT 0,
    round           INT NOT NULL DEFAULT 1,
    priority        INT NOT NULL DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_task_project ON proj_task(project_id);
CREATE INDEX idx_proj_task_status ON proj_task(status);
CREATE INDEX idx_proj_task_assigned ON proj_task(assigned_to);

-- ============================================================
-- PROJECTION: Current annotations (latest state)
-- ============================================================
CREATE TABLE proj_annotation (
    id              UUID PRIMARY KEY,
    task_id         UUID NOT NULL,
    feature_name    VARCHAR(255) NOT NULL,
    annotation_type VARCHAR(50) NOT NULL,
    annotator_id    UUID NOT NULL,
    geometry        JSONB,                   -- spatial data in GeoJSON-like format
    text_span       JSONB,                   -- {start, end, text}
    classification  JSONB,                   -- {value, free_text}
    ranking         JSONB,                   -- {chosen, rationale, criteria}
    is_pre_label    BOOLEAN NOT NULL DEFAULT false,
    confidence      FLOAT,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_ann_task ON proj_annotation(task_id);
CREATE INDEX idx_proj_ann_annotator ON proj_annotation(annotator_id);

-- ============================================================
-- PROJECTION: Quality metrics dashboard
-- ============================================================
CREATE TABLE proj_quality_summary (
    project_id      UUID NOT NULL,
    feature_name    VARCHAR(255),            -- NULL for project-level aggregate
    metric          VARCHAR(50) NOT NULL,
    value           FLOAT NOT NULL,
    sample_size     INT NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (project_id, COALESCE(feature_name, '__all__'), metric)
);

-- ============================================================
-- PROJECTION: Active learning queue
-- ============================================================
CREATE TABLE proj_al_queue (
    data_row_id     UUID NOT NULL,
    project_id      UUID NOT NULL,
    strategy_type   VARCHAR(50) NOT NULL,
    score           FLOAT NOT NULL,
    model_version   VARCHAR(255),
    computed_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (project_id, data_row_id)
);

CREATE INDEX idx_proj_al_score ON proj_al_queue(project_id, score DESC);

-- ============================================================
-- PROJECTION: Data rows (current state)
-- ============================================================
CREATE TABLE proj_data_row (
    id              UUID PRIMARY KEY,
    dataset_id      UUID NOT NULL,
    external_id     VARCHAR(512),
    media_type      VARCHAR(50) NOT NULL,
    file_path       TEXT NOT NULL,
    file_size_bytes BIGINT,
    width           INT,
    height          INT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_data_row_dataset ON proj_data_row(dataset_id);
```

---

## Temporal Queries

The event store enables powerful temporal queries that are impossible with mutable state:

```sql
-- "What did this annotation look like on March 15th?"
SELECT payload
FROM event_store
WHERE stream_id = '{{annotation_uuid}}'
  AND stream_type = 'annotation'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_number DESC
LIMIT 1;

-- "How many annotations did each annotator submit per day last month?"
SELECT
    (metadata->>'actor_id')::UUID AS annotator_id,
    DATE(occurred_at) AS day,
    COUNT(*) AS submissions
FROM event_store
WHERE event_type = 'task.submitted'
  AND occurred_at >= '2026-04-01'
  AND occurred_at < '2026-05-01'
GROUP BY annotator_id, day
ORDER BY day, submissions DESC;

-- "Show all events for a task, in order, for Annex IV provenance export"
SELECT
    event_type,
    payload,
    metadata,
    occurred_at
FROM event_store
WHERE stream_id = '{{task_uuid}}'
  AND stream_type = 'task'
ORDER BY sequence_number;

-- "Which annotations were changed after initial review?"
SELECT
    stream_id AS annotation_id,
    COUNT(*) AS modification_count
FROM event_store
WHERE stream_type = 'annotation'
  AND event_type = 'annotation.updated'
GROUP BY stream_id
HAVING COUNT(*) > 1;

-- Rebuild a projection from scratch
-- (executed by the projection rebuilder service)
-- SELECT * FROM event_store WHERE stream_type = 'task' ORDER BY occurred_at, sequence_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store (append-only), event_snapshot |
| Projection: Identity | 3 | proj_organisation, proj_user, proj_org_member |
| Projection: Projects | 1 | proj_project |
| Projection: Tasks | 1 | proj_task |
| Projection: Annotations | 1 | proj_annotation (all types in JSONB columns) |
| Projection: Quality | 1 | proj_quality_summary |
| Projection: Active Learning | 1 | proj_al_queue |
| Projection: Data | 1 | proj_data_row |
| **Total** | **11** | 2 immutable + 9 rebuildable projections |

---

## Key Design Decisions

1. **Single event store table** — All events across all aggregate types live in one table, partitioned logically by `stream_type`. This simplifies infrastructure (one table to back up, replicate, partition) and enables cross-cutting temporal queries. For very high-volume deployments, the table can be range-partitioned by `occurred_at`.

2. **JSONB payloads with versioned schemas** — Event payloads are JSONB with an `event_version` field. When the payload schema evolves, the version is incremented. Projection builders handle multiple versions via upcasting functions, avoiding the need to migrate historical events.

3. **Optimistic concurrency via sequence numbers** — The `UNIQUE (stream_id, sequence_number)` constraint prevents conflicting concurrent writes to the same aggregate. The application increments `sequence_number` before writing; a constraint violation triggers a retry with the updated state.

4. **Projections are disposable** — Every `proj_*` table can be dropped and rebuilt by replaying the event store. This means read model schema changes are zero-risk: add a new projection, replay events, switch traffic.

5. **Annotations stored as JSONB in projections** — Unlike the normalised model (Suggestion 1), the projection table uses JSONB columns per annotation type (`geometry`, `text_span`, `classification`, `ranking`). This avoids multiple JOIN paths at read time while the event store retains the fully typed data.

6. **Snapshot table for performance** — Aggregates with long event histories (e.g., a project with 100,000 task events) can be snapshotted periodically. The projector loads the latest snapshot and replays only subsequent events, bounding rebuild time.

7. **Audit trail is free** — There is no separate `audit_log` table because the event store already captures every state change with actor, timestamp, and full before/after context. Annex IV compliance export is a query against the event store filtered by stream, not a secondary system.

8. **Active learning as event-driven** — Active learning score computations emit `al.scores_computed` events. The `proj_al_queue` projection maintains the current priority queue. When the model is retrained and new scores are computed, a new event replaces the projection without losing the history of previous scoring runs.
