# Math AI Tutor

A web-based math tutor that teaches through the Socratic method — guiding students to discover solutions rather than handing them answers. Multi-provider authentication, per-subtopic conversation memory, vision-LLM AutoGrader on PDF assignments, math-native chat UI with live LaTeX rendering, and per-student progress tracking.

> Team research project at Rensselaer Polytechnic Institute. Advisor: Dr. Xiaoyang Liu. See [Acknowledgments](#acknowledgments) for contributors.

---

## What it does

Students sign in (six auth providers — Google, Microsoft, Apple, Phone, Email, Anonymous), choose or upload a textbook, and chat with an OpenAI-backed tutor that walks them through problems Socratically. The tutor keeps per-subtopic memory across sessions, scaffolds against the student's tracked progress, renders math inline, and can grade PDF assignments via vision-LLM.

---

## What makes it interesting

### 1. Tool-using LLM with two-tier conversation memory

The tutor keeps per-textbook, per-subtopic memory in `backend/memory/`. For each turn:

- A **12,000-char rolling summary** of past Q&A on the current subtopic is injected into the system prompt (`MAX_SUMMARY_IN_PROMPT_CHARS` in [`backend/api_routes.py`](backend/api_routes.py)).
- The full verbatim event log is **not** in the prompt. Instead, the model is given an OpenAI tool, `get_subtopic_memory_full`, which it can call when it needs the unabridged history. The tool returns the full event log (capped) and the model continues with richer context.

This is a real use of function calling for retrieval — the model decides when it needs more memory, the system isn't padding every prompt with the entire chat history.

### 2. Multi-provider authentication with Firebase

Six providers via Firebase Auth — Google, Microsoft, Apple, Phone, Email/Password, Anonymous. Backend verifies every protected request via Firebase Admin SDK ID-token verification ([`backend/auth.py`](backend/auth.py)). Frontend manages auth state via a React context provider ([`frontend/src/context/AuthContext.tsx`](frontend/src/context/AuthContext.tsx)). When `FIREBASE_SERVICE_ACCOUNT` / `VITE_FIREBASE_*` env vars are unset, auth disables gracefully and the app remains usable.

### 3. Vision-LLM AutoGrader (modular pipeline)

[`backend/AutoGrader/`](backend/AutoGrader/) accepts a problem PDF + an answer PDF. The pipeline:

1. Splits the problem PDF into individual questions (`question_splitter.py`).
2. Pairs each question with the relevant section of the answer PDF.
3. Renders each pair to page images (PyMuPDF + Pillow).
4. Scores each pair via a vision LLM — sees handwritten work, diagrams, and crossed-out steps the way a human grader does.
5. Returns per-question + overall grades.

Two entry points: `POST /api/grade` (lightweight) and `POST /api/autograder/grade` (full pipeline). Vision-LLM-on-page-images is the right primitive for graded math work — OCR loses too much structural information.

### 4. Per-user textbook ingestion + per-student progress

- [`backend/user_textbook_store.py`](backend/user_textbook_store.py) — per-user uploaded textbooks (PDF + FOCS-tree outline JSON + metadata).
- [`backend/student_bar_store.py`](backend/student_bar_store.py) — per-student learning-bar tracking which curriculum sections they've engaged with, auto-updated from message content via token matching.

### 5. Math-native chat UI

React 19 + TypeScript + Vite frontend. LaTeX math renders inline via `react-katex` + `remark-math` + `rehype-katex`. Markdown rendering via `react-markdown` + GFM. Students can type `$\int x^2 dx$` and see it rendered live in the conversation.

### 6. Persistent chat-history with auth-protected sessions

[`frontend/src/ChatHistory.tsx`](frontend/src/ChatHistory.tsx) provides session management — list, resume, delete past chats — gated by Firebase ID-token verification. Sessions persist in MongoDB Atlas ([`backend/database.py`](backend/database.py)).

---

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Frontend  (React 19 + TypeScript + Vite + Firebase SDK)       │
│                                                                │
│  - Home.tsx, Chat (via App.tsx)                                │
│  - SignInModal.tsx   six-provider auth UI                      │
│  - ChatHistory.tsx   auth-protected session list/resume        │
│  - AutoGrader.tsx    PDF upload + per-question grades          │
│  - LearningBarPanel  per-student progress display              │
│  - UserProfile.tsx   profile + settings                        │
│  - UploadTextbook    bind textbook PDF to the session          │
│  - MathText.tsx      LaTeX-in-markdown rendering               │
│  - context/          AuthContext, CurriculumContext,           │
│                      ProfileSettingsContext                    │
└────────────────────────────────────────────────────────────────┘
                            │
                            ▼  REST (axios, Bearer token)
┌────────────────────────────────────────────────────────────────┐
│  Backend  (FastAPI + OpenAI 1.109 + Firebase Admin SDK)        │
│                                                                │
│  - main.py            FastAPI app + CORS + DB init             │
│  - api_routes.py      17 endpoints (chat, grade, sessions,     │
│                       textbooks, student bar, ...)             │
│  - auth.py            Firebase ID-token verification           │
│  - database.py        MongoDB Atlas client + collections       │
│  - AutoGrader/        Modular grading pipeline (splitter +     │
│                       grader + models + service)               │
│  - memory/            per-subtopic event store + Memory        │
│                       protocol                                 │
│  - student_bar_store  per-student learning-bar JSON            │
│  - user_textbook_store per-user textbooks (PDF + outline)      │
│  - learning_resources textbook ingestion helpers               │
└────────────────────────────────────────────────────────────────┘
                            │                  │
                            ▼                  ▼
                ┌──────────────────┐  ┌────────────────────────┐
                │  OpenAI Chat API │  │  MongoDB Atlas         │
                │  (function call) │  │  + Firebase Auth       │
                └──────────────────┘  └────────────────────────┘
```

---

## Tech stack

**Frontend** — React 19, TypeScript, Vite 7, react-router-dom 7, Firebase 12 (client SDK), KaTeX (`react-katex`, `remark-math`, `rehype-katex`), `react-markdown` + `remark-gfm`, axios.

**Backend** — Python, FastAPI, OpenAI Python SDK (function calling), Firebase Admin SDK, PyMongo (MongoDB Atlas), PyMuPDF + Pillow (PDF page-image rendering for AutoGrader), python-dotenv, uvicorn.

**Storage** — MongoDB Atlas for chat sessions + user data; local file system under `backend/data/` for per-user textbooks + per-student learning bars + per-subtopic memory event logs.

**Auth** — Firebase Auth (Google, Microsoft, Apple, Phone, Email/Password, Anonymous); backend verifies ID tokens via Firebase Admin SDK on every protected endpoint.

---

## Quick start

### Prerequisites

- Node.js 18+
- Python 3.9+
- OpenAI API key
- (Optional) MongoDB Atlas URI for persistence beyond a single process
- (Optional) Firebase project for auth — the app runs without it, auth just disables

### Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env               # then edit and add:
#   API_KEY=sk-...your-openai-key...
#   CORS_ORIGINS=http://localhost:5173
#   MONGODB_URI=mongodb+srv://...   (optional)
#   FIREBASE_SERVICE_ACCOUNT='{...service-account-json...}'  (optional)

python main.py                     # serves on http://localhost:8000
```

### Frontend

```bash
cd frontend
npm install

cp .env.example .env.local         # then edit and add (optional, for auth):
#   VITE_FIREBASE_API_KEY=...
#   VITE_FIREBASE_AUTH_DOMAIN=...
#   VITE_FIREBASE_PROJECT_ID=...

npm run dev                        # serves on http://localhost:5173
```

Open `http://localhost:5173`. The chat UI talks to `http://localhost:8000`.

---

## Project structure

```
.
├── backend/
│   ├── main.py                 FastAPI entrypoint + CORS + DB init
│   ├── api_routes.py           17 endpoints (chat, grade, sessions, textbooks, ...)
│   ├── auth.py                 Firebase ID-token verification
│   ├── database.py             MongoDB Atlas connection module
│   ├── deps.py                 shared dependency objects
│   ├── memory/
│   │   ├── memory.py           Memory protocol + open_memory()
│   │   ├── stores/             per-subtopic event/summary files
│   │   └── test.py             memory-store unit tests
│   ├── student_bar_store.py    per-student progress JSON
│   ├── user_textbook_store.py  per-user uploaded textbooks + outlines
│   ├── learning_resources.py   textbook ingestion helpers
│   ├── AutoGrader/             modular grading pipeline
│   │   ├── question_splitter.py
│   │   ├── grader.py
│   │   ├── service.py
│   │   ├── models.py
│   │   ├── public_api.py
│   │   └── tests + design docs
│   ├── data/                   bundled textbooks, learning-tree token map
│   └── requirements.txt
└── frontend/
    ├── src/
    │   ├── App.tsx, Home.tsx              shell + landing
    │   ├── Chat.css, MarkdownMessage.tsx  chat UI + markdown
    │   ├── MathText.tsx                   LaTeX-in-markdown renderer
    │   ├── SignInModal.tsx                six-provider auth UI
    │   ├── ChatHistory.tsx                session list / resume
    │   ├── AutoGrader.tsx                 grading workflow
    │   ├── LearningModel.tsx, MyLearningBar.tsx, LearningBarPanel.tsx
    │   ├── UserProfile.tsx                user profile + settings
    │   ├── UploadTextbook.tsx             textbook upload
    │   ├── firebase.ts                    Firebase SDK init
    │   ├── api.ts, apiBase.ts             axios client + base URL
    │   └── context/                       Auth, Curriculum, ProfileSettings
    ├── package.json
    └── vite.config.ts
```

---

## Design notes

- **Why tool-calling for memory, not RAG?** The corpus per session is tiny (one student's history on one subtopic). Embedding everything and doing semantic retrieval is more machinery than the problem needs. A tool the model can call exactly when it needs more memory is simpler, cheaper, and doesn't require an embedding model in the loop.
- **Why page images and not OCR for grading?** Handwritten math work has structure OCR doesn't preserve — crossed-out steps, drawn diagrams, marginalia. Vision LLMs read what's on the page, full stop.
- **Why six auth providers?** Different student populations have different account preferences. Anonymous lets visitors try the tutor without commitment. Phone/Email/Apple matter for students who don't use Google/Microsoft accounts.
- **Why both file-system and MongoDB persistence?** File-system is great for static per-user assets (uploaded textbooks, learning-bar JSONs). MongoDB handles session state where queryability matters (resume any past chat). Hybrid is honest to the actual data-access patterns.

---

## Status

Research project, in active development. Not production-deployed. Suitable for local development and study use.

---

## Acknowledgments

This is a team project. Primary contributors (by commit volume on the current `function` branch):

- **[@JinBoatus1](https://github.com/JinBoatus1)** — project lead and primary maintainer; tutor logic (`api_routes.py`), per-student progress tracking (`student_bar_store.py`), per-user textbook handling (`user_textbook_store.py`), AutoGrader frontend (`AutoGrader.tsx`), Home/Chat UI, multiple UI refresh rounds.
- **[@lius24 (Shijun Liu)](https://github.com/lius24)** — multi-provider Firebase authentication ([`backend/auth.py`](backend/auth.py) + [`frontend/src/context/AuthContext.tsx`](frontend/src/context/AuthContext.tsx) + [`SignInModal.tsx`](frontend/src/SignInModal.tsx)), MongoDB Atlas integration ([`backend/database.py`](backend/database.py)), auth-protected chat-history UI ([`ChatHistory.tsx`](frontend/src/ChatHistory.tsx)).
- **[@leon-wulin](https://github.com/leon-wulin)** — AutoGrader pipeline ([`backend/AutoGrader/`](backend/AutoGrader/) including `question_splitter.py`, `grader.py`, `service.py`).
- **[@lin](https://github.com/lin)** — additional UI + textbook integration work.

Advisor: **Dr. Xiaoyang Liu**, RPI.

## License

MIT — see [LICENSE](LICENSE).
