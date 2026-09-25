# QuillAI

QuillAI is an AI-powered document analysis and retrieval application that allows users to upload PDF, DOCX, and TXT documents, analyze their contents, perform semantic search, and chat with documents using Retrieval-Augmented Generation (RAG).

The application uses FastAPI as the backend, Supabase for authentication, storage, PostgreSQL and pgvector, Groq for language generation, and Hugging Face Inference for 384-dimensional document embeddings.

## Features

- User registration and login using Supabase Auth
- Secure per-user document access
- PDF, DOCX, and TXT document upload
- Automatic document analysis
- Document summaries
- Document type classification
- Keyword extraction
- Important-point extraction
- Suggested questions
- Security observations
- Semantic vector search
- Retrieval-Augmented Generation (RAG) chat
- Source excerpts for retrieved information
- Confidence information for generated answers
- Private document storage
- Temporary signed download links
- Document deletion
- Dashboard statistics
- Per-user request rate limiting
- Prompt-injection-aware document processing

## Architecture

```text
                         User
                          |
                          | HTTPS
                          v
                +---------------------+
                |       QuillAI       |
                |     FastAPI API     |
                |    Render Free      |
                +----------+----------+
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   +-------------+   +-------------+   +-------------+
   |  Supabase   |   |    Groq     |   | Hugging Face|
   |             |   |             |   |             |
   | Auth        |   | LLM         |   | Embeddings  |
   | Storage     |   | Analysis    |   | all-MiniLM  |
   | PostgreSQL  |   | Chat        |   | L6-v2       |
   | pgvector    |   |             |   | 384 dim     |
   +-------------+   +-------------+   +-------------+
```

### Production architecture

```text
QuillAI Web Application
        |
        | HTTPS / REST
        v
FastAPI Python Web API
        |
        +---- Supabase Auth
        |
        +---- Supabase Storage
        |
        +---- Supabase PostgreSQL + pgvector
        |
        +---- Groq API
        |
        +---- Hugging Face Inference API
```

The production application is hosted as a FastAPI web service on Render.

Supabase provides authentication, private object storage, PostgreSQL, and pgvector.

Groq provides the large language model used for document analysis and RAG responses.

Hugging Face provides remote embeddings using:

```text
sentence-transformers/all-MiniLM-L6-v2
```

The model produces 384-dimensional vectors, matching the application's existing pgvector configuration.

## How QuillAI Works

### 1. User authentication

Users register and log in through Supabase Auth.

The frontend receives the authenticated session and sends the user's access token to the FastAPI backend.

The backend validates the token and obtains the authenticated user's ID.

All document-related database operations are filtered using this user ID.

### 2. Document upload

The user can upload:

- PDF
- DOCX
- TXT

The backend validates the file type and size before processing it.

The maximum default file size is 20 MB.

PDF documents are processed page by page.

DOCX documents are processed paragraph by paragraph.

TXT files are decoded as UTF-8 text.

### 3. Document analysis

After extracting the text, QuillAI sends the relevant document content to the Groq API.

Groq generates structured information including:

- Summary
- Document type
- Keywords
- Important points
- Suggested questions
- Security observations

The response is validated using Pydantic models before being stored.

### 4. Chunking

The extracted document text is divided into overlapping chunks.

The default configuration uses approximately:

```text
Chunk size: 1000 characters
Overlap:    150 characters
```

Chunking allows the application to retrieve only the most relevant parts of a document during search and chat.

### 5. Embedding generation

Each chunk is converted into a numerical vector using Hugging Face Inference.

The embedding model is:

```text
sentence-transformers/all-MiniLM-L6-v2
```

The model produces:

```text
384-dimensional vectors
```

These vectors are compatible with the application's PostgreSQL pgvector column.

Unlike the earlier local implementation, the embedding model is not loaded into the FastAPI process.

This significantly reduces the memory requirements of the backend and allows the application to run on a low-memory hosting environment.

### 6. Vector storage

The document chunks and their embeddings are stored in Supabase PostgreSQL.

The pgvector extension is used for vector similarity search.

The original document is stored separately in Supabase Storage.

### 7. Semantic search

When a user performs a search:

```text
Search question
       |
       v
Hugging Face embedding
       |
       v
384-dimensional query vector
       |
       v
Supabase pgvector
       |
       v
Most similar document chunks
```

The retrieved chunks are returned as search results.

This allows QuillAI to search by meaning rather than requiring exact keyword matches.

### 8. RAG chat

For document chat:

```text
User question
      |
      v
Hugging Face embedding
      |
      v
pgvector similarity search
      |
      v
Relevant document chunks
      |
      v
Groq LLM
      |
      v
Grounded answer
```

The retrieved document passages are supplied to Groq as context.

The application instructs the model to treat document content as untrusted data and not follow instructions contained inside uploaded documents.

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | HTML, CSS, JavaScript | User interface |
| Backend | FastAPI | REST API and application logic |
| Language | Python | Backend implementation |
| Authentication | Supabase Auth | User authentication and sessions |
| File Storage | Supabase Storage | Private document storage |
| Database | Supabase PostgreSQL | Application and document data |
| Vector Database | pgvector | Semantic vector search |
| Embeddings | Hugging Face Inference | 384-dimensional document embeddings |
| Embedding Model | all-MiniLM-L6-v2 | Text-to-vector conversion |
| LLM | Groq API | Document analysis and RAG responses |
| Validation | Pydantic | Structured data validation |
| PDF Processing | pypdf | PDF text extraction |
| DOCX Processing | python-docx | Word document extraction |
| Hosting | Render | FastAPI application hosting |

## Project Structure

```text
QuillAI/
├── .gitignore
├── README.md
├── backend/
│   ├── .env
│   ├── .env.example
│   ├── main.py
│   ├── requirements.txt
│   └── schema.sql
└── frontend/
    ├── app.js
    ├── index.html
    └── styles.css
```

The `.env` file must never be committed to Git.

## Environment Variables

Create:

```text
backend/.env
```

using:

```text
backend/.env.example
```

Required variables:

```env
SUPABASE_URL=
SUPABASE_SECRET_KEY=
SUPABASE_PUBLISHABLE_KEY=

SUPABASE_BUCKET=documents

GROQ_API_KEY=
GROQ_MODEL=openai/gpt-oss-20b

HF_TOKEN=
HF_EMBED_MODEL=sentence-transformers/all-MiniLM-L6-v2

MAX_FILE_SIZE_MB=20
MAX_PAGES=200
```

Never commit real API keys or tokens.

## Local Setup

### 1. Clone the repository

```bash
git clone https://github.com/AtharvaShah-Git/QuillAI.git
cd QuillAI
```

### 2. Create a virtual environment

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
```

On Windows:

```powershell
python -m venv venv
venv\Scripts\activate
```

### 3. Install dependencies

```bash
python -m pip install -r requirements.txt
```

### 4. Configure environment variables

Create:

```text
backend/.env
```

and add the required Supabase, Groq, and Hugging Face credentials.

### 5. Configure Supabase

Run the SQL contained in:

```text
backend/schema.sql
```

in the Supabase SQL Editor.

The database uses pgvector for semantic search.

### 6. Start the backend

From the `backend` directory:

```bash
uvicorn main:app --reload --port 8000
```

Open:

```text
http://localhost:8000
```

## Render Deployment

QuillAI can be deployed as a Python Web Service.

### Build Command

```bash
pip install -r backend/requirements.txt
```

### Start Command

```bash
uvicorn backend.main:app --host 0.0.0.0 --port $PORT
```

### Environment Variables

Configure the following environment variables in Render:

```text
SUPABASE_URL
SUPABASE_SECRET_KEY
SUPABASE_PUBLISHABLE_KEY
SUPABASE_BUCKET
GROQ_API_KEY
GROQ_MODEL
HF_TOKEN
HF_EMBED_MODEL
MAX_FILE_SIZE_MB
MAX_PAGES
```

Do not upload `.env` to Render or GitHub.

Render receives the values through its environment-variable configuration.

## Why Remote Embeddings Are Used

The original implementation loaded the Sentence Transformers model directly into the FastAPI process.

Although this worked locally, the model and its PyTorch dependencies require significantly more memory than a low-memory cloud instance provides.

QuillAI therefore uses Hugging Face Inference for embeddings.

The resulting architecture keeps the FastAPI process lightweight while retaining the same embedding model and 384-dimensional vector space.

This means the existing pgvector schema can continue to be used without changing the embedding dimension.

## Security

QuillAI follows several security practices:

- Authentication is handled by Supabase Auth.
- API credentials are stored in environment variables.
- The Supabase secret key is never exposed to the frontend.
- Every backend database query is associated with the authenticated user.
- Uploaded documents are stored in a private storage bucket.
- Temporary signed links are used for downloads.
- Uploaded files are validated before processing.
- File-size and page-count limits reduce resource abuse.
- API requests have a per-user rate limit.
- Document content is treated as untrusted data in AI prompts.
- AI-generated security observations are not treated as malware detection.
- `.env` files are excluded from version control.

## Limitations

- Scanned or image-only PDFs are not supported because the current implementation does not include OCR.
- Chat questions are processed independently and do not maintain conversational memory.
- The in-memory rate limiter resets when the backend restarts.
- The rate limiter is not shared between multiple application instances.
- Hugging Face embedding inference is an external service and is subject to its current availability and usage limits.
- Groq API availability and usage limits can change.
- AI-generated security observations should not be considered a substitute for dedicated malware or security scanning.
- Large documents may take longer to process because document analysis and embedding generation require external API requests.

## RAG Pipeline

```text
Document
   |
   v
Text Extraction
   |
   v
Document Chunking
   |
   v
Hugging Face Embedding
   |
   v
384-dimensional Vector
   |
   v
Supabase PostgreSQL + pgvector
   |
   v
Similarity Search
   |
   v
Relevant Chunks
   |
   v
Groq LLM
   |
   v
Grounded Answer
```

## Database

The application uses Supabase PostgreSQL with pgvector.

The main logical data components are:

```text
Users
  |
  +-- Documents
        |
        +-- Document metadata
        +-- Extracted pages
        +-- Analysis
        |
        +-- Chunks
              |
              +-- Text
              +-- Page number
              +-- 384-dimensional embedding
```

## API Overview

The backend exposes endpoints for:

```text
GET  /
GET  /api/config

GET  /api/documents
POST /api/documents
GET  /api/documents/{id}
DELETE /api/documents/{id}

POST /api/search
POST /api/chat
```

Authentication is required for user-specific operations.

## Academic Project

QuillAI was developed as an academic Cloud Computing microproject.

The project demonstrates the integration of:

- Cloud-hosted backend services
- Cloud authentication
- Cloud object storage
- Cloud PostgreSQL
- Vector databases
- External AI inference services
- REST APIs
- Retrieval-Augmented Generation
- Container/cloud deployment concepts

## Authors

**Atharva Shah**

Government Polytechnic, Pune

Computer Engineering

**Nishtha Bhange**

Government Polytechnic, Pune

Computer Engineering



## License

This project is intended for academic and educational purposes.
