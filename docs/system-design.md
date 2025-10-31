# In-Store Visibility Platform — System Design Overview

## 1. Vision & Value Proposition
The In-Store Visibility Platform orchestrates retail visibility projects across Pakistan from ideation through execution. It unifies planning, approvals, production, and field execution into a single workflow that blends an infinite-canvas flow builder with SLA-driven project automation, geo-aware store management, and omnichannel (web/email/QR) collaboration.

Key differentiators:
- **Visual Flow Maker:** Low-code infinite canvas for tailoring multi-stage workflows with SLA and permission rules per stage.
- **Email-First Collaboration:** Bi-directional email approvals, quoted-reply ingestion, and magic-link actions keep agencies and brand stakeholders in sync without forcing context switching.
- **Geo-Aware Execution:** Pakistan map overlays, store history, and QR-powered mobile uploads ensure on-ground work is validated and auditable.
- **SLA Automation:** Configurable calendars, escalation tiers, and breach reporting enforce accountability across diverse partner groups.

## 2. Personas & Responsibilities
| Persona | Responsibilities | Key Modules |
| --- | --- | --- |
| Master Admin | Configure groups, flows, SLA calendars, escalation policies, and email routing. | Admin Console, Flow Maker, SLA Maker |
| Visibility Team | Owns projects, approves stage gates, monitors SLA dashboards. | Project Workspace, Dashboards |
| Agency Team (Yellow, Brand Logic) | Submits designs, surveys, production updates, execution proofs. | Project Workspace, Files, QR Uploads |
| AM/TM | Creates opportunities, proposes store lists, liaises with stores. | Project Workspace (Stores tab), Map |
| On-ground Team | Performs surveys, captures geo-tagged before/after media, logs field notes. | QR/OTP Mobile Uploads, Map |
| BU/Brand | Provides briefs, feedback, and final approvals. | Project Workspace, Email approvals |
| Store POC | Supplies access windows, confirms on-site status. | Guest QR uploads, Comments |
| Guest | Single-use uploads or confirmations via scoped QR links. | QR/OTP interface |

## 3. Experience Pillars
1. **Flow Lifecycle Management**
   - Infinite canvas using React Flow for drag-and-drop stage authoring.
   - Stage inspector for assignee groups, SLA policies, entry/exit rules, and required artifacts.
   - Versioned flow publishing (v1, v2…) with projects locked to the version they launch on.
2. **Project Workspace**
   - Stage rail showing status chips, SLA countdown, assignees, and breach indicators.
   - Tabs for Overview, Stages, Stores, Files, Comments, Emails, Audit.
   - Comment composer supports rich text, mentions, and rendered quoted email text.
3. **Omnichannel Approvals**
   - Outbound notifications with magic links (Approve, Reject, Snooze, Reassign).
   - Email reply parsing for command detection (`APPROVE`, `REJECT: reason`, `SNOOZE 24h`, `REASSIGN @group`).
   - Quoted text extraction to maintain threaded context inside comments.
4. **Geo & Mobile Enablement**
   - Pakistan map with PostGIS-backed store layer, heatmaps, and polygon filters.
   - Store drawer surfaces latest media, open tasks, and contact info.
   - QR/OTP guest links allow scoped uploads with EXIF validation and proximity checks.
5. **SLA & Escalation Governance**
   - Business calendars per region (Sindh/Punjab) with working hours and holidays.
   - Tiered escalation policies (80%, 100%, 125%) driving email escalations.
   - Dashboards showing SLA compliance, breach root cause, agency performance.

## 4. Architecture Blueprint
### 4.1 Frontend
- **Framework:** Next.js (React + TypeScript) with TailwindCSS.
- **Infinite Canvas:** React Flow for stage node authoring.
- **State Management:** Zustand for workspace state; React Query for API data fetching.
- **Map:** Mapbox GL JS (with OSM tiles fallback) driven by store GeoJSON.
- **Realtime:** Socket.IO client for live updates on stages and SLA countdowns.
- **Upload Handling:** Pre-signed S3 URLs, dropzone components, image EXIF parsing.

### 4.2 Backend
- **Framework:** NestJS (TypeScript) for modular architecture (Auth, Projects, Flows, Email, SLA, Stores, Files).
- **Database:** PostgreSQL + PostGIS extension; Prisma ORM for schema management.
- **Queues:** Redis + BullMQ for SLA timers, email processing, and file virus scans.
- **Storage:** S3-compatible (AWS S3 / MinIO) with server-side encryption and lifecycle policies.
- **Email:** Microsoft 365 Graph API for outbound; SendGrid/Mailgun inbound parse webhooks.
- **Auth:** OIDC (Azure AD) + email OTP fallback; JWT access + refresh tokens; signed QR JWT for guest flows.
- **Observability:** OpenTelemetry traces, Prometheus metrics, Loki logs via Grafana stack.

### 4.3 Service Boundaries
| Service Module | Responsibilities |
| --- | --- |
| Auth Service | OIDC/OTP login, JWT issuance, QR token generation, RBAC enforcement. |
| Flow Service | CRUD flows, stage configurations, validation, versioning. |
| Project Service | Project lifecycle, stage transitions, audit logging. |
| SLA Service | Timer scheduling, business calendar management, escalation dispatching. |
| Email Service | Outbound templates, inbound parsing, command detection, comment creation. |
| Store Service | Store imports, map overlays, geo-queries, store-project associations. |
| File Service | Pre-signed uploads, virus scanning, metadata extraction, EXIF validation. |
| Notification Service | WebSocket updates, push digest generation. |

### 4.4 Deployment Topology
- Dockerized microservices deployed via Kubernetes (optional) or Docker Compose per environment.
- CI/CD pipeline (GitHub Actions) running lint → tests → build → migrations → deploy.
- Secrets stored in HashiCorp Vault or cloud secrets manager.
- Environments: Dev, Staging, Prod with isolated Redis/S3 buckets and SMTP credentials.

## 5. Data Model Highlights
Entities align with the provided SRS:
- `users`, `groups`, `user_group_memberships` for RBAC.
- `flows`, `flow_stages`, `projects`, `project_stages` for workflow tracking.
- `sla_policies`, `escalation_policies`, `notifications` powering timers and alerts.
- `inbound_emails`, `approvals`, `comments`, `files` for collaboration history.
- `stores`, `store_projects`, `business_calendars`, `exceptions` for geo and SLA nuance.
- `audit_logs` capturing every action for compliance.

Prisma schema fragments will closely mirror these tables with JSONB fields for flexible metadata (`meta_json`, `rules_json`, etc.).

## 6. Key Workflows
### 6.1 Flow Authoring
1. Admin opens Flow Maker, drags stages onto canvas, connects via edges.
2. Assigns groups, SLA policies, required fields/uploads via inspector panel.
3. Saves draft, runs validation (no dangling nodes, rules satisfied).
4. Publishes version; default template is available for new projects.

### 6.2 Project Execution
1. Visibility Lead creates project selecting flow template.
2. Stage activation triggers assignment email + WebSocket update.
3. Assignees complete required fields/uploads; Approver replies via email or web.
4. Approval transitions stage; SLA timers reset for next stage.
5. SLA breaches create escalations and appear on dashboards.
6. Project closes when final QA stage approved; audit log compiled.

### 6.3 Email-In Approval
1. Email service receives webhook, validates signature, stores raw EML on S3.
2. Parser strips signatures, extracts quoted text, detects commands.
3. Resolves sender → user/group; verifies stage permissions.
4. Creates comment with body & quoted blocks; if command found, creates approval and transitions stage.
5. Idempotency ensured via `message_id` uniqueness.

### 6.4 QR/OTP Upload
1. Stage generates QR link with scope (project, stage, store).
2. On-ground user scans QR → obtains OTP or one-time JWT.
3. Mobile upload page enforces EXIF GPS within 200m of store; prompts for required artifacts.
4. Upload stored to S3, metadata saved in `files`; stage entry validated against required uploads.

## 7. SLA & Escalation Engine
- SLA timers scheduled via BullMQ repeatable jobs keyed by `project_stage_id`.
- Business calendars convert working hours into effective durations; pauses applied for `waiting` status or approved exceptions.
- Escalation tiers defined in JSON (tier, offset seconds, email template, target group).
- Breach events recorded in `audit_logs` and surfaced via dashboards and email alerts.

## 8. Dashboards & Analytics
- **SLA Compliance Dashboard:** Stage-level gauges, breach table, trend lines.
- **Agency Performance:** Compare average turnaround vs SLA, penalties/credits.
- **Geo Heatmap:** Active tasks density per region, store-level status.
- **Project Funnel:** Stage throughput, rejection counts, loop frequency.

Power BI or embedded Metabase dashboards can connect to read replicas for additional analytics.

## 9. Security & Compliance
- Role-based access enforced at controller and resolver layers with ABAC checks.
- Signed URLs for uploads/downloads with expiry; ClamAV scanning for viruses.
- PII minimization (store contacts, phone/email) with encryption at rest (Pgcrypto or column-level).
- Rate limiting on inbound webhooks and auth endpoints; audit logs immutable via append-only table.
- Quarterly access review reports exportable for compliance.

## 10. Implementation Roadmap
### Phase 0 (Weeks 1–3)
- Auth foundation (OIDC/OTP), groups, RBAC scaffolding.
- Database schema migration baseline.
- Seed default flows and SLA policies from template definitions.

### Phase 1 (Weeks 4–6)
- Project workspace MVP with stage transitions and outbound notifications.
- SLA engine baseline with timers and Tier-1 escalations.

### Phase 2 (Weeks 7–9)
- Email inbound parser, approvals, quoted comment rendering.
- Tier-2/Tier-3 escalations and breach dashboards.

### Phase 3 (Weeks 10–12)
- Map & Stores module, QR/OTP guest uploads, EXIF validation.

### Phase 4 (Weeks 13–15)
- Advanced dashboards, penalty/credit reporting, hardening, UAT scripts.

## 11. Open Questions
- Single vs multi-tenant SSO for agencies? Impacts OIDC setup and group provisioning.
- Mapbox vs OSM licensing costs and offline tile caching needs.
- Required PSD maximum size and storage retention policy.
- Regional calendar variations (Sindh vs Punjab) to parameterize in business calendars.

## 12. Success Metrics
- ≥95% of stages completed within SLA (post-rollout).
- <5% manual interventions required for email parsing errors in pilot.
- Store survey geo compliance ≥98% (uploads within 200m radius).
- User satisfaction (CSAT) ≥4/5 across Visibility and Agency stakeholders.

## 13. Appendix
- Reference the consolidated SRS for exhaustive field definitions, SLA matrices, and acceptance criteria.
- Future considerations: budgeting module, vendor payment tracking, computer vision QC (beyond v1).
