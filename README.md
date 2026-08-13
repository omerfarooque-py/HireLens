# AI Resume Screening & RAG System

An end-to-end **AI-powered resume screening system** built with **Django REST Framework, PostgreSQL/pgvector, Celery, Redis, embeddings, and LLMs**.

The system allows recruiters to upload and organize candidate resumes, process them asynchronously, search resume content semantically, chat with individual candidates through RAG, generate AI-powered candidate analysis, and rank candidates against a job description.

> **Core idea:** turn unstructured PDF resumes into searchable, persistent vector data that can be used for candidate-specific AI analysis and job matching.

---

## ✨ Features

### 📄 Resume Ingestion

* Upload one or multiple PDF resumes in a single request.
* Validate and process each file independently.
* Return per-file outcomes:

  * `accepted`
  * `rejected`
  * `skipped`
* Extract candidate metadata such as:

  * Name
  * Email
  * Resume creation/modification date
* Detect duplicate resumes using SHA-256 file hashing.
* Detect older resume versions when a newer version of the same candidate already exists.

### ⚡ Asynchronous Processing

Resume processing can involve:

```text
PDF
 ↓
Text extraction
 ↓
Document splitting
 ↓
Embedding generation
 ↓
Vector persistence
```

Instead of keeping the HTTP request open during this work, the system uses:

* **Celery** for background processing
* **Redis** as the message broker/result backend

The upload API can therefore return a task ID while processing continues in the background.

### 🧠 Persistent Vector Search

Resume chunks are converted into embeddings using:

`sentence-transformers/all-MiniLM-L6-v2`

The embeddings are stored directly in PostgreSQL using **pgvector**.

This provides persistent semantic search without requiring an in-memory vector store to be rebuilt every time the application starts.

Conceptually:

```text
Resume PDF
    │
    ▼
Text Extraction
    │
    ▼
Chunking
    │
    ▼
Sentence Transformer
    │
    ▼
Embedding Vector
    │
    ▼
PostgreSQL + pgvector
```

### 💬 RAG Candidate Chat

Users can select an individual candidate and ask questions about their resume.

Example:

```text
"What are this candidate's strongest backend skills?"
```

The system:

1. Converts the question into an embedding.
2. Searches the candidate's stored resume chunks.
3. Retrieves the most relevant content.
4. Builds a context-aware prompt.
5. Sends the context to the LLM.
6. Returns an answer grounded in the candidate's resume.

This prevents the chat system from treating the entire resume as one large prompt.

### 📊 AI Candidate Analysis

The system can generate structured analysis for a candidate using their resume content.

Analysis can be used to identify things such as:

* Candidate strengths
* Relevant experience
* Technical skills
* Missing or weaker areas
* Overall candidate assessment

### 🎯 Candidate Ranking

Candidates can be evaluated against a job description.

Example:

```text
Job:
Python Backend Engineer with Django,
REST APIs, PostgreSQL and NLP experience.
```

The system retrieves relevant candidate information and asks the LLM to produce a structured ranking result.

The ranking flow can be scoped to a particular batch, allowing recruiters to compare candidates within a specific hiring group.

### 🗂️ Candidate Batches

Resumes can be organized into named batches.

For example:

```text
Q4 Backend Hiring
├── Candidate A
├── Candidate B
├── Candidate C
└── Candidate D
```

This allows ranking and document retrieval to be scoped to the intended candidate group rather than processing every resume in the system.

### 💾 LLM Response Caching

LLM calls can be expensive and unnecessarily repeated when the same candidate is analyzed against the same input.

The system therefore caches AI responses for operations such as:

* Candidate analysis
* Candidate ranking

Cache keys incorporate relevant document/job information so that a changed resume or job description can result in a new evaluation.

---

# 🏗️ Architecture

The main processing architecture is:

```text
                         ┌─────────────────────┐
                         │      Frontend       │
                         │ Upload / Chat / UI  │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Django + DRF    │
                         │      REST API      │
                         └──────────┬──────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
                 ▼                  ▼                  ▼
          PostgreSQL             Redis             Groq LLM
          + pgvector               │
                 │                 ▼
                 │              Celery
                 │                 │
                 │                 ▼
                 │          Background Tasks
                 │                 │
                 │          ┌──────┴───────┐
                 │          │              │
                 │       PDF Load       Embeddings
                 │          │              │
                 │          ▼              ▼
                 │       Chunking      Vector Storage
                 │          │              │
                 └──────────┴──────────────┘
                              │
                              ▼
                       RAG Retrieval
                              │
                              ▼
                         LLM Response
```

---

# 🔄 Resume Processing Flow

A typical upload follows this pipeline:

```text
1. User uploads PDF
          ↓
2. Django validates file
          ↓
3. SHA-256 hash generated
          ↓
4. Duplicate/version checks
          ↓
5. Document record created
          ↓
6. Celery task queued
          ↓
7. PDF text extracted
          ↓
8. Text split into chunks
          ↓
9. Embeddings generated
          ↓
10. Chunks + vectors persisted
          ↓
11. Resume becomes searchable
```

This separates **request handling** from **heavy document processing**.

---

# 🔎 RAG Flow

Candidate chat and AI analysis use the stored vector representations.

```text
User Question
      │
      ▼
Question Embedding
      │
      ▼
pgvector Similarity Search
      │
      ▼
Relevant Resume Chunks
      │
      ▼
Context Construction
      │
      ▼
Prompt Template
      │
      ▼
Groq / Llama LLM
      │
      ▼
AI Response
```

The important part is that retrieval is performed against the selected candidate's stored document chunks rather than blindly sending unrelated resumes to the model.

---

# 🎯 Candidate Ranking Flow

```text
                 Job Description
                        │
                        ▼
               ┌────────────────┐
               │ Selected Batch │
               └───────┬────────┘
                       │
                       ▼
              Candidate Documents
                       │
                       ▼
              Relevant Resume Data
                       │
                       ▼
                 LLM Evaluation
                       │
                       ▼
              Structured JSON Result
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          Score              Recommendation
             │                   │
             └─────────┬─────────┘
                       ▼
                  Ranking Results
```

---

# 🧩 Tech Stack

| Layer           | Technology                               |
| --------------- | ---------------------------------------- |
| Backend         | Django 6                                 |
| API             | Django REST Framework                    |
| Database        | PostgreSQL                               |
| Vector Database | pgvector                                 |
| Background Jobs | Celery                                   |
| Message Broker  | Redis                                    |
| Embeddings      | `sentence-transformers/all-MiniLM-L6-v2` |
| LLM             | Groq / `llama-3.3-70b-versatile`         |
| PDF Processing  | `pdfplumber`, `pypdf`                    |
| Frontend        | Django Templates                         |
| Language        | Python 3.12+                             |

---

# 📁 Project Structure

```text
Resume-Builder/
│
├── config/
│   ├── settings.py
│   ├── urls.py
│   ├── celery.py
│   └── ...
│
├── core/
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── urls.py
│   ├── tests.py
│   └── ...
│
├── rag/
│   ├── loader.py
│   ├── splitter.py
│   ├── embeddings.py
│   ├── vectordb.py
│   ├── retrieval.py
│   ├── engine.py
│   ├── orchestrator.py
│   ├── tasks.py
│   └── ...
│
├── prompts/
│   └── ...
│
├── frontend/
│   └── templates/
│
├── docs/
│   └── images/
│
├── media/
│   └── documents/
│
├── manage.py
├── requirements.txt
└── README.md
```

### Responsibility Breakdown

**`core/`**

Contains the Django application layer:

* Database models
* API serializers
* REST views
* URL routing
* Tests

**`rag/`**

Contains the AI/RAG pipeline:

* PDF loading
* Document splitting
* Embedding generation
* Vector persistence
* Similarity retrieval
* LLM orchestration
* Background processing

**`prompts/`**

Keeps LLM prompt templates separate from application logic.

**`frontend/`**

Contains the server-rendered UI for interacting with the system.

---

# 🚀 Getting Started

## Prerequisites

Make sure the following are installed:

* Python 3.12+
* PostgreSQL 14+
* PostgreSQL `pgvector` extension
* Redis
* Git
* A Groq API key

---

## 1. Clone the Repository

```bash
git clone https://github.com/omerfarooque-clentro/HireLens.git
cd HireLens
```

---

## 2. Create a Virtual Environment

### Windows

```powershell
python -m venv chatbot_env
.\chatbot_env\Scripts\Activate.ps1
```

### macOS / Linux

```bash
python -m venv chatbot_env
source chatbot_env/bin/activate
```

---

## 3. Install Dependencies

```bash
pip install -r requirements.txt
```

---

# 🗄️ PostgreSQL + pgvector

Create a PostgreSQL database and enable the vector extension:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The application uses PostgreSQL both for conventional relational data and for storing resume embeddings.

Example environment configuration:

```env
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_HOST=127.0.0.1
DB_PORT=5432

GROQ_API_KEY=your_groq_api_key
```

> Do not commit real credentials or API keys to the repository.

---

# 🔧 Database Migration

Run:

```bash
python manage.py migrate
```

---

# ▶️ Running the Application

The application requires Django, Redis, and a Celery worker.

Run each service in a separate terminal.

### Terminal 1 — Django

```bash
python manage.py runserver
```

### Terminal 2 — Redis

```bash
redis-server
```

### Terminal 3 — Celery

```bash
python -m celery -A config worker -l info -P solo --concurrency=1
```

Once running, open:

```text
http://127.0.0.1:8000/
```

---

# 🖥️ Web Interface

## Home

The home page provides access to the main resume processing workflows.

<img width="1366" height="768" alt="Resume Builder Home Page" src="https://github.com/user-attachments/assets/bf1c04da-307b-428c-9ebb-1a04549a6cb9" />

---

## Resume Upload

Upload one or multiple resumes and optionally assign them to a batch.

<img width="1366" height="768" alt="Resume Upload Page" src="https://github.com/user-attachments/assets/e403e38a-2702-4abe-84b0-8471c7521453" />

The upload system reports individual outcomes instead of treating the entire request as one success/failure operation.

---

## Candidate Chat

Select a candidate and ask questions about their resume.

<img width="1366" height="768" alt="Candidate RAG Chat" src="https://github.com/user-attachments/assets/1f053652-c0ec-4d68-a646-dd14e387ad74" />

---

## Candidate Analysis

Generate an AI-powered analysis of a candidate based on their resume.

<img width="1366" height="768" alt="Candidate Analysis" src="https://github.com/user-attachments/assets/4dd03665-9f99-4093-a0c8-4f7f5b15ef52" />

---

## Candidate Ranking

Compare candidates against a job description.

<img width="1366" height="768" alt="Candidate Ranking" src="https://github.com/user-attachments/assets/9cf161be-8d54-4198-912b-fd4762553326" />

---

# 🌐 Web Routes

| Route        | Purpose                     |
| ------------ | --------------------------- |
| `/`          | Home page                   |
| `/upload/`   | Resume upload               |
| `/chat/`     | Candidate-specific RAG chat |
| `/analysis/` | Candidate analysis          |
| `/ranking/`  | Candidate ranking           |

---

# 🔌 REST API

Base API path:

```text
/api/
```

## List Batches

```http
GET /api/batches/
```

Returns available candidate batches.

---

## List Documents

```http
GET /api/documents/?batch_id=<id>
```

Returns candidate documents and optionally filters them by batch.

---

## Upload Resumes

```http
POST /api/upload/
```

Uses multipart form data.

The request can contain:

* One or more PDF files
* An optional batch name

The response provides aggregate totals and individual file results.

---

## Candidate Chat

```http
POST /api/chat/
```

Example request:

```json
{
  "document_id": 1,
  "question": "What are this candidate's strongest backend skills?"
}
```

The system retrieves relevant chunks from that candidate's resume before generating the response.

---

## Candidate Analysis

```http
POST /api/resume/analyze/
```

Example:

```json
{
  "document_id": 1
}
```

---

## Candidate Ranking

```http
POST /api/resume/rank/
```

Example:

```json
{
  "job_description": "Python backend engineer with NLP and Django",
  "batch": "Q4-hiring"
}
```

The ranking operation evaluates candidates within the selected batch.

---

# 📦 Upload Response

A successful upload request can contain different outcomes for different files.

Example:

```json
{
  "totals": {
    "accepted": 2,
    "rejected": 1,
    "skipped": 1
  },
  "details": [
    {
      "filename": "resume_a.pdf",
      "status": "accepted",
      "document_id": 12,
      "task_id": "..."
    },
    {
      "filename": "resume_b.pdf",
      "status": "rejected",
      "reason": "Rejected: Missing required email contact info."
    },
    {
      "filename": "resume_c.pdf",
      "status": "skipped",
      "reason": "Duplicate resume skipped."
    }
  ]
}
```

This design allows a batch upload to partially succeed without losing the result of individual files.

---

# 🛡️ Duplicate & Version Handling

The ingestion pipeline uses the uploaded file's SHA-256 hash to identify exact duplicates.

Conceptually:

```text
PDF
 │
 ▼
SHA-256
 │
 ├── Existing hash → Duplicate → Skip
 │
 └── New hash
       │
       ▼
   Metadata extraction
       │
       ▼
   Candidate matching
       │
       ├── Older version → Skip
       │
       └── New version → Process
```

This is particularly useful when recruiters repeatedly upload resumes during a hiring process.

---

# 🧠 Why pgvector?

An earlier/simple RAG architecture could keep a vector store in application memory or rebuild it from the source documents.

This project instead persists embeddings in PostgreSQL.

That gives the application:

* Persistent vectors
* Database-backed retrieval
* Document-level filtering
* Easier integration with existing relational data
* No requirement to rebuild the entire vector index after every restart

The database effectively becomes the source of truth for both candidate metadata and their searchable resume chunks.

---

# ⚙️ Why Celery?

PDF extraction and embedding generation are potentially expensive operations.

Doing everything inside the upload request would create a flow like:

```text
HTTP Request
    │
    ├── Read PDF
    ├── Extract text
    ├── Split chunks
    ├── Generate embeddings
    └── Save vectors
          │
          ▼
      HTTP Response
```

This makes the user wait for the complete processing pipeline.

With Celery:

```text
HTTP Request
    │
    ▼
Create Document
    │
    ▼
Queue Celery Task
    │
    ▼
HTTP Response
    │
    │
    └───────────────► Celery Worker
                           │
                           ├── Extract
                           ├── Split
                           ├── Embed
                           └── Persist
```

The API and processing workload are therefore separated.

---

# 💰 LLM Caching

AI requests can become expensive when the same candidate is repeatedly analyzed.

The application caches relevant LLM responses.

For example:

```text
Candidate Resume
       +
Job Description
       │
       ▼
   Cache Key
       │
   ┌───┴────┐
   │        │
 Cache Hit  Cache Miss
   │        │
   ▼        ▼
 Return    LLM
 Result     │
            ▼
         Store Result
```

This reduces unnecessary repeated LLM requests and improves response time for repeated operations.

---

# 🧪 Testing

Run the Django test suite with:

```bash
python manage.py test
```

Testing covers application behavior around the core resume-processing functionality.

Areas worth testing include:

* Resume upload
* Invalid files
* Duplicate detection
* Candidate metadata extraction
* Batch filtering
* Document retrieval
* RAG retrieval
* Candidate analysis
* Candidate ranking
* Cache behavior
* Background processing

---

# ⚠️ Common Issues

### Celery command not found

Instead of:

```bash
celery -A config worker
```

use:

```bash
python -m celery -A config worker -l info -P solo --concurrency=1
```

This ensures the Celery executable is resolved from the active Python environment.

### Upload succeeds but processing does not finish

Check:

1. Redis is running.
2. The Celery worker is running.
3. The worker is using the correct virtual environment.
4. The Django application can connect to PostgreSQL.

### PostgreSQL vector errors

Make sure pgvector is installed and enabled:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### LLM requests fail

Verify:

```env
GROQ_API_KEY=your_groq_api_key
```

and make sure the key is available to the Django process.

---

# 🔐 Security Notes

This project is intended for development/portfolio use.

Before deploying publicly:

* Move `SECRET_KEY` to environment variables.
* Disable `DEBUG`.
* Configure `ALLOWED_HOSTS`.
* Keep API keys outside the repository.
* Restrict access to uploaded resume files.
* Configure production storage for uploaded documents.
* Add authentication/authorization around candidate data.
* Avoid exposing personally identifiable resume data through unrestricted endpoints.

Resumes can contain sensitive personal information, so production deployments should treat uploaded documents and generated candidate analysis accordingly.

---

# 🚧 Current Scope & Future Improvements

Potential improvements include:

* Authentication and role-based access control
* Production object storage for uploaded PDFs
* Better resume metadata extraction
* Structured LLM outputs using schema validation
* Hybrid vector + keyword retrieval
* Retrieval reranking
* Better task status/progress tracking
* More granular candidate comparison
* Production Docker setup
* CI/CD
* API authentication and rate limiting
* More comprehensive integration tests
* Production monitoring and logging

---

# 📌 What This Project Demonstrates

This project focuses on more than simply integrating an LLM API.

It demonstrates an end-to-end backend architecture involving:

```text
Django / DRF
     │
     ├── REST APIs
     ├── PostgreSQL
     └── Application models
             │
             ▼
          Celery
             │
             ▼
           Redis
             │
             ▼
       PDF Processing
             │
             ▼
         Embeddings
             │
             ▼
        pgvector
             │
             ▼
       Semantic Search
             │
             ▼
            RAG
             │
             ▼
            LLM
             │
             ├── Candidate Chat
             ├── Candidate Analysis
             └── Candidate Ranking
```

The result is an AI application where the **backend infrastructure, data persistence, asynchronous processing, retrieval system, and LLM layer work together as one system**.

---

# 📄 License

This project is currently provided without a finalized open-source license.

If the repository is intended to be publicly reused or distributed, add an appropriate license such as MIT before publishing it as an open-source project.

---
# main GitHub profile

https://github.com/omerfarooque-py
