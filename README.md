# 🎯 Smart Job Hunter

Job search automation built with **n8n + AI (Gemini) + Telegram**, running 24/7 on a **Raspberry Pi 5 (4 GB RAM)** with Ubuntu and Docker.

Every day the workflow pulls job listings from several sources, filters them, scores how well each one matches my profile, and for the best matches automatically generates a **cover letter** and a **tailored ATS-friendly CV**, then notifies me on Telegram. I review everything and decide whether to apply.

---

## ✨ What it does

1. **Daily trigger** (n8n's built-in scheduler).
2. **Job extraction** from public APIs:
   - RemoteOK
   - Arbeitnow
   - Adzuna (optional, requires a free `app_id` / `app_key`)
3. **Normalization** into a common schema: `title`, `url`, `description`, `budget`, `datePosted`, `source`.
4. **Merge and deduplication**: previously processed listings are read from Google Sheets and skipped.
5. **Keyword filter** to remove noise before spending any AI tokens.
6. **Loop + Wait** to process one listing at a time and respect Gemini rate limits.
7. **AI evaluation** (AI Agent + Gemini + Structured Output Parser) returning JSON with
   `score`, `justificacion` (justification) and `keywords`.
8. **Score filter**:
   - ❌ Low score → saved to the *Discarded* sheet.
   - ✅ High score → continues through the flow.
9. **Content generation** (two Gemini agents):
   - **Cover letter**: first person, max. 120 words, written in the language of the listing.
   - **Plain-text ATS-friendly CV**: no tables, columns or icons, reordered and enriched with the listing's keywords.
10. **CV converted to a `.txt` file** and sent through **Telegram** as a document.
11. **Telegram notification** with score, link, justification and cover letter.
12. **Logging** of the approved listing in Google Sheets.

---

## 🧩 Workflow architecture

```
Daily Trigger
 ├─ 1a. Fetch RemoteOK  ──► Normalize RemoteOK ──┐
 ├─ 1b. Fetch Arbeitnow ──► Normalize Arbeitnow ─┼─► Normalize Job Fields (append)
 ├─ (1c. Adzuna) ─────────► Normalize Adzuna ────┘            │
 └─ Read Already-Processed Jobs (Google Sheets) ──────────────┤
                                                              ▼
                                      Merge Data (append) ► Filter Duplicates (Code)
                                                              ▼
                                                 Keywords (Filter) ► Filter
                                                              ▼
                                                       Loop Over Items ◄──────────────┐
                                                              ▼                       │
                                  2. Evaluation - Match Score vs Profile (AI Agent)   │
                                     ├─ Gemini (Chat Model)                           │
                                     └─ Structured Output Parser (score JSON)         │
                                                              ▼                       │
                                                       Job Data (Set)                 │
                                                              ▼                       │
                                                  3. Filter - Score >= threshold?     │
                                         ┌────────── false ─┴─ true ──────────┐       │
                                         ▼                                    ▼       │
                             Save Discarded (Sheet)               Code in JavaScript  │
                                                                              ▼       │
                                                        4. Tailoring - Generate Cover Letter
                                                                              ▼       │
                                                        6. Tailoring - Generate ATS CV │
                                                                              ▼       │
                                                         Convert CV to TXT File        │
                                                          ├─► 7. Send CV (Telegram sendDocument)
                                                          └─► Save Approved Match (Sheet)
                                                                    ▼
                                                     5. Notification - Telegram Alert ──► Wait ─┘
```

---

## 🖥️ Hardware and environment

| Component | Details |
|---|---|
| Device | Raspberry Pi 5, 4 GB RAM |
| Storage | Kingston microSD card |
| OS | Ubuntu |
| Containers | Docker + docker-compose |
| Orchestrator | n8n (official `docker.n8n.io/n8nio/n8n` image) |
| AI | Google Gemini (via n8n's LangChain nodes) |
| Notifications | Telegram bot |
| Data persistence | Google Sheets |

---

## 🚀 Installation

### 1. Folder structure and permissions

```bash
mkdir -p ~/smart-job-hunter/{n8n-data,local-files}
chmod -R 775 smart-job-hunter/
sudo chown -R dylan:docker smart-job-hunter/
cd smart-job-hunter
```

> ⚠️ The `n8n-data` folder is used by the `node` user (UID 1000) inside the container. If n8n fails to start because of permissions, run `sudo chown -R 1000:1000 n8n-data`.

### 2. `docker-compose.yml`

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_automation
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=<YOUR_PI_IP>
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_SECURE_COOKIE=false
      - GENERIC_TIMEZONE=America/Costa_Rica
      - TZ=America/Costa_Rica

      # SD card protection: minimize writes
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=24
      - EXECUTIONS_DATA_PRUNE_TIMEOUT=3600
      - EXECUTIONS_SAVE_DATA_SUCCESS=none
      - EXECUTIONS_SAVE_DATA_ERROR=all
    volumes:
      - ./n8n-data:/home/node/.n8n
      - ./local-files:/files
```

> The `version: '3.8'` line is obsolete in Docker Compose v2 and only triggers a warning, so it can be omitted.

### 3. Start n8n

```bash
sudo docker-compose up -d
```

Open `http://<YOUR_PI_IP>:5678` and create the admin account.

### 4. Create the Telegram bot

1. Talk to [@BotFather](https://t.me/BotFather) → `/newbot`.
2. Choose a name (e.g. `Job Hunter Dylan`) and a username ending in `bot` (e.g. `dylan_jobhunter_bot`).
3. Save the **token** and create the *Telegram API* credential in n8n.
4. Send a message to the bot and get your **chat ID** (for example with a *Telegram Trigger* node or `getUpdates`).

### 5. Required credentials in n8n

- Google Gemini (PaLM) API key
- Google Sheets OAuth2
- Telegram Bot API
- (Optional) Adzuna `app_id` / `app_key` → <https://developer.adzuna.com>

### 6. Import the workflow

`Workflows → Import from file → workflow.json`, then reassign the credentials on each node.

---

## 🔧 Configuration

| What | Where |
|---|---|
| Frequency | **Daily Trigger** node |
| Search keywords | **Keywords** node (Filter) |
| Score threshold | **3. Filter - Score** node |
| Profile / base CV | Prompts of the **2. Evaluation**, **4. Cover Letter** and **6. ATS CV** nodes |
| Spreadsheets | Google Sheets nodes (*Read Processed*, *Save Discarded*, *Save Approved Match*) |
| Telegram chat ID | **5. Notification** and **7. Send CV** nodes |

---

## 📝 Notes and lessons learned

A summary of the issues I had to deal with while building this.

### Raspberry Pi / Docker
- **SD card wear**: n8n stores every execution in SQLite. I reduced writes with `EXECUTIONS_SAVE_DATA_SUCCESS=none`, 24 h pruning, and saving only failed executions. For serious long-term use, consider a **USB/NVMe SSD** instead of a microSD card.
- **Volume permissions**: the container runs as `node` (UID 1000); host permissions must match or n8n can't write to `./n8n-data`.
- **`N8N_SECURE_COOKIE=false`** is required to access over plain HTTP on a local network. If exposed to the internet, use HTTPS (reverse proxy / Cloudflare Tunnel) and turn it back on.
- **RAM (4 GB)**: enough because the AI runs in the cloud (Gemini). Running large local models alongside n8n on this device isn't realistic.
- **ARM64**: the official n8n image is multi-arch, so no changes are needed.
- n8n shows a notice about migrating to **n8n 3.0 (v2)**; review the *Migration report* before upgrading, and pin the image version instead of `latest` to avoid surprises.

### Workflow / n8n
- **Gemini rate limits**: `Loop Over Items` + `Wait` process listings one at a time.
- **Output Parser**: the evaluation agent uses *Require Specific Output Format* with a *Structured Output Parser* to get valid JSON (`score`, `justificacion`, `keywords`). It must be connected on the canvas or the node shows a warning.
- **Source normalization**: each API returns different fields; every source has its own Code node mapping to `title/url/description/budget/datePosted/source`, and they are then combined with *Merge (append)*.
- **Deduplication**: listings are compared against the processed-jobs Google Sheet so notifications aren't repeated.
- **CV as a file**: no PDF is generated (it would need extra tooling); a plain `.txt` ATS-friendly file is produced with the *Convert to File* node, ready to copy into a Word template.

### Known issues / improvements
- ⚠️ **Inconsistent score threshold**: the workflow note says `>= 80` approved, but the filter node uses `>= 60`. Unify the value.
- ⚠️ **Telegram shows literal `<b>` tags**: the node's *Parse Mode* (Markdown) doesn't match the text (HTML). Use `Parse Mode: HTML` or switch the text to Markdown.
- ⚠️ **Attachment file name**: the `.txt` takes the first words of the CV content as its name, producing long, truncated names. Set a fixed name with an expression, e.g. `CV_{{ $json.title }}.txt`.
- ⚠️ **Repeated caption** on every document sent; consider including the job title.
- Finish the **Adzuna** source (`app_id`, `app_key`, `keyword`).
- Add error handling (*Error Trigger*) with a Telegram alert.
- Back up `n8n-data` periodically and export the workflow into this repository.

---

## 📁 Repository structure

```
smart-job-hunter/
├── docker-compose.yml
├── workflow.json          # n8n workflow export (without credentials)
├── n8n-data/              # (git-ignored)
├── local-files/           # (git-ignored)
└── README.md
```

Recommended `.gitignore`:

```
n8n-data/
local-files/
.env
```

> 🔒 **Never commit** tokens, API keys, chat IDs or your personal CV. n8n exports don't include credentials, but check that your prompts don't contain sensitive data before publishing.

---

## 📜 License

MIT — use it and adapt it freely.
