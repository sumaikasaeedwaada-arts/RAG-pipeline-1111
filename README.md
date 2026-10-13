# AI Business Assistant

An authenticated FastAPI business assistant with conversation history, document ingestion, semantic retrieval, and source citations. The browser UI is served by the API, and Gemini is configured through environment variables.

## Architecture

```text
.
├── AI_service/
│   ├── api.py              # New authenticated FastAPI/RAG application
│   ├── database.py         # SQLite schema and transactional connections
│   ├── rag.py              # Parsers, chunking, embeddings, vector retrieval
│   ├── security.py         # Password hashing and signed bearer tokens
│   ├── settings.py         # Environment-based service configuration
│   ├── ai_service.py       # Gemini chat client and legacy CLI
│   ├── app.py              # Legacy /ask endpoint
│   ├── chat_api.py         # Legacy /chat endpoint
│   └── static/             # Responsive browser client
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── README.md
```

```text
Browser UI (HTML/CSS/JS)
        │ same-origin bearer-token requests
        ▼
FastAPI ── auth + validation + SQLite-backed rate limits
   ├── SQLite: users, conversations, messages, documents, chunks, vectors
   ├── RAG: PDF/DOCX/TXT parsing → overlapping chunks → Gemini embeddings
   │       → user-scoped cosine similarity search → numbered citations
   └── Gemini LLM: history + retrieved source context → grounded answer
```

### Database schema

| Table | Important fields and relations |
|---|---|
| `users` | `id`, unique case-insensitive `email`, PBKDF2 `password_hash`, `created_at` |
| `conversations` | `id`, `user_id → users.id`, `title`, `created_at`, `updated_at` |
| `messages` | `id`, `conversation_id → conversations.id`, `role`, `content`, `citations_json`, `created_at` |
| `documents` | `id`, `user_id → users.id`, `filename`, `media_type`, `status`, `created_at` |
| `chunks` | `document_id → documents.id`, `chunk_index`, `page_number`, `content`, `embedding_json` |
| `rate_limit_buckets` | Shared SQLite fixed-window request counters keyed by client and route class |

SQLite provides durable storage and exact cosine vector search for a single-node or modest dataset deployment. It scans the requesting user's chunks; larger corpora or multi-node deployments should migrate this table/search operation to a managed vector database. The rate limiter shares counters among workers that use the same SQLite file.

## API

Interactive OpenAPI documentation is available at `/docs`; the schema is at `/openapi.json`.

| Method | Path | Auth | Purpose |
|---|---|---:|---|
| `GET` | `/api/health` | No | Health probe |
| `POST` | `/api/auth/register` | No | Register (password minimum 10 characters) |
| `POST` | `/api/auth/login` | No | Login; returns a bearer access token |
| `GET` | `/api/auth/me` | Yes | Current account |
| `GET`, `POST` | `/api/conversations` | Yes | List or create conversations |
| `GET` | `/api/conversations/{id}/messages` | Yes | Read owned conversation history |
| `POST` | `/api/conversations/{id}/messages` | Yes | Ask a question; returns answer and cited sources |
| `DELETE` | `/api/conversations/{id}` | Yes | Delete an owned conversation and its messages |
| `GET`, `POST` | `/api/documents` | Yes | List or upload PDF/DOCX/TXT files (8 MB max) |
| `DELETE` | `/api/documents/{id}` | Yes | Delete an owned document and its chunks |

Authenticated endpoints expect `Authorization: Bearer <access_token>`. The API rate limits each client IP to 10 auth requests/minute and 120 other API requests/minute. Configure trusted proxy handling at the ingress if deployed behind a reverse proxy.

## Run locally

Requires Python 3.11+ and a Gemini API key for actual chat and embedding requests.

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Edit `.env` and set `GEMINI_API_KEY` and a long, random `AUTH_SECRET`. Never commit `.env`; it is excluded by `.gitignore`. Set environment variables in the current PowerShell session and start the app:

```powershell
$env:GEMINI_API_KEY = "your-key"
$env:AUTH_SECRET = "a-long-random-secret"
$env:DATABASE_PATH = "data/assistant.db"
uvicorn AI_service.api:app --reload --env-file .env
```

Open <http://127.0.0.1:8000>. Create an account, upload a file, and ask a question whose answer appears in it. The service returns an explicit 503 when the Gemini key/provider is unavailable; it does not silently generate embeddings or fabricated answers.

## Tests

```powershell
python -m unittest AI_service.test_chat_api AI_service.test_rag_api -v
```

Tests mock model calls and use a temporary SQLite database; they do not require a real provider key.

## Docker deployment

Create a local `.env` based on `.env.example`, set the actual key and a cryptographically random `AUTH_SECRET`, then run:

```powershell
docker compose up --build -d
docker compose logs -f assistant
```

The database persists in the `assistant-data` volume. Back up that volume regularly. Terminate TLS at a trusted reverse proxy, restrict direct access to the container port, use a stable high-entropy `AUTH_SECRET` shared by all workers, rotate provider credentials via environment/secret management, and monitor `/api/health` and application logs. Configure proxy IP forwarding only for a trusted ingress. Do not expose the SQLite volume publicly.

## Security and operational notes

- Passwords are PBKDF2-HMAC-SHA256 hashes with per-user random salts; bearer tokens are HMAC-signed and expire after one hour by default.
- Set `AUTH_SECRET` in deployment. When omitted in local development, a per-process random key is used and a warning is logged; tokens then do not survive restarts or work across workers.
- Conversation and document queries are scoped to the authenticated user. Uploaded files are parsed in memory and their text/vector data is persisted; original upload bytes are not retained.
- Upload size, supported suffix, empty content, chunk count, and request field lengths are bounded and validated. Avoid logging uploaded document bodies, passwords, tokens, or provider credentials.
- SQLite-backed exact vector search is appropriate as a straightforward durable baseline, not a replacement for ANN indexes at large scale.Read skill `python-fact-grounded-coding`

I’ll first inspect the existing starter files, then lay out the architecture/schema/API and implementation sequence before making the changes.

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\chatbot.py"
}

Find files matching `**/*`

pylanceWorkspaceRoots
Completed with input: {
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

## Architecture proposal

The starter is a small Python CLI rather than an existing web app. I’ll evolve it into a single FastAPI service with a lightweight responsive browser UI. For an implementable, locally runnable baseline: SQLite stores users, conversations, messages, documents, and chunk metadata; local persistent Chroma stores vectors; an environment-selected LLM/embedding provider keeps credentials out of source. Authentication uses hashed passwords and bearer tokens, with per-user ownership checks. Uploaded files are parsed (PDF/DOCX/TXT), split into overlapping chunks, embedded, and retrieved with citations before answer generation. Provider and storage boundaries remain behind small service modules so deployments can replace SQLite/vector storage later.

**Initial schema**
- `users(id, email UNIQUE, password_hash, created_at)`
- `conversations(id, user_id, title, created_at, updated_at)`
- `messages(id, conversation_id, role, content, citations_json, created_at)`
- `documents(id, user_id, filename, media_type, status, created_at)`, plus vector-store chunk metadata (`document_id`, chunk index, text, page/source metadata).

**API surface**
- `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`
- `GET/POST /api/conversations`, `GET/DELETE /api/conversations/{id}`
- `GET/POST /api/conversations/{id}/messages` (message creation returns answer and citations)
- `POST /api/documents`, `GET /api/documents`, `DELETE /api/documents/{id}`
- `GET /health`; FastAPI OpenAPI/Swagger docs at `/docs`.

**Implementation sequence**
1. Inspect current modules/dependencies and existing tests.
2. Add settings, database/models, auth, and focused tests.
3. Implement document parsing/chunking/embedding/search and test those units.
4. Wire chat/history/citations, API validation, logging, and rate limiting; test API integration.
5. Add responsive UI, Docker setup, README/deployment instructions, then run the relevant test suite.

I’m proceeding with that scope; any API provider secrets will remain in environment variables.

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\app.py"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\chat_api.py"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\ai_service.py"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_chat_api.py"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_gemini.py"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\chatbot_analysis_report.md"
}

Read 
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\.gitignore"
}

Read , lines 1 to 180
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\chat_api.py",
  "view_range": [
    1,
    180
  ]
}

Read , lines 1 to 260
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_chat_api.py",
  "view_range": [
    1,
    260
  ]
}

Read , lines 1 to 240
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\ai_service.py",
  "view_range": [
    1,
    240
  ]
}

Read , lines 181 to 360
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\chat_api.py",
  "view_range": [
    181,
    360
  ]
}

Ran terminal command: git status --short; git branch --show-current; Get-ChildItem -Force

The repo already has a legacy in-memory Gemini chat API and tests, so I’ll preserve its existing routes/CLI while adding durable, authenticated `/api` workflows alongside them. One tracked deletion is already present (`chatbot_report_screenshot.png`); I’ll leave it untouched. I’m checking the selected Python environment and existing test behavior before choosing compatible dependencies.

runTests
Completed with input: {
  "files": [
    "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_chat_api.py"
  ]
}
Tool execution failed: No tests found in the files

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylancePythonEnvironments
Completed with input: {
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceImports
Completed with input: {
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Ran terminal command: C:\Users\'Five Star Computer'\AppData\Local\Programs\Python\Python314\python.exe -m unittest AI_service.test_chat_api -v

configurePythonEnvironment
Completed with input: {
  "resourcePath": "C:\\Users\\Five Star Computer\\Ai-chatbot",
  "pythonPath": "C:\\Users\\Five Star Computer\\AppData\\Local\\Programs\\Python\\Python314\\python.exe"
}

The existing 10-test suite passes, and Pylance confirms the current code already depends on FastAPI, Google GenAI, NumPy, and Pydantic. I’m keeping compatibility with those tests, avoiding a new vector database by persisting embeddings in SQLite for exact cosine search, and adding only the document/upload dependencies in the project manifest.

Edit , , , , , , , , , , , , , , 

Edit , , , 
Completed with input: "*** Begin Patch\n*** Update File: C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\ai_service.py\n@@\n+import os\n import time\n \n from google import genai\n from google.genai import types\n@@\n-MODEL = \"gemini-3.5-flash-lite\"\n+MODEL = os.getenv(\"GEMINI_MODEL\", \"gemini-3.5-flash-lite\")\n+\n+\n+def generate_answer(prompt):\n+    response = client.models.generate_content(\n+        model=MODEL,\n+        contents=prompt,\n+        config=types.GenerateContentConfig(\n+            system_instruction=SYSTEM_PROMPT\n+        )\n+    )\n+    if not response.text:\n+        raise ValueError(\"Gemini returned an empty response.\")\n+    return response.text\n*** Update File: C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\database.py\n@@\n     try:\n         yield connection\n+        connection.commit()\n+    except Exception:\n+        connection.rollback()\n+        raise\n     finally:\n         connection.close()\n*** Update File: C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\api.py\n@@\n import json\n import logging\n import os\n-import secrets\n import sqlite3\n import time\n import uuid\n@@\n-from fastapi.responses import FileResponse\n+from fastapi.responses import FileResponse, JSONResponse\n from fastapi.staticfiles import StaticFiles\n from pydantic import BaseModel, EmailStr, Field\n from starlette.middleware.base import BaseHTTPMiddleware\n@@\n-                    return HTTPExceptionResponse(\n+                    return JSONResponse(\n                         status_code=429,\n-                        detail=\"Rate limit exceeded. Please try again shortly.\",\n-                        headers={\"Retry-After\": str(retry_after)}\n+                        content={\n+                            \"detail\": \"Rate limit exceeded. Please try again shortly.\"\n+                        },\n+                        headers={\"Retry-After\": str(retry_after)}\n                     )\n         return await call_next(request)\n \n \n-from starlette.responses import JSONResponse\n-\n-\n-def HTTPExceptionResponse(status_code: int, detail: str, headers: dict):\n-    return JSONResponse(\n-        status_code=status_code,\n-        content={\"detail\": detail},\n-        headers=headers\n-    )\n-\n-\n app = FastAPI(\n*** Update File: C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_rag_api.py\n@@\n-        with (\n+        with (\n             patch.object(rag, \"create_embedding\", side_effect=embedding),\n-            patch.object(api.ai_service, \"generate_answer\", return_value=\"Revenue was $42 [1].\")\n+            patch.object(\n+                api.ai_service,\n+                \"generate_answer\",\n+                return_value=\"Revenue was $42 [1].\"\n+            ) as generate\n         ):\n@@\n         self.assertEqual(answer.status_code, 200, answer.text)\n         self.assertEqual(answer.json()[\"citations\"][0][\"filename\"], \"annual.txt\")\n-        prompt = json.loads(api.ai_service.generate_answer.call_args.args[0])\n+        prompt = json.loads(generate.call_args.args[0])\n*** End Patch\n"
Tool execution failed: Failed to apply patch: Error: Failed to find expected lines in C:\Users\Five Star Computer\Ai-chatbot\AI_service\ai_service.py:
import time

from google import genai
from google.genai import types

Read , lines 1 to 80
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\ai_service.py",
  "view_range": [
    1,
    80
  ]
}

Read , lines 56 to 100
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\database.py",
  "view_range": [
    56,
    100
  ]
}

Read , lines 1 to 150
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\api.py",
  "view_range": [
    1,
    150
  ]
}

Read , lines 65 to 160
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\AI_service\\test_rag_api.py",
  "view_range": [
    65,
    160
  ]
}

Edit , , , 

installPythonPackage
Completed with input: {
  "packageList": [
    "python-multipart",
    "pypdf",
    "python-docx",
    "email-validator"
  ],
  "resourcePath": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

The end-to-end pieces are now in place: durable auth/conversation/document tables, upload→parse→chunk→embed→search, history-aware cited chat, static UI, rate limiting, container files, and docs. I’ve also made SQLite writes transactional and kept the legacy entry points; now I’m running the old and new tests together and checking diagnostics before fixing any failures.

Ran terminal command: & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m unittest AI_service.test_chat_api AI_service.test_rag_api -v

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceFileSyntaxErrors
Completed with input: {
  "fileUri": "file:///C:/Users/Five%20Star%20Computer/Ai-chatbot/AI_service/api.py",
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceFileSyntaxErrors
Completed with input: {
  "fileUri": "file:///C:/Users/Five%20Star%20Computer/Ai-chatbot/AI_service/rag.py",
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceImports
Completed with input: {
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Edit , , , , , , 

installPythonPackage
Completed with input: {
  "packageList": [
    "python-dotenv"
  ],
  "resourcePath": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Ran terminal command: & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m unittest AI_service.test_chat_api AI_service.test_rag_api -v

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceFileSyntaxErrors
Completed with input: {
  "fileUri": "file:///C:/Users/Five%20Star%20Computer/Ai-chatbot/AI_service/security.py",
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceFileSyntaxErrors
Completed with input: {
  "fileUri": "file:///C:/Users/Five%20Star%20Computer/Ai-chatbot/AI_service/test_rag_api.py",
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Search for `FAIL:|AssertionError|ERROR:|Ran [0-9]+ tests|FAILED`

Edit 

Edit , , 

Ran terminal command: & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m unittest AI_service.test_chat_api AI_service.test_rag_api -v

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceSyntaxErrors
Completed with input: {
  "code": "from contextlib import asynccontextmanager\nfrom collections.abc import AsyncGenerator\nfrom fastapi import FastAPI\n\n@asynccontextmanager\nasync def lifespan(_: FastAPI) -> AsyncGenerator[None, None]:\n    yield\n",
  "pythonVersion": "3.14",
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

pylanceImports
Completed with input: {
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

The first validation pass exposed only compatibility/test adjustments (the existing suite asserts a specific prompt phrase, and signup counts toward the shared auth rate limit); both are corrected. The full 19-test suite now exits successfully, and workspace diagnostics are clean. I’m doing a live server/UI smoke check and a final diff review next.

Search for `Ran [0-9]+ tests|^OK$|^FAILED`

Ran terminal command: $env:DATABASE_PATH = Join-Path $env:TEMP 'ai-business-assistant-smoke.sqlite'; $env:GEMINI_API_KEY = 'smoke-test-not-a-real-key'; $env:AUTH_SECRET = 'smoke-test-secret-not-for-production'; & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m uvicorn AI_service.api:app --host 127.0.0.1 --port 8765

Ran terminal command: git diff --check; git status --short; git diff --stat

openBrowserPage
Completed with input: {
  "url": "http://127.0.0.1:8765"
}

clickElement
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "element": "Create an account",
  "ref": "e14"
}

typeInPage
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "element": "Email",
  "ref": "e10",
  "text": "[demo@example.com](mailto:demo@example.com)"
}

typeInPage
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "element": "Password",
  "ref": "e12",
  "text": "a-secure-smoke-password"
}

clickElement
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "element": "Create account",
  "ref": "e16"
}

Read , lines 1 to 180
Completed with input: {
  "path": "C:\\Users\\Five Star Computer\\Ai-chatbot\\README.md",
  "view_range": [
    1,
    180
  ]
}

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Ran terminal command: $health = Invoke-RestMethod -Uri 'http://127.0.0.1:8765/api/health'; $schema = Invoke-RestMethod -Uri 'http://127.0.0.1:8765/openapi.json'; $ui = Invoke-WebRequest -Uri 'http://127.0.0.1:8765/' -UseBasicParsing; Write-Output "health=$($health.status); openapi_paths=$($schema.paths.Count); ui_status=$($ui.StatusCode); ui_title=$($ui.Content.Contains('AI Business Assistant'))"

The live app serves the UI and completes account creation successfully; `/api/health` is healthy. During that smoke check I noticed the small-screen layout hides the conversation/documents sidebar, so I’m fixing mobile access to history and ensuring the legacy chat API honors the same environment-selected Gemini models before final validation.

Edit , , , , 

Edit , 

Ran terminal command: $schema = Invoke-RestMethod -Uri 'http://127.0.0.1:8765/openapi.json'; $paths = $schema.paths.PSObject.Properties.Name; Write-Output "openapi_path_count=$($paths.Count); includes_auth=$($paths -contains '/api/auth/register'); includes_upload=$($paths -contains '/api/documents')"

pylanceLSP
Completed with input: {
  "method": "workspace/diagnostic",
  "params": {},
  "workspaceRoot": "C:\\Users\\Five Star Computer\\Ai-chatbot"
}

Ran terminal command: & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m unittest AI_service.test_chat_api AI_service.test_rag_api -v

runPlaywrightCode
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "code": "await page.setViewportSize({ width: 390, height: 844 }); await page.reload(); await page.getByRole('button', { name: 'Show conversations' }).click(); return { title: await page.title(), sidebarVisible: await page.locator('#sidebar').isVisible(), newConversationVisible: await page.getByRole('button', { name: /New conversation/ }).isVisible() };",
  "timeoutMs": 10000
}

Edit 

Edit 

clickElement
Completed with input: {
  "pageId": "e1f19a1a-ad5c-4270-9727-8b1abef2d0a4",
  "element": "New conversation",
  "ref": "e9"
}

Search for `Ran [0-9]+ tests|^OK$|^FAILED`

Stop Terminal Session
Completed with input: {
  "shellId": "6"
}

Ran terminal command: git diff --check; git status --short

Ran terminal command: & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m pip check; if ($?) { & 'C:\Users\Five Star Computer\AppData\Local\Programs\Python\Python314\python.exe' -m compileall -q AI_service chatbot.py; if ($?) { Write-Output 'compileall=ok' } }

## Implemented

The starter now includes a FastAPI business assistant with a responsive browser UI, persistent conversations, document-backed RAG, authentication, and deployment files.

- **Backend and storage:** `api.py` provides authenticated APIs; `database.py` defines SQLite tables for users, conversations, messages, documents, chunks, and rate-limit counters.
- **Security and RAG:** `security.py` handles password hashing and signed bearer tokens. `rag.py` parses PDF, DOCX, and TXT files, chunks text, creates Gemini embeddings, and searches user-owned chunks with cosine similarity. Chat responses include citations and bounded conversation history.
- **Frontend:** `index.html`, `styles.css`, and `app.js` implement sign-in, history, uploads, chat, and a mobile navigation drawer.
- **Configuration and deployment:** `requirements.txt`, `.env.example`, `Dockerfile`, `docker-compose.yml`, and `README.md` cover setup, API endpoints, schema, tests, and deployment. Gemini credentials are read from environment variables; no real secrets were added.

## Verification

- **19 tests passed**, including the existing tests and new auth, ownership, rate-limit, upload/parser, RAG citation, and history tests.
- `pip check`, Python compilation, `git diff --check`, and Pylance diagnostics completed without issues.
- Live smoke checks verified health, OpenAPI endpoints, account creation, and mobile navigation.
- Gemini generation and embeddings were mocked in tests; **a live provider request was not tested** because no real API key was available.
- Vector search uses an exact SQLite scan, suitable for a modest single-node deployment; the README notes when to migrate to a vector database.

The pre-existing deletion of `chatbot_report_screenshot.png` was left unchanged.
