# eSign UIN Raden Fatah — Client App Integration Skill

Agent skill untuk membantu AI coding agent (Antigravity, Claude Code, Codex, Cursor, dll.) mengintegrasikan aplikasi klien (SIMAK, E-Office, SIPEG, maupun aplikasi eksternal/unit) dengan **Portal Tanda Tangan Elektronik Terpusat (eSign) UIN Raden Fatah Palembang**.

## Lingkungan & Base URL

| Environment | Portal UI | Integration API Base URL |
| :--- | :--- | :--- |
| **Production** | `https://esign.radenfatah.ac.id` | `https://esign.radenfatah.ac.id/api/v1/integration` |
| **Staging** | `https://esign-stg.radenfatah.ac.id` | `https://esign-stg.radenfatah.ac.id/api/v1/integration` |

## Cakupan Skill

- Autentikasi Client App (`Basic Auth` kompatibel BSrE v2 & header `X-Client-App-Id` / `X-Client-App-Secret`)
- Penemuan katalog jenis dokumen yang diizinkan (`GET /document-types`) & daftar pejabat aktif (`GET /signers`, cek `bsre_status === "ISSUE"`)
- Pengunggahan dokumen PDF dalam JSON Base64 (`POST /documents`, tanpa `multipart/form-data`), dukungan format `file[]` ala BSrE v2 dan `files[]` terstruktur
- Idempotensi `external_ref_id`, penyimpanan & pengembalian utuh `custom_fields` (JSON hingga 64 KB), dan opsi `include_bsre_footer`
- Konfigurasi workflow `paraf` dan `sign` (sekuensial & paralel), serta penempatan QR tanda tangan (`coordinate`, `anchor` teks, atau `INVISIBLE`, hingga 20 visualisasi per penandatangan)
- Penanganan callback webhook (`document.completed` & `document.rejected`) dengan verifikasi HMAC-SHA256 (`X-eSign-Timestamp` & `X-eSign-Signature`), polling fallback, dan pengunduhan PDF final (`?include_files_base64=true`) beserta verifikasi SHA-256

## Cara Memasang Skill

### Opsi 1 — Menggunakan `skills` CLI
```bash
npx skills add https://github.com/grim13/esign-uinrf-integration-skill
```

### Opsi 2 — Clone Manual ke Proyek atau Global Config
- **Per-proyek (Antigravity / lintas-agent)**:
  ```bash
  git clone https://github.com/grim13/esign-uinrf-integration-skill.git .agents/skills/esign-uinrf-integration
  ```
- **Per-proyek (Claude Code)**:
  ```bash
  git clone https://github.com/grim13/esign-uinrf-integration-skill.git .claude/skills/esign-uinrf-integration
  ```
- **Global Antigravity**:
  ```bash
  git clone https://github.com/grim13/esign-uinrf-integration-skill.git ~/.gemini/config/skills/esign-uinrf-integration
  ```

## Struktur Repositori

```text
.
├── README.md
├── SKILL.md                                  # Panduan utama & checklist verifikasi
├── references/
│   ├── api-endpoints.md                      # Spesifikasi 5 endpoint REST, limit, & kode error HTTP
│   └── payload-and-webhook-spec.md           # Varian payload, aturan QR/workflow, & kontrak HMAC webhook
└── examples/
    └── client-snippets.md                    # Template siap pakai (Node.js/TS, Laravel/PHP, Go, Python)
```
