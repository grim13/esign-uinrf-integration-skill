# eSign UIN Raden Fatah — REST API Endpoints Reference

## Base URLs

| Environment | Portal UI | Integration API Base URL |
| :--- | :--- | :--- |
| **Production** | `https://esign.radenfatah.ac.id` | `https://esign.radenfatah.ac.id/api/v1/integration` |
| **Staging** | `https://esign-stg.radenfatah.ac.id` | `https://esign-stg.radenfatah.ac.id/api/v1/integration` |

---

## Authentication & Security Rules

Every endpoint under `/api/v1/integration/*` requires Client App authentication. Two equivalent methods are supported:

1. **HTTP Basic Auth (Recommended / BSrE Client v2 Compatible)**
   ```http
   Authorization: Basic <base64(app_id:app_secret)>
   ```
2. **Explicit Headers**
   ```http
   X-Client-App-Id: <app_id>
   X-Client-App-Secret: <app_secret>
   ```

Additional checks enforced on every request:
- `app_id` is normalized to lowercase (`^[a-z][a-z0-9_-]{2,63}$`) and `is_active` must be `true`.
- If `allowed_ips` is non-empty on the Client App record, the caller's client IP must match one of the configured IPv4/IPv6 addresses or CIDR blocks.
- Every API call (including `401`/`403` failures against a known `app_id`) is recorded in `client_app_api_logs` and viewable in Portal eSign (`/client-apps/:id/logs`).

---

## Endpoints

### 1. List Allowed Document Types
Returns active document types permitted for the authenticated Client App.

```http
GET /api/v1/integration/document-types
```

**Response `200 OK`**:
```json
{
  "app_id": "app_simak",
  "data": [
    {
      "id": 1,
      "code": "ijazah",
      "name": "Ijazah",
      "classification_code": "AKD",
      "classification_name": "Dokumen Akademik"
    }
  ]
}
```

---

### 2. List Active Signers (Cross-University)
Returns all active registered signers across the university (`work_unit_id` is not restricted for Client Apps).

```http
GET /api/v1/integration/signers
```

**Response `200 OK`**:
```json
{
  "data": [
    {
      "id": 12,
      "name": "Prof. Dr. Contoh Pejabat, M.Ag.",
      "nip": "197001011995031001",
      "bsre_email": "rektor.uinrf@kemenag.go.id",
      "position_title": "Rektor",
      "bsre_status": "ISSUE"
    }
  ]
}
```

**Valid `bsre_status` values**:
- `ISSUE` *(Ready to sign — certificate active)*
- `EXPIRED`, `RENEW`, `WAITING_FOR_VERIFICATION`, `NEW`, `NO_CERTIFICATE`, `NOT_REGISTERED`, `SUSPEND`, `REVOKE`

---

### 3. Upload & Submit PDF Document
Creates a document from a Base64 PDF payload and optionally submits it immediately into the signing queue.

```http
POST /api/v1/integration/documents
Content-Type: application/json
```

#### Request Body Schema

| Field | Type | Required | Constraints / Notes |
| :--- | :--- | :--- | :--- |
| `document_type_code` | `string` | Yes* | Active document type `code` permitted for this Client App. *Or pass `document_type_id` (`integer`). |
| `document_type_id` | `integer` | Yes* | Alternative to `document_type_code`. |
| `title` | `string` | Yes | Document title/subject, max `255` characters. |
| `external_ref_id` | `string` | No | Unique reference ID in your client app, max `150` chars. Unique per `client_app_id` (`409 Conflict` if duplicated). |
| `document_number` | `string` | No | Official letter/document number, max `150` chars. |
| `custom_fields` | `object` | No | Arbitrary JSON object up to `64 KB` (`65536` bytes). Alias: `metadata`. Default `{}`. Returned verbatim on `GET` & webhook. |
| `callback_url` | `string` | No | Valid `http://` or `https://` URL overriding the Client App's default `webhook_url`. |
| `submit_now` | `boolean` | No | Default `true`. When `true`, transitions immediately to `in_paraf` or `in_signing` (requires >= 1 `sign` step). When `false`, saves as `draft`. |
| `include_bsre_footer` | `boolean` | No | Default `false`. When `true`, stamps BSrE footer + verification token at bottom of every PDF page right before the first `sign` execution. |
| `file` | `string[]` | Yes* | BSrE v2 style array containing exactly 1 Base64 PDF string (`data:application/pdf;base64,...` prefix or raw Base64). *Or use `files`. |
| `file_names` | `string[]` | No | Display filename corresponding to `file[]` (defaults to `dokumen-1.pdf`). |
| `files` | `object[]` | Yes* | Structured alternative to `file`: `[{ "slot_key": "naskah_utama", "file_name": "doc.pdf", "file_base64": "..." }]` (`base64` alias accepted). Max 1 file. |
| `workflows` | `object[]` | Conditional | Required if `submit_now` is `true` (must include at least 1 `sign` step). See [payload-and-webhook-spec.md](./payload-and-webhook-spec.md). |

#### Hard Limits
- **JSON Request Body**: max `80 MB` (`83,886,080` bytes)
- **Files per Submission**: `1` PDF (`MaxPDFFilesPerDocument = 1`)
- **Decoded PDF Size**: max `25 MB` (`26,214,400` bytes)
- **Custom Fields JSON**: max `64 KB` (`65,536` bytes), must be a JSON object (not array/primitive)
- **Visible QR Placements per Signer**: `1` to `20` (`MaxPlacementsPerSignerFile = 20`)
- **Default QR Size**: `88 x 88` pt (valid range `10`–`1200` pt for anchor; must fit within PDF page width/height for coordinate)

#### Response `201 Created`
```json
{
  "data": {
    "id": "0196e561-b6b6-7000-8000-000000000142",
    "app_id": "app_simak",
    "external_ref_id": "APP_SIMAK-DOC-2026-00142",
    "document_number": "IJZ/2026/FST/00142",
    "title": "Ijazah Sarjana Sistem Informasi - Mahasiswa Uji",
    "include_bsre_footer": false,
    "status": "in_signing",
    "current_stage": "sign",
    "current_step_order": 1,
    "document_type": {
      "id": 1,
      "code": "ijazah",
      "name": "Ijazah"
    },
    "callback_url": "https://simak.radenfatah.ac.id/api/tte/callback",
    "callback_status": "pending",
    "custom_fields": {
      "id_dokumen_client": 98142,
      "id_wisuda": 92
    },
    "files": [
      {
        "id": 1,
        "file_order": 1,
        "file_name": "ijazah-22041450012.pdf",
        "file_size_bytes": 184320,
        "page_count": 1,
        "page_width_pt": 595.28,
        "page_height_pt": 841.89,
        "original_sha256": "a94a8fe5...",
        "current_sha256": "a94a8fe5...",
        "verification_code": "8F3A12B9C4"
      }
    ],
    "workflows": [
      {
        "id": 10,
        "signer_id": 12,
        "signer_name": "Prof. Dr. Contoh Pejabat, M.Ag.",
        "nip": "197001011995031001",
        "bsre_email": "rektor.uinrf@kemenag.go.id",
        "stage_type": "sign",
        "step_order": 1,
        "status": "pending",
        "location": "Palembang",
        "placements": [
          {
            "id": 55,
            "workflow_id": 10,
            "document_file_id": 1,
            "visual_mode": "coordinate",
            "page_number": 1,
            "origin_x": 120,
            "origin_y": 640,
            "width": 88,
            "height": 88
          }
        ]
      }
    ],
    "created_at": "2026-11-19T16:30:00Z",
    "updated_at": "2026-11-19T16:30:00Z"
  }
}
```

---

### 4. Get Document Status & Download Final Signed PDF
Retrieves the current document state, workflow progress, `custom_fields`, and optionally the signed PDF in Base64.

```http
GET /api/v1/integration/documents/:id
GET /api/v1/integration/documents/:id?include_files_base64=true
```
- `:id` is the document UUID (`data.id` from `POST /documents`).
- `include_files_base64`: accepts `"true"` or `"1"`.
  - Only permitted when `status === "completed"`.
  - If called while `status` is `draft`, `in_paraf`, `in_signing`, `rejected`, or `failed`, returns **`409 Conflict`**.
  - When `true` and `completed`, each item in `data.files[]` includes `"file_base64": "<base64_signed_pdf>"`.

**Status Enums**:
- `DocumentStatus`: `draft` | `in_paraf` | `in_signing` | `completed` | `rejected` | `failed`
- `WorkflowStageType`: `paraf` | `sign`
- `WorkflowStatus`: `waiting` | `pending` | `signed` | `rejected`

---

### 5. Submit a Draft Document
Submits a document previously uploaded with `submit_now: false` into the active `paraf`/`sign` queue.

```http
POST /api/v1/integration/documents/:id/submit
```
- Returns `200 OK` with updated `ClientAppDocumentOutput` (`status: "in_paraf"` or `"in_signing"`).
- Returns `409 Conflict` if document is no longer `draft`.
- Returns `400 Bad Request` if document has no `sign` workflow step.

---

## HTTP Error Codes & Exact Error Responses

All errors return JSON: `{ "error": "<message>" }`.

| HTTP Status | Trigger & Exact Error Message(s) |
| :--- | :--- |
| **400 Bad Request** | Invalid JSON (`invalid JSON request body`), invalid UUID (`invalid document id`), missing/long title (`document title is required and must not exceed 255 characters`), missing doc type (`valid active document_type_id is required`), missing PDF (`at least one PDF file is required`), > 1 PDF (`a single submission may contain at most 1 PDF file`), invalid PDF (`uploaded file must be a valid PDF document`), bad Base64 (`invalid base64 PDF payload`), duplicate slot key (`invalid or duplicate file slot reference in document draft`), invalid `custom_fields` (`custom_fields must be a valid JSON object up to 64 KB`), invalid `callback_url` (`webhook_url must be a valid http or https URL`), missing `sign` step (`at least one main signer (sign) step is required to submit the document`), duplicate signer in same stage (`the same signer cannot appear more than once in the same stage`), unknown/inactive signer (`requested signer was not found or is inactive`), bad placement (`invalid signature placement configuration`). |
| **401 Unauthorized** | `missing client app credentials` or `invalid client app credentials`. |
| **403 Forbidden** | `client app is inactive`, `source IP address is not in client app allowed_ips`, or `document type is not permitted for this client app`. |
| **404 Not Found** | `document not found` (UUID does not exist, `source_method != "api"`, or belongs to a different Client App). |
| **409 Conflict** | `external_ref_id already exists for this client app`, `only draft documents can be modified, deleted, or submitted`, or `document can only be downloaded after all paraf and signing stages are completed`. |
| **413 Payload Too Large** | `JSON request body exceeds 80 MB limit` or `each PDF file size must not exceed 25 MB`. |
| **415 Unsupported Media Type** | `Content-Type must be application/json with base64-encoded PDF file(s)`. |
| **500 Internal Server Error** | `failed to authenticate client app`, `failed to load active signers`, or `failed to process integration request`. |
