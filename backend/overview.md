# Backend Overview

**Directory**: `/siapp` (first build) → rebuild per `syap/docs/plan`

## 🎯 Purpose
The **SiAPP Backend** is the server of the **PERSIST Security add-on**: one Go API that powers the SiAPP mobile app (field staff) and the web dashboard (admins) for both the `security` and `cleaning` business lines. It runs **on top of Persist Personalia**, which owns people, passwords, companies and the official attendance record. SiAPP owns the field operations layered on top: teams, shifts, patrol, panic, reports and the rest.

## 🔑 Key Responsibilities
*   **Session Provider**: Signs people in with their Personalia email + password (checked by Personalia), enforces one bound phone per person, and issues SiAPP's own access and refresh tokens. SiAPP stores no passwords.
*   **Tenant Guard**: Every row belongs to an organization (the person's employing company in Personalia). Postgres row-level security keeps one company's data invisible to another.
*   **Field Operations**: Attendance capture (selfie + rotating QR), patrol and guard posts, panic, reports, instructions, CCTV, assets, and the cleaning extras (E-Memo, QR).
*   **Organization Overlay**: Teams (regu), DANRU/member roles and shifts on top of Personalia's projects and placements.
*   **Personalia Bridge**: The only component that talks to Personalia: login, people lookup by NIK, attendance push (outbox), leave, and change webhooks.
*   **Real-time Communication**: FCM push (with a dedicated panic channel), WebSocket rooms for live tracking and panic, an in-app inbox.

## 🏗 Tech Stack
*   **Language**: Go
*   **HTTP**: `net/http` + chi
*   **Database**: PostgreSQL 16 with row-level security (via sqlc + pgx, migrations with goose)
*   **Cache / Queue**: Redis, asynq (jobs and cron)
*   **Realtime**: `coder/websocket` with Redis pub/sub fan-out; FCM for push
*   **Storage**: S3-compatible (MinIO locally), presigned URLs
*   **API Contract**: `persist-ecosystem/contracts/openapi.yaml`, types generated with oapi-codegen
*   **Logging**: `log/slog` (JSON)

## 📍 Status
*   `/siapp` holds the **first build** (MySQL, local users): platform scaffold and auth only. It is being realigned to the plan (Postgres + RLS, login through Personalia).
*   Everything else in [features.md](features.md) is **planned**, phase by phase, in `syap/docs/plan/`.
*   The Personalia side of the integration is built by us in `kpi-personalia-api` (`syap/docs/plan/personalia-integration.md`).
