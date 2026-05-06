# Standards & API Reference

> Project: Data Labeling Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 25012:2008 — Data Quality Model**
https://www.iso.org/standard/35736.html
Defines fifteen data quality characteristics (accuracy, completeness, consistency, credibility, currentness, accessibility, compliance, confidentiality, efficiency, precision, traceability, understandability, availability, portability, recoverability). Essential reference for defining quality metrics within a data labeling platform's QA framework.

**ISO/IEC 25024:2015 — Measurement of Data Quality**
https://www.iso.org/standard/35749.html
Specifies 63 measures for quantifying data quality levels relative to the characteristics in ISO/IEC 25012. Applicable to building inter-annotator agreement scoring, accuracy dashboards, and automated quality gates.

**ISO/IEC 5259-2:2024 — Data Quality for Analytics and Machine Learning**
https://aistandardshub.org/ai-standards/artificial-intelligence-data-quality-for-analytics-and-machine-learning-ml-part-2-data-quality-measures/
The AI-specific evolution of the ISO 25012/25024 family. Extends quality measures to cover dataset-level characteristics required for ML model training, validation, and evaluation. Directly applicable to building a training-data quality platform.

**ISO/IEC 27001:2022 — Information Security Management Systems**
https://www.iso.org/standard/82875.html
The baseline information security certification expected by enterprise buyers of data labeling platforms. Governs data handling, access control, audit logging, and incident management. Essential for platforms handling sensitive training data (medical, financial, government).

**ISO 9001:2015 — Quality Management Systems**
https://www.iso.org/standard/62085.html
Applied by annotation service providers (Appen, DIGI-TEXX, iMerit) as a process quality certification. Relevant for platforms providing managed workforce services or quality SLAs.

---

### W3C & IETF Standards

**W3C Web Annotation Data Model (Recommendation, 2017)**
https://www.w3.org/TR/annotation-model/
Defines a structured JSON-LD model for annotations on web resources. Provides a semantic framework (Body, Target, Motivation) that generalises across text, image, and audio annotations. A valuable reference data model for designing platform-agnostic annotation schemas.

**W3C Web Annotation Protocol (Recommendation, 2017)**
https://www.w3.org/TR/annotation-protocol/
Specifies HTTP-based transport mechanisms for creating, retrieving, updating, and deleting annotations, consistent with REST and Web Architecture principles. Relevant to designing the platform's annotation CRUD API.

**W3C Web Annotation Vocabulary (Recommendation, 2017)**
https://www.w3.org/TR/annotation-vocab/
Specifies RDF classes, predicates, and named entities for use with the Web Annotation Data Model. Provides vocabulary terms for annotation types, roles, and states.

**RFC 7517 — JSON Web Key (JWK)**
https://datatracker.ietf.org/doc/html/rfc7517
Defines the JSON structure for representing cryptographic keys. Relevant to Encord's public-private key authentication model and any API key management system.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
https://datatracker.ietf.org/doc/html/rfc6749
Industry standard for delegated authorization. All major commercial labeling platforms (Labelbox, Encord, Scale, Kili) rely on OAuth 2.0 for third-party integrations and SSO. Essential for any platform API that needs user delegation and enterprise SSO.

**RFC 7519 — JSON Web Tokens (JWT)**
https://datatracker.ietf.org/doc/html/rfc7519
Standard token format used in OAuth 2.0 / OIDC authentication flows. Widely used for API authentication across the labeling platform ecosystem.

**OpenID Connect Core 1.0**
https://openid.net/specs/openid-connect-core-1_0.html
Identity layer on top of OAuth 2.0 providing user authentication and single sign-on. Enterprise buyers expect OIDC-compatible SSO from SaaS annotation platforms.

---

### Data Model & API Specifications

**COCO Dataset Format (Microsoft, 2015)**
https://cocodataset.org/#format-data
De facto standard JSON annotation format for computer vision datasets. Covers five annotation types: object detection (bounding boxes), keypoint detection, stuff segmentation, panoptic segmentation, and image captioning. All major annotation platforms (CVAT, Label Studio, Labelbox, Encord, SuperAnnotate) export to COCO. Structure: `{"images": [...], "annotations": [...], "categories": [...]}`.

**Pascal VOC Format**
https://host.robots.ox.ac.uk/pascal/VOC/
XML-based annotation format originating from the PASCAL Visual Object Classes challenge. Second most common CV export format; supported by all major platforms. Relevant for interoperability with legacy ML pipelines.

**YOLO Annotation Format (Ultralytics)**
https://docs.ultralytics.com/datasets/
Plain-text, per-image annotation format used natively by YOLO model family. Each line: `<class_id> <x_center> <y_center> <width> <height>` (normalised). Essential for real-time object detection training pipelines.

**Darwin JSON Format (V7 Labs)**
https://docs.v7labs.com/reference/introduction
Open annotation format from V7 Labs that preserves ontology metadata and multi-stage workflow state beyond what COCO supports. Not yet a formal standard but notable as the most serious attempt at a richer, workflow-aware annotation interchange format.

**OpenAPI Specification 3.1**
https://spec.openapis.org/oas/v3.1.0.html
The current industry standard for describing RESTful APIs in machine-readable YAML or JSON. OpenAPI 3.1 is a full superset of JSON Schema Draft 2020-12. All major platforms (Scale, Labelbox, CVAT, Encord) publish or generate OpenAPI-compatible API documentation. Essential for building a discoverable, developer-friendly platform API.

**JSON Schema (Draft 2020-12)**
https://json-schema.org/specification.html
Standard for describing the structure and validation rules for JSON data. Used in OpenAPI 3.1 schema definitions and relevant to designing annotation schema validation within the platform.

**DICOM (Digital Imaging and Communications in Medicine)**
https://www.dicomstandard.org/
The dominant standard for medical imaging data exchange. DICOM defines both the file format and the network communication protocol for radiology images (CT, MRI, X-ray). Required for any data labeling platform targeting healthcare or medical AI use cases. Supported by Encord and Kili.

**NIfTI (Neuroimaging Informatics Technology Initiative)**
https://nifti.nimh.nih.gov/nifti1
Standard file format for neuroimaging data (MRI, fMRI). Required alongside DICOM for neuroscience and brain imaging annotation use cases. Supported by Encord.

---

### Security & Authentication Standards

**OWASP API Security Top 10 (2023)**
https://owasp.org/www-project-api-security/
Framework for identifying and mitigating the ten most critical API security risks. Directly applicable to securing annotation platform APIs (broken object-level authorisation, excessive data exposure, etc.).

**NIST Cybersecurity Framework 2.0 (2024)**
https://www.nist.gov/cyberframework
The US federal reference framework for cybersecurity risk management. Relevant for platforms serving US government or defence customers (Scale AI's core market).

**SOC 2 Type II (AICPA)**
https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-excellence/system-and-organization-controls-soc-suite-of-services
The de facto SaaS security audit standard. Trust Service Criteria: Security, Availability, Processing Integrity, Confidentiality, Privacy. SOC 2 Type II compliance is expected by enterprise buyers; held by Encord, Labelbox, Scale, SuperAnnotate.

**HIPAA (Health Insurance Portability and Accountability Act)**
https://www.hhs.gov/hipaa/index.html
US healthcare data privacy law. Governs annotation workflows involving patient data (medical imaging, clinical notes). Platforms targeting healthcare AI must be HIPAA-compliant (Encord, Kili).

**GDPR (General Data Protection Regulation)**
https://gdpr.eu/
EU data privacy regulation. Governs annotation workflows involving EU personal data; requires data minimisation, subject access rights, and data processing agreements. All major commercial platforms claim GDPR compliance.

---

### Regulatory Frameworks for AI Training Data

**EU AI Act — Article 10 & Annex IV**
https://artificialintelligenceact.eu/article/10/
https://artificialintelligenceact.eu/annex/4/
Article 10 mandates data governance for high-risk AI systems: training, validation, and testing datasets must be relevant, representative, and as free of errors as possible. Annex IV requires technical documentation including training data provenance, labeling methodology, preprocessing steps, and data lineage. High-risk AI systems under Annex III must comply by 2 August 2026. A data labeling platform that captures and exports Annex IV-compliant metadata would be highly differentiated for EU enterprise customers.

**RLHF / RLAIF (de facto industry practice)**
No formal standard body; originating papers: Christiano et al. (2017) "Deep Reinforcement Learning from Human Preferences"; Bai et al. (2022) "Constitutional AI". The dominant paradigm for LLM alignment training driving the largest growth segment of the data labeling market. Key annotation types: pairwise preference ranking, multi-turn conversation rating, instruction following demonstration.

---

## Similar Products — Developer Documentation & APIs

### Scale AI

- **Description:** The largest commercial data labeling platform offering managed workforce annotation, RLHF data collection, and LLM evaluation at enterprise scale. Acquired a $15B Meta investment in 2024.
- **API Documentation:** https://scale.com/docs/api-reference/introduction-to-scale-api
- **SDKs/Libraries:** Python SDK via `pip install scale-api`; CLI tools
- **Developer Guide:** https://scale.com/docs/overview
- **Standards:** REST/JSON; task-oriented resource model (Projects, Tasks)
- **Authentication:** API key passed as HTTP Basic Auth username; Bearer token for newer endpoints

### Labelbox

- **Description:** Enterprise annotation and LLM evaluation platform with native pairwise comparison, Model Foundry, and RLHF preference collection workflows.
- **API Documentation:** https://docs.labelbox.com/reference/getting-started
- **SDKs/Libraries:** Python SDK — `pip install labelbox`; https://github.com/Labelbox/labelbox-python; PyPI: https://pypi.org/project/labelbox/
- **Developer Guide:** https://docs.labelbox.com/
- **Standards:** GraphQL primary API (REST supplementary); Python SDK is the recommended integration path as GraphQL endpoints may be deprecated without notice
- **Authentication:** API key (Bearer token); OAuth 2.0 for SSO integrations

### Encord

- **Description:** AI-assisted annotation platform with leading medical imaging support (DICOM, NIfTI), video-native annotation, and SAM 2 integration. GDPR, SOC 2, and HIPAA compliant.
- **API Documentation:** https://docs.encord.com/platform-documentation/Annotate/annotate-api-overview
- **SDKs/Libraries:** Python SDK — `pip install encord`; https://github.com/encord-team/encord-client-python; PyPI: https://pypi.org/project/encord/; API reference: https://python.docs.encord.com/
- **Developer Guide:** https://docs.encord.com/
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** Public-private key pair; service account model — public key uploaded to platform, private key used to sign SDK requests

### CVAT (Computer Vision Annotation Tool)

- **Description:** Open-source (MIT) computer vision annotation tool maintained by CVAT.ai and OpenCV Foundation. Self-hostable via Docker Compose or Kubernetes; supports bounding boxes, polygons, polylines, keypoints, 3D cuboids.
- **API Documentation:** https://docs.cvat.ai/docs/api_sdk/ ; Swagger UI at `<host>/api/docs/`
- **SDKs/Libraries:** Python SDK — `pip install cvat-sdk`; PyPI: https://pypi.org/project/cvat-sdk/; GitHub: https://github.com/cvat-ai/cvat
- **Developer Guide:** https://docs.cvat.ai/docs/api_sdk/sdk/developer-guide/
- **Standards:** REST/JSON with Swagger/OpenAPI schema; Django REST Framework
- **Authentication:** API key or session-based authentication; CVAT.ai cloud uses OAuth 2.0 / OIDC

### Label Studio

- **Description:** Open-source (Apache 2.0) multi-modal annotation platform for text, images, audio, video, time-series, and agent traces. The most flexible open-source annotation schema system (XML-based custom interfaces).
- **API Documentation:** https://api.labelstud.io/api-reference/introduction/getting-started ; https://labelstud.io/guide/api
- **SDKs/Libraries:** Python SDK — `pip install label-studio-sdk`; PyPI: https://pypi.org/project/label-studio-sdk/; GitHub: https://github.com/HumanSignal/label-studio-sdk; SDK v2.0.19 (March 2026)
- **Developer Guide:** https://labelstud.io/guide/
- **Standards:** REST/JSON; OpenAPI-compatible; JSON export schema with configurable structure
- **Authentication:** API token (Bearer); enterprise edition supports OAuth 2.0 / OIDC SSO

### V7 Labs Darwin

- **Description:** AI-assisted annotation platform with SAM 3 auto-detection, conditional workflow routing, and the Darwin open annotation format. Supports image, video, and document annotation.
- **API Documentation:** https://docs.v7labs.com/reference/introduction
- **SDKs/Libraries:** darwin-py Python SDK — `pip install darwin-py`; PyPI: https://pypi.org/project/darwin-py/; GitHub: https://github.com/v7labs/darwin-py; SDK docs: https://darwin-py-sdk.v7labs.com/
- **Developer Guide:** https://docs.v7labs.com/
- **Standards:** REST/JSON; Darwin JSON format for annotation interchange; exports COCO, YOLO, Pascal VOC
- **Authentication:** API key (Bearer token)

### Kili Technology

- **Description:** European (Paris-based) annotation platform with class-level consensus analysis, programmatic QA scripts, and active learning. Supports image, video, text, PDF, and satellite imagery.
- **API Documentation:** https://python-sdk-docs.kili-technology.com/latest/
- **SDKs/Libraries:** Python SDK — `pip install kili`; PyPI: https://pypi.org/project/kili/; GitHub: https://github.com/kili-technology/kili-python-sdk
- **Developer Guide:** https://python-sdk-docs.kili-technology.com/latest/
- **Standards:** GraphQL API; Python SDK wraps GraphQL; exports COCO, Pascal VOC, GeoJSON
- **Authentication:** API key; supports custom webhooks for pipeline integration

### Argilla (Hugging Face)

- **Description:** Open-source (Apache 2.0) NLP and LLM dataset curation platform with native Hugging Face Hub integration and Distilabel synthetic data generation framework. Purpose-built for LLM alignment and RLHF workflows.
- **API Documentation:** https://docs.argilla.io/
- **SDKs/Libraries:** Python SDK — `pip install argilla`; PyPI: https://pypi.org/project/argilla/; distilabel: https://github.com/argilla-io/distilabel
- **Developer Guide:** https://docs.argilla.io/dev/getting_started/quickstart/
- **Standards:** REST API; Hugging Face Hub datasets API integration; Apache Arrow / Parquet for dataset storage
- **Authentication:** API key; OAuth sign-in via Hugging Face Spaces deployment

### Roboflow

- **Description:** End-to-end computer vision platform from annotation to deployment with freemium tier, SAM-based zero-shot auto-labeling, and Roboflow Universe community dataset repository (250K+ public datasets).
- **API Documentation:** https://docs.roboflow.com/
- **SDKs/Libraries:** Python SDK — `pip install roboflow`; JavaScript SDK; PyPI: https://pypi.org/project/roboflow/
- **Developer Guide:** https://docs.roboflow.com/
- **Standards:** REST/JSON; exports COCO, YOLO, Pascal VOC, TensorFlow; OpenAPI-compatible
- **Authentication:** API key (Bearer token); workspace-scoped keys

---

## Notes

**Annotation format fragmentation:** The market lacks a widely adopted, workflow-aware interchange format. COCO is ubiquitous for CV data but loses ontology metadata, workflow state, and quality scores. The Darwin format is the closest to addressing this, but it is a proprietary format from a single vendor and not governed by a standards body. A new open-source platform has an opportunity to publish an openly governed annotation interchange format that includes quality metadata and provenance information.

**LLM annotation data models:** There is no established standard for representing LLM preference annotation data (pairwise comparisons, rankings, multi-turn conversation ratings). Each platform (Labelbox, Scale, Argilla) uses a proprietary data model. This is an area where a platform could publish and contribute an open standard.

**W3C Web Annotation adoption:** Despite being a W3C Recommendation since 2017, the Web Annotation Data Model is not used by any major labeling platform as their primary format. Its semantic richness (JSON-LD, RDF-backed) makes it a strong candidate for a future open standard in this domain.

**EU AI Act compliance tooling:** Annex IV compliance documentation for training data is a mandatory requirement for high-risk AI systems as of August 2026. No current annotation platform provides automated Annex IV-compliant audit export as a built-in feature — this represents a clear product gap and regulatory tailwind.
