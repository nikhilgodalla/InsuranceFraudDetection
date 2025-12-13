# Insurance Claim Extractor Backend (FastAPI + Gemini)

This backend accepts insurance claim **PDFs**, classifies the document type (e.g., `health`, `auto`, `property`), and extracts structured key-value data using **Google Gemini 2.5-Flash** or other supported models.

Built with:
- **FastAPI** (Python 3.11+)
- **Google GenAI SDK (`google-genai`)**
- **Snowflake** for data persistence
- **Pydantic v2** for schemas
- **Structlog** for JSON logging
- **uv** (optional) for modern dependency & environment management

---

## Folder Overview
```
app/
├─ main.py
├─ routes/extract.py
├─ extractors/extractor.py
├─ classifiers/doc_type.py
├─ claim_types/
│  ├─ registry.py
│  ├─ health/
│  │  ├─ schema.py
│  │  └─ prompt.txt
│  ├─ auto/
│  └─ property/
├─ llm/
│  ├─ client.py
│  ├─ pricing.py
│  └─ usage.py
├─ db/                     ← NEW
│  └─ snowflake_utils.py
├─ config.py
├─ logging_conf.py
├─ middleware.py
└─ common/
```

---

## 1. Environment Setup

### Required Environment Variables (`.env`)

Create a `.env` file in your project root:
```bash
# For Gemini Developer API
USE_VERTEXAI=false
GEMINI_API_KEY=your_api_key_here

GENAI_MODEL=gemini-2.5-flash
MAX_OUTPUT_TOKENS=4096
APP_ENV=dev

# Snowflake Configuration (Optional - for data persistence)
SNOWFLAKE_ACCOUNT=your_account_identifier
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=INSURANCE_FRAUD_DB
SNOWFLAKE_SCHEMA=CLAIMS
```

> Get your Gemini API key from https://aistudio.google.com/app/apikey.

### Snowflake Setup (Optional)

If using Snowflake for data persistence:

1. Sign up at https://signup.snowflake.com/ (free trial)
2. Create database and table:
```sql
CREATE DATABASE IF NOT EXISTS INSURANCE_FRAUD_DB;
CREATE SCHEMA IF NOT EXISTS INSURANCE_FRAUD_DB.CLAIMS;
USE DATABASE INSURANCE_FRAUD_DB;
USE SCHEMA CLAIMS;

CREATE TABLE IF NOT EXISTS extracted_claims (
    claim_id VARCHAR(100) PRIMARY KEY,
    transaction_id VARCHAR(100),
    doc_type VARCHAR(50),
    customer_name VARCHAR(500),
    claim_amount VARCHAR(50),
    severity VARCHAR(50),
    extracted_data VARIANT,
    model_used VARCHAR(100),
    total_tokens INTEGER,
    cost_usd FLOAT,
    created_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

**Note:** Extracted claims are automatically saved to Snowflake if configured. The system works without Snowflake (saves are non-blocking).

---

## 2. Installation

### Option A — using **uv** (recommended)

If you already have [`uv`](https://github.com/astral-sh/uv):
```bash
# Install dependencies (reads pyproject.toml)
uv pip install --system .

# Run development server
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8080
```

**Common uv commands:**
```bash
uv add fastapi uvicorn[standard] google-genai structlog pydantic python-dotenv
uv remove <package>
uv run pytest             # run tests
uv pip freeze             # list deps
```

---

### Option B — using classic **Python venv + pip**
```bash
# Create and activate a venv
python3 -m venv .venv
source .venv/bin/activate        # macOS/Linux

# Install dependencies
pip install -r requirements.txt
# or, if you have pyproject.toml:
pip install .
```

Run the server:
```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload
```

---

## 3. Running the API

Once started, you'll see:
```
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```

Check health:
```bash
curl http://localhost:8080/healthz
# → {"status":"ok"}
```

---

## 4. Testing with Postman or curl

### Endpoint
```
POST http://localhost:8080/v1/extract
```

### Query parameters

| Key | Type | Default | Description |
|------|------|----------|-------------|
| `doc_type` | string | *optional* | One of `health`, `auto`, `property`. If omitted, classifier will attempt detection. |
| `classify_if_missing` | bool | `true` | Whether to auto-classify when `doc_type` is missing. |

---

### Body (form-data)

| Key | Type | Description |
|------|------|-------------|
| `file` | *File* | Choose your claim PDF to upload. |

Example:  

#### Postman  
1. Select **Body → form-data**  
2. Add key `file` → type **File** → select your PDF.  
3. Optional: add `doc_type=health` as a query param.  
4. Click **Send**

#### curl
```bash
curl -X POST "http://localhost:8080/v1/extract?doc_type=health" \
     -F "file=@Claim_Documents/TXN00000001_health.pdf"
```

---

### Successful Response Example
```json
{
  "doc_type": "health",
  "model": "gemini-2.5-flash",
  "result": {
    "company_name": "ACME INSURANCE COMPANY",
    "transaction_id": "TXN-TXN00000001",
    "policyholder_info": {
      "policy_number": "PLC00008468",
      "customer_name": "Christopher Demarest",
      "insurance_type": "Health",
      "customer_id": "A00003822"
    },
    "claim_summary": {
      "amount": "$9,000.00",
      "reported_date": "2020-05-21",
      "severity": "Major Loss"
    },
    "summary": "Health insurance claim for $9,000 by Christopher Demarest."
  },
  "usage": {
    "input_tokens": 597,
    "output_tokens": 830,
    "total_tokens": 1427
  },
  "cost_estimate": {
    "input_cost_usd": 0.000179,
    "output_cost_usd": 0.002075,
    "total_cost_usd": 0.002254
  }
}
```

---

## 5. Common Troubleshooting

| Symptom | Cause / Fix |
|----------|-------------|
| `Missing key inputs argument!` | Your `.env` is missing `GEMINI_API_KEY` or `USE_VERTEXAI` flags are wrong. |
| `415 Unsupported file type` | You sent JSON; must use `multipart/form-data` with a file. |
| `422 Could not classify document` | Classifier couldn't decide. Pass `doc_type` explicitly (e.g., `?doc_type=health`). |
| `doc_type 'property' not enabled` | Enable the type in `app/claim_types/registry.py`. |
| `_async_httpx_client` AttributeError | Older `google-genai` bug — upgrade: `uv add -U google-genai`. |
| `Snowflake connection failed` | Check `.env` credentials. Ensure warehouse is running in Snowflake. |

---

## 6. Cost Calculation

Each response includes an estimated cost based on the current **Gemini 2.5-Flash** pricing  
(as of Nov 2025):

| Type | Rate per 1M tokens |
|------|--------------------|
| Input | $0.30 |
| Output | $2.50 |

You can adjust these constants in `app/config.py` if pricing changes.

---

## 7. Production Deployment (optional)

### Docker
```bash
docker build -t claim-extractor .
docker run -p 8080:8080 --env-file .env claim-extractor
```

### GCP Cloud Run (outline)
1. Push the image to Artifact Registry.  
2. Deploy a Cloud Run service with:  
   - `PORT=8080`  
   - Secret `GEMINI_API_KEY` mounted as env var.  
3. Configure concurrency = 1–2 per CPU for best latency.

---

## 8. Extending for New Claim Types

To add a new claim type (`auto`, `property`, etc.):

1. Create a folder under `app/claim_types/<type>/`
```
   schema.py     # Define <Type>Claim (Pydantic model)
   prompt.txt    # Instructions for Gemini extraction
```
2. Register it in `app/claim_types/registry.py`:
```python
   REGISTRY["auto"] = ("app.claim_types.auto.schema", "AutoClaim")
```
3. Restart the server — it's now available for classification and extraction

---

## 9. Snowflake Data Queries

If Snowflake is configured, query stored claims:
```sql
-- View recent claims
SELECT * FROM INSURANCE_FRAUD_DB.CLAIMS.extracted_claims 
ORDER BY created_at DESC LIMIT 10;

-- Count by type
SELECT doc_type, COUNT(*) FROM extracted_claims GROUP BY doc_type;
```