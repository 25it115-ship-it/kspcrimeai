# kspcrimeai
 Intelligent Conversational AI for Karnataka State Police (KSP) Crime Database

A production-ready, full-stack web application that lets authorized KSP personnel
query the crime database in natural language (English, Kannada, Hindi) and get
accurate, source-grounded answers — alongside a full case-management suite
(FIRs, crime records, missing persons, stolen vehicles), analytics dashboards,
and admin/audit tooling.

## Tech stack

| Layer       | Technology |
|-------------|------------|
| Frontend    | Next.js 14 (App Router), React 18, Tailwind CSS, Recharts, Web Speech API |
| Backend     | FastAPI, SQLAlchemy, PostgreSQL, JWT auth, Pydantic |
| Generative AI | Anthropic Claude API, SentenceTransformers embeddings, FAISS vector store, NL2SQL, RAG |
| Infra       | Docker & Docker Compose |

## Project structure

```
ksp-crime-ai/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI app entrypoint
│   │   ├── config.py, database.py, security.py, dependencies.py
│   │   ├── models/                 # SQLAlchemy models
│   │   ├── schemas/                # Pydantic request/response schemas
│   │   ├── routers/                # auth, crime_records, fir, missing_persons,
│   │   │                           #   stolen_vehicles, analytics, chatbot, admin,
│   │   │                           #   notifications, reports
│   │   ├── ai/                     # claude_client, prompts, embeddings,
│   │   │                           #   rag_engine (FAISS), nl2sql, conversation_memory,
│   │   │                           #   orchestrator (ties it all together)
│   │   ├── utils/                  # audit logging, validators, encryption
│   │   └── seed_data.py            # demo users + sample crime data + RAG index build
│   ├── requirements.txt, Dockerfile, .env.example
├── frontend/
│   ├── src/app/                    # login, register, dashboard, chat, records, fir,
│   │                                #   missing-persons, stolen-vehicles, admin
│   ├── src/components/             # Sidebar, AppShell, MessageBubble, VoiceInput,
│   │                                #   FileUploadFIR
│   ├── src/context/                # AuthContext, ThemeContext (dark/light)
│   ├── src/lib/api.js              # axios client with JWT interceptor
│   ├── package.json, Dockerfile, .env.local.example
├── docs/
│   ├── API.md                      # endpoint reference
│   └── ARCHITECTURE.md             # system design & data flow
├── docker-compose.yml
└── README.md
```

## Quick start (Docker — recommended)

```bash
cd ksp-crime-ai

# 1. Configure secrets
cp backend/.env.example backend/.env
# edit backend/.env and set ANTHROPIC_API_KEY (required for AI features)

# 2. Launch the stack
docker compose up --build

# 3. Seed demo data (first run only)
docker compose exec backend python -m app.seed_data
```

- Frontend: http://localhost:3000
- Backend API + Swagger docs: http://localhost:8000/docs
- Postgres: localhost:5432 (user `ksp_admin` / db `ksp_crime_db`)

## Manual local development (without Docker)

### Backend
```bash
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env   # set ANTHROPIC_API_KEY and a local DATABASE_URL

# Make sure a local Postgres is running with a `ksp_crime_db` database, then:
python -m app.seed_data          # creates tables + demo data + vector index
uvicorn app.main:app --reload --port 8000
```

### Frontend
```bash
cd frontend
npm install
cp .env.local.example .env.local   # points NEXT_PUBLIC_API_URL at the backend
npm run dev
```

## Demo credentials (created by `seed_data.py`)

| Role | Email | Password | Scope |
|------|-------|----------|-------|
| Admin | admin@ksp.gov.in | Admin@12345 | State-wide, full access |
| Supervisor | supervisor@ksp.gov.in | Super@12345 | State-wide, read/write |
| Analyst | analyst@ksp.gov.in | Analyst@12345 | State-wide, read-only analytics |
| Investigator | officer1@ksp.gov.in … officer5@ksp.gov.in | Officer@12345 | District-scoped |

**Change all demo passwords before any real deployment.**

## Try the chatbot

Once logged in, open **AI Chatbot** in the sidebar and try:
- "Show theft cases in Bengaluru"
- "Who is investigating FIR 4587?"
- "Cybercrime statistics for 2025"
- "List open investigations in Mysuru"
- Switch the language selector to ಕನ್ನಡ or हिन्दी and ask the same questions.

Each answer is labeled **Database Fact**, **AI Summary**, or **No Result / Restricted**,
and you can expand "sources" to see the exact SQL query or retrieved document
snippets that grounded the response — nothing is presented as fact unless it
came from the database.

## Key AI/RAG design decisions

- **NL2SQL first, RAG second**: exact, statistical, or lookup-style questions are
  answered with a validated, read-only SQL query for zero-hallucination accuracy;
  fuzzy/descriptive questions fall back to FAISS semantic search over indexed
  FIR/crime text, then Claude composes the final answer *strictly* from what was
  retrieved (never from its own general knowledge).
- **Authorization-aware retrieval**: both the SQL generation and the semantic
  search are scoped to the caller's district unless their role is state-wide, so
  the AI cannot leak another district's case data even if asked directly.
- **SQL safety**: generated SQL must be a single `SELECT` against a whitelisted
  table/column set with no DDL/DML keywords, or it's rejected before execution.
- See `docs/ARCHITECTURE.md` for the full request lifecycle and data model.

## Security features

JWT authentication · bcrypt password hashing · role-based access control (5 roles,
enforced server-side) · Fernet field-level encryption for sensitive columns ·
immutable audit log on all writes/AI queries/exports · Pydantic input validation ·
SQL-injection-safe NL2SQL pipeline · CORS restricted to configured origins.

## Notes & next steps for a real deployment

- Swap `Base.metadata.create_all()` for Alembic migrations.
- Put the interactive crime-hotspot map behind a real GIS/Maps API key
  (`/api/analytics/hotspots` already returns the lat/lng + severity payload).
- Move file uploads and the FAISS index to persistent/networked storage (already
  Docker-volume-backed in `docker-compose.yml`) for multi-instance deployments.
- Rotate `JWT_SECRET_KEY` and `FIELD_ENCRYPTION_KEY` and store them in a secrets
  manager, not `.env`, in production.
