# CareerCoach AI

An AI-powered career coaching platform that helps software engineers 
prepare for job interviews by analysing their resume against a job 
description and conducting personalised mock interviews with voice support.

> **Status:** In Progress — backend complete, frontend in development

---

## What it does

1. **Resume Analysis** — Upload your PDF resume and paste a job description.
   The platform extracts skills from both and identifies gaps.

2. **Mock Interview** — Based on the analysis, AI generates personalised
   interview questions (behavioural, technical, situational). Answer by
   voice or text.

3. **Feedback** — Each answer is evaluated and scored. A session summary
   shows strengths, weaknesses, and suggested improvements.

---

## Architecture
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   React Frontend │────▶│  FastAPI Backend  │────▶│  Java Parser    │
│   (Vite, port   │     │  (Python, port   │     │  (Spring Boot,  │
│    5173)        │     │   8000)          │     │   port 8001)    │
└─────────────────┘     └──────────────────┘     └─────────────────┘
│
▼
┌──────────────────┐
│   PostgreSQL DB   │
│  (job_queue,     │
│   resumes,       │
│   users, ...)    │
└──────────────────┘

**FastAPI** handles authentication, resume storage, job queue management,
and coordinates between services.

**Java Spring Boot** microservice handles PDF text extraction using
Apache PDFBox 3.x. Kept separate for fault isolation — a crash in the
parser doesn't affect auth or interview endpoints.

**Background Worker** runs alongside FastAPI, polling the job queue every
5 seconds. When a parse_resume job appears, it calls the Java service,
gets extracted text, and updates the resume record.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, React Router, Axios |
| Backend | Python 3.13, FastAPI, SQLAlchemy (async), Pydantic |
| Parser | Java 21, Spring Boot 3.3.5, Apache PDFBox 3.0.3 |
| Database | PostgreSQL, Alembic migrations |
| Auth | JWT (access + refresh tokens), bcrypt |
| Async | asyncio background worker, job queue pattern |
| DevOps | Docker, Docker Compose |

---

## Key Engineering Decisions

**Why a separate Java service for PDF parsing?**
Fault isolation. If the parser runs out of memory on a large PDF, it
doesn't bring down the auth service or the interview endpoints. Each
service can be scaled and deployed independently.

**Why a job queue instead of parsing synchronously?**
PDF parsing can take 2-5 seconds. Blocking the HTTP request during that
time gives a poor user experience. The upload endpoint returns immediately
with a 201, and the worker processes the job in the background. The
frontend can poll for completion.

**Why pessimistic locking on the job queue?**
The worker marks a job as 'processing' before doing any work. If two
worker instances run simultaneously, the second one sees 'processing'
and skips it. This prevents duplicate processing without needing
distributed locks.

---

## Running Locally

**Prerequisites:** Docker, Docker Compose

```bash
# Clone the repo
git clone https://github.com/ichetansethi/careercoach-ai
cd careercoach-ai

# Copy environment variables
cp careercoach-api/.env.example careercoach-api/.env
# Edit .env and set JWT_SECRET_KEY

# Start everything
docker compose up --build

# Run database migrations
docker compose run --rm migrate

# Frontend (separate terminal)
cd careercoach-frontend
npm install
npm run dev
```

**Services:**
- Frontend: http://localhost:5173
- FastAPI docs: http://localhost:8000/docs
- Java parser health: http://localhost:8001/api/health

---

## Project Structure
careercoach-ai/
├── careercoach-api/          # FastAPI backend
│   ├── app/
│   │   ├── routers/          # auth, resumes, interviews
│   │   ├── models/           # SQLAlchemy ORM models
│   │   ├── schemas/          # Pydantic request/response schemas
│   │   ├── workers/          # async job queue worker
│   │   └── core/             # config, database, security
│   └── tests/                # 32 passing tests
├── resume-parser/            # Java Spring Boot PDF parser
│   └── src/main/java/com/careercoach/parser/
│       ├── controller/       # REST endpoint
│       └── service/          # PDFBox extraction logic
└── careercoach-frontend/     # React frontend
└── src/
├── pages/            # Login, Dashboard, Upload, Results, Interview
├── components/       # Layout (sidebar + topbar)
└── api/              # axios client with auth interceptor

---

## Tests

```bash
cd careercoach-api
pytest tests/
# 32 passed
```

Tests use SQLite in-memory with type shims for PostgreSQL-specific
columns (JSONB, UUID). No live database required to run tests.

---

## What's next

- [ ] LLM integration for question generation and answer evaluation
- [ ] NLP skill extractor using spaCy
- [ ] Docker Compose production configuration
- [ ] CI/CD pipeline with GitHub Actions
