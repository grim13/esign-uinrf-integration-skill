# Payload Variants, Workflows, QR Placement & Webhook Specification

## 1. PDF Upload Formats

Each submission accepts **exactly 1 PDF file** (`<= 25 MB` decoded). Strip or keep `data:application/pdf;base64,` data-URI prefixes and whitespace/newlines—eSign supports standard, raw, URL-safe, and raw URL-safe Base64 encodings.

### Option A: BSrE v2 Flat Format (`file` + `file_names`)
```json
{
  "file": ["JVBERi0xLjQK..."],
  "file_names": ["ijazah-22041450012.pdf"]
}
```
- Internally assigned `slot_key = "file_1"`.

### Option B: Structured Array Format (`files`)
```json
{
  "files": [
    {
      "slot_key": "naskah_utama",
      "file_name": "surat-tugas.pdf",
      "file_base64": "JVBERi0xLjQK..."
    }
  ]
}
```
- `base64` is accepted as an alias for `file_base64`.
- `slot_key` defaults to `"file_1"` if omitted; if provided, match it in placement `slot_key` (or omit placement `slot_key` for single-file payloads, as eSign auto-binds it to the single uploaded file).

---

## 2. `custom_fields` & `include_bsre_footer`

- **`custom_fields`** (alias `metadata`): Must be a JSON **object** (e.g. `{"id_dokumen": 123, "nim": "22041450012"}`), max `64 KB`. Arrays, strings, or numbers at root level return `400 Bad Request`. Stored in PostgreSQL JSONB and returned unchanged in `POST /documents`, `GET /documents/:id`, and webhook callbacks.
- **`include_bsre_footer`**: Boolean, default `false`. When `true`, eSign stamps a BSrE footer (separator line, BSrE logo, electronic signature notice, and verification code) at the bottom of every PDF page once, immediately before the first `sign` execution.

---

## 3. Workflow Ordering (`paraf` and `sign`)

### Signer Resolution Priority
For each entry in `workflows[]`, eSign resolves the active signer in this order:
1. `signer_id` (integer > 0)
2. `nip` (string, case-insensitive trimmed)
3. `bsre_email` (string, case-insensitive trimmed)

### Stage & Step Rules
- `stage_type`: `"paraf"` or `"sign"` (defaults to `"sign"` if omitted).
- **No Duplicate Signer per Stage**: The same signer cannot appear twice in `paraf` or twice in `sign`. (A signer *can* appear once in `paraf` and once in `sign`, though typically they are distinct.)
- **Stage Sequence**: All `paraf` steps always execute before `sign` steps.
- **Parallel vs Sequential (`step_order`)**:
  - Entries with the **same** `step_order` within the same `stage_type` are active simultaneously (**parallel**).
  - Increasing `step_order` values execute **sequentially** (eSign automatically compacts gaps like `1, 3` into `1, 2`).
  - If `step_order` is omitted (`<= 0`), each entry gets an incrementing sequential order (`1, 2, ...`).
- **`paraf` Stage**:
  - Has **no visual QR** (any placements on `paraf` are ignored).
  - Example:
    ```json
    {
      "stage_type": "paraf",
      "step_order": 1,
      "bsre_email": "kabiro.aakk.uinrf@kemenag.go.id",
      "location": "Palembang"
    }
    ```

---

## 4. Signature QR Placement Rules (`sign` Stage)

Every `stage_type: "sign"` workflow entry **must** include at least one placement via `signatureProperties`, `signature_properties`, or `placements` (checked in reverse alias priority: `placements` -> `signature_properties` -> `signatureProperties`).

### Mode 1: Coordinate Placement (`coordinate`)
Explicit X/Y coordinates in **PDF points (pt)** on a specific page (`1`-indexed):
```json
{
  "tampilan": "VISIBLE",
  "page": 1,
  "originX": 120,
  "originY": 640,
  "width": 88,
  "height": 88
}
```
Supported aliases: `visual_mode: "coordinate"`, `page_number`, `origin_x`, `origin_y`.
- If `page`/`page_number` is omitted in coordinate mode, defaults to the **last page** of the PDF.
- `width` and `height` default to `88` pt when omitted (`<= 0`), and must be `>= 10` pt and fit within the page bounds (`x + width <= pageWidth + 4`, `y + height <= pageHeight + 4`, `x >= -1`, `y >= -1`).

### Mode 2: Text Anchor Placement (`anchor`)
Automatically places the QR code at a literal placeholder text tag inside the PDF (e.g., `#tte_rektor#`):
```json
{
  "tampilan": "VISIBLE",
  "tag": "#tte_rektor#",
  "width": 88,
  "height": 88
}
```
Supported aliases: `visual_mode: "anchor"`, `anchor_tag`.
- `tag` must be 1–100 characters.
- `width` and `height` default to `88` pt (valid range `10`–`1200` pt).
- Ensure the anchor tag text is actually embedded as selectable text in the PDF (not rasterized inside an image) and matches the string sent to BSrE.

### Mode 3: Invisible Signature (`invisible`)
Cryptographic BSrE signature without any visible QR stamp on the PDF:
```json
{
  "tampilan": "INVISIBLE"
}
```
Supported alias: `visual_mode: "invisible"`.

### Multi-Placement Rules
- A single `sign` workflow step can specify **1 to 20** `VISIBLE` placements (combining `coordinate` and `anchor` entries as needed) on the same PDF. eSign executes them sequentially against BSrE when the signer submits their passphrase.
- **Forbidden**: Mixing `INVISIBLE` with `VISIBLE` placements, or sending multiple `INVISIBLE` placements for the same signer on the same PDF (`400 Bad Request`: `invalid signature placement configuration`).

---

## 5. Webhook Callback & HMAC-SHA256 Specification

### Events
- `document.completed`: All `paraf` and `sign` stages finished successfully; PDF is finalized and ready to download.
- `document.rejected`: A parafer or signer rejected the document; `rejection_note` contains the reason.

### Request Headers
| Header | Description |
| :--- | :--- |
| `Content-Type` | `application/json` |
| `X-eSign-Event` | `document.completed` or `document.rejected` |
| `X-eSign-Timestamp` | Unix timestamp in seconds (string, e.g., `"1795106000"`) |
| `X-eSign-Signature` | `sha256=<lowercase_hex>` HMAC-SHA256 signature (sent when `webhook_secret` is configured on the Client App) |

### Payload Body Examples

**`document.completed`**:
```json
{
  "event": "document.completed",
  "timestamp": "2026-11-19T17:15:20Z",
  "app_id": "app_simak",
  "document_id": "0196e561-b6b6-7000-8000-000000000142",
  "external_ref_id": "APP_SIMAK-DOC-2026-00142",
  "document_number": "IJZ/2026/FST/00142",
  "title": "Ijazah Sarjana Sistem Informasi - Mahasiswa Uji",
  "status": "completed",
  "completed_at": "2026-11-19T17:15:19Z",
  "custom_fields": {
    "id_dokumen_client": 98142,
    "id_wisuda": 92
  },
  "files": [
    {
      "id": 1,
      "file_order": 1,
      "file_name": "ijazah-22041450012.pdf",
      "file_size_bytes": 219480,
      "page_count": 1,
      "original_sha256": "a94a8fe5...",
      "current_sha256": "f12904ac...",
      "verification_code": "8F3A12B9C4"
    }
  ]
}
```

**`document.rejected`**:
Includes `"status": "rejected"` and `"rejection_note": "<alasan penolakan pejabat>"` (with `completed_at` omitted).

### HMAC-SHA256 Verification Formula
Compute the signature over the **exact raw UTF-8 request body bytes** (do **not** re-serialize parsed JSON, as key ordering or whitespace differences will break verification):

```text
signed_payload = "<X-eSign-Timestamp>" + "." + <raw_request_body_bytes>
expected_signature = "sha256=" + lowercase_hex(HMAC_SHA256(key = ESIGN_WEBHOOK_SECRET, message = signed_payload))
```

Security best practices for your webhook receiver:
1. Validate `X-eSign-Timestamp` is within an acceptable window (e.g., `abs(now - ts) <= 300` seconds) to prevent replay attacks.
2. Compare `expected_signature` and `X-eSign-Signature` using constant-time comparison (`crypto.timingSafeEqual`, `hash_equals`, `hmac.Equal`, `hmac.compare_digest`).
3. Note that the webhook payload intentionally omits `file_base64` to keep callbacks lightweight. On `document.completed`, fetch the signed PDF via `GET /api/v1/integration/documents/{document_id}?include_files_base64=true` and verify `sha256(decoded_pdf) === files[0].current_sha256`.
4. Always keep a **polling fallback** (`GET /api/v1/integration/documents/{document_id}`) so documents still reconcile if a webhook delivery fails.
