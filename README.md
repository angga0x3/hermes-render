# Hermes Agent di Render (Blueprint)

Deploy **Hermes Agent** (Nous Research — AI agent self-hosted dengan learning
loop, skills, dan memory persisten) sebagai satu web service Docker di Render
menggunakan Render Blueprint (`render.yaml`).

Blueprint ini berbasis template resmi
[`render-examples/hermes-render`](https://github.com/render-examples/hermes-render)
(MIT, Render Services) dan diuji terhadap image upstream yang di-pin.

## Isi folder

| File | Fungsi |
|---|---|
| `render.yaml` | Blueprint Render: web service Docker + disk 5 GB + health check + env vars |
| `Dockerfile` | extends image upstream `nousresearch/hermes-agent` yang di-pin, plus: skills Render, wrapper MCP Render, fix bug Chat-tab upstream |
| `scripts/bootstrap.sh` | Entrypoint: chown disk → patch config → lanjut ke entrypoint upstream |
| `scripts/patch-config.py` | Patch idempotent `config.yaml`: daftarkan MCP Render + direktori skills (INSERT-only, tidak menimpa editan Anda) |
| `skills/render-on-hermes/SKILL.md` | Skill overlay: memberi tahu agent bahwa MCP Render sudah ter-wire |
| `tests/test_patch_config.py` | Test suite patcher (`python -m unittest discover -s tests`) |
| `.env.example` | Referensi env var (hanya dokumentasi — blueprint tidak membacanya) |
| `README-upstream.md` | README lengkap asli dari template resmi (detail arsitektur, troubleshooting, security) |
| `LICENSE` | MIT (Render Services, Inc.) |

## Langkah deploy

1. Push isi folder ini ke **root** sebuah repo GitHub (`render.yaml` **harus** di root).
2. Dashboard Render → **New + → Blueprint** → pilih repo → Apply.
   (Alternatif: klik tombol *Deploy to Render* di `README-upstream.md` —
   deploy langsung dari template resmi tanpa fork.)
3. Saat apply, isi env var yang diminta:
   - `RENDER_MCP_API_KEY` — **opsional**. API key Render
     (dashboard.render.com → Settings → API Keys) agar agent bisa mengelola
     akun Render Anda lewat MCP (`mcp_render_*`). Bisa dibiarkan kosong dan
     diisi belakangan di tab Environment.
   - `HERMES_GATEWAY_TOKEN` — di-generate otomatis oleh blueprint. Catat nilainya;
     dipakai sebagai `API_SERVER_KEY` jika suatu saat mengaktifkan API server
     OpenAI-compatible (`API_SERVER_ENABLED=true`).
4. Build ±3–5 menit, boot ±1 menit. Health check: `GET /api/status` (port 10000).
5. Auto-deploy **mati** (`autoDeployTrigger: off`) — deploy manual setelah update.

## Setup setelah deploy (wajib)

Buka `https://<nama-service>.onrender.com` → dashboard Hermes:

1. **API Keys** → paste minimal satu LLM key:
   `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`,
   `HF_TOKEN`, `GLM_API_KEY`, dll. (tersimpan di `/opt/data/.env` di disk —
   aman antar-deploy).
2. **Config** → pilih model.
3. **Status** → pastikan gateway berjalan → **Restart gateway** bila perlu.
4. (Opsional) **API Keys** lagi → token platform chat: Telegram, Discord,
   Slack, dll.
5. Chat dengan agent lewat tab **Chat** (TUI penuh via PTY + xterm.js).

## ⚠️ Keamanan — baca sebelum paste API key

- **Dashboard Hermes TIDAK punya autentikasi bawaan.** Siapa pun yang tahu URL
  bisa membaca key Anda, mengubah config, dan menyuruh agent apa pun. Lindungi
  URL dulu (auth gateway / OAuth proxy / private network seperti Tailscale)
  sebelum isi production key. Jangan pakai domain default `.onrender.com`
  mentah-mentah untuk production.
- Jika `RENDER_MCP_API_KEY` diisi, agent dapat akses **penuh ke akun Render**
  (semua tool MCP yang diizinkan role key Anda) — termasuk delete service.
  Role key menentukan batasnya.
- Render CLI sengaja **tidak** di-install di image ini; agent memakai jalur MCP.

## Biaya

- Plan **Standard** (1 CPU / 2 GB): selalu aktif, ±$25/bln. Free (512 MB)
  tidak cukup untuk Hermes.
- Disk **5 GB** di `/opt/data`: ±$1.25/bln (skills, memory, sesi, config
  bertahan antar-deploy).
- Plan Pro (2 CPU / 4 GB) disarankan jika sering memakai subagent/Playwright.

## Update versi

- **Upgrade Hermes**: ubah `ARG HERMES_IMAGE` di `Dockerfile`, commit, lalu
  deploy manual. Pin saat ini: `v2026.5.7` — versi yang **sudah diuji** terhadap
  workaround di Dockerfile (fix Chat-tab, ownership `ui-tui`).
- **PENTING**: sejak v0.23.x upstream ganti skema boot dari tini ke
  **s6-overlay** (PID 1 `/init`, `stage2-hook`, TUI prebuilt, `/opt/hermes`
  immutable). Tag stabil terbaru saat blueprint ini dibuat: `v0.23.3`
  (2026-11-20). Bump ke v0.23.x **tidak bisa asal ganti tag** — wrapper
  `bootstrap.sh` + workaround Dockerfile perlu diadaptasi dan diuji dulu.
  Sementara itu tetap di pin yang teruji.
- **Upgrade skills Render**: ubah `RENDER_SKILLS_REF` (commit SHA) di
  `Dockerfile`.
- **Rotasi `RENDER_MCP_API_KEY`**: edit env var di tab Environment → restart.
  Header MCP lazy-resolve dari env var — tidak perlu rebuild.
- **Rotasi key provider** (LLM/chat): lewat tab API Keys dashboard, atau tab
  Environment Render (env Render override `/opt/data/.env`).

## Troubleshooting singkat

- **Log**: tab Events/Logs di dashboard Render, atau CLI
  `render logs <service-id>`.
- **Shell**: `render ssh <service-id>` → inspeksi disk
  (`ls /opt/data`), jalankan CLI Hermes langsung.
- **Clean rebuild**: tab Manual Deploy → *Clear build cache & deploy*.
- `command not found: render` dari agent → by design (CLI tidak di-install;
  pakai tool `mcp_render_*`).
- `tirith security scanner enabled but not available` → harmless, abaikan.
