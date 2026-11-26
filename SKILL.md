---
name: esign-uinrf-integration
description: >-
  Guides AI agents on integrating external and campus client applications (SIMAK, E-Office, SIPEG, etc.) with the Centralized Digital Signature Portal (eSign UIN Raden Fatah Palembang) via /api/v1/integration/*. Activate this skill whenever asked to integrate a client app with eSign UIN Raden Fatah, upload Base64 PDFs for digital signing, configure paraf/sign workflows (sequential/parallel, coordinate/anchor/invisible QR placements), pass custom_fields or external_ref_id, handle or verify HMAC-SHA256 webhooks, or poll and download signed PDFs.
---

# eSign UIN Raden Fatah — Client App Integration Guide

Use this skill when integrating a server-side client application (e.g., SIMAK, E-Office, SIPEG, or custom campus apps) with **Portal Tanda Tangan Elektronik Terpusat UIN Raden Fatah Palembang** (`/api/v1/integration/*`).

---

## 1. Architecture & Environments (Read First)

- **Protocol Style**: **Pure JSON REST API (`Content-Type: application/json`) + Base64-encoded PDFs** inspired by BSrE API v2. Do **NOT** use `multipart/form-data`.
- **Server-to-Server Only**: All `/api/v1/integration/*` requests MUST originate from your backend server. Never expose `ESIGN_APP_SECRET` or `ESIGN_WEBHOOK_SECRET` in browser JavaScript or mobile apps.
- **Environments**:
  - **Production**:
    - Portal UI: `https://esign.radenfatah.ac.id`
    - Integration Base URL: `https://esign.radenfatah.ac.id/api/v1/integration`
  - **Staging**:
    - Portal UI: `https://esign-stg.radenfatah.ac.id`
    - Integration Base URL: `https://esign-stg.radenfatah.ac.id/api/v1/integration`
- **Cross-University Signer Scope**: Documents uploaded by a Client App are stored with `source_method = "api"` and `work_unit_id = null`. Unlike manual portal uploads, Client Apps are **not** restricted to a single work unit—you can assign any active signer or parafer across the university using `nip`, `bsre_email`, or `signer_id`.
- **eSign Owns QR Generation & BSrE Signing**:
  - Client apps only specify QR **placement** (`coordinate`, `anchor`, or `INVISIBLE`). eSign automatically generates the verification QR code (linking to the public `/verify` page with a unique 10-char Base58 `verification_code`) and stamps it during BSrE signing.
  - Client apps **never** send `imageBase64` or signer BSrE passphrases. Signers review and sign documents inside the eSign Portal (`paraf` / `signing` queues) using SSO + MFA + their personal BSrE passphrase.

---

## 2. Step-by-Step Client Integration Workflow

### Step 1: Register Client App & Configure Environment Variables
An eSign Superadmin (or assigned `admin_client_app`) registers your application in **Portal eSign → Aplikasi Terintegrasi** (`/client-apps`) and provides:
1. `app_id` (3–64 lowercase chars matching `^[a-z][a-z0-9_-]{2,63}$`, e.g., `app_simak`)
2. `app_secret` (prefix `tte_live_...`, shown once at creation/rotation)
3. Allowed document type codes (`allowed_document_types`, e.g., `ijazah`, `transkrip`, `surat_tugas`)
4. Optional `allowed_ips` (IPv4/IPv6/CIDR whitelist enforced on every API call)
5. Optional `webhook_url` and `webhook_secret` (prefix `whsec_...`)

Store these in your client backend `.env` (never hardcode secrets):
```env
# Use staging or production base URL
ESIGN_BASE_URL=https://esign-stg.radenfatah.ac.id/api/v1/integration
# ESIGN_BASE_URL=https://esign.radenfatah.ac.id/api/v1/integration

ESIGN_APP_ID=app_simak
ESIGN_APP_SECRET=tte_live_xxxxxxxxxxxxxxxxxxxxxxxx
ESIGN_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxxxxx
```

### Step 2: Authenticate Every Request
Every `/api/v1/integration/*` endpoint accepts **either** authentication scheme (both are equivalent):
- **Option A — HTTP Basic Auth** (compatible with existing BSrE Client v2 libraries):
  ```http
  Authorization: Basic <base64(ESIGN_APP_ID:ESIGN_APP_SECRET)>
  ```
- **Option B — Explicit Credential Headers**:
  ```http
  X-Client-App-Id: <ESIGN_APP_ID>
  X-Client-App-Secret: <ESIGN_APP_SECRET>
  ```

### Step 3: Discover Allowed Document Types & Active Signers
Before submitting documents (or to populate dropdowns / validate mappings in your client app):
1. `GET /api/v1/integration/document-types` — Returns only active document types permitted for your `app_id`. Passing an unlisted `document_type_code` or `document_type_id` returns `403 Forbidden`.
2. `GET /api/v1/integration/signers` — Returns all active university signers (`id`, `name`, `nip`, `bsre_email`, `position_title`, `bsre_status`).
   - Only assign signers whose **`bsre_status === "ISSUE"`** for `sign` steps, otherwise BSrE signing will fail when the signer attempts to sign.

### Step 4: Upload & Submit PDF Document (`POST /documents`)
Send `POST <ESIGN_BASE_URL>/documents` with `Content-Type: application/json`:
- **PDF Limit**: **Exactly 1 PDF file** per document submission (`max 25 MB` decoded PDF, `max 80 MB` total JSON body).
- **Two Supported PDF Formats** (pick one):
  - **BSrE v2 flat style**: `file: ["<base64_pdf>"]` + `file_names: ["document.pdf"]`
  - **Structured array style**: `files: [{ "slot_key": "naskah_utama", "file_name": "document.pdf", "file_base64": "<base64_pdf>" }]`
- **Idempotency via `external_ref_id`**: Pass your app's unique reference ID (max 150 chars, e.g., `SIMAK-IJZ-2026-00142`). Duplicate `external_ref_id` for the same Client App returns `409 Conflict` so retries never create duplicate documents.
- **Metadata Echo via `custom_fields`**: Pass any JSON object up to **64 KB** (e.g., `{"id_dokumen_client": 98142, "id_wisuda": 92, "nim": "22041450012"}`). eSign stores it intact and echoes it verbatim on `GET /documents/:id` and webhook callbacks.
- **Submission Mode**:
  - `submit_now: true` (default): Immediately queues stage 1 (`in_paraf` or `in_signing`) and notifies stage-1 signers. Requires at least one `stage_type: "sign"` step.
  - `submit_now: false`: Saves as `draft`; later call `POST /api/v1/integration/documents/:id/submit` to start signing.
- **BSrE Footer (`include_bsre_footer`)**: Optional boolean (default `false`). When `true`, stamps a BSrE verification footer bar + token at the bottom of every page right before the first `sign` execution.
- **Workflows (`paraf` & `sign`)**:
  - Identify each signer using **one** of `signer_id`, `nip`, or `bsre_email`.
  - A signer can appear **at most once** in `paraf` and **at most once** in `sign`.
  - All `paraf` steps finish before `sign` steps begin. Within a stage, equal `step_order` executes in **parallel**; increasing `step_order` executes **sequentially**.
  - **`paraf` steps** have **no visual QR** (do not pass placements).
  - **`sign` steps** MUST specify `signatureProperties` (or aliases `placements` / `signature_properties`):
    - **Coordinate**: `{ "tampilan": "VISIBLE", "page": 1, "originX": 120, "originY": 640, "width": 88, "height": 88 }`
    - **Text Anchor**: `{ "tampilan": "VISIBLE", "tag": "#tte_rektor#", "width": 88, "height": 88 }`
    - **Invisible**: `[{ "tampilan": "INVISIBLE" }]`
    - One signer can have **1 to 20** visible placements on the PDF (processed sequentially on BSrE), and visible placements cannot be mixed with `INVISIBLE`.

Save the returned `data.id` (document UUID), `data.status`, and `data.files[0].verification_code` in your client database.

### Step 5: Track Completion (Webhook + Polling Fallback)
1. **Webhook Callback**: When all stages finish (`document.completed`) or a signer rejects (`document.rejected`), eSign sends `POST` JSON to `callback_url` (or the app's default `webhook_url`) with:
   - `X-eSign-Event`: `document.completed` | `document.rejected`
   - `X-eSign-Timestamp`: Unix epoch seconds
   - `X-eSign-Signature`: `sha256=<hex(HMAC_SHA256(webhook_secret, "<timestamp>.<raw_body>"))>`
   - Always verify the signature using constant-time comparison (`timingSafeEqual` / `hash_equals` / `hmac.Equal` / `hmac.compare_digest`) over the **raw request body bytes** and reject stale timestamps (e.g., > 5 minutes).
   - Return HTTP `2xx` quickly, then asynchronously fetch the finalized PDF.
2. **Polling Fallback (Always Implement)**: Periodically (or via manual "Sync Status" button) call `GET /api/v1/integration/documents/:id` in case a webhook failed, timed out, or was not configured.

### Step 6: Download Final Signed PDF & Verify Integrity
1. Call `GET /api/v1/integration/documents/:id?include_files_base64=true` **only after `status === "completed"`** (calling it earlier returns `409 Conflict`).
2. Base64-decode `data.files[0].file_base64` and verify its SHA-256 hash matches `data.files[0].current_sha256`.
3. Store the signed PDF in your client app and mark your local record as signed (`verification_code`, `completed_at`).

---

## 3. Reference Documentation & Code Templates

Read these files on demand for exact schemas, error codes, and language templates:
- **REST API Endpoints, Schemas, Limits & Error Codes**: [references/api-endpoints.md](./references/api-endpoints.md)
- **Payload Variants, QR Placement Rules & Webhook HMAC Spec**: [references/payload-and-webhook-spec.md](./references/payload-and-webhook-spec.md)
- **Ready-to-Use Client & Webhook Snippets (Node.js/TS, Laravel/PHP, Go, Python)**: [examples/client-snippets.md](./examples/client-snippets.md)

---

## 4. Integration Verification Checklist

Before finishing an integration task, verify all of the following:
- [ ] All requests target `/api/v1/integration/*` with `Content-Type: application/json` (never `multipart/form-data`) and send credentials from backend server only (`Basic` or `X-Client-App-Id` + `X-Client-App-Secret`).
- [ ] Environment configuration supports switching cleanly between Staging (`esign-stg.radenfatah.ac.id`) and Production (`esign.radenfatah.ac.id`).
- [ ] `document_type_code` (or `document_type_id`) is verified against `GET /document-types` and signers have `bsre_status === "ISSUE"` from `GET /signers`.
- [ ] `external_ref_id` is sent for idempotency (`409 Conflict` handled gracefully) and `custom_fields` is a valid JSON object <= 64 KB.
- [ ] PDF payload contains exactly 1 valid PDF <= 25 MB (base64 without breaking JSON body 80 MB limit).
- [ ] `sign` workflow placements follow coordinate/anchor/invisible rules (<= 20 visible placements per signer, no mixing `VISIBLE` + `INVISIBLE`).
- [ ] Webhook handler verifies `X-eSign-Signature` over `"<X-eSign-Timestamp>.<raw_body>"` using timing-safe comparison and validates timestamp freshness.
- [ ] Polling fallback (`GET /documents/:id`) and final download (`?include_files_base64=true` + SHA-256 verification against `current_sha256`) are implemented.
- [ ] Test calls are confirmed in **Portal eSign → Aplikasi Terintegrasi → Log API** (`/client-apps/:id/logs`).
