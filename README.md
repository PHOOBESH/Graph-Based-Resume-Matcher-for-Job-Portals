# 🔗 Graph-Based Resume Matcher for Job Portals

**AI-powered resume-to-job matching using a live skill knowledge graph.**  
Candidates upload resumes, recruiters post job descriptions — the system uses Gemini AI to extract skills from both, builds a dynamic ontology in Neo4j, and performs **weighted graph traversal** to produce explainable, ranked matches.

> **Disclaimer:** This tool is intended for research, education, and recruitment-tech development. It does not constitute professional HR or legal advice.

---

## ✨ Features

- **Dual-role system** — separate flows for candidates (resume upload) and recruiters (JD posting), protected by login with bcrypt-hashed passwords
- **Gemini 2.5 Flash parsing** — extracts structured data (name, email, skills, experience, projects) from raw PDF resumes and free-text job descriptions
- **Live Neo4j knowledge graph** — resumes, jobs, and skills are stored as graph nodes with typed relationships (`HAS`, `REQUIRES`, `RELATED_TO`, `IS_A`)
- **Dynamic skill ontology** — after every upload, Gemini automatically expands the graph by generating `IS_A` and `RELATED_TO` links for each new skill (e.g., Flask → IS_A → Web Framework, Flask → RELATED_TO → Django)
- **Weighted graph matching** — scoring uses direct skill hits (×1.0) and one-hop ontology matches (×0.5), so a candidate who knows PyTorch can partially match a TensorFlow role
- **XAI explanation engine** — the `/explain_match` endpoint traces shortest paths in the graph and returns human-readable explanations of why each match was made
- **Skill Graph Explorer** — interactive frontend for exploring any skill's ontology neighbourhood
- **Metrics evaluator** — `metrics_evaluator.py` computes Precision, Recall, F1, Accuracy, Skill Expansion Rate (SER), Semantic Match Rate (SMR), Ontology Coverage (OC), and ΔNDCG
- **Resume PDF download** — raw PDFs are stored in MongoDB GridFS and streamable on demand

---

## 🏗️ Architecture

The diagram above shows the full pipeline. Briefly:

1. **Candidate** uploads a PDF → FastAPI calls Gemini to parse it → normalized data saved to MongoDB → Neo4j synced → ontology auto-expanded → weighted job recommendations returned
2. **Recruiter** pastes a JD → FastAPI calls Gemini to extract required skills → saved to MongoDB → Neo4j synced → ontology auto-expanded → ranked eligible applicants returned
3. **Neo4j graph core** holds `Resume`, `Job`, and `Skill` nodes; the ontology builder populates `RELATED_TO`/`IS_A` edges that power the 1-hop expanded matching

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Backend API | Python 3.11+, FastAPI |
| AI / LLM | Google Gemini 2.5 Flash (`gemini-2.5-flash`) |
| Graph Database | Neo4j (Bolt protocol) |
| Document Database | MongoDB + GridFS |
| Embeddings (eval) | `sentence-transformers` — `all-MiniLM-L6-v2` |
| Frontend | React 19, React Router v7, Axios |
| Auth | bcrypt via `passlib` |
| PDF Parsing | PyMuPDF (`fitz`) |

---

## 📂 Project Structure

```
.
├── backend/
│   ├── main.py                     # FastAPI app — all routes and core logic
│   ├── graph.py                    # Standalone Neo4j graph sync utilities
│   ├── resume_parser.py            # Sentence-transformer-based resume parser (legacy)
│   ├── resume_parser_llm.py        # Gemini-based resume parser (standalone script)
│   ├── job_description_extract.py  # MongoDB connection check utility
│   ├── job_description_extract_llm.py  # Gemini JD extractor (standalone script)
│   ├── classify.py                 # Ontology evaluation against synthetic ground truth
│   ├── metrics_evaluator.py        # Full metrics suite (P/R/F1, SER, SMR, OC, NDCG)
│   └── start_backend.bat           # Windows helper to launch uvicorn
├── resume-analyzer-frontend/
│   ├── src/
│   │   ├── App.js                  # React Router entry point
│   │   ├── api.js                  # Axios base URL config (http://127.0.0.1:8000)
│   │   └── pages/
│   │       ├── Login.js            # Login page (admin / user roles)
│   │       ├── Signup.js           # User registration
│   │       ├── ParseResume.js      # Candidate dashboard — upload + view recommendations
│   │       ├── ResumeDisplay.js    # Parsed resume viewer
│   │       ├── ExtractJD.js        # Recruiter dashboard — post JD + view applicants
│   │       ├── JobRecommendationCard.js  # Reusable job card component
│   │       ├── SkillGraphExplorer.js     # Interactive ontology explorer
│   │       └── Home.js             # Home page
│   └── package.json
├── start.bat                       # Windows: starts both backend and frontend
├── start_frontend.bat              # Windows: starts frontend only
├── .env                            # API keys (not committed)
└── package.json                    # Root JS dependencies (axios, react-router-dom)
```

---

## ⚙️ Setup & Installation

### Prerequisites

- Python 3.11+
- Node.js 18+ and npm
- A running Neo4j instance (local or AuraDB)
- A running MongoDB instance (local or Atlas)
- A Google Gemini API key — get one at [aistudio.google.com](https://aistudio.google.com)

---

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/Graph-Based-Resume-Matcher-for-Job-Portals.git
cd Graph-Based-Resume-Matcher-for-Job-Portals
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
GEMINI_API_KEY=your_google_gemini_api_key_here

# MongoDB (defaults shown)
MONGO_URI=mongodb://localhost:27017

# Neo4j (defaults shown)
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_neo4j_password_here
```

> **Security note:** Never commit your `.env` file. It is already listed in `.gitignore`.

### 3. Set up the Python backend

```bash
cd backend

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate       # macOS/Linux
# .\venv\Scripts\activate      # Windows

# Install dependencies
pip install fastapi uvicorn pymongo motor gridfs \
            python-multipart passlib[bcrypt] \
            google-generativeai python-dotenv \
            PyMuPDF sentence-transformers \
            neo4j prettytable scikit-learn numpy pandas
```

### 4. Set up the React frontend

```bash
cd resume-analyzer-frontend
npm install
```

---

## 🚀 Running the Application

### Start the backend

```bash
cd backend
uvicorn main:app --reload --port 8000
```

The API will be available at `http://127.0.0.1:8000`. Interactive docs at `http://127.0.0.1:8000/docs`.

### Start the frontend

```bash
cd resume-analyzer-frontend
npm start
```

The React app will open at `http://localhost:3000`.

### Windows shortcut

From the project root, double-click or run:

```bat
start.bat
```

---

## 👤 User Roles & Login

| Username | Password | Role | Landing page |
|---|---|---|---|
| `admin` | `admin` | Recruiter | `/extract_jd_skills` — post JDs, view applicants |
| Any registered user | their password | Candidate | `/parse_resume` — upload resume, view job recommendations |

New candidates can self-register via `/signup`.

---

## 🔌 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/signup/` | Register a new user |
| `POST` | `/login/` | Authenticate; returns role + redirect path |
| `POST` | `/parse_resume/` | Upload PDF resume; returns parsed data + job recommendations |
| `GET` | `/my_resume/` | Retrieve current user's saved resume |
| `DELETE` | `/my_resume/` | Delete current user's resume |
| `POST` | `/extract_jd_skills/` | Post a job description; returns extracted skills + eligible applicants |
| `GET` | `/recommend_jobs/` | Get job recommendations for a resume (`?resume_id=...&mode=expanded`) |
| `GET` | `/eligible_applicants/` | Get ranked applicants for a job (`?job_id=...`) |
| `GET` | `/download_resume/{file_id}` | Stream the raw PDF from GridFS |
| `POST` | `/ontology/expand` | Manually trigger ontology expansion for a skill list |
| `POST` | `/ontology/rebuild` | Rebuild the full skill ontology from scratch |
| `GET` | `/ontology/explore` | Explore a skill's graph neighbourhood (`?skill=Python`) |
| `GET` | `/explain_match/` | XAI path explanation for a resume-job pair (`?resume_id=...&job_id=...`) |

---

## 🧠 How Graph Matching Works

### Graph schema

```
(:Resume)-[:HAS]->(:Skill)
(:Job)-[:REQUIRES]->(:Skill)
(:Skill)-[:IS_A]->(:Skill)          # e.g. Flask IS_A Web Framework
(:Skill)-[:RELATED_TO]->(:Skill)    # e.g. Flask RELATED_TO Django
```

### Weighted scoring

For each resume-job pair, the system queries the graph for:

- **Direct matches** — candidate's skill node is identical to a required skill (score weight: 1.0)
- **Related matches** — candidate's skill is 1 hop away via `RELATED_TO` or `IS_A` from a required skill (score weight: 0.5)

```
weightedScore = (directMatches × 1.0) + (relatedMatches × 0.5)
```

This means a candidate who knows PyTorch is partially credited when a job requires TensorFlow, because the ontology links them as related ML frameworks.

### Ontology expansion

After every resume or JD upload, the system calls Gemini once per new skill to generate up to 5 ontology relations. Relations with confidence < 0.6 are discarded. This builds the graph incrementally without requiring a pre-built taxonomy.

---

## 📊 Evaluation & Metrics

`metrics_evaluator.py` connects directly to your Neo4j instance and computes:

| Metric | Description |
|---|---|
| Precision / Recall / F1 | Standard classification metrics for direct vs expanded matching |
| Accuracy | Per-mode match accuracy |
| SER (Skill Expansion Rate) | Average ratio of ontology-related skills to explicit skills per resume |
| SMR (Semantic Match Rate) | Proportion of job matches contributed by ontology (not direct) links |
| OC (Ontology Coverage) | Share of all skill nodes that have at least one outgoing relation |
| ΔNDCG | Improvement in ranking quality from direct to expanded scoring |

`classify.py` benchmarks Gemini's ontology generation against a synthetic ground truth CSV using semantic similarity (`all-MiniLM-L6-v2`, threshold 0.7), reporting per-skill and aggregated Precision, Recall, F1.

To run the evaluator:

```bash
cd backend
python metrics_evaluator.py
```

> **Note:** `metrics_evaluator.py` requires `(:Resume)-[:ACTUAL_MATCH]->(:Job)` ground truth edges in Neo4j. Seed these manually for your test dataset.

---

## 🛣️ Roadmap

- [ ] Async ontology expansion (background task queue) to avoid blocking API responses
- [ ] JWT-based authentication to replace session-less form auth
- [ ] Resume file upload support for DOCX and TXT formats
- [ ] Recruiter dashboard with job management (edit/delete postings)
- [ ] Expand ontology to BNS/BNSS-adjacent domains (legal, finance, healthcare)
- [ ] Visualize the skill graph directly in the browser using D3.js or Cytoscape.js
- [ ] Add feedback loop — let recruiters confirm/reject matches to fine-tune scoring weights
- [ ] Dockerize the full stack (backend + MongoDB + Neo4j)
- [ ] Deploy to cloud (Railway / Render / AWS) with environment-based config

---

## 🙏 Acknowledgements

- [Google Gemini](https://ai.google.dev/) — LLM for structured parsing and ontology generation
- [Neo4j](https://neo4j.com/) — graph database powering skill relationship traversal
- [MongoDB](https://www.mongodb.com/) — document store and GridFS for PDF storage
- [sentence-transformers](https://www.sbert.net/) — semantic similarity for evaluation

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
