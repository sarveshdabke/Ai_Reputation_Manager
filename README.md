<div align="center">

# 🛡️ AI Reputation Manager

### Automated Customer Feedback Monitoring & Intelligent AI Response Pipeline

[![n8n](https://img.shields.io/badge/Orchestration-n8n-FF6C37?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io/)
[![Groq](https://img.shields.io/badge/LLM_Engine-Groq_Llama3.3-f34f29?style=for-the-badge)](https://groq.com/)
[![Brevo](https://img.shields.io/badge/Alerts-Brevo_SMTP-00a2aa?style=for-the-badge&logo=brevo&logoColor=white)](https://www.brevo.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

**An automation workflow that ingests local business review data, uses a generative LLM to draft responses to low-rated reviews, sends email alerts for approval, and writes processed state back to disk.**

[Overview](#-workflow-overview--architecture) •
[Setup](#️-infrastructure-provisioning--api-setup) •
[Installation](#-deployment--installation) •
[Project Structure](#-file-structure) •
[Troubleshooting](#-troubleshooting) •
[License](#-license)

</div>

---

## 📊 Workflow Overview & Architecture

Built around `review-automation-workflow.json`, the pipeline is organized into three sequential stages:

```
[ Manual Trigger ]
      │
      ▼
[ 1. Ingestion & Filtering ]
      │   Reads local Reviews & Business Master sheets,
      │   filters for low-rated, unprocessed reviews
      ▼
[ 2. AI Engine & Alerts ]
      │   Groq LLM agent drafts a response,
      │   Brevo sends an email for review/approval
      ▼
[ 3. Data Synchronization ]
      │   Marks reviews as processed,
      │   rewrites the master Excel file safely
      ▼
    [ Done ]
```

If any stage fails, an `Error Trigger` node intercepts the failure and emails the administrator with a system health alert — see [Fault Tolerance](#️-fault-tolerance).

---

## 🔍 Step-by-Step Pipeline Execution

### Phase 1 — Data Ingestion & Filtering

| Step | Node(s) | Description |
|------|---------|-------------|
| 1 | `When clicking 'Execute workflow'` | Manual trigger starts the run |
| 2 | Binary read nodes | Reads `MockData/Reviews.xlsx` and `MockData/Business_Master.xlsx` |
| 3 | `Parse_Reviews_XLSX`, `Parse_BusinessDetails_XLSX` | Converts spreadsheet rows into JSON |
| 4 | `Filter_Unprocessed_Reviews` | Keeps rows where `rating < 3` **and** `processed == false` |

### Phase 2 — AI Engine & Notifications

| Step | Node(s) | Description |
|------|---------|-------------|
| 1 | `Edit Fields1`, `Merge` | Joins review and business records on `shop_id` |
| 2 | `AI_Response_Generator` | LangChain agent node receives the merged review context |
| 3 | `Groq_Llama3_LLM` | Calls `llama-3.3-70b-versatile` to draft an empathetic, on-brand reply (capped at 70 words) |
| 4 | `Dispatch_Alert_Email` | Sends an HTML email via Brevo SMTP containing the original complaint and the AI-drafted response for human approval |

### Phase 3 — Data Synchronization

| Step | Node(s) | Description |
|------|---------|-------------|
| 1 | `Update_Processed_Flags` | Sets `processed = true`, `alert_sent = true` on handled rows |
| 2 | `Merge5`, `Combine_All_Rows_For_Export` | Recombines updated rows with untouched historical rows |
| 3 | `Generate_Final_Excel_Data`, `Overwrite_Local_Excel_File` | Rebuilds the workbook and writes it back to disk |

### 🛡️ Fault Tolerance

> If any node in the sequence fails mid-run, a root-level `Error Trigger` catches the failure and routes it to a `Send an Email` node, notifying the administrator immediately instead of failing silently.

---

## 🛠️ Infrastructure Provisioning & API Setup

### 1. Groq API (LLM Engine)

Used by the LangChain chat node to generate review responses.

1. Go to the [Groq Cloud Console](https://console.groq.com/) and sign in or register.
2. Open **API Keys** from the left navigation.
3. Generate a new secret key (starts with `gsk_...`).
4. In n8n: **Credentials → New → Groq Cloud API**, and paste the key.

### 2. Brevo SMTP (Email Alerts)

Used to send status alerts and approval emails.

1. Create an account at [Brevo](https://www.brevo.com/).
2. From your profile menu, go to **SMTP & API → SMTP tab**.
3. Click **Generate a new SMTP key** and copy it.
4. In n8n: **Credentials → New → SMTP**, and fill in:

   | Field | Value |
   |-------|-------|
   | SMTP Server | `smtp-relay.brevo.com` |
   | Port | `587` (TLS) |
   | Username | Your registered Brevo email |
   | Password | The SMTP key generated above |

> ⚠️ **Never commit API keys or SMTP credentials to the repository.** Store them only in n8n's credential manager, and make sure `.env` / credential export files are listed in `.gitignore`.

---

## 🚀 Deployment & Installation

### Prerequisites

- [n8n](https://docs.n8n.io/getting-started/installation/) installed locally or self-hosted (`npx n8n` or Docker)
- A Groq API key
- A Brevo account with SMTP access
- Node.js 18+ (only if running n8n via npm)

### File Structure

```text
Ai_Reputation_Manager/
├── MockData/
│   ├── Business_Master.xlsx
│   └── Reviews.xlsx
├── .gitignore
├── LICENSE
├── README.md
└── review-automation-workflow.json
```

### Import Steps

1. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/Ai_Reputation_Manager.git
   cd Ai_Reputation_Manager
   ```
2. Start n8n and open the editor UI (default: `http://localhost:5678`).
3. Create a new, blank workflow.
4. Open `review-automation-workflow.json`, copy its contents, then paste directly onto the n8n canvas (`Ctrl+V` / `Cmd+V`) to import all nodes and connections.
5. Open the `Groq_Llama3_LLM` and `Dispatch_Alert_Email` nodes and attach your own Groq and SMTP credentials (see [setup](#️-infrastructure-provisioning--api-setup) above).
6. Confirm the file-read/write nodes point at your local `MockData/` paths.
7. Click **Execute Workflow** to run the pipeline end-to-end.

---

## 🧪 Testing It Out

- Add a row to `MockData/Reviews.xlsx` with `rating < 3` and `processed = FALSE`.
- Run the workflow manually.
- Check the inbox tied to your Brevo/SMTP account for the alert email.
- Confirm the row's `processed` flag flips to `TRUE` in the rewritten Excel file.

---

## 🩹 Troubleshooting

| Issue | Likely Cause |
|-------|---------------|
| Workflow fails at the LLM node | Invalid or expired Groq API key |
| No email received | Incorrect SMTP credentials, or the alert landed in spam |
| Excel file not updating | File is open elsewhere / locked, or path is misconfigured |
| Rows processed twice | `processed` flag not being read correctly — check column name matches exactly |

---

## 🗺️ Roadmap

- [ ] Add support for multi-language review responses
- [ ] Replace local Excel storage with a database (e.g., Postgres/Airtable)
- [ ] Add Slack/Teams alert option alongside email
- [ ] Add a review-response approval step before sending publicly

---

## 🤝 Contributing

Contributions are welcome. Please open an issue first to discuss any significant change, then submit a pull request.

## 📄 License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.