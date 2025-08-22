# Smart Receipt Assistant

Smart Receipt Assistant is a multi-agent system designed to extract, normalize, store and query information from retail receipts.  It combines an on-device orchestrator (Complex LangGraph) with a remote A2A agent (ADK Agent) to perform OCR on receipt images, convert currencies, translate text and provide relevant information from external knowledge bases.  The project also exposes a FastAPI backend and a Gradio front-end for end-user interaction.

## Table of Contents

1. [Introduction](#introduction)
2. [Quick Start](#quick-start)
   1. [Start the ADK Agent](#start-the-adk-agent)
   2. [Start Complex LangGraph MCP Servers](#start-complex-langgraph-mcp-servers)
   3. [Start the FastAPI Backend](#start-the-fastapi-backend)
   4. [Start the Gradio UI](#start-the-gradio-ui)
3. [Architecture](#architecture)
   1. [Complex LangGraph (core flow)](#complex-langgraph-core-flow)
   2. [ADK Agent (remote a2a)](#adk-agent-remote-a2a)
   3. [RAG Section](#rag-section)
   4. [API (FastAPI)](#api-fastapi)
   5. [Gradio UI](#gradio-ui)
4. [Project Layout](#project-layout)

## Introduction

Smart Receipt Assistant was built to automate the tedious process of organizing and analysing purchase receipts.  Users can upload receipt images or paste OCR text, and the system will:

* **Recognize text** using Azure Computer Vision via a remote A2A agent.
* **Normalize** the unstructured OCR output into a clean schema (date, total amount, currency, merchant, line items).
* **Store** the normalized data in a local SQLite database for persistent queries.
* **Answer questions** about receipts (e.g. *“What is my total spend on groceries this month?”*) by generating SQL via a Smart Query Agent.
* **Retrieve relevant context** using a Retrieval-Augmented Generation (RAG) pipeline over both articles about Lebanon’s economy and structured price data.

The system is composed of several agents that communicate.  Each agent runs in its own process and is orchestrated by a central router graph.  The instructions below describe how to run the components locally.

## Quick Start

> **Note**: Commands below assume a Unix-like shell.  For Windows, replace forward slashes with backslashes and ensure the appropriate Python interpreter is available.  Each numbered subsection should be run in a separate terminal.

### Start the ADK Agent

The **ADK Agent** is responsible for OCR, currency conversion and translation.  It exposes an A2A interface consumed by Complex LangGraph.

```bash
cd adk_agent

# Start the agent server
python __main__.py
```

### In a second terminal (MCP server for the ADK agent)


```bash
cd adk_agent

# Start the mcp server
python __main__.py
```


```bash
cd adk_agent

# Start the agent server
python __main__.py
```

### Start Complex LangGraph MCP Servers

Complex LangGraph contains multiple sub-agents. Each agent must be started in its own terminal from the `complex_langgraph/` directory.

```bash
cd complex_langgraph

# Terminal A – Query Agent
cd query_agent
python mcp_server.py

# Terminal B – Store Agent
cd ../store_agent
python mcp_server.py

# Terminal C – RAG
cd ../rag
python mcp_server.py
```

### Start the Backend
The API layer orchestrates the agents and exposes a single POST endpoint for user queries or receipt uploads.
It should be started in another terminal from the project root.

``` bash
uvicorn app.main:api --host 0.0.0.0 --port 8001 --reload
```
### Start the Gradio UI

Finally, run the Gradio interface to interact with the assistant through a web browser.
It provides widgets for uploading receipts and asking questions.

``` bash

python -m ui.gradio_ui

```
# 🌐 Complex Langraph

The **Complex LangGraph** orchestrates local agents through a series of nodes connected via a graph structure.  
A **router node** examines the current conversation state and routes the input to one of four downstream nodes:

| **Node**    | **Purpose** |
|-------------|-------------|
| **Normalize** | Cleans raw OCR text, removes noise (e.g., headers, footers) and maps the data to a relational schema. |
| **Store**     | Inserts normalized receipts into a local SQLite database for persistence. |
| **Query**     | Converts natural-language questions into SQL using a Smart Query Agent and answers queries. |
| **RAG**       | Retrieves relevant context from external sources (articles or SQL tables) to enrich responses. |

When the input is an **image**, a separate **OCR Node** triggers the remote **ADK Agent** to extract text.  
The router then decides whether to perform normalization, storage, or querying based on the current state of the conversation.

> **Note:** The Query Agent is **highly reliant on prompt engineering**.  
> It is explicitly told that when searching for a keyword, it must use `%%LIKE%%` — otherwise it may become confused and produce incorrect SQL.

Most agents use small (flash) language models to reduce resource usage.  
However, the architecture is **modular**—each node can be swapped for a more powerful model if required.

---

# 🌐 ADK Agent (Remote A2A)

The **ADK Agent** lives outside the Complex LangGraph and communicates via the **A2A protocol**.  

### Responsibilities
- **OCR**: Uses Azure Computer Vision to extract text from receipt images.  
- **Extraction**: Parses the OCR text to extract the total amount, purchase date, and currency symbol.  
- **Currency conversion**: Calls `exchangerate.host` to convert amounts to USD.  
- **Translation**: Translates non-English text into English where necessary.  
- **Response format**: Returns a clean JSON payload with the extracted fields.  

The ADK Agent runs as its own server with:
- A **dedicated agent card** (executor class).  
- A `send_ocr` function to handle messages.  

It must be started **before any OCR requests** are made.

---
# 📚 RAG (Retrieval-Augmented Generation) Section

The **retrieval-augmented generation (RAG)** component supports two distinct data sources:  

---

## 📰 Articles RAG

- **Data**: Articles and reports about Lebanon’s economy and inflation.  
- **Chunking**: Multiple techniques were tested, including semantic chunking and recursive strategies.  
  - The optimal results came from the **SpaCy text splitter**.  
- **Embeddings**: Both Hugging Face embeddings with an in-memory vector store and OpenAI embeddings were evaluated.  
  - Best performance was achieved with **OpenAI embeddings + FAISS**.  
- **Vector DB**: FAISS is used to store and search embeddings efficiently.  

---

## 📊 SQL-based RAG

- **Data**: Excel sheets containing prices of products and services in Lebanon.  
- **Storage**: Loaded into a **LiteSQL (SQLite)** database with two structured tables.  
- **Querying**: Uses **LangChain’s SQL Toolkit**, which includes:
  - `ListSQLDatabaseTool(db)` – List available tables and schemas.  
  - `InfoSQLDatabaseTool(db)` – Retrieve detailed schema information.  
  - `QuerySQLDataBaseTool(db)` – Execute parameterized SQL queries.  

---

These components can be **mixed and matched**:  
The retrieval pipeline intelligently chooses the appropriate source (Articles RAG or SQL-based RAG) depending on the user’s question.  


# ⚡ API (FastAPI)

The **FastAPI** backend exposes a single `POST /` endpoint.  
Users can send either **free-form text (questions)** or **receipt images**.

- Swagger/OpenAPI documentation is automatically generated when running the server.  
- Navigate to: [http://localhost:8001/docs](http://localhost:8001/docs) to view and test the endpoint.  

---

# 🎨 Gradio UI

The **Gradio interface** is a lightweight client that communicates with the FastAPI backend.  
It allows users to:

- 📤 Upload image files (PNG/JPEG) containing receipts  
- 📝 Paste OCR text directly into a text box  
- ❓ Ask natural-language questions about their receipt history  
- 👀 View the assistant’s answers and any extracted entities  

---

# 🗂️ Project Layout

```plaintext
Smart_Receipt_Assistant/
├── adk_agent/                  # Remote A2A agent for OCR, extraction and currency conversion
│   ├── __main__.py            # Entry point for the ADK agent server
│   ├── mcp_server.py           # MCP server exposing the agent over the protocol
│   ├── agent.py                # Business logic for OCR and extraction
│   ├── agent_executor.py       # Agent card / executor class
│   └── ...                     # Other helpers and __pycache__
├── app/                        # FastAPI backend
│   └── main.py                 # Defines the API routes and orchestrates the agents
├── assets/                     # Static assets (e.g. demo receipts, images)
├── complex_langgraph/          # Core LangGraph orchestration
│   ├── query_agent/            # Smart Query agent (natural language → SQL)
│   │   └── mcp_server.py       # MCP server for the query agent
│   ├── store_agent/            # Agent responsible for inserting receipts into SQLite
│   │   └── mcp_server.py
│   ├── rag/                    # RAG node retrieving context
│   │   └── mcp_server.py
│   ├── normalize_agent/        # Normalization logic (cleaning OCR text)
│   │   └── ...
│   ├── a2a_client/             # Client to call the remote ADK agent
│   └── mcp_server.py           # Root MCP server coordinating the router graph
├── fine_tuned_model/           # Optional fine-tuned models (e.g. LoRA for SQL generation)
│   └── my_n12sql_lora/         # Pretrained weights and config
├── ui/                         # User interfaces
│   ├── gradio_ui.py            # Gradio front-end implementation
│   └── credentials.json        # API keys / secrets used by the UI (excluded in .gitignore)
├── lebanon_prices.db           # Example database of product/service prices (SQL-based RAG)
├── receipts.db                 # SQLite database storing normalised receipts
├── graph.py                    # Defines the LangGraph router and node connections
├── .gradio/                    # State/configuration generated by Gradio
├── .gitignore                  # Specifies ignored files (e.g. .env, *.json credentials)
└── README.md



