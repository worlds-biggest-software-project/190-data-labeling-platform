# Data Labeling Platform — Phased Development Plan

> Project: 190-data-labeling-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12+ | ML ecosystem alignment — annotators, active learning, and model integration all use Python natively; all competitor SDKs (Label Studio, CVAT, Argilla, Labelbox) are Python; Pydantic + FastAPI provide type-safe API development with automatic OpenAPI 3.1 generation |
| API framework | FastAPI 0.115+ | Async-first for handling concurrent annotation sessions; native OpenAPI 3.1 spec generation (standards.md requirement); Pydantic v2 integration for request/response validation; WebSocket support for real-time collaborative annotation |
| Database | PostgreSQL 16 | Required by all four data model suggestions; JSONB with GIN indexes for flexible annotation content (Suggestion 3 hybrid approach); UUID primary keys; full-text search; ltree extension available for ontology hierarchies; battle-tested at Label Studio and CVAT scale |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | Async support via asyncpg; declarative models map directly to the data model; Alembic for versioned migrations; JSONB column support |
| Task queue | Celery 5 + Redis | Async workloads: pre-labeling inference, IAA computation, export generation, webhook delivery, active learning scoring; Redis doubles as cache layer for session data and annotation UI state |
| Frontend | React 18 + TypeScript + Vite | Canvas-based annotation requires client-side rendering; react-konva for spatial annotation tools; TypeScript for annotation type safety; Vite for fast dev builds; TanStack Query for server state management |
| Annotation canvas | Konva.js (via react-konva) | Hardware-accelerated 2D canvas; supports bounding boxes, polygons, keypoints, freehand; zoom/pan; layer management; used in production annotation tools |
| CSS / UI framework | Tailwind CSS 4 + Radix UI | Utility-first for rapid UI development; Radix for accessible, unstyled primitives (dialogs, dropdowns, tooltips); consistent design without heavy component library lock-in |
| Authentication | FastAPI Users + PyJWT | Local auth (email/password) + OAuth 2.0 / OIDC for SSO; JWT access tokens + refresh tokens; API key support for SDK/pipeline access; aligns with RFC 6749, RFC 7519, OpenID Connect Core 1.0 |
| Cloud storage SDK | boto3 + google-cloud-storage + azure-storage-blob | S3, GCS, Azure Blob connectors as specified in MVP feature set; abstracted behind a storage interface |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target per README; multi-service orchestration (API, worker, frontend, PostgreSQL, Redis) |
| Testing | pytest + pytest-asyncio + Playwright | pytest for unit/integration; pytest-asyncio for async endpoint tests; Playwright for E2E annotation UI tests; factory_boy for test fixtures |
| Code quality | Ruff (linter + formatter) + mypy | Ruff replaces flake8/black/isort in one tool; mypy for static type checking; pre-commit hooks |
| Python SDK | httpx + Pydantic | SDK uses httpx for async HTTP; Pydantic models shared between API and SDK; published to PyPI as `datalabel` |
| Export formats | COCO, YOLO, Pascal VOC, JSON | COCO Dataset Format is the de facto standard (standards.md); YOLO for real-time detection pipelines; Pascal VOC for legacy compatibility |
| IAA metrics | scikit-learn + statsmodels + krippendorff | Cohen's Kappa and Fleiss' Kappa from scikit-learn/statsmodels; Krippendorff's Alpha from the krippendorff package; IoU from custom implementation |
| API documentation | OpenAPI 3.1 (auto-generated) | FastAPI generates OpenAPI 3.1 spec automatically; Swagger UI and ReDoc served at /docs and /redoc; aligns with standards.md OpenAPI 3.1 requirement |

### Project Structure

```
data-labeling-platform/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   └── versions/
├── src/
│   └── datalabel/
│       ├── __init__.py
│       ├── main.py                      # FastAPI application entry point
│       ├── config.py                    # Settings via pydantic-settings
│       ├── database.py                  # SQLAlchemy engine + session
│       ├── models/                      # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── organisation.py
│       │   ├── user.py
│       │   ├── project.py
│       │   ├── ontology.py
│       │   ├── dataset.py
│       │   ├── data_row.py
│       │   ├── task.py
│       │   ├── annotation.py
│       │   ├── quality.py
│       │   ├── active_learning.py
│       │   ├── export.py
│       │   ├── webhook.py
│       │   └── audit.py
│       ├── schemas/                     # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── organisation.py
│       │   ├── user.py
│       │   ├── project.py
│       │   ├── ontology.py
│       │   ├── dataset.py
│       │   ├── task.py
│       │   ├── annotation.py
│       │   ├── quality.py
│       │   └── export.py
│       ├── api/                         # FastAPI route modules
│       │   ├── __init__.py
│       │   ├── deps.py                  # Dependency injection (auth, db session)
│       │   ├── auth.py
│       │   ├── organisations.py
│       │   ├── projects.py
│       │   ├── datasets.py
│       │   ├── tasks.py
│       │   ├── annotations.py
│       │   ├── quality.py
│       │   ├── active_learning.py
│       │   ├── exports.py
│       │   └── webhooks.py
│       ├── services/                    # Business logic
│       │   ├── __init__.py
│       │   ├── auth_service.py
│       │   ├── project_service.py
│       │   ├── task_service.py
│       │   ├── annotation_service.py
│       │   ├── quality_service.py
│       │   ├── active_learning_service.py
│       │   ├── export_service.py
│       │   ├── storage_service.py
│       │   ├── webhook_service.py
│       │   └── prelabel_service.py
│       ├── workers/                     # Celery task definitions
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── export_tasks.py
│       │   ├── quality_tasks.py
│       │   ├── prelabel_tasks.py
│       │   ├── active_learning_tasks.py
│       │   └── webhook_tasks.py
│       └── exporters/                   # Format-specific export logic
│           ├── __init__.py
│           ├── base.py
│           ├── coco.py
│           ├── yolo.py
│           ├── pascal_voc.py
│           └── json_export.py
├── sdk/                                 # Python SDK package
│   ├── pyproject.toml
│   └── src/
│       └── datalabel_sdk/
│           ├── __init__.py
│           ├── client.py
│           ├── models.py
│           └── exceptions.py
├── frontend/                            # React annotation UI
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── api/                         # API client (generated from OpenAPI)
│       ├── components/
│       │   ├── annotation/              # Canvas-based annotation tools
│       │   │   ├── AnnotationCanvas.tsx
│       │   │   ├── BoundingBoxTool.tsx
│       │   │   ├── PolygonTool.tsx
│       │   │   ├── KeypointTool.tsx
│       │   │   └── ToolSidebar.tsx
│       │   ├── text/                    # Text annotation components
│       │   │   ├── TextAnnotator.tsx
│       │   │   ├── NERHighlighter.tsx
│       │   │   └── ClassificationPanel.tsx
│       │   ├── ranking/                 # RLHF pairwise comparison
│       │   │   ├── PairwiseComparison.tsx
│       │   │   └── RankingEditor.tsx
│       │   ├── review/                  # Review workflow UI
│       │   │   ├── ReviewQueue.tsx
│       │   │   └── ReviewPanel.tsx
│       │   ├── project/                 # Project management
│       │   ├── dashboard/               # Quality dashboards
│       │   └── common/                  # Shared UI components
│       ├── hooks/
│       ├── stores/                      # Zustand state management
│       └── types/
├── tests/
│   ├── conftest.py
│   ├── factories/                       # factory_boy factories
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                        # Test data files (images, JSON)
└── docs/
    ├── api/                             # Auto-generated from OpenAPI
    └── guides/
```

---

## Phase 1: Foundation — Project Setup & Core Data Models

### Purpose

Establish the project skeleton, database schema, authentication system, and configuration infrastructure. After this phase, the application boots, connects to PostgreSQL, runs migrations, and serves authenticated API endpoints. All subsequent phases build on this foundation without restructuring.

### Tasks

#### 1.1 — Project Scaffolding & Configuration

**What**: Create the Python project with pyproject.toml, Docker Compose stack, and environment-based configuration.

**Design**:

```python
# src/datalabel/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "DataLabel Platform"
    debug: bool = False
    secret_key: str  # required, no default
    allowed_origins: list[str] = ["http://localhost:5173"]

    # Database
    database_url: str = "postgresql+asyncpg://datalabel:datalabel@localhost:5432/datalabel"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Storage
    storage_backend: str = "local"  # local, s3, gcs, azure
    storage_local_path: str = "./data/uploads"
    aws_access_key_id: str | None = None
    aws_secret_access_key: str | None = None
    aws_s3_bucket: str | None = None
    aws_s3_region: str = "us-east-1"
    gcs_bucket: str | None = None
    gcs_credentials_file: str | None = None
    azure_storage_connection_string: str | None = None
    azure_container_name: str | None = None

    # Auth
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    model_config = {"env_prefix": "DATALABEL_", "env_file": ".env"}

settings = Settings()
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [db, redis]
    volumes: ["./data:/app/data"]

  worker:
    build: .
    command: celery -A datalabel.workers.celery_app worker -l info
    env_file: .env
    depends_on: [db, redis]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: datalabel
      POSTGRES_USER: datalabel
      POSTGRES_PASSWORD: datalabel
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

**Testing**:
- `Unit: test_settings_from_env — environment variables populate Settings fields correctly`
- `Unit: test_settings_required_secret — missing DATALABEL_SECRET_KEY raises ValidationError`
- `Unit: test_settings_defaults — unset optional fields use documented defaults`
- `Integration: test_docker_compose_up — docker compose up --build succeeds and API responds on port 8000`
- `Integration: test_database_connection — SQLAlchemy engine connects to PostgreSQL and executes SELECT 1`

#### 1.2 — Database Models & Migrations (Core Identity)

**What**: Implement SQLAlchemy ORM models for organisation, user, organisation_member, and api_key tables; create initial Alembic migration.

**Design**:

Based on Data Model Suggestion 3 (Hybrid Relational + JSONB), which balances schema flexibility with relational integrity. The hybrid approach is chosen because: (1) annotation content varies across modalities, making JSONB the right fit; (2) the single annotation table simplifies cross-modality queries; (3) JSONB ontology definitions avoid migrations when the label schema evolves; (4) it has the fastest path to MVP with 20 tables vs. 30 in the fully normalised model.

```python
# src/datalabel/models/base.py
import uuid
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False
    )

# src/datalabel/models/organisation.py
from sqlalchemy import String, ForeignKey, UniqueConstraint
from sqlalchemy.dialects.postgresql import JSONB, UUID as PGUUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
import uuid

class Organisation(Base, TimestampMixin):
    __tablename__ = "organisation"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(128), unique=True, nullable=False)
    plan: Mapped[str] = mapped_column(String(50), default="free", nullable=False)
    settings: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)

    members: Mapped[list["OrganisationMember"]] = relationship(back_populates="organisation")
    projects: Mapped[list["Project"]] = relationship(back_populates="organisation")

class User(Base, TimestampMixin):
    __tablename__ = "user"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    display_name: Mapped[str] = mapped_column(String(255), nullable=False)
    password_hash: Mapped[str | None] = mapped_column(String(255))
    auth_provider: Mapped[str] = mapped_column(String(50), default="local", nullable=False)
    auth_provider_id: Mapped[str | None] = mapped_column(String(255))
    profile: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)

class OrganisationMember(Base):
    __tablename__ = "organisation_member"
    __table_args__ = (UniqueConstraint("organisation_id", "user_id"),)

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id", ondelete="CASCADE"))
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id", ondelete="CASCADE"))
    role: Mapped[str] = mapped_column(String(50), default="annotator", nullable=False)
    permissions: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    joined_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    organisation: Mapped["Organisation"] = relationship(back_populates="members")
    user: Mapped["User"] = relationship()

class ApiKey(Base):
    __tablename__ = "api_key"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id", ondelete="CASCADE"))
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id", ondelete="CASCADE"))
    key_hash: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    scopes: Mapped[dict] = mapped_column(JSONB, default=list, nullable=False)
    expires_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

**Testing**:
- `Unit: test_organisation_defaults — Organisation() has plan="free", settings={}`
- `Unit: test_user_slug_uniqueness — two organisations with same slug raise IntegrityError`
- `Integration: test_alembic_upgrade_head — alembic upgrade head creates all tables`
- `Integration: test_alembic_downgrade — alembic downgrade base removes all tables cleanly`
- `Integration: test_create_organisation_with_member — insert organisation + member + verify FK relationship`
- `Integration: test_cascade_delete_organisation — deleting organisation cascades to members and api_keys`

#### 1.3 — Authentication & Authorization

**What**: Implement JWT-based authentication with local email/password registration, login, token refresh, and API key authentication for SDK access.

**Design**:

```python
# src/datalabel/schemas/auth.py
from pydantic import BaseModel, EmailStr
import uuid

class RegisterRequest(BaseModel):
    email: EmailStr
    password: str  # min 8 chars, validated
    display_name: str

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # seconds

class ApiKeyCreateRequest(BaseModel):
    name: str
    scopes: list[str] = ["read", "write"]
    expires_in_days: int | None = None

class ApiKeyResponse(BaseModel):
    id: uuid.UUID
    name: str
    key: str  # returned only on creation, never again
    scopes: list[str]
    expires_at: str | None

# src/datalabel/services/auth_service.py
from passlib.context import CryptContext
import jwt
from datetime import datetime, timedelta, timezone

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    async def register(self, request: RegisterRequest) -> User: ...
    async def login(self, request: LoginRequest) -> TokenResponse: ...
    async def refresh_token(self, refresh_token: str) -> TokenResponse: ...
    async def create_api_key(self, user_id: uuid.UUID, org_id: uuid.UUID, request: ApiKeyCreateRequest) -> ApiKeyResponse: ...
    async def verify_api_key(self, key: str) -> tuple[User, Organisation]: ...

# src/datalabel/api/deps.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, APIKeyHeader

bearer_scheme = HTTPBearer(auto_error=False)
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def get_current_user(
    token: str | None = Depends(bearer_scheme),
    api_key: str | None = Security(api_key_header),
    db: AsyncSession = Depends(get_db),
) -> User:
    """Authenticate via JWT bearer token OR API key header."""
    ...

async def require_role(min_role: str):
    """Dependency factory: require the user has at least min_role in the current organisation."""
    ...
```

API endpoints:
- `POST /api/v1/auth/register` — create account, return tokens
- `POST /api/v1/auth/login` — authenticate, return tokens
- `POST /api/v1/auth/refresh` — exchange refresh token for new access token
- `POST /api/v1/auth/api-keys` — create API key (authenticated)
- `DELETE /api/v1/auth/api-keys/{key_id}` — revoke API key
- `GET /api/v1/auth/me` — return current user profile

**Testing**:
- `Unit: test_password_hashing — hash and verify password round-trips correctly`
- `Unit: test_jwt_encode_decode — token contains user_id, org_id, exp claims`
- `Unit: test_jwt_expired_token — expired token raises HTTPException 401`
- `Integration: test_register_new_user — POST /auth/register returns 201 with tokens`
- `Integration: test_register_duplicate_email — POST /auth/register with existing email returns 409`
- `Integration: test_login_valid_credentials — POST /auth/login returns 200 with tokens`
- `Integration: test_login_invalid_password — POST /auth/login with wrong password returns 401`
- `Integration: test_api_key_auth — request with valid X-API-Key header returns 200`
- `Integration: test_api_key_expired — request with expired API key returns 401`
- `Integration: test_api_key_scopes — read-only key cannot POST to write endpoints`

#### 1.4 — Organisation & User Management API

**What**: CRUD endpoints for organisations, organisation membership, and user profiles.

**Design**:

```python
# src/datalabel/schemas/organisation.py
from pydantic import BaseModel
import uuid

class OrganisationCreate(BaseModel):
    name: str
    slug: str  # validated: lowercase, alphanumeric + hyphens

class OrganisationResponse(BaseModel):
    id: uuid.UUID
    name: str
    slug: str
    plan: str
    member_count: int
    created_at: str

class OrganisationMemberAdd(BaseModel):
    email: str
    role: str = "annotator"  # owner, admin, manager, reviewer, annotator

class OrganisationMemberResponse(BaseModel):
    id: uuid.UUID
    user_id: uuid.UUID
    display_name: str
    email: str
    role: str
    joined_at: str
```

API endpoints:
- `POST /api/v1/organisations` — create organisation (creator becomes owner)
- `GET /api/v1/organisations` — list user's organisations
- `GET /api/v1/organisations/{org_id}` — get organisation details
- `PATCH /api/v1/organisations/{org_id}` — update organisation (admin+)
- `POST /api/v1/organisations/{org_id}/members` — invite member
- `GET /api/v1/organisations/{org_id}/members` — list members
- `PATCH /api/v1/organisations/{org_id}/members/{member_id}` — update role
- `DELETE /api/v1/organisations/{org_id}/members/{member_id}` — remove member

**Testing**:
- `Unit: test_slug_validation — slugs with uppercase or spaces are rejected`
- `Integration: test_create_organisation — POST returns 201, creator is owner`
- `Integration: test_list_organisations — only returns orgs where user is a member`
- `Integration: test_invite_member — POST /members with valid email creates pending membership`
- `Integration: test_remove_last_owner — deleting the sole owner returns 400`
- `Integration: test_role_hierarchy — annotator cannot update organisation settings`

---

## Phase 2: Data Management — Datasets, Data Rows, & Storage

### Purpose

Build the data ingestion pipeline: creating datasets, uploading data rows (images, text files, LLM output pairs), and connecting to cloud storage backends. After this phase, users can upload data through the API, browse datasets, and the storage abstraction supports local, S3, GCS, and Azure backends.

### Tasks

#### 2.1 — Storage Abstraction Layer

**What**: Implement a pluggable storage interface with local filesystem, S3, GCS, and Azure Blob backends.

**Design**:

```python
# src/datalabel/services/storage_service.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import AsyncIterator
import uuid

@dataclass
class StorageObject:
    key: str
    size_bytes: int
    content_type: str
    etag: str | None = None

class StorageBackend(ABC):
    @abstractmethod
    async def upload(self, key: str, data: bytes | AsyncIterator[bytes], content_type: str) -> StorageObject: ...

    @abstractmethod
    async def download(self, key: str) -> AsyncIterator[bytes]: ...

    @abstractmethod
    async def get_presigned_url(self, key: str, expires_in: int = 3600) -> str: ...

    @abstractmethod
    async def delete(self, key: str) -> None: ...

    @abstractmethod
    async def exists(self, key: str) -> bool: ...

class LocalStorageBackend(StorageBackend):
    def __init__(self, base_path: str): ...

class S3StorageBackend(StorageBackend):
    def __init__(self, bucket: str, region: str, access_key: str, secret_key: str): ...

class GCSStorageBackend(StorageBackend):
    def __init__(self, bucket: str, credentials_file: str | None): ...

class AzureBlobStorageBackend(StorageBackend):
    def __init__(self, connection_string: str, container_name: str): ...

def create_storage_backend(config: Settings) -> StorageBackend:
    """Factory function based on settings.storage_backend."""
    ...
```

**Testing**:
- `Unit: test_local_upload_download — upload bytes, download same bytes, content matches`
- `Unit: test_local_delete — delete file, exists() returns False`
- `Unit: test_local_presigned_url — returns file:// URL for local backend`
- `Integration (mocked): test_s3_upload — moto mock, upload succeeds, object exists in bucket`
- `Integration (mocked): test_s3_presigned_url — returns valid S3 presigned URL format`
- `Integration (mocked): test_gcs_upload — mock GCS client, upload succeeds`
- `Integration (mocked): test_azure_upload — mock Azure client, upload succeeds`
- `Unit: test_factory_local — create_storage_backend with storage_backend="local" returns LocalStorageBackend`
- `Unit: test_factory_s3 — create_storage_backend with storage_backend="s3" returns S3StorageBackend`

#### 2.2 — Dataset & Data Row Models

**What**: Database models and migrations for dataset, data_row, and dataset_version tables.

**Design**:

```python
# src/datalabel/models/dataset.py
class Dataset(Base, TimestampMixin):
    __tablename__ = "dataset"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id", ondelete="CASCADE"))
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    storage_config: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    created_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))

    data_rows: Mapped[list["DataRow"]] = relationship(back_populates="dataset")
    versions: Mapped[list["DatasetVersion"]] = relationship(back_populates="dataset")

class DataRow(Base):
    __tablename__ = "data_row"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    dataset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("dataset.id", ondelete="CASCADE"), index=True)
    external_id: Mapped[str | None] = mapped_column(String(512), index=True)
    media_type: Mapped[str] = mapped_column(String(100), nullable=False)
    file_path: Mapped[str] = mapped_column(nullable=False)
    file_size_bytes: Mapped[int | None] = mapped_column()
    metadata: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    dataset: Mapped["Dataset"] = relationship(back_populates="data_rows")

class DatasetVersion(Base):
    __tablename__ = "dataset_version"
    __table_args__ = (UniqueConstraint("dataset_id", "version_number"),)

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    dataset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("dataset.id", ondelete="CASCADE"))
    version_number: Mapped[int] = mapped_column(nullable=False)
    description: Mapped[str | None] = mapped_column()
    row_count: Mapped[int] = mapped_column(default=0)
    snapshot: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    created_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    dataset: Mapped["Dataset"] = relationship(back_populates="versions")
```

```python
# src/datalabel/schemas/dataset.py
class DatasetCreate(BaseModel):
    name: str
    storage_config: dict = {}

class DataRowCreate(BaseModel):
    external_id: str | None = None
    media_type: str  # validated against allowed types
    metadata: dict = {}

class DataRowBatchCreate(BaseModel):
    rows: list[DataRowCreate]  # max 1000 per batch

class DataRowResponse(BaseModel):
    id: uuid.UUID
    external_id: str | None
    media_type: str
    file_path: str
    file_size_bytes: int | None
    metadata: dict
    presigned_url: str | None  # for rendering in the UI
    created_at: str
```

API endpoints:
- `POST /api/v1/organisations/{org_id}/datasets` — create dataset
- `GET /api/v1/organisations/{org_id}/datasets` — list datasets (paginated)
- `GET /api/v1/datasets/{dataset_id}` — get dataset details with row count
- `POST /api/v1/datasets/{dataset_id}/rows` — upload single data row (multipart)
- `POST /api/v1/datasets/{dataset_id}/rows/batch` — batch create data rows (JSON metadata + pre-uploaded file paths)
- `GET /api/v1/datasets/{dataset_id}/rows` — list data rows (paginated, filterable)
- `GET /api/v1/datasets/{dataset_id}/rows/{row_id}` — get data row with presigned URL
- `POST /api/v1/datasets/{dataset_id}/versions` — create immutable version snapshot
- `GET /api/v1/datasets/{dataset_id}/versions` — list versions

**Testing**:
- `Unit: test_media_type_validation — reject unsupported media types (e.g., application/exe)`
- `Unit: test_batch_create_limit — batch with >1000 rows returns 400`
- `Integration: test_upload_image — POST multipart with PNG, data_row created, file stored`
- `Integration: test_upload_text — POST multipart with text file, metadata parsed correctly`
- `Integration: test_list_rows_pagination — 50 rows inserted, page_size=20 returns 20 with next cursor`
- `Integration: test_create_version — version snapshot records current row IDs`
- `Integration: test_presigned_url — GET row returns presigned URL that resolves to the file`
- `Fixture: test_upload_sample_image — upload tests/fixtures/sample_640x480.png, verify width/height in metadata`

---

## Phase 3: Project & Ontology Management

### Purpose

Implement project creation with ontology (label schema) definitions. After this phase, users can create annotation projects, define label schemas with features (bounding box classes, text classification options, RLHF ranking criteria), and version their ontologies. This is the prerequisite for the annotation workflow.

### Tasks

#### 3.1 — Project CRUD & Configuration

**What**: Database models and API endpoints for creating, updating, and managing annotation projects.

**Design**:

```python
# src/datalabel/models/project.py
class Project(Base, TimestampMixin):
    __tablename__ = "project"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id", ondelete="CASCADE"), index=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str | None] = mapped_column()
    modality: Mapped[str] = mapped_column(String(50), nullable=False)  # image, text, llm_pair, multi
    status: Mapped[str] = mapped_column(String(50), default="draft", nullable=False)
    config: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    created_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))

    organisation: Mapped["Organisation"] = relationship(back_populates="projects")
    ontologies: Mapped[list["Ontology"]] = relationship(back_populates="project")
    tasks: Mapped[list["Task"]] = relationship(back_populates="project")

# Status state machine:
# draft -> active -> paused -> active (resume)
#                 -> completed
#                 -> archived
# draft -> archived (cancel without starting)
VALID_STATUS_TRANSITIONS = {
    "draft":     {"active", "archived"},
    "active":    {"paused", "completed", "archived"},
    "paused":    {"active", "archived"},
    "completed": {"archived"},
    "archived":  set(),
}
```

```python
# src/datalabel/schemas/project.py
class ProjectCreate(BaseModel):
    name: str
    description: str | None = None
    modality: Literal["image", "text", "llm_pair", "multi"]
    config: ProjectConfig = ProjectConfig()
    dataset_id: uuid.UUID  # link to data source

class ProjectConfig(BaseModel):
    assignment_strategy: Literal["manual", "round_robin", "priority"] = "round_robin"
    min_annotations_per_task: int = 1  # for consensus tasks, set to 2+
    auto_accept_confidence: float | None = None  # pre-label auto-accept threshold
    enable_active_learning: bool = False
    export_formats: list[str] = ["coco", "json"]
    guidelines_md: str | None = None
    guidelines_hash: str | None = None  # computed from guidelines_md

class ProjectResponse(BaseModel):
    id: uuid.UUID
    name: str
    description: str | None
    modality: str
    status: str
    config: ProjectConfig
    task_count: int
    completed_count: int
    ontology_version: int
    created_at: str
    updated_at: str
```

API endpoints:
- `POST /api/v1/organisations/{org_id}/projects` — create project
- `GET /api/v1/organisations/{org_id}/projects` — list projects (paginated, filterable by status)
- `GET /api/v1/projects/{project_id}` — get project with task statistics
- `PATCH /api/v1/projects/{project_id}` — update project config
- `POST /api/v1/projects/{project_id}/status` — transition project status
- `DELETE /api/v1/projects/{project_id}` — soft-delete (archive)

**Testing**:
- `Unit: test_valid_status_transitions — active->paused allowed, completed->active rejected`
- `Unit: test_guidelines_hash — updating guidelines_md recomputes SHA-256 hash`
- `Integration: test_create_project — POST returns 201, project linked to organisation`
- `Integration: test_list_projects_by_status — filter status=active returns only active projects`
- `Integration: test_status_transition — POST /status with "active" transitions draft->active`
- `Integration: test_invalid_status_transition — POST /status with "completed" on draft returns 400`
- `Integration: test_project_task_stats — project response includes correct task_count and completed_count`

#### 3.2 — Ontology Definition & Versioning

**What**: Ontology model with JSON Schema-based feature definitions, version management, and validation of annotation content against the ontology schema.

**Design**:

```python
# src/datalabel/models/ontology.py
class Ontology(Base):
    __tablename__ = "ontology"
    __table_args__ = (UniqueConstraint("project_id", "version"),)

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"), index=True)
    version: Mapped[int] = mapped_column(default=1, nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)
    schema_definition: Mapped[dict] = mapped_column(JSONB, nullable=False)
    created_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    project: Mapped["Project"] = relationship(back_populates="ontologies")
```

```python
# src/datalabel/schemas/ontology.py
class OntologyFeature(BaseModel):
    """A single label class or classification in the ontology."""
    id: str  # unique within ontology, e.g., "feat_vehicle"
    name: str  # human-readable, e.g., "Vehicle"
    type: Literal["object", "classification", "relationship", "ranking"]
    tool: Literal[
        "bounding_box", "polygon", "keypoint", "segmentation",
        "ner_span", "text_class", "radio", "checkbox", "free_text",
        "pairwise_comparison", "ranking_list"
    ]
    color: str | None = None  # hex color, e.g., "#FF0000"
    attributes: list[FeatureAttribute] = []
    options: list[str] | None = None  # for classification tools
    criteria: list[str] | None = None  # for ranking tools

class FeatureAttribute(BaseModel):
    name: str
    type: Literal["enum", "boolean", "number", "text"]
    options: list[str] | None = None  # for enum type
    min: float | None = None  # for number type
    max: float | None = None  # for number type
    required: bool = False

class OntologyCreate(BaseModel):
    features: list[OntologyFeature]

class OntologyResponse(BaseModel):
    id: uuid.UUID
    project_id: uuid.UUID
    version: int
    is_active: bool
    schema_definition: dict
    created_at: str

# src/datalabel/services/ontology_service.py
class OntologyService:
    async def create_ontology(self, project_id: uuid.UUID, request: OntologyCreate, user_id: uuid.UUID) -> Ontology:
        """Create version 1 of the ontology for a project."""
        ...

    async def create_new_version(self, project_id: uuid.UUID, request: OntologyCreate, user_id: uuid.UUID) -> Ontology:
        """Deactivate current ontology, create next version. Existing annotations retain their ontology_version reference."""
        ...

    def validate_annotation_content(self, content: dict, feature: OntologyFeature) -> list[str]:
        """Validate annotation JSONB content against the feature definition. Returns list of validation errors."""
        ...
```

API endpoints:
- `POST /api/v1/projects/{project_id}/ontology` — create or update ontology (creates new version if one exists)
- `GET /api/v1/projects/{project_id}/ontology` — get active ontology
- `GET /api/v1/projects/{project_id}/ontology/versions` — list all ontology versions
- `GET /api/v1/projects/{project_id}/ontology/versions/{version}` — get specific version

**Testing**:
- `Unit: test_ontology_feature_validation — bounding_box feature must have type="object" and tool="bounding_box"`
- `Unit: test_validate_bbox_content — valid bbox content passes; missing "w" field fails`
- `Unit: test_validate_ranking_content — ranking must have "chosen" field with value A, B, or tie`
- `Unit: test_feature_attribute_enum — enum attribute with value not in options fails validation`
- `Integration: test_create_ontology — POST with features list creates version 1`
- `Integration: test_create_second_version — second POST creates version 2, deactivates version 1`
- `Integration: test_get_active_ontology — GET returns highest active version`
- `Fixture: test_sample_cv_ontology — create ontology with Vehicle(bbox) + Pedestrian(keypoint) features`
- `Fixture: test_sample_rlhf_ontology — create ontology with ResponsePreference(pairwise_comparison) feature`

---

## Phase 4: Task Management & Annotation Workflow

### Purpose

Implement the core annotation workflow: creating tasks from dataset rows, assigning tasks to annotators, submitting annotations, and routing tasks through review stages. This is the heart of the platform — after this phase, the complete label-review-accept cycle works end-to-end through the API.

### Tasks

#### 4.1 — Task Creation & Assignment

**What**: Create tasks from dataset data rows, with assignment strategies (manual, round-robin, priority-based).

**Design**:

```python
# src/datalabel/models/task.py
class Task(Base, TimestampMixin):
    __tablename__ = "task"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"), index=True)
    data_row_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("data_row.id"))
    status: Mapped[str] = mapped_column(String(50), default="queued", nullable=False, index=True)
    priority: Mapped[int] = mapped_column(default=0, nullable=False)
    assigned_to: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("user.id"), index=True)
    reviewer_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("user.id"))
    round: Mapped[int] = mapped_column(default=1, nullable=False)
    workflow_state: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    provenance: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    submitted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    reviewed_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))

# Task status state machine:
# queued -> assigned -> in_progress -> submitted -> in_review -> accepted
#                                                              -> rejected -> assigned (re-annotation)
#                    -> skipped (annotator skips)
TASK_STATUS_TRANSITIONS = {
    "queued":      {"assigned"},
    "assigned":    {"in_progress", "skipped"},
    "in_progress": {"submitted", "skipped"},
    "submitted":   {"in_review", "accepted"},  # accepted = auto-accept for single-review
    "in_review":   {"accepted", "rejected"},
    "rejected":    {"assigned"},  # re-assignment for another round
    "accepted":    set(),
    "skipped":     {"queued"},  # can be re-queued
}
```

```python
# src/datalabel/services/task_service.py
class TaskService:
    async def create_tasks_from_dataset(
        self, project_id: uuid.UUID, dataset_id: uuid.UUID, filters: dict | None = None
    ) -> list[Task]:
        """Create one task per data row (optionally filtered). If min_annotations_per_task > 1, create multiple tasks per row."""
        ...

    async def assign_next_task(self, project_id: uuid.UUID, annotator_id: uuid.UUID) -> Task | None:
        """Get the next unassigned task based on the project's assignment strategy. Returns None if no tasks available."""
        ...

    async def submit_task(self, task_id: uuid.UUID, annotator_id: uuid.UUID) -> Task:
        """Transition task to submitted. Validates that annotations exist."""
        ...

    async def get_annotator_queue(self, project_id: uuid.UUID, annotator_id: uuid.UUID, limit: int = 20) -> list[Task]:
        """Get tasks assigned to this annotator, ordered by priority."""
        ...
```

API endpoints:
- `POST /api/v1/projects/{project_id}/tasks/generate` — create tasks from dataset (body: {dataset_id, filters?})
- `GET /api/v1/projects/{project_id}/tasks` — list tasks (paginated, filterable by status, assignee)
- `GET /api/v1/projects/{project_id}/tasks/next` — get next available task for current user
- `GET /api/v1/tasks/{task_id}` — get task with data row and annotations
- `POST /api/v1/tasks/{task_id}/assign` — assign task to user
- `POST /api/v1/tasks/{task_id}/start` — transition to in_progress
- `POST /api/v1/tasks/{task_id}/submit` — submit completed task
- `POST /api/v1/tasks/{task_id}/skip` — skip task with reason

**Testing**:
- `Unit: test_task_status_transitions — validate all legal and illegal transitions`
- `Unit: test_round_robin_assignment — 3 annotators get tasks in rotating order`
- `Unit: test_priority_assignment — highest priority task assigned first`
- `Integration: test_generate_tasks — 10 data rows produce 10 tasks`
- `Integration: test_generate_tasks_with_consensus — min_annotations_per_task=3 produces 30 tasks for 10 rows`
- `Integration: test_assign_next_task — returns queued task, status becomes assigned`
- `Integration: test_submit_task_without_annotations — returns 400`
- `Integration: test_submit_task_with_annotations — transitions to submitted, submitted_at set`
- `Integration: test_skip_task — status becomes skipped, task re-queueable`

#### 4.2 — Annotation CRUD

**What**: Create, read, update, and delete annotations on tasks. Single table with JSONB content, supporting all annotation types.

**Design**:

```python
# src/datalabel/models/annotation.py
class Annotation(Base, TimestampMixin):
    __tablename__ = "annotation"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("task.id", ondelete="CASCADE"), index=True)
    feature_id: Mapped[str] = mapped_column(String(255), nullable=False)
    feature_name: Mapped[str] = mapped_column(String(255), nullable=False)
    annotation_type: Mapped[str] = mapped_column(String(50), nullable=False, index=True)
    annotator_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"), index=True)
    content: Mapped[dict] = mapped_column(JSONB, nullable=False)
    is_pre_label: Mapped[bool] = mapped_column(default=False, nullable=False)
    confidence: Mapped[float | None] = mapped_column()
    version: Mapped[int] = mapped_column(default=1, nullable=False)

class AnnotationHistory(Base):
    __tablename__ = "annotation_history"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    annotation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("annotation.id", ondelete="CASCADE"), index=True)
    version: Mapped[int] = mapped_column(nullable=False)
    content: Mapped[dict] = mapped_column(JSONB, nullable=False)
    changed_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))
    change_reason: Mapped[str | None] = mapped_column()
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

```python
# src/datalabel/schemas/annotation.py
class AnnotationCreate(BaseModel):
    feature_id: str
    annotation_type: str
    content: dict  # validated against ontology at the service layer

class BoundingBoxContent(BaseModel):
    x: float  # normalised 0-1
    y: float
    w: float
    h: float
    rotation: float = 0.0
    attributes: dict = {}

class PolygonContent(BaseModel):
    points: list[list[float]]  # [[x1,y1], [x2,y2], ...]
    attributes: dict = {}

class KeypointContent(BaseModel):
    points: list[KeypointPoint]

class KeypointPoint(BaseModel):
    x: float
    y: float
    name: str
    visible: bool = True

class NERSpanContent(BaseModel):
    start: int
    end: int
    text: str
    attributes: dict = {}

class ClassificationContent(BaseModel):
    value: str
    free_text: str | None = None

class RankingContent(BaseModel):
    prompt: str
    response_a: str
    response_b: str
    chosen: Literal["A", "B", "tie"]
    rationale: str | None = None
    criteria_scores: dict[str, int] = {}

class AnnotationResponse(BaseModel):
    id: uuid.UUID
    task_id: uuid.UUID
    feature_id: str
    feature_name: str
    annotation_type: str
    content: dict
    is_pre_label: bool
    confidence: float | None
    version: int
    created_at: str
    updated_at: str
```

API endpoints:
- `POST /api/v1/tasks/{task_id}/annotations` — create annotation
- `GET /api/v1/tasks/{task_id}/annotations` — list annotations for task
- `PATCH /api/v1/annotations/{annotation_id}` — update annotation content (saves history)
- `DELETE /api/v1/annotations/{annotation_id}` — delete annotation
- `GET /api/v1/annotations/{annotation_id}/history` — get annotation version history

**Testing**:
- `Unit: test_bbox_content_validation — x+w > 1.0 fails validation`
- `Unit: test_polygon_minimum_points — polygon with <3 points fails`
- `Unit: test_ner_span_offsets — end <= start fails validation`
- `Unit: test_ranking_chosen_values — chosen must be A, B, or tie`
- `Integration: test_create_bbox_annotation — POST with valid bbox content, annotation saved with correct feature_name`
- `Integration: test_create_ranking_annotation — POST with RLHF ranking, content stored correctly`
- `Integration: test_update_annotation_history — PATCH creates history record with previous content`
- `Integration: test_annotation_ontology_validation — annotation with feature_id not in ontology returns 400`
- `Integration: test_delete_annotation — DELETE removes annotation, subsequent GET returns 404`
- `Fixture: test_sample_image_annotations — create 5 bbox + 2 polygon annotations on sample image task`

#### 4.3 — Review Workflow

**What**: Multi-stage review with accept/reject/fix/escalate decisions, driving task status transitions.

**Design**:

```python
# src/datalabel/models/quality.py
class ReviewDecision(Base):
    __tablename__ = "review_decision"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("task.id", ondelete="CASCADE"), index=True)
    annotation_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("annotation.id"))
    reviewer_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))
    decision: Mapped[str] = mapped_column(String(50), nullable=False)  # accept, reject, fix, escalate
    comment: Mapped[str | None] = mapped_column()
    corrections: Mapped[dict | None] = mapped_column(JSONB)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# src/datalabel/services/review_service.py
class ReviewService:
    async def start_review(self, task_id: uuid.UUID, reviewer_id: uuid.UUID) -> Task:
        """Transition task to in_review. Only tasks with status=submitted can be reviewed."""
        ...

    async def submit_review(self, task_id: uuid.UUID, reviewer_id: uuid.UUID, decision: ReviewDecisionCreate) -> ReviewDecision:
        """Record review decision and transition task status.
        - accept: task -> accepted
        - reject: task -> rejected, round incremented, re-queued for annotation
        - fix: reviewer's corrections applied, task -> accepted
        - escalate: task stays in_review, notification sent to manager
        """
        ...

    async def get_review_queue(self, project_id: uuid.UUID, reviewer_id: uuid.UUID) -> list[Task]:
        """Get submitted tasks awaiting review for this reviewer."""
        ...
```

API endpoints:
- `GET /api/v1/projects/{project_id}/review-queue` — get tasks awaiting review
- `POST /api/v1/tasks/{task_id}/review/start` — claim task for review
- `POST /api/v1/tasks/{task_id}/review` — submit review decision
- `GET /api/v1/tasks/{task_id}/reviews` — list all review decisions for task

**Testing**:
- `Integration: test_accept_task — review with decision=accept transitions task to accepted`
- `Integration: test_reject_task — decision=reject transitions to rejected, round incremented`
- `Integration: test_reject_requeue — rejected task can be re-assigned for round 2`
- `Integration: test_fix_task — decision=fix applies corrections, task accepted`
- `Integration: test_review_unsubmitted_task — reviewing a queued task returns 400`
- `Integration: test_review_queue_filtering — only submitted tasks appear in review queue`
- `Integration: test_multiple_reviews — two reviewers both review same task (for consensus projects)`

---

## Phase 5: Annotation UI — Frontend Foundation

### Purpose

Build the React frontend with annotation canvas for spatial annotations (bounding boxes, polygons, keypoints), text annotation for NER/classification, and the RLHF pairwise comparison interface. After this phase, annotators can complete the full label-submit cycle through a web browser.

### Tasks

#### 5.1 — Frontend Scaffolding & Auth

**What**: React + TypeScript + Vite project setup with authentication flow, API client generated from OpenAPI spec, and route structure.

**Design**:

```typescript
// frontend/src/types/index.ts
export interface User {
  id: string;
  email: string;
  display_name: string;
  is_active: boolean;
}

export interface Organisation {
  id: string;
  name: string;
  slug: string;
  plan: string;
}

export interface Project {
  id: string;
  name: string;
  modality: "image" | "text" | "llm_pair" | "multi";
  status: "draft" | "active" | "paused" | "completed" | "archived";
  task_count: number;
  completed_count: number;
}

// frontend/src/stores/authStore.ts
import { create } from "zustand";

interface AuthState {
  user: User | null;
  accessToken: string | null;
  organisation: Organisation | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshToken: () => Promise<void>;
}

// Route structure:
// /login — login page
// /register — registration page
// /org/:orgSlug — organisation dashboard
// /org/:orgSlug/projects — project list
// /org/:orgSlug/projects/:projectId — project detail
// /org/:orgSlug/projects/:projectId/annotate — annotation workspace
// /org/:orgSlug/projects/:projectId/review — review workspace
// /org/:orgSlug/datasets — dataset list
// /org/:orgSlug/settings — organisation settings
```

**Testing**:
- `E2E: test_login_flow — enter email/password, click login, redirected to org dashboard`
- `E2E: test_login_invalid — wrong password shows error message`
- `E2E: test_token_refresh — after token expires, next API call refreshes transparently`
- `Unit (vitest): test_auth_store_login — login sets user and accessToken`
- `Unit (vitest): test_auth_store_logout — logout clears state`

#### 5.2 — Image Annotation Canvas

**What**: Canvas-based annotation workspace using Konva.js for bounding boxes, polygons, and keypoints on images.

**Design**:

```typescript
// frontend/src/components/annotation/AnnotationCanvas.tsx
interface AnnotationCanvasProps {
  imageUrl: string;
  annotations: Annotation[];
  ontology: OntologyFeature[];
  activeTool: AnnotationTool;
  activeFeature: OntologyFeature | null;
  onAnnotationCreate: (annotation: AnnotationCreate) => void;
  onAnnotationUpdate: (id: string, content: AnnotationContent) => void;
  onAnnotationDelete: (id: string) => void;
  readonly?: boolean;
}

type AnnotationTool = "select" | "bounding_box" | "polygon" | "keypoint" | "pan" | "zoom";

interface AnnotationContent {
  bounding_box?: { x: number; y: number; w: number; h: number; rotation: number };
  polygon?: { points: [number, number][] };
  keypoint?: { points: { x: number; y: number; name: string; visible: boolean }[] };
}

// Key interactions:
// - Click+drag to create bounding box
// - Click vertices to create polygon, double-click to close
// - Click to place keypoints in ontology-defined order
// - Select existing annotation to resize/move/delete
// - Mouse wheel to zoom, middle-click to pan
// - Keyboard shortcuts: B(box), P(polygon), K(keypoint), Delete, Ctrl+Z(undo), Ctrl+Y(redo)

// frontend/src/components/annotation/ToolSidebar.tsx
interface ToolSidebarProps {
  ontology: OntologyFeature[];
  activeTool: AnnotationTool;
  activeFeature: OntologyFeature | null;
  onToolChange: (tool: AnnotationTool) => void;
  onFeatureChange: (feature: OntologyFeature) => void;
  annotations: Annotation[];  // for annotation list panel
}
```

**Testing**:
- `E2E: test_draw_bounding_box — select bbox tool, drag on canvas, annotation appears in list`
- `E2E: test_draw_polygon — click 4 vertices, double-click to close, polygon rendered`
- `E2E: test_select_and_move — click existing bbox, drag to new position, coordinates updated`
- `E2E: test_delete_annotation — select annotation, press Delete, annotation removed`
- `E2E: test_undo_redo — create bbox, Ctrl+Z removes it, Ctrl+Y restores it`
- `E2E: test_zoom_pan — mouse wheel zooms, annotations scale correctly`
- `E2E: test_keyboard_shortcuts — press B switches to bbox tool, P to polygon`
- `Unit (vitest): test_coordinate_normalisation — canvas pixel coords converted to 0-1 normalised`

#### 5.3 — Text Annotation & Classification UI

**What**: Text annotation interface for NER span selection and document-level classification.

**Design**:

```typescript
// frontend/src/components/text/TextAnnotator.tsx
interface TextAnnotatorProps {
  text: string;
  annotations: Annotation[];
  ontology: OntologyFeature[];
  activeFeature: OntologyFeature | null;
  onAnnotationCreate: (annotation: AnnotationCreate) => void;
  onAnnotationDelete: (id: string) => void;
}

// Interaction: select text with mouse -> popup shows available NER labels -> click label to create annotation
// Highlighted spans are color-coded by feature type
// Overlapping spans are rendered with nested highlighting

// frontend/src/components/text/ClassificationPanel.tsx
interface ClassificationPanelProps {
  ontology: OntologyFeature[];  // filtered to classification features
  currentValues: Record<string, string>;
  onClassify: (featureId: string, value: string) => void;
}
// Renders radio buttons / checkboxes / free-text inputs based on feature tool type
```

**Testing**:
- `E2E: test_select_text_span — highlight text, select "ORG" label, span annotated`
- `E2E: test_overlapping_spans — two overlapping NER spans render correctly with nested highlights`
- `E2E: test_classification_radio — select sentiment label from radio buttons`
- `E2E: test_classification_free_text — type free-text classification, saved correctly`
- `Unit (vitest): test_span_offset_calculation — selected text range maps to correct start/end offsets`

#### 5.4 — RLHF Pairwise Comparison UI

**What**: Side-by-side comparison interface for LLM output preference annotation.

**Design**:

```typescript
// frontend/src/components/ranking/PairwiseComparison.tsx
interface PairwiseComparisonProps {
  prompt: string;
  responseA: string;
  responseB: string;
  criteria: string[];  // from ontology feature definition
  onSubmit: (ranking: RankingContent) => void;
}

interface RankingContent {
  chosen: "A" | "B" | "tie";
  rationale: string;
  criteria_scores: Record<string, number>;  // 1-5 per criterion
}

// Layout:
// Top: prompt display (scrollable, markdown-rendered)
// Middle: two columns showing Response A and Response B (markdown-rendered)
// Bottom: criteria scoring (1-5 slider per criterion), chosen selector (A/B/Tie), rationale text area
// Keyboard: 1=choose A, 2=choose B, 3=tie, Tab to move between criteria
```

**Testing**:
- `E2E: test_pairwise_select_a — click "Response A" button, chosen=A saved`
- `E2E: test_pairwise_criteria_scoring — slide helpfulness to 4, harmlessness to 5, values saved`
- `E2E: test_pairwise_rationale — type rationale text, included in submission`
- `E2E: test_pairwise_keyboard — press 1, Response A selected`
- `E2E: test_pairwise_markdown_rendering — responses with code blocks render correctly`
- `Unit (vitest): test_ranking_validation — submission without choosing A/B/tie shows error`

---

## Phase 6: Quality Control — IAA Metrics & QA Rules

### Purpose

Implement inter-annotator agreement computation (Cohen's Kappa, Fleiss' Kappa, Krippendorff's Alpha), quality dashboards, and the programmatic QA rules engine. After this phase, project managers can measure annotation quality, identify low-performing annotators, and run automated quality checks on submissions.

### Tasks

#### 6.1 — Inter-Annotator Agreement Computation

**What**: Background task that computes IAA metrics for consensus-annotated tasks, with per-feature and per-annotator-pair breakdowns.

**Design**:

```python
# src/datalabel/services/quality_service.py
from sklearn.metrics import cohen_kappa_score
from statsmodels.stats.inter_rater import fleiss_kappa
import krippendorff
import numpy as np

class QualityService:
    async def compute_iaa_for_project(self, project_id: uuid.UUID) -> list[QualityMetric]:
        """Compute IAA for all consensus tasks in the project.
        For each pair of annotators who labeled the same data row:
          - Classification: Cohen's Kappa on label values
          - Bounding box: IoU-based agreement
          - Ranking: agreement on chosen value
        For n>2 annotators: Fleiss' Kappa or Krippendorff's Alpha.
        """
        ...

    def compute_cohen_kappa(self, labels_a: list[str], labels_b: list[str]) -> float:
        """Cohen's Kappa for two annotators on classification tasks."""
        return cohen_kappa_score(labels_a, labels_b)

    def compute_fleiss_kappa(self, rating_matrix: np.ndarray) -> float:
        """Fleiss' Kappa for n annotators on classification tasks.
        rating_matrix: (n_subjects, n_categories) count matrix.
        """
        return fleiss_kappa(rating_matrix, method="fleiss")

    def compute_iou(self, bbox_a: dict, bbox_b: dict) -> float:
        """Intersection over Union for two bounding boxes.
        Each bbox: {"x": float, "y": float, "w": float, "h": float}
        """
        ...

    def compute_krippendorff_alpha(self, reliability_data: np.ndarray, level: str = "nominal") -> float:
        """Krippendorff's Alpha for ordinal or nominal data."""
        return krippendorff.alpha(reliability_data=reliability_data, level_of_measurement=level)

# src/datalabel/models/quality.py
class QualityMetric(Base):
    __tablename__ = "quality_metric"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"), index=True)
    scope: Mapped[dict] = mapped_column(JSONB, nullable=False)
    metric_type: Mapped[str] = mapped_column(String(50), nullable=False)
    value: Mapped[float] = mapped_column(nullable=False)
    sample_size: Mapped[int] = mapped_column(nullable=False)
    details: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    computed_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# src/datalabel/workers/quality_tasks.py
@celery_app.task
def compute_project_iaa(project_id: str):
    """Celery task to compute IAA metrics asynchronously."""
    ...
```

API endpoints:
- `POST /api/v1/projects/{project_id}/quality/compute` — trigger IAA computation (async, returns job ID)
- `GET /api/v1/projects/{project_id}/quality/metrics` — get computed IAA metrics (filterable by feature, annotator pair)
- `GET /api/v1/projects/{project_id}/quality/summary` — aggregated quality dashboard data

**Testing**:
- `Unit: test_cohen_kappa_perfect — identical labels return kappa=1.0`
- `Unit: test_cohen_kappa_random — random labels return kappa near 0.0`
- `Unit: test_fleiss_kappa_computation — known 4-annotator matrix returns expected value`
- `Unit: test_iou_full_overlap — identical bboxes return IoU=1.0`
- `Unit: test_iou_no_overlap — non-intersecting bboxes return IoU=0.0`
- `Unit: test_iou_partial_overlap — known partial overlap returns expected value (0.25)`
- `Unit: test_krippendorff_alpha — known reliability data returns expected alpha`
- `Integration: test_compute_iaa_classification — 2 annotators on 10 classification tasks, metrics stored correctly`
- `Integration: test_compute_iaa_bbox — IoU computed for overlapping bboxes on same data row`
- `Integration: test_quality_api_filtering — filter metrics by feature_name returns only matching records`

#### 6.2 — Programmatic QA Rules Engine

**What**: User-defined quality check rules (Python scripts, threshold checks, regex patterns) that run on annotation submission and flag violations.

**Design**:

```python
# src/datalabel/models/quality.py (addition)
class QARule(Base):
    __tablename__ = "qa_rule"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"))
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    rule_definition: Mapped[dict] = mapped_column(JSONB, nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

class QAResult(Base):
    __tablename__ = "qa_result"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    qa_rule_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("qa_rule.id", ondelete="CASCADE"))
    task_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("task.id", ondelete="CASCADE"))
    passed: Mapped[bool] = mapped_column(nullable=False)
    message: Mapped[str | None] = mapped_column()
    executed_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# src/datalabel/services/qa_engine.py
class QARuleEngine:
    """Executes QA rules against annotations in a sandboxed environment."""

    async def execute_rules(self, task_id: uuid.UUID) -> list[QAResult]:
        """Run all active QA rules for the task's project against the task's annotations."""
        ...

    def execute_threshold_rule(self, rule: dict, annotation: Annotation) -> QAResult:
        """Check if an annotation field meets a threshold condition.
        Example rule: {"type": "threshold", "field": "content.w", "operator": ">=", "value": 0.01}
        """
        ...

    def execute_regex_rule(self, rule: dict, annotation: Annotation) -> QAResult:
        """Check if a text field matches a regex pattern.
        Example rule: {"type": "regex", "field": "content.text", "pattern": "^[A-Z]", "message": "Text must start with uppercase"}
        """
        ...

    def execute_python_rule(self, rule: dict, annotations: list[Annotation]) -> QAResult:
        """Execute a sandboxed Python function against annotations.
        Example rule: {"type": "python_script", "code": "def check(annotations): return len(annotations) >= 1"}
        Executed in RestrictedPython sandbox with no I/O access.
        """
        ...

# Rule definition schema:
# {
#   "type": "threshold" | "regex" | "python_script",
#   "applies_to": ["bounding_box", "polygon"],  # annotation types
#   ...type-specific fields
# }
```

API endpoints:
- `POST /api/v1/projects/{project_id}/qa-rules` — create QA rule
- `GET /api/v1/projects/{project_id}/qa-rules` — list QA rules
- `PATCH /api/v1/qa-rules/{rule_id}` — update rule
- `DELETE /api/v1/qa-rules/{rule_id}` — delete rule
- `POST /api/v1/tasks/{task_id}/qa/run` — run QA rules on task (returns results)
- `GET /api/v1/tasks/{task_id}/qa/results` — get QA results

**Testing**:
- `Unit: test_threshold_rule_pass — bbox width=0.05 passes "width >= 0.01" rule`
- `Unit: test_threshold_rule_fail — bbox width=0.005 fails "width >= 0.01" rule`
- `Unit: test_regex_rule_match — text "Hello World" passes "^[A-Z]" rule`
- `Unit: test_regex_rule_no_match — text "hello world" fails "^[A-Z]" rule`
- `Unit: test_python_rule_sandbox — rule attempting import os raises SecurityError`
- `Unit: test_python_rule_execution — simple annotation count check passes`
- `Integration: test_qa_on_submit — submitting task triggers QA rules, results stored`
- `Integration: test_qa_rule_crud — create, list, update, delete rule lifecycle`
- `Integration: test_qa_results_api — GET results returns pass/fail with messages`

#### 6.3 — Quality Dashboard API

**What**: API endpoints that aggregate quality data for project manager dashboards.

**Design**:

```python
# src/datalabel/schemas/quality.py
class QualityDashboard(BaseModel):
    project_id: uuid.UUID
    total_tasks: int
    completed_tasks: int
    completion_rate: float
    avg_iaa_score: float | None
    iaa_by_feature: list[FeatureIAA]
    annotator_stats: list[AnnotatorStats]
    qa_pass_rate: float | None

class FeatureIAA(BaseModel):
    feature_name: str
    metric: str
    value: float
    sample_size: int

class AnnotatorStats(BaseModel):
    user_id: uuid.UUID
    display_name: str
    tasks_completed: int
    avg_annotation_time_ms: int | None
    acceptance_rate: float
    avg_iaa_score: float | None
```

API endpoints:
- `GET /api/v1/projects/{project_id}/quality/dashboard` — aggregated quality dashboard
- `GET /api/v1/projects/{project_id}/quality/annotators` — per-annotator quality breakdown
- `GET /api/v1/projects/{project_id}/quality/features` — per-feature quality breakdown

**Testing**:
- `Integration: test_dashboard_empty_project — returns zeros for all metrics`
- `Integration: test_dashboard_with_data — correct completion_rate, avg_iaa computed`
- `Integration: test_annotator_stats — per-annotator acceptance_rate correct after 5 accepted, 2 rejected tasks`
- `Integration: test_feature_iaa — per-feature kappa scores returned correctly`

---

## Phase 7: Export Pipeline

### Purpose

Build the format-specific export pipeline for COCO, YOLO, Pascal VOC, and JSON. Export runs as an async Celery task and produces downloadable files. After this phase, ML engineers can export their labeled datasets in industry-standard formats for model training.

### Tasks

#### 7.1 — Export Framework & COCO Export

**What**: Async export framework with the COCO JSON format as the first implementation.

**Design**:

```python
# src/datalabel/exporters/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class ExportConfig:
    project_id: uuid.UUID
    format: str
    filters: dict = field(default_factory=dict)  # e.g., {"status": "accepted"}
    include_provenance: bool = False  # EU AI Act Annex IV metadata
    dataset_version: int | None = None

class BaseExporter(ABC):
    @abstractmethod
    async def export(self, config: ExportConfig, annotations: list, data_rows: list, ontology: dict) -> str:
        """Export annotations to the target format. Returns file path of the generated export."""
        ...

# src/datalabel/exporters/coco.py
class COCOExporter(BaseExporter):
    """Export annotations in COCO Dataset Format.

    Output structure (per COCO spec):
    {
        "info": {"description": "...", "version": "1.0", "year": 2026, ...},
        "images": [{"id": 1, "file_name": "...", "width": 1920, "height": 1080}],
        "annotations": [{"id": 1, "image_id": 1, "category_id": 1, "bbox": [x, y, w, h], "area": ..., "iscrowd": 0}],
        "categories": [{"id": 1, "name": "Vehicle", "supercategory": "object"}]
    }
    """
    async def export(self, config: ExportConfig, annotations: list, data_rows: list, ontology: dict) -> str:
        ...

    def annotation_to_coco(self, annotation: Annotation, image_id: int, category_map: dict) -> dict:
        """Convert a platform annotation to COCO annotation format.
        - bounding_box: normalised coords -> absolute pixel coords
        - polygon: normalised points -> absolute pixel coords, compute area
        - keypoint: normalised -> absolute, visibility flags
        """
        ...

# src/datalabel/models/export.py
class ExportJob(Base):
    __tablename__ = "export_job"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"))
    format: Mapped[str] = mapped_column(String(50), nullable=False)
    status: Mapped[str] = mapped_column(String(50), default="pending", nullable=False)
    config: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    file_path: Mapped[str | None] = mapped_column()
    row_count: Mapped[int | None] = mapped_column()
    requested_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("user.id"))
    completed_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

API endpoints:
- `POST /api/v1/projects/{project_id}/exports` — request export (async, returns export job ID)
- `GET /api/v1/projects/{project_id}/exports` — list export jobs
- `GET /api/v1/exports/{export_id}` — get export job status
- `GET /api/v1/exports/{export_id}/download` — download export file (presigned URL)

**Testing**:
- `Unit: test_coco_bbox_conversion — normalised bbox [0.1, 0.2, 0.3, 0.4] on 1920x1080 image -> [192, 216, 576, 432]`
- `Unit: test_coco_category_mapping — 3 ontology features map to category IDs 1,2,3`
- `Unit: test_coco_output_structure — exported JSON has info, images, annotations, categories keys`
- `Integration: test_export_coco — 10 annotated tasks exported, valid COCO JSON produced`
- `Integration: test_export_job_lifecycle — pending -> running -> completed with file_path set`
- `Integration: test_export_download — completed export returns presigned download URL`
- `Fixture: test_export_coco_sample — export test fixture annotations, validate against COCO schema`

#### 7.2 — YOLO & Pascal VOC Exporters

**What**: YOLO plain-text and Pascal VOC XML format exporters.

**Design**:

```python
# src/datalabel/exporters/yolo.py
class YOLOExporter(BaseExporter):
    """Export in YOLO annotation format.

    Per image: one .txt file with lines:
    <class_id> <x_center> <y_center> <width> <height>
    All values normalised 0-1.

    Also generates:
    - data.yaml: {train, val, nc, names}
    - classes.txt: class names
    """
    ...

# src/datalabel/exporters/pascal_voc.py
class PascalVOCExporter(BaseExporter):
    """Export in Pascal VOC XML format.

    Per image: one .xml file:
    <annotation>
      <folder>...</folder>
      <filename>...</filename>
      <size><width>1920</width><height>1080</height><depth>3</depth></size>
      <object>
        <name>Vehicle</name>
        <bndbox><xmin>192</xmin><ymin>216</ymin><xmax>768</xmax><ymax>648</ymax></bndbox>
      </object>
    </annotation>
    """
    ...
```

**Testing**:
- `Unit: test_yolo_line_format — bbox produces correct "<class_id> <cx> <cy> <w> <h>" line`
- `Unit: test_yolo_data_yaml — generated data.yaml has correct nc and names`
- `Unit: test_voc_xml_structure — generated XML parses with correct element hierarchy`
- `Unit: test_voc_bbox_absolute — normalised bbox converted to absolute pixel xmin/ymin/xmax/ymax`
- `Integration: test_export_yolo — 10 annotated tasks produce 10 .txt files + data.yaml`
- `Integration: test_export_voc — 10 annotated tasks produce 10 .xml files`
- `Fixture: test_yolo_roundtrip — export then re-import, annotations match`

---

## Phase 8: Pre-labeling & AI-Assisted Annotation

### Purpose

Add AI-assisted pre-labeling: send unannotated data to a configurable model endpoint, receive predictions, and store them as pre-label annotations for human review. After this phase, the platform can generate draft annotations that humans correct rather than create from scratch, reducing labeling time.

### Tasks

#### 8.1 — Pre-label Model Integration

**What**: Pluggable model backend that sends data rows to a prediction endpoint and converts responses to platform annotations.

**Design**:

```python
# src/datalabel/services/prelabel_service.py
from abc import ABC, abstractmethod

@dataclass
class PredictionResult:
    feature_id: str
    annotation_type: str
    content: dict
    confidence: float

class PrelabelBackend(ABC):
    @abstractmethod
    async def predict(self, data_row: DataRow, ontology: dict) -> list[PredictionResult]: ...

class HTTPPrelabelBackend(PrelabelBackend):
    """Send data to an HTTP endpoint and parse predictions.

    Request: POST to model_endpoint
    {
        "image_url": "presigned_url" | "text": "content",
        "ontology": { ... },
        "task_type": "object_detection" | "classification" | "ner"
    }

    Expected response:
    {
        "predictions": [
            {"feature_id": "feat_vehicle", "type": "bounding_box", "content": {"x": 0.1, "y": 0.2, "w": 0.3, "h": 0.4}, "confidence": 0.92}
        ]
    }
    """
    def __init__(self, endpoint_url: str, auth_header: str | None = None): ...
    async def predict(self, data_row: DataRow, ontology: dict) -> list[PredictionResult]: ...

class PrelabelService:
    async def prelabel_tasks(self, project_id: uuid.UUID, strategy: dict) -> int:
        """Pre-label all queued tasks in a project. Returns count of pre-labeled tasks.
        Strategy config:
        {
            "model_endpoint": "https://model.internal/predict",
            "confidence_threshold": 0.5,
            "auto_accept_above": 0.95,
            "batch_size": 50
        }
        """
        ...

    async def apply_predictions(self, task: Task, predictions: list[PredictionResult], auto_accept_threshold: float | None) -> list[Annotation]:
        """Create pre-label annotations from predictions.
        If confidence > auto_accept_threshold, mark task as accepted directly.
        Otherwise, task remains queued for human review.
        """
        ...

# src/datalabel/workers/prelabel_tasks.py
@celery_app.task
def prelabel_project_tasks(project_id: str, strategy: dict):
    """Celery task: pre-label all queued tasks for a project."""
    ...
```

API endpoints:
- `POST /api/v1/projects/{project_id}/prelabel` — trigger pre-labeling (async, body: strategy config)
- `GET /api/v1/projects/{project_id}/prelabel/status` — get pre-labeling progress
- `POST /api/v1/projects/{project_id}/prelabel/test` — test pre-label on a single task (synchronous)

**Testing**:
- `Unit: test_prediction_to_annotation — PredictionResult maps to Annotation with is_pre_label=True`
- `Unit: test_confidence_threshold — predictions below threshold are discarded`
- `Unit: test_auto_accept — prediction with confidence 0.97 and threshold 0.95 auto-accepts task`
- `Integration (mocked): test_http_backend — mock model endpoint returns predictions, annotations created`
- `Integration (mocked): test_prelabel_batch — 10 queued tasks pre-labeled via mocked endpoint`
- `Integration: test_prelabel_test_endpoint — POST /prelabel/test returns predictions without saving`
- `Integration: test_prelabel_preserves_manual — pre-labels don't overwrite existing human annotations`

#### 8.2 — Pre-label Review UI

**What**: Frontend modifications to display pre-label annotations with confidence indicators and accept/correct/reject controls.

**Design**:

```typescript
// frontend/src/components/annotation/PrelabelOverlay.tsx
interface PrelabelOverlayProps {
  annotation: Annotation;  // where is_pre_label=true
  onAccept: () => void;
  onCorrect: (updated: AnnotationContent) => void;
  onReject: () => void;
}

// Visual design:
// - Pre-labels rendered with dashed borders and reduced opacity
// - Confidence badge (e.g., "92%") on each pre-label annotation
// - Toolbar: Accept All (above threshold) | Review Each | Reject All
// - Individual annotation: Accept (checkmark) | Correct (edit) | Reject (X)
// - Accepted pre-labels become solid-border annotations
// - Color coding: green (high confidence) | yellow (medium) | red (low)
```

**Testing**:
- `E2E: test_prelabel_display — pre-label annotations show with dashed borders and confidence badge`
- `E2E: test_accept_prelabel — click Accept, annotation border becomes solid, is_pre_label set to false`
- `E2E: test_correct_prelabel — adjust bbox size, save, annotation updated with new coordinates`
- `E2E: test_reject_prelabel — click Reject, annotation removed`
- `E2E: test_accept_all — Accept All button accepts all pre-labels above threshold`

---

## Phase 9: Active Learning

### Purpose

Implement active learning query strategies that intelligently select the most informative samples for human annotation. After this phase, the platform prioritises uncertain or diverse samples, reducing the total number of annotations needed to achieve target model performance.

### Tasks

#### 9.1 — Active Learning Strategies & Scoring

**What**: Implement uncertainty sampling and diversity sampling strategies that score unlabeled data rows and prioritise them for annotation.

**Design**:

```python
# src/datalabel/models/active_learning.py
class ALStrategy(Base):
    __tablename__ = "al_strategy"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("project.id", ondelete="CASCADE"))
    config: Mapped[dict] = mapped_column(JSONB, nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

class ALScore(Base):
    __tablename__ = "al_score"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    strategy_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("al_strategy.id", ondelete="CASCADE"), index=True)
    data_row_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("data_row.id"))
    score: Mapped[float] = mapped_column(nullable=False)
    model_version: Mapped[str | None] = mapped_column(String(255))
    details: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    computed_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# src/datalabel/services/active_learning_service.py
class ActiveLearningService:
    async def compute_scores(self, project_id: uuid.UUID) -> int:
        """Score all unlabeled data rows using the active strategy. Returns count scored."""
        ...

    async def uncertainty_sampling(self, model_endpoint: str, data_rows: list[DataRow], ontology: dict) -> list[ALScore]:
        """Query the model for prediction probabilities, compute uncertainty.
        Uncertainty = 1 - max(class_probabilities)  (least confident)
        Or entropy = -sum(p * log(p))
        """
        ...

    async def diversity_sampling(self, data_rows: list[DataRow], labeled_rows: list[DataRow], k: int) -> list[ALScore]:
        """Select k samples that maximise diversity relative to already-labeled data.
        Uses feature embeddings from model and k-center-greedy selection.
        """
        ...

    async def get_next_batch(self, project_id: uuid.UUID, batch_size: int = 50) -> list[DataRow]:
        """Return the top-scored unlabeled data rows for the next annotation batch."""
        ...

# Strategy config schema:
# {
#     "strategy_type": "uncertainty" | "diversity" | "hybrid",
#     "model_endpoint": "https://...",
#     "batch_size": 50,
#     "selection_criteria": "least_confident" | "entropy" | "margin",
#     "diversity_weight": 0.3,  # for hybrid: weight between uncertainty and diversity
#     "recompute_interval_tasks": 100  # recompute after every N completed tasks
# }
```

API endpoints:
- `POST /api/v1/projects/{project_id}/active-learning/strategies` — configure AL strategy
- `POST /api/v1/projects/{project_id}/active-learning/compute` — trigger score computation (async)
- `GET /api/v1/projects/{project_id}/active-learning/scores` — get scored samples (sorted by informativeness)
- `POST /api/v1/projects/{project_id}/active-learning/next-batch` — get next batch of samples to label

**Testing**:
- `Unit: test_uncertainty_least_confident — [0.9, 0.1] has uncertainty 0.1; [0.5, 0.5] has uncertainty 0.5`
- `Unit: test_uncertainty_entropy — uniform distribution has maximum entropy`
- `Unit: test_diversity_selection — k=3 from 10 points selects well-separated samples`
- `Unit: test_hybrid_scoring — combined score uses diversity_weight correctly`
- `Integration (mocked): test_compute_scores — mock model endpoint, 100 rows scored, stored in al_score`
- `Integration: test_next_batch — returns top-50 highest-scored unlabeled rows`
- `Integration: test_recompute_after_labeling — scores recomputed after 100 tasks completed`

---

## Phase 10: Webhooks, Audit, & Pipeline Integration

### Purpose

Implement webhook delivery for pipeline integration (trigger model training on batch completion), comprehensive audit logging for ISO/IEC 27001 and EU AI Act Annex IV compliance, and dataset versioning. After this phase, the platform integrates into MLOps pipelines and produces audit-grade provenance records.

### Tasks

#### 10.1 — Webhook System

**What**: Configurable webhooks that fire on platform events (task.completed, export.ready, batch.completed) with HMAC signature verification and retry logic.

**Design**:

```python
# src/datalabel/models/webhook.py
class Webhook(Base):
    __tablename__ = "webhook"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id", ondelete="CASCADE"))
    url: Mapped[str] = mapped_column(nullable=False)
    events: Mapped[list] = mapped_column(ARRAY(String), nullable=False)
    secret_hash: Mapped[str | None] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)
    config: Mapped[dict] = mapped_column(JSONB, default=dict, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

class WebhookDelivery(Base):
    __tablename__ = "webhook_delivery"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    webhook_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("webhook.id", ondelete="CASCADE"))
    event_type: Mapped[str] = mapped_column(String(100), nullable=False)
    payload: Mapped[dict] = mapped_column(JSONB, nullable=False)
    response_code: Mapped[int | None] = mapped_column()
    response_body: Mapped[str | None] = mapped_column()
    delivered_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    retry_count: Mapped[int] = mapped_column(default=0, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

# src/datalabel/services/webhook_service.py
import hmac
import hashlib

class WebhookService:
    async def dispatch(self, event_type: str, payload: dict, organisation_id: uuid.UUID) -> None:
        """Find all active webhooks subscribed to this event, enqueue delivery tasks."""
        ...

    def sign_payload(self, payload: bytes, secret: str) -> str:
        """HMAC-SHA256 signature for webhook verification."""
        return hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()

# src/datalabel/workers/webhook_tasks.py
@celery_app.task(bind=True, max_retries=5, default_retry_delay=60)
def deliver_webhook(self, webhook_id: str, event_type: str, payload: dict):
    """Deliver webhook with exponential backoff retry. Log response."""
    ...

# Supported events:
# task.submitted, task.accepted, task.rejected
# annotation.created, annotation.updated
# export.completed, export.failed
# project.status_changed
# quality.iaa_computed
# al.scores_computed
```

API endpoints:
- `POST /api/v1/organisations/{org_id}/webhooks` — create webhook
- `GET /api/v1/organisations/{org_id}/webhooks` — list webhooks
- `PATCH /api/v1/webhooks/{webhook_id}` — update webhook
- `DELETE /api/v1/webhooks/{webhook_id}` — delete webhook
- `GET /api/v1/webhooks/{webhook_id}/deliveries` — list delivery history
- `POST /api/v1/webhooks/{webhook_id}/test` — send test payload

**Testing**:
- `Unit: test_hmac_signature — signed payload matches expected HMAC-SHA256 hex digest`
- `Unit: test_event_filtering — webhook subscribed to ["task.accepted"] does not receive "task.submitted"`
- `Integration (mocked): test_webhook_delivery — mock target URL, task.accepted event delivered with 200 response`
- `Integration (mocked): test_webhook_retry — mock 500 response, task retried up to 5 times with backoff`
- `Integration: test_webhook_delivery_logging — delivery recorded with response_code, retry_count`
- `Integration: test_webhook_test_endpoint — POST /test sends test payload, delivery logged`

#### 10.2 — Audit Logging & Provenance

**What**: Comprehensive audit log capturing all state changes, plus EU AI Act Annex IV compliant provenance export.

**Design**:

```python
# src/datalabel/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisation.id"), index=True)
    actor_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("user.id"))
    action: Mapped[str] = mapped_column(String(100), nullable=False)
    resource_type: Mapped[str] = mapped_column(String(100), nullable=False)
    resource_id: Mapped[uuid.UUID] = mapped_column(nullable=False)
    changes: Mapped[dict | None] = mapped_column(JSONB)
    ip_address: Mapped[str | None] = mapped_column(String(45))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), index=True)

# src/datalabel/services/audit_service.py
class AuditService:
    async def log(self, org_id: uuid.UUID, actor_id: uuid.UUID | None, action: str,
                  resource_type: str, resource_id: uuid.UUID,
                  changes: dict | None = None, ip_address: str | None = None) -> None:
        """Write an audit log entry. Called by service layer on every state mutation."""
        ...

    async def export_annex_iv(self, project_id: uuid.UUID) -> dict:
        """Generate EU AI Act Annex IV compliant documentation for a project.

        Output structure:
        {
            "project": { "name", "description", "modality", "created_at" },
            "dataset": { "name", "row_count", "media_types", "storage_location" },
            "ontology": { "version", "features", "guidelines_hash" },
            "labeling_methodology": {
                "assignment_strategy", "review_stages", "qa_rules",
                "pre_label_models": [{"name", "version", "confidence_threshold"}]
            },
            "annotator_pool": [{"id", "expertise", "tasks_completed", "acceptance_rate"}],
            "quality_metrics": { "iaa_scores", "qa_pass_rate" },
            "data_lineage": [
                {"event", "actor", "timestamp", "details"}
            ]
        }
        """
        ...
```

API endpoints:
- `GET /api/v1/organisations/{org_id}/audit-log` — list audit entries (paginated, filterable by resource_type, action, date range)
- `GET /api/v1/projects/{project_id}/provenance/annex-iv` — export Annex IV documentation (JSON)

**Testing**:
- `Integration: test_audit_on_task_create — creating a task writes audit log with action="task.created"`
- `Integration: test_audit_on_annotation_update — updating annotation logs old and new values`
- `Integration: test_audit_pagination — 100 entries, page_size=20, correct pagination`
- `Integration: test_audit_filter_by_resource — filter resource_type="annotation" returns only annotation events`
- `Integration: test_annex_iv_export — project with annotations produces valid Annex IV JSON`
- `Integration: test_annex_iv_includes_provenance — export includes pre-label model versions and annotator pool`

---

## Phase 11: Python SDK & Natural Language Task Specification

### Purpose

Build the official Python SDK for programmatic access and implement natural-language task specification: describe a labeling task in plain English and have the platform generate an annotation schema, labeling guide, and quality rubric. After this phase, ML engineers can automate their labeling pipelines via code and non-technical users can create projects by describing them.

### Tasks

#### 11.1 — Python SDK

**What**: Client library published to PyPI that wraps the REST API with Pydantic models.

**Design**:

```python
# sdk/src/datalabel_sdk/client.py
import httpx
from typing import AsyncIterator

class DataLabelClient:
    """Synchronous client for the Data Labeling Platform API."""

    def __init__(self, api_url: str, api_key: str):
        self._client = httpx.Client(
            base_url=api_url,
            headers={"X-API-Key": api_key},
            timeout=30.0
        )

    # Organisation
    def list_organisations(self) -> list[Organisation]: ...
    def get_organisation(self, org_id: str) -> Organisation: ...

    # Projects
    def create_project(self, org_id: str, name: str, modality: str, dataset_id: str, **kwargs) -> Project: ...
    def list_projects(self, org_id: str, status: str | None = None) -> list[Project]: ...
    def get_project(self, project_id: str) -> Project: ...

    # Datasets
    def create_dataset(self, org_id: str, name: str, storage_config: dict | None = None) -> Dataset: ...
    def upload_data_row(self, dataset_id: str, file_path: str, media_type: str, metadata: dict | None = None) -> DataRow: ...
    def upload_batch(self, dataset_id: str, rows: list[dict]) -> list[DataRow]: ...

    # Ontology
    def set_ontology(self, project_id: str, features: list[dict]) -> Ontology: ...

    # Tasks
    def generate_tasks(self, project_id: str, dataset_id: str) -> list[Task]: ...
    def get_next_task(self, project_id: str) -> Task | None: ...
    def submit_task(self, task_id: str) -> Task: ...

    # Annotations
    def create_annotation(self, task_id: str, feature_id: str, annotation_type: str, content: dict) -> Annotation: ...
    def list_annotations(self, task_id: str) -> list[Annotation]: ...

    # Export
    def request_export(self, project_id: str, format: str = "coco", filters: dict | None = None) -> ExportJob: ...
    def wait_for_export(self, export_id: str, timeout: int = 300) -> ExportJob: ...
    def download_export(self, export_id: str, output_path: str) -> str: ...

    # Quality
    def compute_iaa(self, project_id: str) -> list[QualityMetric]: ...
    def get_quality_dashboard(self, project_id: str) -> QualityDashboard: ...

class AsyncDataLabelClient:
    """Async variant using httpx.AsyncClient."""
    ...

# sdk/src/datalabel_sdk/exceptions.py
class DataLabelError(Exception): ...
class AuthenticationError(DataLabelError): ...
class NotFoundError(DataLabelError): ...
class ValidationError(DataLabelError): ...
class RateLimitError(DataLabelError): ...
```

**Testing**:
- `Unit: test_client_auth_header — client sends X-API-Key header on all requests`
- `Unit: test_client_error_handling — 404 response raises NotFoundError`
- `Unit: test_client_rate_limit — 429 response raises RateLimitError`
- `Integration (mocked): test_create_project_flow — create org, create dataset, upload rows, create project, set ontology`
- `Integration (mocked): test_export_flow — request export, poll status, download file`
- `Integration (mocked): test_annotation_flow — get task, create annotations, submit task`
- `Unit: test_async_client — AsyncDataLabelClient methods are coroutines`

#### 11.2 — Natural Language Task Specification

**What**: LLM-powered feature that converts a plain-English description of a labeling task into an ontology schema, markdown labeling guidelines, and QA rules.

**Design**:

```python
# src/datalabel/services/nlp_task_service.py
from openai import AsyncOpenAI  # or anthropic

class NLPTaskService:
    """Generate annotation schemas from natural language descriptions."""

    SYSTEM_PROMPT = """You are an annotation schema designer for a data labeling platform.
Given a natural language description of a labeling task, generate:
1. An ontology schema with features, tools, and attributes
2. Markdown labeling guidelines for annotators
3. QA rules to check annotation quality

Respond in JSON matching the provided schema."""

    async def generate_from_description(self, description: str, modality: str) -> TaskSpecification:
        """Send the description to an LLM and parse the structured output.

        Args:
            description: e.g., "Label all vehicles in dashcam images. Mark cars, trucks, and motorcycles
                          with bounding boxes. Note if they are partially occluded."
            modality: "image", "text", or "llm_pair"

        Returns:
            TaskSpecification with ontology features, guidelines, and QA rules.
        """
        ...

@dataclass
class TaskSpecification:
    ontology_features: list[dict]  # OntologyFeature-compatible dicts
    guidelines_md: str             # Markdown labeling instructions
    qa_rules: list[dict]           # QARule-compatible dicts
    confidence: float              # LLM's self-assessed confidence in the spec

# Example input/output:
# Input: "Classify customer support emails as positive, negative, or neutral sentiment.
#          Also extract any product names mentioned."
# Output:
# {
#   "ontology_features": [
#     {"id": "feat_sentiment", "name": "Sentiment", "type": "classification", "tool": "radio",
#      "options": ["positive", "negative", "neutral"]},
#     {"id": "feat_product", "name": "Product Name", "type": "object", "tool": "ner_span",
#      "attributes": [{"name": "product_category", "type": "enum", "options": ["hardware", "software", "service"]}]}
#   ],
#   "guidelines_md": "## Sentiment Classification\n\n...",
#   "qa_rules": [
#     {"name": "sentiment_required", "type": "threshold", "field": "annotation_count", "operator": ">=", "value": 1}
#   ]
# }
```

API endpoints:
- `POST /api/v1/projects/generate-spec` — generate task specification from description
- `POST /api/v1/projects/{project_id}/apply-spec` — apply generated specification to project (creates ontology + guidelines + QA rules)

**Testing**:
- `Integration (mocked LLM): test_generate_cv_spec — "Label vehicles" produces bounding_box feature with vehicle class`
- `Integration (mocked LLM): test_generate_nlp_spec — "Classify sentiment" produces classification feature`
- `Integration (mocked LLM): test_generate_rlhf_spec — "Compare responses" produces ranking feature`
- `Integration: test_apply_spec — generated spec creates ontology, sets guidelines, creates QA rules`
- `Unit: test_spec_validation — generated spec validates against OntologyFeature schema`
- `Unit: test_spec_confidence_threshold — low-confidence spec warns user before applying`

---

## Phase 12: Cloud Storage Connectors, Docker Deployment, & Hardening

### Purpose

Production-ready deployment with cloud storage connectors, Docker optimization, health checks, rate limiting, CORS configuration, and security hardening. After this phase, the platform can be deployed in production for teams using it with real data.

### Tasks

#### 12.1 — Cloud Storage Connectors (S3, GCS, Azure)

**What**: Complete the storage abstraction with production-ready S3, GCS, and Azure Blob connectors including multipart upload, presigned URLs, and connection pooling.

**Design**:

```python
# Full implementation of storage backends from Phase 2.1
# with production additions:

class S3StorageBackend(StorageBackend):
    async def upload_multipart(self, key: str, file_path: str, part_size: int = 10 * 1024 * 1024) -> StorageObject:
        """Multipart upload for files > 100MB."""
        ...

    async def generate_upload_presigned_url(self, key: str, content_type: str, expires_in: int = 3600) -> str:
        """Generate presigned PUT URL for direct client-side upload."""
        ...

# Environment variables:
# DATALABEL_STORAGE_BACKEND=s3
# DATALABEL_AWS_ACCESS_KEY_ID=...
# DATALABEL_AWS_SECRET_ACCESS_KEY=...
# DATALABEL_AWS_S3_BUCKET=my-datalabel-bucket
# DATALABEL_AWS_S3_REGION=us-east-1
```

**Testing**:
- `Integration (mocked): test_s3_multipart_upload — file >100MB uploaded in parts`
- `Integration (mocked): test_s3_presigned_upload_url — client can PUT directly to S3`
- `Integration (mocked): test_gcs_upload_download — roundtrip upload/download on GCS`
- `Integration (mocked): test_azure_upload_download — roundtrip upload/download on Azure`
- `Unit: test_storage_factory — each backend string creates correct class`

#### 12.2 — Production Docker & Health Checks

**What**: Optimized multi-stage Dockerfile, health check endpoints, graceful shutdown, and production Docker Compose configuration.

**Design**:

```dockerfile
# Dockerfile (multi-stage)
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir build && python -m build

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /app/dist/*.whl .
RUN pip install --no-cache-dir *.whl && rm *.whl
COPY alembic.ini .
COPY alembic/ alembic/
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "datalabel.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```python
# Health check endpoint
# GET /health -> {"status": "healthy", "database": "connected", "redis": "connected", "version": "1.0.0"}
# GET /ready  -> {"status": "ready"} (only after migrations applied and services connected)

# Rate limiting middleware
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address, default_limits=["100/minute"])

# CORS configuration
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Testing**:
- `Integration: test_health_endpoint — GET /health returns 200 with database and redis status`
- `Integration: test_ready_endpoint — GET /ready returns 200 after startup`
- `Integration: test_rate_limiting — 101st request in 1 minute returns 429`
- `Integration: test_cors_allowed_origin — request from allowed origin includes CORS headers`
- `Integration: test_cors_blocked_origin — request from blocked origin has no CORS headers`
- `E2E: test_docker_compose_production — docker compose -f docker-compose.prod.yml up, all services healthy`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Data Management              ─── requires Phase 1
    │
Phase 3: Project & Ontology           ─── requires Phase 1
    │
    ├── Phase 4: Task & Annotation     ─── requires Phase 2 + Phase 3
    │       │
    │       ├── Phase 5: Annotation UI ─── requires Phase 4
    │       │       │
    │       │       └── Phase 8: Pre-labeling ─── requires Phase 5 (UI) + Phase 4 (API)
    │       │                │
    │       │                └── Phase 9: Active Learning ─── requires Phase 8
    │       │
    │       ├── Phase 6: Quality Control ─── requires Phase 4
    │       │
    │       └── Phase 7: Export Pipeline ─── requires Phase 4
    │
    └── Phase 10: Webhooks & Audit     ─── requires Phase 4 + Phase 6
            │
            └── Phase 11: SDK & NLP Spec ─── requires Phase 4 + Phase 7 (SDK needs stable API)
                    │
                    └── Phase 12: Production Hardening ─── requires all previous phases

Parallelism opportunities:
  - Phases 2 and 3 can be developed concurrently after Phase 1
  - Phases 5, 6, and 7 can be developed concurrently after Phase 4
  - Phase 8 can start once Phase 4 API is stable (Phase 5 UI can follow)
  - Phase 10 can be developed in parallel with Phases 8-9
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with working code.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. E2E tests pass where applicable (`pytest tests/e2e/`).
5. Ruff linting passes with zero errors (`ruff check src/`).
6. Ruff formatting passes (`ruff format --check src/`).
7. mypy type checking passes (`mypy src/datalabel/`).
8. Database migrations are created and apply cleanly (`alembic upgrade head`).
9. Docker build succeeds (`docker build -t datalabel .`).
10. Docker Compose stack starts and all services are healthy.
11. New API endpoints appear in auto-generated OpenAPI spec (`/docs`).
12. New configuration options have defaults and are documented in config.py.
13. No TODO or FIXME markers left in committed code for the phase's scope.
