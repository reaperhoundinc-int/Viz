# Viz — In-Store Visibility Platform

Comprehensive design and architecture outline for a responsive Node.js-based workflow system that manages in-store visibility projects across Pakistan from ideation through execution.

## Highlights
- Infinite-canvas flow maker with drag-and-drop stages, SLA rules, and role-based permissions.
- Group and SLA makers for configuring organizations, escalation policies, and calendars.
- Project workspace with omnichannel approvals, quoted email capture, and media/file management.
- Geo-aware store module with Pakistan map overlays, QR/OTP guest uploads, and EXIF validation.
- SLA automation engine with tiered escalations, dashboards, and audit trails.

## Documentation
- [System Design Overview](docs/system-design.md): Detailed architecture, workflows, roadmap, and success metrics aligned with the merged SRS/SoW v1.1.

## Tech Stack (Proposed)
- **Frontend:** Next.js (React + TypeScript), TailwindCSS, React Flow, Mapbox GL JS, Socket.IO.
- **Backend:** NestJS (TypeScript), PostgreSQL + PostGIS, Redis/BullMQ, S3-compatible storage, Microsoft Graph / SendGrid.
- **Infra:** Dockerized services, CI/CD (GitHub Actions), OpenTelemetry + Grafana/Loki/Tempo observability.

## Getting Started (Discovery)
1. Review the [System Design Overview](docs/system-design.md).
2. Identify open questions (SSO tenancy, mapping vendor, PSD limits, regional calendars).
3. Plan engineering discovery & estimation against the proposed 5-phase rollout.
