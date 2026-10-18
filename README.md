# Marketing Database Agent

A Gemini Enterprise chat agent that builds prospect lists from the 66degrees marketing
database in BigQuery and hands the user a downloadable CSV, with a full admin audit trail.

**Owners:** Deepanjan Dasgupta · Neelkant Ekbote
**GCP project:** set your own (see [Configuration](#configuration))

---

> 👉 New here? Start with **[examples/EXAMPLES.md](examples/EXAMPLES.md)** to see the agent in action.

## What a user experiences

1. Opens **Marketing Database Agent** in Gemini Enterprise and clicks **🎯 Build a Prospect Database**.
2. Fills in Event Name, Solution, Database Count (max 5,000), Job Title(s), City, State, Country.
3. Clicks **Send** on the BigQuery confirmation.
4. Gets a short reply with a **Download CSV** link. The file saves as
   `Database - <Event Name>.csv` with 11 columns:
   Company Name, Website, First Name, Last Name, Job Title, Email ID, Industry,
   Prospect LinkedIn, City, State, Country.

No prospect rows are shown in the chat.

## What the admins get

Every generation, by anyone, automatically:
- **Emails** the admins (who, when, how many, event, solution, titles,
  location, monthly total for that user, and a 7-day download copy). See
  `docs/admin-email-preview.png`.
- **Adds a row** to the permanent log table
  `your-gcp-project-id.prospect_agent_logs.generation_log`.

The user's identity comes from BigQuery's own job history, so it cannot be skipped or faked.

---

## How it works

```
User ──► Gemini Enterprise agent (instructions only, BigQuery connector)
           │  Query 1: count matches
           │  Query 2: EXPORT DATA ──► gs://your-gcp-project-id-prospect-exports/exports/<run_id>/<Event>_000000000000.csv
           ▼
        Reply with Download CSV link

New file in bucket ──► Cloud Run function `prospect-csv-rename`
                         ├─ sets download name "Database - <Event>.csv"
                         ├─ looks up who ran the export (INFORMATION_SCHEMA.JOBS)
                         ├─ counts the real rows
                         ├─ inserts a row into prospect_agent_logs.generation_log
                         └─ POSTs to the Apps Script web app ──► HTML email to admins

Hourly scheduled query ──► refresh_prospect_view() rebuilds v_prospects_clean
                           from EVERY table in raw_marketing_vendor_database
```

### Job title matching
- The user's titles are matched with word-boundary regex (`\bcto\b`, so "director" is never matched as CTO).
- **Solution** (Agentic AI, Data & AI, Cloud, Security, Marketing, Sales…): if the user's titles
  can't fill the requested count, senior titles related to the solution fill the gap.
  The user's own titles always come first.

---

## Repository layout

| Folder | What | Runs where |
|---|---|---|
| `agent/` | Agent instructions + starter prompt | Gemini Enterprise agent editor |
| `bigquery/` | `setup_auto_view.sh`: clean view over all tables, refreshed hourly | Cloud Shell |
| `csv-helper/` | Cloud Run function: rename + log + email | Cloud Run (`prospect-csv-rename`, us-central1) |
| `apps-script/` | `Code.gs`: HTML email sender (web app) | script.google.com |
| `mcp-server-optional/` | MCP server, built but blocked by org policy | Not in use |
| `examples/` | **Worked examples**: chats, Solution fill, title matching, CSV sample, SQL, email, log queries | — |
| `docs/` | Email preview, [troubleshooting](docs/TROUBLESHOOTING.md) | — |

---

## Google Cloud resources

| Resource | Name |
|---|---|
| Source dataset | `raw_marketing_vendor_database` (Marketing_Database_0001…, any future tables) |
| Clean view | `raw_marketing_vendor_database.v_prospects_clean` |
| Refresh procedure | `raw_marketing_vendor_database.refresh_prospect_view` |
| Scheduled query | "Refresh prospect view" (every 1 hour, US) |
| Export bucket | `your-gcp-project-id-prospect-exports` (files auto-delete after 7 days) |
| Log table | `prospect_agent_logs.generation_log` |
| Cloud Run function | `prospect-csv-rename` (trigger: object finalized in the export bucket) |
| Runtime service account | `YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com` |
| Apps Script | "Prospect Database Admin Emails" (web app, execute as owner, access: Anyone) |

The export bucket and its 7-day auto-delete were created earlier by `mcp-server-optional/deploy.sh`. A fresh project must create them first (see step 3 below).

---

## Configuration

This repo is public, so it contains **no real project IDs, emails, URLs or secrets**.
Before running anything, replace these placeholders in the files:

| Placeholder | Replace with | Where |
|---|---|---|
| `your-gcp-project-id` | Your Google Cloud project ID | All scripts, `agent/agent_instructions.md` |
| `YOUR_PROJECT_NUMBER` | Your project number (`gcloud projects describe <id> --format="value(projectNumber)"`) | `bigquery/`, `csv-helper/`, `mcp-server-optional/` |
| `admin-one@example.com`, `admin-two@example.com` | The admin emails that receive alerts | `csv-helper/deploy_logging.sh`, `csv-helper/index.js`, `apps-script/Code.gs` |
| `PASTE_YOUR_SECRET_HERE` | A random secret (`openssl rand -hex 16`) | `apps-script/Code.gs` (and passed to `deploy_logging.sh`) |

The Apps Script web app URL and the secret are passed to `deploy_logging.sh` as arguments
at deploy time and are never committed.

Quick replace (run in the repo folder):
```bash
grep -rlI --exclude-dir=.git -e your-gcp-project-id -e YOUR_PROJECT_NUMBER -e admin-one@example.com -e admin-two@example.com . | xargs sed -i \
  -e 's/your-gcp-project-id/<PROJECT_ID>/g' \
  -e 's/YOUR_PROJECT_NUMBER/<PROJECT_NUMBER>/g' \
  -e 's/admin-one@example.com/<ADMIN_EMAIL_1>/g' \
  -e 's/admin-two@example.com/<ADMIN_EMAIL_2>/g'
```

---

## Setup from scratch (Cloud Shell)

1. **Clean view over all tables**
   ```bash
   cd bigquery && bash setup_auto_view.sh
   ```
2. **Email sender**: create an Apps Script project, paste `apps-script/Code.gs`, set
   `SHARED_SECRET` (generate with `openssl rand -hex 16`), run `sendTestEmail` once,
   then **Deploy → Web app** (Execute as: Me, Access: Anyone) and copy the URL.
3. **Export bucket** (only if it does not exist yet). The helper deploy fails without it:
   ```bash
   gcloud storage buckets create gs://your-gcp-project-id-prospect-exports --location=us-central1 --uniform-bucket-level-access --public-access-prevention
   gcloud storage buckets add-iam-policy-binding gs://your-gcp-project-id-prospect-exports --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" --role="roles/storage.objectAdmin"
   gcloud iam service-accounts add-iam-policy-binding YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
   ```
   Add the 7-day auto-delete rule as in `mcp-server-optional/deploy.sh`. On a brand-new project,
   also grant `roles/pubsub.publisher` to the Cloud Storage service agent
   (`gcloud storage service-agent`) and `roles/eventarc.eventReceiver` to the runtime
   service account, because the helper is triggered by bucket events.
4. **Rename + log + email helper**
   ```bash
   cd csv-helper
   bash deploy_logging.sh "<APPS_SCRIPT_WEB_APP_URL>" "<SECRET>"
   bash test_logging.sh
   ```
5. **Agent**: paste `agent/agent_instructions.md` into the agent's Instructions, and
   `agent/starter_prompt.txt` as a starter prompt. Keep the BigQuery connector attached.

> Secrets are **never** stored in this repo. `deploy_logging.sh` writes them to a local
> `env.yaml`, which is git-ignored.

---

## Useful admin queries

Who took how much this month:
```sql
SELECT user_email, COUNT(*) AS downloads, SUM(delivered_count) AS records
FROM `your-gcp-project-id.prospect_agent_logs.generation_log`
WHERE DATE(generated_at, 'Asia/Kolkata') >= DATE_TRUNC(CURRENT_DATE('Asia/Kolkata'), MONTH)
GROUP BY user_email
ORDER BY records DESC;
```

Latest generations:
```sql
SELECT FORMAT_TIMESTAMP('%d %b %Y %I:%M %p', generated_at, 'Asia/Kolkata') AS time_ist,
       user_email, event_name, solution, requested_count, delivered_count
FROM `your-gcp-project-id.prospect_agent_logs.generation_log`
ORDER BY generated_at DESC
LIMIT 20;
```

Rebuild the view immediately after uploading a new table (otherwise it happens within the hour):
```sql
CALL `your-gcp-project-id.raw_marketing_vendor_database.refresh_prospect_view`();
```

---

## Known limits

- Maximum 5,000 records per request.
- Download links in the chat require the user to have read access to the export bucket.
- Gmail can't load the Lexend font in the admin email and falls back to its default font.
- Custom MCP connectors are blocked by org policy (see `mcp-server-optional/`).
- **Sharing with more users** needs: Gemini Enterprise access to the agent, BigQuery Job
  User + Data Viewer, and object create/read on the export bucket.
