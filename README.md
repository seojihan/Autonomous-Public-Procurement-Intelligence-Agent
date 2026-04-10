# ⚡ BidRadar AI
### *Autonomous Public Procurement Intelligence Agent*

> Continuously monitors Korea's national procurement portal (KONEPS / 나라장터),  
> cross-validates bid data against original RFP documents using Google Gemini,  
> and delivers a daily strategy report — straight to your inbox.

---

> 🎓 **This project was independently designed and built by a high school student.**  
> Every component — from multi-API orchestration to AI-powered report generation — was self-taught and implemented without professional guidance.

---

## 🧭 Overview

**BidRadar AI** is a fully automated n8n workflow that acts as a 24/7 procurement analyst for your business. It pulls live bid announcements from Korea's Public Data Portal, enriches them with license, region, and item constraints, runs a deep cross-validation analysis via Gemini AI, and compiles everything into a formatted PDF report delivered by email — four times a day.

No dashboards to check. No manual searches. Just intelligence, on schedule.

---

## ✨ Key Features

| Feature | Description |
|---|---|
| 📡 **Multi-API Collection** | Fetches bid listings, license limits, regional restrictions, and item categories from KONEPS in parallel |
| 🧠 **Gemini AI Analysis** | Cross-validates API data against original RFP document text to detect inconsistencies |
| 🚨 **Data Integrity Alerts** | Flags conflicts between database records and source documents with trust scores |
| 🗄️ **Supabase Storage** | Persists all bid snapshots and tracks report delivery status per user |
| 📄 **PDF Report Generation** | Converts AI-generated strategy reports to PDF via Google Drive |
| 📬 **Scheduled Email Delivery** | Sends branded HTML reports with PDF attachments at 4 fixed times daily |
| 🔁 **Per-User Loop** | Iterates across multiple company profiles stored in Supabase |
| 🕐 **KST-Aware Scheduling** | Time labels and filtering are calculated in Korean Standard Time regardless of server timezone |

---

## 🏗️ Architecture

```
Schedule Trigger (×4 daily)
        │
        ▼
  Get Keywords          ← Supabase: monitored_keywords
        │
        ▼
  Loop Over Items       ← Per keyword
        │
   ┌────┴────┐
   │         │
   ▼         ▼
[COLLECT]  [REPORT] (Morning pass only)
   │         │
   │         ▼
   │    Get Unsent Bids   ← Supabase: bid_snapshots (is_sent = false)
   │         │
   │         ▼
   │    Loop: Per User    ← Supabase: company_profiles
   │         │
   │         ▼
   │    DataAggregator    ← Formats bid data for AI prompt
   │         │
   │         ▼
   │    Gemini AI         ← RFP cross-validation + strategy report
   │         │
   │         ▼
   │    ReportJoiner      ← Merges multi-user reports
   │         │
   │         ▼
   │    Google Drive      ← Creates & downloads PDF
   │         │
   │         ▼
   │    Gmail             ← Sends HTML email + PDF attachment
   │         │
   │         ▼
   │    Mark as Sent      ← Supabase: bid_snapshots (is_sent = true)
   │
   ▼
[Parallel API Calls]
  ├─ Main Bid Info       (getBidPblancListInfoServcPPSSrch)
  ├─ License Limits      (getBidPblancListInfoLicenseLimit)
  ├─ Regional Limits     (getBidPblancListInfoPrtcptPsblRgn)
  ├─ Purchase Items      (getBidPblancListInfoServcPurchsObjPrdct)
  └─ File Attachments    (getBidPblancListInfoEorderAtchFileInfo)
        │
        ▼
  Code: Aggregate        ← Combines all API responses into one object
        │
        ▼
  Supabase: Upsert       ← Saves bid_snapshots
```

---

## 🗂️ Supabase Schema

### `monitored_keywords`
| Column | Type | Description |
|---|---|---|
| `id` | uuid | Primary key |
| `user_id` | uuid | Owner reference |
| `keyword` | text | Search keyword for bid filtering |

### `bid_snapshots`
| Column | Type | Description |
|---|---|---|
| `id` | uuid | Primary key |
| `bid_no` | text | KONEPS bid announcement number |
| `bid_name` | text | Bid title |
| `price` | numeric | Estimated budget |
| `detail_url` | text | Link to original announcement |
| `license_limit` | jsonb | Required license data (from API) |
| `region_limit` | jsonb | Eligible region data (from API) |
| `purchase_items` | jsonb | Procurement item list |
| `rfp_text` | text | Extracted RFP document content |
| `check_time` | text | Collection time label (Morning / Afternoon / Evening) |
| `keyword_used` | text | Keyword that triggered the collection |
| `user_id` | uuid | Owner reference |
| `is_sent` | boolean | Whether the report has been delivered |

### `company_profiles`
| Column | Type | Description |
|---|---|---|
| `id` | uuid | Primary key |
| `name` | text | Company name |
| `licenses` | text[] | Held license types |
| `regions` | text[] | Operating regions |
| `specialties` | text[] | Core competency keywords |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Workflow Engine** | [n8n](https://n8n.io) (self-hosted or cloud) |
| **Bid Data Source** | [Korea Public Data Portal](https://www.data.go.kr) — KONEPS API |
| **Database** | [Supabase](https://supabase.com) (PostgreSQL) |
| **AI Engine** | Google Gemini 2.5 Flash (`@n8n/n8n-nodes-langchain.googleGemini`) |
| **File Storage** | Google Drive (report archival) |
| **Email Delivery** | Gmail via OAuth2 |
| **Report Format** | HTML email + PDF attachment |

---

## 🚀 Setup Guide

### 1. Import the Workflow

In your n8n instance:
```
Workflows → Import from file → workflow.json
```

### 2. Configure Credentials

Set up the following credentials in n8n's **Credentials** panel:

| Credential | Type | Used By |
|---|---|---|
| `Supabase account` | Supabase API | Get / Create / Update rows |
| `Google Gemini API` | Google PaLM API | AI report generation |
| `Gmail account` | Gmail OAuth2 | Email delivery |
| `Google Drive account` | Google Drive OAuth2 | PDF creation & download |

### 3. Set Your API Key

In each HTTP Request node, replace the placeholder with your actual key:

```
YOUR_PUBLIC_DATA_API_KEY  →  your real key from data.go.kr
```

> Get your key at: [https://www.data.go.kr](https://www.data.go.kr)  
> Search for: `나라장터 입찰공고정보서비스`

### 4. Set Recipient Email

In the **"Send a message"** (Gmail) node, update `sendTo`:
```
YOUR_EMAIL_ADDRESS  →  your actual email address
```

### 5. Populate Supabase Tables

Add at least one row to each of:
- `monitored_keywords` — keywords you want to track (e.g. `시스템`, `유지보수`)
- `company_profiles` — your company info (used to personalize AI analysis)

### 6. Activate

Toggle the workflow **Active** — the Schedule Trigger fires automatically at:

| Time (KST) | Label | Action |
|---|---|---|
| 01:00 | Dawn | Collect overnight bids |
| 08:00 | Morning | Collect + **Send daily report** |
| 12:00 | Afternoon | Collect midday bids |
| 19:00 | Evening | Collect end-of-day bids |

---

## 🧠 AI Analysis Logic

BidRadar AI uses a **two-source cross-validation** strategy:

**Source A — API Facts**: structured data from the KONEPS REST API (license codes, region codes, item names)

**Source B — RFP Source Truth**: raw text extracted from the original procurement document

When the two sources conflict (e.g., API says "agricultural license" but RFP says "software vendor"), Gemini **always defers to the RFP** and flags the discrepancy as a data integrity warning.

### Report Structure (per bid)
```
■ Data Integrity Summary
■ BEST MATCH — Cross-validated bid
  ├─ Eligibility Matrix (API vs. RFP comparison table)
  ├─ Fact-Check Commentary
  ├─ Winning Strategy Recommendation
  ├─ ROI & Resource Simulation
  └─ Source Links
⚠️  Legal Disclaimer
```

---

## ⚠️ Security Notes

- **Never commit your actual API key.** Use n8n's built-in credential store instead.
- **Never hardcode email addresses.** Use n8n environment variables or expressions.
- All Supabase credential IDs in `workflow.json` (`AXfDBvuu6X36mRaL`, etc.) are **n8n-internal references only** — they carry no sensitive value outside your own n8n instance.
- Recommended: self-host n8n behind a reverse proxy with HTTPS and authentication enabled.

---

## 📁 Repository Structure

```
├── workflow.json        # n8n workflow export (sanitized — no real credentials)
└── README.md            # This file
```

---

## 📜 License

This project is for **educational and personal automation purposes**.  
KONEPS API data is subject to the [Korea Public Data Portal Terms of Use](https://www.data.go.kr/ugs/selectPublicDataUseTermsView.do).

---

*Built with n8n · Powered by Gemini · Fueled by coffee ☕*
