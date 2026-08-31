# ClinicalBrief AI

An agentic Clinical Decision Support (CDS) pipeline built in n8n. Patient lab data enters via Google Sheet. GPT-4o searches NIH PubMed in real time, retrieves clinical protocol guidelines via RAG, scores confidence, and delivers a source-cited clinical brief to the clinician's inbox — routing uncertain or extreme cases to human review before any document is signed.

---

## What it does

1. **Patient or pharmacy staff fills a Google Sheet** with lab values — HbA1c, LDL, BMI, blood pressure, medications, diagnoses. No app. No login.
2. **SerpApi queries Google Scholar** in real time using the patient's diagnoses. Three current peer-reviewed publications are retrieved and injected into the agent prompt before any generation happens.
3. **GPT-4o agent reasons** over patient data + live PubMed evidence + RAG-retrieved clinical guidelines. Produces a structured, source-cited brief with a confidence score.
4. **Confidence Gate** evaluates the combined score (agent score + extraction confidence) / 2. At or above 0.80: brief is sent for electronic signature. Below 0.80: routed to human review.
5. **Clinician receives an HTML email** — full summary, diagnosis, risk level, numbered recommendations with evidence citations, medications, and physician notes.

---

## Pipeline architecture

```
Google Sheets Trigger (Form Responses 1 tab)
    ↓
Input Validator — validates required fields, passes all lab columns
    ↓
Nutrient — Extract Lab Data — structures lab values, scores confidence, flags extreme values
    ↓
Format Extraction Output — cleans data, calculates extraction confidence, flags clinical thresholds
    ↓
SerpApi — Clinical Search — real-time Google Scholar query using patient diagnoses
    ↓
Build Agent Prompt — merges patient data + PubMed evidence into structured prompt
    ↓
Clinical Intelligence Agent (GPT-4o, temperature 0.1)
    ├── Tool: retrieve_clinical_protocols (RAG — in-memory vector store)
    └── Output: Clinical Brief Parser (structured JSON schema)
    ↓
Validate & Merge Output — combines agent score + extraction confidence
    ↓
Confidence Gate (IF node — threshold 0.80)
    ├── True (≥ 0.80)  → Foxit eSign → Deliver Signed Brief (Gmail) → Audit Log
    └── False (< 0.80) → Human Review Queue (Gmail) → Audit Log
```

---

## Confidence scoring

```
confidence_score = (agent_score + extraction_confidence) / 2
```

**agent_score** — GPT-4o's self-assessed confidence that evidence supports each recommendation (0–1).

**extraction_confidence** — average field-level confidence score across lab values. Penalised by:
- Vague text in medications or diagnoses (e.g. "unknown", "possible")
- Missing optional fields
- Extreme clinical values above safety thresholds:
  - HbA1c > 10.0% → score drops to 0.60
  - LDL > 190 mg/dL → score drops to 0.60
  - BMI > 40 → score drops to 0.62
  - Systolic BP > 160 mmHg → score drops to 0.62

Clinical flags are passed through to the human review email so the clinician knows exactly why the case was flagged.

---

## Google Sheet schema

Tab name: `Form Responses 1`

| Column | Required | Description | Example |
|---|---|---|---|
| patient_id | ✅ | Unique patient identifier | patient_001 |
| clinician_email | ✅ | Where the brief is delivered | dr.smith@clinic.org |
| document_type | ✅ | lab_report / discharge_summary / radiology / pathology | lab_report |
| hba1c | ✅ | Glycated haemoglobin — key diabetes marker | 7.2% |
| ldl | ✅ | LDL cholesterol | 145 mg/dL |
| diagnoses | ✅ | Comma-separated active diagnoses | Type 2 Diabetes, Hypertension |
| bmi | optional | Body Mass Index | 28.4 |
| blood_pressure | optional | Systolic/diastolic | 138/88 mmHg |
| medications | optional | Comma-separated current medications | metformin 1000mg, lisinopril 10mg |
| notes | optional | Free-text clinical notes | quarterly review |

---

## Clinical knowledge base

6 clinical protocol documents embedded using OpenAI text-embedding-3-small (1,536 dimensions, cosine similarity, top-K=3):

| Document | Covers |
|---|---|
| American Diabetes Association — Standards of Medical Care 2024 | HbA1c targets, metformin, GLP-1, SGLT2 thresholds |
| American College of Cardiology / AHA — Cardiovascular Risk Guidelines 2023 | LDL targets, statin therapy, cardiovascular risk scoring |
| Joint National Committee 8th Report — Hypertension Guidelines | Blood pressure targets, ACE inhibitors, first-line medications |
| World Health Organization — Metabolic Syndrome Diagnostic Criteria | Metabolic syndrome definition, HDL, triglycerides |
| National Kidney Foundation KDIGO — CKD Guidelines 2023 | Kidney disease staging, diabetic nephropathy management |
| US Preventive Services Task Force — Preventive Care 2024 | Screening intervals, aspirin use, preventive recommendations |

All sources are open-access or public domain.

---

## Test data — one row per use case

```csv
patient_id,clinician_email,document_type,hba1c,ldl,bmi,blood_pressure,medications,diagnoses,notes
patient_normal,your@email.com,lab_report,7.2%,125 mg/dL,26.8,128/82 mmHg,"metformin 1000mg, atorvastatin 20mg","Type 2 Diabetes, Dyslipidemia",stable quarterly review
patient_review_extreme,your@email.com,lab_report,11.2%,210 mg/dL,42.5,168/104 mmHg,"metformin 1000mg, lisinopril 10mg","Type 2 Diabetes, Severe Hypertension",all values critically elevated
patient_review_vague,your@email.com,lab_report,6.4%,101 mg/dL,24.2,122/78 mmHg,unknown supplement,possible metabolic issue,patient unsure of diagnosis
patient_high_risk,your@email.com,lab_report,9.1%,162 mg/dL,31.5,144/92 mmHg,"metformin 500mg, amlodipine 5mg","Type 2 Diabetes, CKD Stage 3",worsening from last visit
patient_well_controlled,your@email.com,lab_report,6.2%,88 mg/dL,23.4,118/74 mmHg,"sitagliptin 100mg, rosuvastatin 10mg","Type 2 Diabetes, Dyslipidemia",annual review — excellent adherence
```

Expected routing:
- `patient_normal` → green auto-sign
- `patient_review_extreme` → red review (4 clinical flags)
- `patient_review_vague` → red review (vague medications and diagnoses)
- `patient_high_risk` → red review (HbA1c borderline, CKD complicates)
- `patient_well_controlled` → green auto-sign

---

## Setup and run instructions

### Prerequisites

- n8n cloud account (n8n.cloud) or self-hosted n8n
- OpenAI API key — platform.openai.com
- SerpApi API key — serpapi.com (free trial: 100 searches)
- Foxit eSign API credentials — account.foxit.com
- Google account (Sheets trigger + Gmail delivery)

### Step 1 — Create credentials in n8n

Go to n8n → Credentials → New and create:

| Credential | Type | Where to get |
|---|---|---|
| OpenAI | OpenAI API | platform.openai.com → API keys |
| Google Sheets | Google Sheets Trigger OAuth2 | Sign in with Google |
| Gmail | Gmail OAuth2 | Sign in with Google |

SerpApi and Foxit credentials are embedded directly in the HTTP Request node parameters — update them there after import.

### Step 2 — Import workflows

In n8n → Workflows → Import from file:

1. Import `ClinicalBrief_AI_Main_Workflow_REDACTED.json`
2. Import `ClinicalBrief_AI_Seed_KB_REDACTED.json`

### Step 3 — Replace placeholders

In the main workflow, replace these values in the relevant nodes:

| Placeholder | Replace with | Node |
|---|---|---|
| `YOUR_OPENAI_CREDENTIAL_ID` | Your n8n OpenAI credential ID | GPT-4o Clinical Model, OpenAI Embeddings |
| `YOUR_GMAIL_CREDENTIAL_ID` | Your n8n Gmail credential ID | Deliver Signed Brief, Human Review Queue |
| `YOUR_SHEETS_CREDENTIAL_ID` | Your n8n Sheets credential ID | Google Sheets Trigger |
| `YOUR_SERPAPI_KEY` | Your SerpApi API key | SerpApi — Clinical Search node → query params |
| `YOUR_FOXIT_API_KEY` | Your Foxit API key | Foxit node → X-API-KEY header |
| `YOUR_FOXIT_API_SECRET` | Your Foxit API secret | Foxit node → X-API-SECRET header |
| `YOUR_NUTRIENT_BASIC_AUTH_BASE64` | Base64 of username:password | Nutrient node → Authorization header |

### Step 4 — Create Google Sheet

Create a Google Sheet with a tab named `Form Responses 1` and the columns listed in the schema above. Copy the Sheet ID from the URL and set it in the Google Sheets Trigger node.

### Step 5 — Seed the clinical knowledge base

Open the Seed KB workflow → click Test Workflow. Wait for green checkmark.

**Important:** The knowledge base uses n8n's in-memory vector store. It resets every time n8n restarts. Re-run the Seed KB workflow before every session.

### Step 6 — Activate and test

Toggle the main workflow active. Add a test row to the Google Sheet. Within 60 seconds check the clinician email inbox.

---

## Known caveats and limitations

### Nutrient DWS integration — simulated

The `Nutrient — Extract Lab Data` node is currently a **Code node** that reads lab values directly from the Google Sheet columns. It does not make an API call to the Nutrient DWS Data Extraction API.

**Why:** The Nutrient hackathon credentials returned authentication errors on the extraction endpoint during development. The sheet-based approach provides equivalent structured output for the demo.

**Production path:** Replace the Code node with an HTTP Request node calling `POST https://api.nutrient.io/extraction/extract` with a Bearer token and a JSON schema defining the fields to extract. The upstream node would accept a PDF binary upload rather than sheet columns.

### Foxit eSign — sandbox endpoint

The Foxit eSign node is configured with the sandbox base URL `https://na1.fusion.foxit.com/api/esign/v1/envelopes`. This endpoint currently returns 404 on the sandbox account. The node is set to `onError: continueRegularOutput` so the pipeline continues and delivers the email regardless.

**Production path:** Verify the correct sandbox endpoint from the Foxit developer portal and update the URL.

### RAG store — in-memory only

The clinical knowledge base uses n8n's Simple In-Memory Vector Store. It does not persist across n8n restarts.

**Production path:** Replace with Supabase pgvector for persistent storage — no reseed required.

### SerpApi tool limitation inside agent

SerpApi is called as a regular HTTP Request node outside the agent, not as an agent tool. The `toolHttpRequest` node type cannot execute from inside the n8n AI Agent node. SerpApi results are injected into the agent prompt via the Build Agent Prompt Code node.

### Single-patient sequential processing

The Google Sheets trigger polls once every 60 seconds and processes rows sequentially. Not suitable for high-volume concurrent use.

**Production path:** Replace Sheets trigger with a webhook trigger + n8n queue mode for parallel execution.

---

## Project

Built and voluntarily contributed to the open innovation community.
Published under LearnHive Labs · learnhive.org

Author: Uday Shankar Bhowal
Lead Analyst @ Cigna Evernorth · MS Healthcare Informatics · Austin TX
ORCID: 0009-0006-6948-4914
