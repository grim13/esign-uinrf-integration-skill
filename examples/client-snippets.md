# Ready-to-Use Client App Integration Snippets

Production Base URL: `https://esign.radenfatah.ac.id/api/v1/integration`  
Staging Base URL: `https://esign-stg.radenfatah.ac.id/api/v1/integration`

---

## 1. Node.js / TypeScript (Native `fetch` + Express/Next.js Webhook)

```typescript
// lib/esign.ts
import crypto from "node:crypto";
import fs from "node:fs/promises";

const ESIGN_BASE_URL =
  process.env.ESIGN_BASE_URL || "https://esign-stg.radenfatah.ac.id/api/v1/integration";
const ESIGN_APP_ID = process.env.ESIGN_APP_ID!;
const ESIGN_APP_SECRET = process.env.ESIGN_APP_SECRET!;
const ESIGN_WEBHOOK_SECRET = process.env.ESIGN_WEBHOOK_SECRET || "";

function authHeaders(): Record<string, string> {
  const basic = Buffer.from(`${ESIGN_APP_ID}:${ESIGN_APP_SECRET}`).toString("base64");
  return {
    Authorization: `Basic ${basic}`,
    "Content-Type": "application/json",
  };
}

export async function listSigners() {
  const res = await fetch(`${ESIGN_BASE_URL}/signers`, { headers: authHeaders() });
  if (!res.ok) throw new Error(`eSign signers error ${res.status}: ${await res.text()}`);
  return res.json();
}

export async function submitPdfForSigning(params: {
  externalRefId: string;
  documentTypeCode: string;
  documentNumber?: string;
  title: string;
  pdfPath: string;
  fileName: string;
  signerNip: string;
  anchorTag?: string;
  customFields?: Record<string, unknown>;
  callbackUrl?: string;
}) {
  const pdfBytes = await fs.readFile(params.pdfPath);
  const payload = {
    external_ref_id: params.externalRefId,
    document_type_code: params.documentTypeCode,
    document_number: params.documentNumber,
    title: params.title,
    callback_url: params.callbackUrl,
    submit_now: true,
    include_bsre_footer: false,
    custom_fields: params.customFields ?? {},
    file: [pdfBytes.toString("base64")],
    file_names: [params.fileName],
    workflows: [
      {
        stage_type: "sign",
        step_order: 1,
        nip: params.signerNip,
        location: "Palembang",
        signatureProperties: params.anchorTag
          ? [{ tampilan: "VISIBLE", tag: params.anchorTag, width: 88, height: 88 }]
          : [{ tampilan: "VISIBLE", page: 1, originX: 120, originY: 640, width: 88, height: 88 }],
      },
    ],
  };

  const res = await fetch(`${ESIGN_BASE_URL}/documents`, {
    method: "POST",
    headers: authHeaders(),
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`eSign submit error ${res.status}: ${await res.text()}`);
  return res.json();
}

export async function downloadCompletedPdf(documentId: string): Promise<{
  pdfBuffer: Buffer;
  verificationCode: string;
  currentSha256: string;
}> {
  const res = await fetch(
    `${ESIGN_BASE_URL}/documents/${encodeURIComponent(documentId)}?include_files_base64=true`,
    { headers: authHeaders() }
  );
  if (!res.ok) throw new Error(`eSign download error ${res.status}: ${await res.text()}`);

  const { data } = await res.json();
  const file = data.files[0];
  const pdfBuffer = Buffer.from(file.file_base64, "base64");
  const actualSha = crypto.createHash("sha256").update(pdfBuffer).digest("hex");
  if (actualSha !== file.current_sha256) {
    throw new Error(`SHA-256 mismatch: expected ${file.current_sha256}, got ${actualSha}`);
  }
  return {
    pdfBuffer,
    verificationCode: file.verification_code,
    currentSha256: file.current_sha256,
  };
}

export function verifyESignWebhook(
  rawBody: Buffer,
  timestampHeader: string,
  signatureHeader: string
): boolean {
  if (!ESIGN_WEBHOOK_SECRET || !timestampHeader || !signatureHeader) return false;
  const ts = Number(timestampHeader);
  if (!Number.isFinite(ts) || Math.abs(Math.floor(Date.now() / 1000) - ts) > 300) {
    return false;
  }
  const signedPayload = `${timestampHeader}.${rawBody.toString("utf8")}`;
  const expected =
    "sha256=" +
    crypto.createHmac("sha256", ESIGN_WEBHOOK_SECRET).update(signedPayload).digest("hex");

  const a = Buffer.from(expected, "utf8");
  const b = Buffer.from(signatureHeader, "utf8");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

---

## 2. PHP / Laravel (`Http` Facade + Webhook Controller)

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use RuntimeException;

class ESignClient
{
    private string $baseUrl;
    private string $appId;
    private string $appSecret;

    public function __construct()
    {
        $this->baseUrl = rtrim(config('services.esign.base_url'), '/'); // e.g. https://esign-stg.radenfatah.ac.id/api/v1/integration
        $this->appId = config('services.esign.app_id');
        $this->appSecret = config('services.esign.app_secret');
    }

    public function submitDocument(array $payload): array
    {
        $response = Http::withBasicAuth($this->appId, $this->appSecret)
            ->acceptJson()
            ->timeout(60)
            ->post("{$this->baseUrl}/documents", $payload);

        if ($response->failed()) {
            throw new RuntimeException("eSign upload failed ({$response->status()}): " . $response->body());
        }

        return $response->json('data');
    }

    public function getDocument(string $documentUuid, bool $includeFilesBase64 = false): array
    {
        $query = $includeFilesBase64 ? ['include_files_base64' => 'true'] : [];
        $response = Http::withBasicAuth($this->appId, $this->appSecret)
            ->acceptJson()
            ->timeout(60)
            ->get("{$this->baseUrl}/documents/{$documentUuid}", $query);

        if ($response->failed()) {
            throw new RuntimeException("eSign fetch failed ({$response->status()}): " . $response->body());
        }

        return $response->json('data');
    }

    public function downloadVerifiedSignedPdf(string $documentUuid): string
    {
        $doc = $this->getDocument($documentUuid, true);
        $file = $doc['files'][0] ?? null;
        if (!$file || empty($file['file_base64'])) {
            throw new RuntimeException('Signed PDF payload missing from eSign response.');
        }

        $bytes = base64_decode($file['file_base64'], true);
        if ($bytes === false || hash('sha256', $bytes) !== $file['current_sha256']) {
            throw new RuntimeException('Signed PDF SHA-256 verification failed.');
        }

        return $bytes;
    }
}
```

Webhook verification in Laravel Controller:
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;

class ESignWebhookController extends Controller
{
    public function handle(Request $request)
    {
        $secret = (string) config('services.esign.webhook_secret');
        $timestamp = (string) $request->header('X-eSign-Timestamp', '');
        $signature = (string) $request->header('X-eSign-Signature', '');
        $rawBody = $request->getContent();

        if ($secret === '' || $timestamp === '' || !ctype_digit($timestamp) || abs(time() - (int) $timestamp) > 300) {
            return response()->json(['error' => 'invalid timestamp'], Response::HTTP_UNAUTHORIZED);
        }

        $expected = 'sha256=' . hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret);
        if (!hash_equals($expected, $signature)) {
            return response()->json(['error' => 'invalid signature'], Response::HTTP_UNAUTHORIZED);
        }

        $payload = json_decode($rawBody, true);
        // Dispatch queued job to handle $payload['event'] ('document.completed' or 'document.rejected')
        // using $payload['document_id'], $payload['external_ref_id'], and $payload['custom_fields'].

        return response()->json(['received' => true]);
    }
}
```

---

## 3. Go (`net/http` + `crypto/hmac` Webhook Handler)

```go
package esign

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"net/http"
	"strconv"
	"time"
)

type Client struct {
	BaseURL       string
	AppID         string
	AppSecret     string
	WebhookSecret string
	HTTP          *http.Client
}

func (c *Client) DoJSON(ctx context.Context, method, path string, body any, out any) error {
	var reader io.Reader
	if body != nil {
		raw, err := json.Marshal(body)
		if err != nil {
			return err
		}
		reader = bytes.NewReader(raw)
	}
	req, err := http.NewRequestWithContext(ctx, method, c.BaseURL+path, reader)
	if err != nil {
		return err
	}
	req.SetBasicAuth(c.AppID, c.AppSecret)
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.HTTP.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	respBytes, _ := io.ReadAll(resp.Body)
	if resp.StatusCode >= 300 {
		return fmt.Errorf("esign %s %s (%d): %s", method, path, resp.StatusCode, string(respBytes))
	}
	if out != nil {
		return json.Unmarshal(respBytes, out)
	}
	return nil
}

func (c *Client) VerifyWebhookSignature(rawBody []byte, timestampHdr, signatureHdr string) bool {
	ts, err := strconv.ParseInt(timestampHdr, 10, 64)
	if err != nil || math.Abs(float64(time.Now().Unix()-ts)) > 300 {
		return false
	}
	mac := hmac.New(sha256.New, []byte(c.WebhookSecret))
	mac.Write([]byte(timestampHdr))
	mac.Write([]byte("."))
	mac.Write(rawBody)
	expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))
	return hmac.Equal([]byte(expected), []byte(signatureHdr))
}

func VerifyAndDecodePDF(b64, expectedSHA256 string) ([]byte, error) {
	pdfBytes, err := base64.StdEncoding.DecodeString(b64)
	if err != nil {
		return nil, err
	}
	sum := sha256.Sum256(pdfBytes)
	if hex.EncodeToString(sum[:]) != expectedSHA256 {
		return nil, fmt.Errorf("sha256 mismatch")
	}
	return pdfBytes, nil
}
```

---

## 4. Python (`requests` + Webhook Verification)

```python
import base64
import hashlib
import hmac
import os
import time
import requests

ESIGN_BASE_URL = os.getenv(
    "ESIGN_BASE_URL", "https://esign-stg.radenfatah.ac.id/api/v1/integration"
).rstrip("/")
ESIGN_APP_ID = os.environ["ESIGN_APP_ID"]
ESIGN_APP_SECRET = os.environ["ESIGN_APP_SECRET"]
ESIGN_WEBHOOK_SECRET = os.getenv("ESIGN_WEBHOOK_SECRET", "")


def submit_pdf(
    external_ref_id: str,
    doc_type_code: str,
    title: str,
    pdf_path: str,
    signer_nip: str,
    custom_fields: dict | None = None,
) -> dict:
    with open(pdf_path, "rb") as f:
        pdf_b64 = base64.b64encode(f.read()).decode("ascii")

    payload = {
        "external_ref_id": external_ref_id,
        "document_type_code": doc_type_code,
        "title": title,
        "submit_now": True,
        "custom_fields": custom_fields or {},
        "files": [
            {
                "slot_key": "naskah_utama",
                "file_name": os.path.basename(pdf_path),
                "file_base64": pdf_b64,
            }
        ],
        "workflows": [
            {
                "stage_type": "sign",
                "step_order": 1,
                "nip": signer_nip,
                "location": "Palembang",
                "placements": [
                    {
                        "slot_key": "naskah_utama",
                        "visual_mode": "anchor",
                        "anchor_tag": "#tte_pejabat#",
                        "width": 88,
                        "height": 88,
                    }
                ],
            }
        ],
    }

    res = requests.post(
        f"{ESIGN_BASE_URL}/documents",
        auth=(ESIGN_APP_ID, ESIGN_APP_SECRET),
        json=payload,
        timeout=60,
    )
    res.raise_for_status()
    return res.json()["data"]


def download_completed_pdf(document_id: str) -> bytes:
    res = requests.get(
        f"{ESIGN_BASE_URL}/documents/{document_id}",
        params={"include_files_base64": "true"},
        auth=(ESIGN_APP_ID, ESIGN_APP_SECRET),
        timeout=60,
    )
    res.raise_for_status()
    file_info = res.json()["data"]["files"][0]
    pdf_bytes = base64.b64decode(file_info["file_base64"])
    if hashlib.sha256(pdf_bytes).hexdigest() != file_info["current_sha256"]:
        raise ValueError("Signed PDF SHA-256 verification failed")
    return pdf_bytes


def verify_webhook_signature(raw_body: bytes, timestamp_hdr: str, signature_hdr: str) -> bool:
    if not ESIGN_WEBHOOK_SECRET or not timestamp_hdr.isdigit():
        return False
    if abs(int(time.time()) - int(timestamp_hdr)) > 300:
        return False
    signed = f"{timestamp_hdr}.".encode("utf-8") + raw_body
    expected = "sha256=" + hmac.new(
        ESIGN_WEBHOOK_SECRET.encode("utf-8"), signed, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_hdr)
```
