# PostGraphrag
Python Flask based Graphrag implementation using Ollama and Postgres PGAI



Below is an example technical specification for a proof-of-value Flask application integrating GraphRAG with a local LLM (Ollama) and Timescale’s PGAI for embeddings. This specification document outlines a suggested architecture, data flows, and functional modules. Feel free to adapt or expand on any elements to suit real-world constraints or goals.

## 1. High-Level Overview

###	1.	User File Upload

Users can upload multiple text-based files (PDFs, TXT, CSV, or similar). These files are parsed and stored in a PostgreSQL database.

###	2.	Indexing (Graph Generation)

The system runs a GraphRAG pipeline on the uploaded documents:
- Document segmentation into text chunks
- Local LLM (Ollama) for entity/relationship extraction and summarization
- Communities detection / Graph creation
Results (entities, relationships, communities) stored in Postgres.

###	3.	Embeddings
The text chunks and knowledge graph outputs are embedded using Timescale’s PGAI extension (vector store in Postgres).


###	4.	Query
The user can submit queries in two modes:
- Local Search (targeting specific entities & relationships)
- Global Search (operating on pre-summarized communities)


###	5.	LLM Inference
All LLM inference calls route to Ollama running locally (with CPU/GPU execution).
- The system orchestrates queries + context building
- Then returns the final answer to the user


###	6.	Frontend & API
A lightweight Flask web front-end (HTML/JS or React/Vue for advanced UI) that:
- Accepts file uploads
- Displays job progress/status
- Provides a query interface, letting users select local or global mode
- Renders answers (and possibly partial sub-answers or sources)


###	7.	Assumptions
- Ollama is installed and runs as a local service (REST or gRPC).
- PGAI is installed on the same PostgreSQL server or a separate instance.
- For demonstration, we focus on a single Dockerized environment or a local dev environment.

## 2. Proposed Architecture

```
flowchart TB
    subgraph Client
      UI[Browser-based UI] --> FlaskAPI
    end
    
    subgraph Server[Flask Web Server]
      FlaskAPI[Flask Routes] --> IngestionModule
      FlaskAPI --> GraphRAGModule
      FlaskAPI --> QueryModule
    end
    
    IngestionModule --->|Store Files+Metadata| Postgres[(PostgreSQL+PGAI)]
    GraphRAGModule --->|Read/Write Entities/Relationships| Postgres
    QueryModule -->|Retrieve Vector Embeddings/Graph Data| Postgres
    
    subgraph Services
      OllamaLLM[Ollama Local LLM Service] <--- GraphRAGModule
      OllamaLLM <--- QueryModule
    end
```

### Key Points:

**1.	Flask Routes handle requests from the frontend:**
- File Upload (Ingestion)
- Start/Check Indexing (GraphRAG)
- Perform Query (Local/Global)

**2. Ingestion Module reads uploaded files, stores them in Postgres as raw text (possibly chunked or partially chunked for indexing).**

**3.	GraphRAG Module orchestrates:**

- Segmenting text into chunks
- Calling Ollama for entity/relationship extraction, gleaning, summarization
- Storing graph outputs (entities, relationships, communities) in Postgres
- Storing embeddings via PGAI


**4.	Query Module:**
- Accepts user queries
- Dynamically retrieves relevant context (entities, relationships, community summaries) using embeddings or direct PG queries
- Generates final answer with Ollama
- Returns the result to Flask

**5.	PostgreSQL with Timescale PGAI extension is used for:**
- Document storage (raw text)
- Graph tables (entities, relationships, community reports)
- Vector embeddings (pgvector or PGAI)


**6.	Ollama local LLM service is used for:**
- Summarization and extraction prompts during indexing
- Query-time text generation

## 3. Functional Modules & Responsibilities

### 3.1 Ingestion Module
- Endpoints:
    - POST /upload
- Workflow:
	1.	Receive file(s) via Flask.
	2.	Extract text from each file (if PDF, convert pages to text; if CSV, read lines, etc.).
	3.	Normalize text (cleaning, optional language detection).
	4.	Store the file metadata (filename, user ID, etc.) in Postgres.
	5.	Store extracted text in a “documents” table (with a reference to the file ID).

- Key Tables (Postgres):
    1. files(id, filename, user_id, upload_datetime, status, etc.)
    2. documents(doc_id, file_id, text_content, tokens_estimated, ingestion_timestamp, etc.)

### 3.2 GraphRAG Module

Responsible for the core pipeline:

1.	Chunking
- Given the documents table, segment text into chunks (size in tokens, e.g., 600).
- Store chunks in a text_units table.

2.	LLM-based Extraction
- For each chunk, call Ollama with “entity extraction prompt.”
- Use gleaning approach to re-check missed entities.
- Persist partial entity references in e.g., temp_entity_extraction or a staging table.

3.	Consolidation & Summaries
- Merge partial extractions into canonical entities in entities table.
- Summarize entity descriptions to produce a single description per entity.
- Similarly, create a relationships table by merging relationship references.

4.	Community Detection
- Build adjacency matrix from entities and relationships (weighted by frequency or LLM confidence).
- Run Leiden or a similar community detection algorithm (in Python or a separate library).
- Insert communities hierarchy (level 0..N) in a communities table.

5.	Community Summaries
- For each community, gather all relevant node & edge descriptions.
- Summarize them with Ollama.
- Insert summarized text into community_summaries table.

6.	Embeddings via PGAI
- For each text chunk, entity description, or community summary, store vector embeddings in Postgres with PGAI.
- Create embedding indices for search efficiency.

- Key Tables (Postgres):
    - text_units(id, doc_id, chunk_text, vector_embedding, etc.)
    - entities(id, name, type, summary, vector_embedding, ... )
    - relationships(id, source_entity_id, target_entity_id, description, weight, ... )
    - communities(id, level, parent, ... )
    - community_summaries(community_id, summary_text, vector_embedding, ... )

### 3.3 Query Module

Goal: Provide a user interface (and endpoints) to retrieve an answer from the knowledge store.
- Endpoints:
    - POST /query/local
    - POST /query/global

- Local Query Flow:
	1.	Accept a query text.
	2.	Retrieve top-k relevant entities (via PG vector embeddings or direct usage of local metadata).
	3.	Gather relationships, text_units, etc. from the database.
	4.	Construct a context prompt.
	5.	Send to Ollama for final answer.
	6.	Return the answer.

- Global Query Flow (Map-Reduce style):
	1.	Accept a query text.
	2.	For the chosen community level (or user can specify which level?), read the community_summaries.
	3.	Split them into feasible context windows.
	4.	For each batch chunk, ask Ollama for a partial answer & a “helpfulness score.”
	5.	Filter out unhelpful partial answers, then combine the top partial answers into a final prompt.
	6.	Summarize them to produce the final response.

- Considerations:
    - Provide chain-of-thought or partial references if needed.
    - Return sources or entity citations to help the user trust the answer.

### 3.4 Ollama LLM Module

Role: Interacts with the local LLM server from Python.
- Provides a Python-based client (either via an ollama PyPi package or direct HTTP calls).
- Accepts a prompt string plus parameters (temperature, max tokens).
- Returns raw JSON or text output to the calling module.

### 3.5 PGAI Integration

Role: Manage embeddings in Postgres.
	1.	Whenever we need to embed text (document chunk, entity summary, community summary), call PGAI.
	2.	Store the resulting vector in Postgres.
	3.	Use pgvector or timescale vector indexing for similarity search.

Key Steps:
- “embedding_function(text)” -> vector
- Insert into table with a vector column
- For retrieving top-k relevant chunks: SELECT * FROM text_units ORDER BY embeddings <-> query_embedding LIMIT k;

## 4. Data Model (Draft)

Below is a simplified schema outline. Actual schema can be refined once you see real data shapes.
```sql
CREATE TABLE files (
  id SERIAL PRIMARY KEY,
  filename TEXT NOT NULL,
  user_id INT,
  upload_datetime TIMESTAMP DEFAULT NOW(),
  status TEXT
);

CREATE TABLE documents (
  doc_id SERIAL PRIMARY KEY,
  file_id INT REFERENCES files(id),
  text_content TEXT NOT NULL,
  ingestion_timestamp TIMESTAMP DEFAULT NOW(),
  tokens_estimated INT
);

CREATE TABLE text_units (
  id SERIAL PRIMARY KEY,
  doc_id INT REFERENCES documents(doc_id),
  chunk_text TEXT NOT NULL,
  embedding VECTOR(768), -- Example dimension
  token_count INT
);

CREATE TABLE entities (
  id SERIAL PRIMARY KEY,
  name TEXT,
  type TEXT,
  summary TEXT,
  embedding VECTOR(768)
);

CREATE TABLE relationships (
  id SERIAL PRIMARY KEY,
  source_entity_id INT REFERENCES entities(id),
  target_entity_id INT REFERENCES entities(id),
  description TEXT,
  weight FLOAT DEFAULT 1.0
);

CREATE TABLE communities (
  id SERIAL PRIMARY KEY,
  level INT,
  parent INT,
  size INT, -- number of entities
  name TEXT
);

CREATE TABLE community_summaries (
  community_id INT REFERENCES communities(id),
  summary_text TEXT,
  embedding VECTOR(768)
);
```

## 5. Example User Flows

### 5.1 Ingest & Index

1.	Upload: User drags multiple files into the app or calls /upload.
2.	Start Indexing:
    - The system asynchronously chunkifies the documents.
    - Calls Ollama for entity/relationship extraction, gleanings, and summarization.
    - Builds and stores the communities.
    - Embeds relevant text in Timescale PGAI.
3.	View Progress: The UI polls for job status: “Indexing in progress”, “Completed”, “Error”, etc.

### 5.2 Query

1.	User Query: “What are the key themes across all these regulatory policy articles?”
2.	Mode: User selects “Global query.”
3.	Process:
    - The system fetches the relevant community summaries from a specific level.
    - It does a map-reduce flow to produce a final summarized answer.
4.	Return: The user sees a structured text answer, possibly a short summary, bullet points, or references.

### 5.3 Local Search Example
1.	User Query: “What did Kevin Scott say about Privacy in Episode #12?”
2.	Mode: Local query.
3.	Process:
    - The system vector-searches entities for “Kevin Scott,” “Privacy,” etc.
    - Gathers local text units from Episode #12 and relevant relationships from the graph.
    - Assembles them in a final prompt for Ollama.
4.	Return: The user sees a targeted answer with citations or chunk references.

## 6. Infrastructure & Deployment

### 6.1 Local Dev
- Docker Compose with containers for:
- Flask App
- Postgres + Timescale + PGAI
- Ollama
- .env for environment-specific configs.

### 6.2 Cloud Deployment / Self-hosted Deployment
- Potentially replicate the same Docker Compose to an Azure or AWS VM or container instance.
- Scale out if usage demands.
- Ensure secure connections, SSL for Postgres.

## 7. Key Prompts & Config

### 7.1 LLM Prompts

1.	Entity Extraction
    - A default system prompt that instructs Ollama to output a JSON or delimited list of (entity_name, type, short_description).
2.	Relationship Extraction
    - Possibly combined with entity extraction or as a second step: “For each pair of entities in the text, define the relationship, if any.”
3.	Summarize Descriptions
    - “Combine all descriptions of this entity into a single cohesive paragraph.”
4.	Community Summaries
    - “Given these N entity/relationship/covariate descriptions, write a short summary capturing the key topics, events, participants, etc.”

5.	Query Time
    - Local: “Given the user query and these entities/relationships, produce a short answer referencing the text units.”
    - Global: “We have partial summaries from each community. For each batch, refine the partial answer. Then we finalize the answer from the partials.”

### 7.2 Configuration Variables

- MAX_CHUNK_SIZE = 600 (tokens)
- MAX_GLEANINGS = 1
- MAX_QUERY_CONTEXT_TOKENS = 8000 (or 4096, etc.)
- LLM_ENDPOINT = "http://localhost:11411" (example for Ollama)
- POSTGRES_CONNECTION_STRING
- PGAI_EMBED_MODEL = 'some-timescale-embedding-model'

## 8. Conclusion & Next Steps

This specification outlines the main modules, data flows, and architecture choices for a proof-of-value Flask application integrating:
- Local LLM (Ollama) for language tasks
- GraphRAG for entity/relationship extraction and global/local queries
- Timescale’s PGAI for embeddings and vector search within PostgreSQL

## Next Steps:
	1.	Stand up local Postgres + PGAI environment.
	2.	Implement minimal ingestion flow with sample documents.
	3.	Implement a skeleton pipeline in Python (chunking, LLM calls, storing results).
	4.	Create a minimal UI with endpoints for upload, indexing, and query.
	5.	Iterate on prompt tuning for domain-specific data sets.
	6.	Add advanced features (e.g., user login, advanced visualization, logging of queries, or chain-of-thought outputs).


