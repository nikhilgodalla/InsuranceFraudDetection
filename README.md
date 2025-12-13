# Insurance Fraud Detection System

A full-stack application for detecting insurance fraud using AI-powered document analysis, graph-based pattern detection, and real-time monitoring.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Flask, Tailwind CSS, Vanilla JavaScript |
| Backend | FastAPI, Google Gemini 2.5-Flash |
| Database | Neo4j (Graph), Snowflake (Persistence) |
| AI/ML | Google GenAI SDK, LLM-based analysis |

## Features

- 📄 **Document Upload & Analysis** - Upload PDF claim documents for automated processing
- 🔍 **Real-time Fraud Detection** - 3-step pipeline (Extract → Graph Ingestion → LLM Analysis)
- 📊 **Interactive Dashboard** - View fraud statistics and trends
- 💬 **AI Chatbot** - Ask questions about the fraud detection system
- 📈 **System Monitoring** - Track LLM usage, costs, and performance metrics
- ✅ **Evaluation Tools** - Assess model performance and accuracy

---

## Project Structure

```
insurance-fraud-detection/
├── frontend/
│   ├── main.py                 # Flask application and API routes
│   ├── templates/              # HTML templates (Jinja2)
│   │   ├── base.html          # Base layout with navigation
│   │   ├── upload.html        # Document upload and analysis
│   │   ├── dashboard.html     # Analytics dashboard
│   │   ├── chat.html          # AI chatbot interface
│   │   ├── monitoring.html    # System monitoring
│   │   └── evaluation.html    # Model evaluation
│   ├── static/                 # Static assets
│   ├── requirements.txt
│   └── .env
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── routes/extract.py
│   │   ├── extractors/extractor.py
│   │   ├── classifiers/doc_type.py
│   │   ├── claim_types/
│   │   │   ├── registry.py
│   │   │   ├── health/
│   │   │   ├── auto/
│   │   │   └── property/
│   │   ├── llm/
│   │   │   ├── client.py
│   │   │   ├── pricing.py
│   │   │   └── usage.py
│   │   ├── db/snowflake_utils.py
│   │   ├── config.py
│   │   └── logging_conf.py
│   ├── requirements.txt
│   └── .env
```

---

## Quick Start

### Prerequisites

- Python 3.11+
- Neo4j database (for fraud detection)
- Gemini API key ([Get one here](https://aistudio.google.com/app/apikey))
- Snowflake account (optional, for persistence)

### 1. Clone the Repository

```bash
git clone https://github.com/your-repo/insurance-fraud-detection.git
cd insurance-fraud-detection
```

### 2. Backend Setup

```bash
cd backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate   # Windows

# Install dependencies
pip install -r requirements.txt
```

Create `backend/.env`:

```env
# Gemini Configuration
USE_VERTEXAI=false
GEMINI_API_KEY=your_api_key_here
GENAI_MODEL=gemini-2.5-flash
MAX_OUTPUT_TOKENS=4096
APP_ENV=dev

# Snowflake (Optional)
SNOWFLAKE_ACCOUNT=your_account
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=INSURANCE_FRAUD_DB
SNOWFLAKE_SCHEMA=CLAIMS
```

Start the backend:

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8080
```

### 3. Frontend Setup

```bash
cd frontend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install flask python-dotenv requests
```

Create `frontend/.env`:

```env
BACKEND_URL=http://localhost:8080
```

Start the frontend:

```bash
python3 main.py
```

### 4. Access the Application

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5001 |
| Backend API | http://localhost:8080 |
| API Docs | http://localhost:8080/docs |

---

## API Reference

### Backend Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/healthz` | GET | Health check |
| `/v1/extract` | POST | Extract and classify document |
| `/v1/fraud/ingest_to_graph` | POST | Ingest data to Neo4j |
| `/v1/fraud/analyze` | POST | LLM-based fraud analysis |
| `/v1/chat` | POST | Chatbot queries |

### Frontend Routes

| Page | URL | Description |
|------|-----|-------------|
| Dashboard | `/` | Fraud statistics and trends |
| Upload & Detect | `/upload` | Upload PDFs and run detection |
| Chatbot | `/chat` | Ask questions about fraud patterns |
| Monitoring | `/monitoring` | System performance and LLM usage |
| Evaluation | `/evaluation` | Model performance metrics |

### Example API Call

```bash
curl -X POST "http://localhost:8080/v1/extract?doc_type=health" \
     -F "file=@claim_document.pdf"
```

Response:

```json
{
  "doc_type": "health",
  "model": "gemini-2.5-flash",
  "result": {
    "company_name": "ACME INSURANCE COMPANY",
    "transaction_id": "TXN-TXN00000001",
    "policyholder_info": {
      "policy_number": "PLC00008468",
      "customer_name": "Christopher Demarest"
    },
    "claim_summary": {
      "amount": "$9,000.00",
      "severity": "Major Loss"
    }
  },
  "usage": {
    "total_tokens": 1427
  },
  "cost_estimate": {
    "total_cost_usd": 0.002254
  }
}
```

---

## Fraud Detection Pipeline

The system uses a 3-step pipeline:

1. **Extract & Classify** - Upload PDF → LLM extracts structured data and classifies document type
2. **Graph Ingestion** - Data ingested into Neo4j → Graph algorithms detect fraud patterns
3. **LLM Analysis** - AI generates detailed fraud analysis with risk scores and recommendations

---

## Database Setup

### Neo4j (Required)

Ensure Neo4j is running for graph-based fraud detection. Configure connection in backend environment.

### Snowflake (Optional)

For persistent storage of extracted claims:

```sql
CREATE DATABASE IF NOT EXISTS INSURANCE_FRAUD_DB;
CREATE SCHEMA IF NOT EXISTS INSURANCE_FRAUD_DB.CLAIMS;

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

---

## Cost Estimation

Gemini 2.5-Flash pricing (as of Nov 2025):

| Type | Rate per 1M tokens |
|------|-------------------|
| Input | $0.30 |
| Output | $2.50 |

---

## Extending Claim Types

To add a new claim type (e.g., `life`):

1. Create folder: `backend/app/claim_types/life/`
2. Add `schema.py` (Pydantic model) and `prompt.txt`
3. Register in `registry.py`:
   ```python
   REGISTRY["life"] = ("app.claim_types.life.schema", "LifeClaim")
   ```
4. Restart the server

---

## Production Deployment

### Docker Compose

```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    env_file: ./backend/.env
    
  frontend:
    build: ./frontend
    ports:
      - "5001:5001"
    environment:
      - BACKEND_URL=http://backend:8080
    depends_on:
      - backend
```

### Using Gunicorn (Frontend)

```bash
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:5001 main:app
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Connection refused" | Ensure backend is running on port 8080 |
| "Graph ingest failed" | Check Neo4j is running with correct credentials |
| "Missing key inputs argument!" | Verify `GEMINI_API_KEY` in backend `.env` |
| "415 Unsupported file type" | Use `multipart/form-data` with file upload |
| "422 Could not classify document" | Pass `doc_type` explicitly (e.g., `?doc_type=health`) |
| Port already in use | Change port in `main.py` or use different port |

---

## System Requirements

- **OS**: macOS, Linux, or Windows
- **Python**: 3.11+
- **RAM**: 4GB minimum (8GB recommended)
- **Browser**: Chrome, Firefox, Safari, or Edge

---

## Links

- **Gemini API Keys**: https://aistudio.google.com/app/apikey
- **Snowflake Signup**: https://signup.snowflake.com/

---

## License

MIT License