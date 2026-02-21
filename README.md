# Memory & Context Engineering for AI Agents with Oracle AI Database

Build AI agents that **remember, learn, and adapt** across conversations using Oracle AI Database as the memory backbone, LangChain for orchestration, and Tavily for web search.

## What This Does

This project implements a complete **Memory Manager** with six distinct memory types — each serving a specific cognitive function for an AI agent. Instead of treating every interaction as a blank slate, the agent persists knowledge, tracks entities, learns workflows, and manages its own context window.

| Memory Type | Purpose | Storage |
|---|---|---|
| **Conversational** | Chat history per thread | SQL Table |
| **Knowledge Base** | Searchable documents & facts | Vector Store |
| **Workflow** | Learned action patterns | Vector Store |
| **Toolbox** | Dynamic tool definitions | Vector Store |
| **Entity** | People, places, systems extracted from context | Vector Store |
| **Summary** | Compressed context for long conversations | Vector Store |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            USER QUERY                                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         AGENT LOOP                                      │
│                                                                         │
│  1. Build Context (programmatic reads from all memory types)            │
│  2. Check Context Window Usage → auto-summarize if >80%                 │
│  3. Retrieve Relevant Tools (semantic search on toolbox)                │
│  4. Call LLM with context + tools                                       │
│  5. Execute tool calls if needed (agentic)                              │
│  6. Save results (conversation, workflow, entities)                     │
│                                                                         │
└──────┬──────────────┬──────────────┬──────────────┬─────────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌────────────┐ ┌────────────┐ ┌───────────┐ ┌────────────┐
│  LLM       │ │  Toolbox   │ │  Tavily   │ │  Context   │
│ (HF API or │ │ (Semantic  │ │ (Web      │ │  Engine    │
│  Local     │ │  Tool      │ │  Search)  │ │ (Summarize │
│  Qwen3)    │ │  Retrieval)│ │           │ │  & Offload)│
└────────────┘ └────────────┘ └───────────┘ └────────────┘
       │              │              │              │
       └──────────────┴──────┬───────┴──────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ORACLE AI DATABASE (Docker)                           │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────────────────────────────┐     │
│  │   SQL Table      │  │          Vector Stores (OracleVS)       │     │
│  │                  │  │                                          │     │
│  │  Conversational  │  │  Knowledge Base  │  Workflow  │ Toolbox  │     │
│  │  Memory          │  │  Entity          │  Summary             │     │
│  │  (thread_id,     │  │                                          │     │
│  │   role, content) │  │  IVF Indexes for fast similarity search  │     │
│  └──────────────────┘  └──────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
               ┌──────────┐
               │  User    │
               │  Query   │
               └────┬─────┘
                    │
        ┌───────────▼───────────┐
        │  PROGRAMMATIC READS   │  (every turn, automatic)
        │                       │
        │  • Conversation History───────► SQL Table
        │  • Knowledge Base ────────────► Vector Search
        │  • Workflows ─────────────────► Vector Search
        │  • Entities ──────────────────► Vector Search
        │  • Summaries (IDs only) ──────► Vector Search
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │  CONTEXT WINDOW CHECK │
        │  >80%? → Summarize &  │
        │  offload to Summary   │
        │  Memory               │
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │  LLM INFERENCE        │
        │  + Semantic Tool      │
        │    Retrieval          │
        └───────────┬───────────┘
                    │
              ┌─────┴─────┐
              │           │
     ┌────────▼──┐  ┌─────▼──────┐
     │ AGENTIC   │  │ FINAL      │
     │ TOOL CALLS│  │ ANSWER     │
     │           │  └─────┬──────┘
     │ • search  │        │
     │ • expand  │  ┌─────▼──────────┐
     │ • summarize│ │ PROGRAMMATIC   │
     └────────┬──┘  │ WRITES         │
              │     │                │
              │     │ • Conversation │──► SQL Table
              └─────│ • Workflow     │──► Vector Store
                    │ • Entities     │──► Vector Store
                    └────────────────┘
```

## Quick Start

**Prerequisites:** Python 3.10+, Docker, a free HuggingFace token (optional), Tavily API key

```bash
# 1. Start Oracle Database Free in Docker
docker pull ghcr.io/gvenzl/oracle-free:23.26.0
docker run -d --name oracle-free -p 1521:1521 -p 5500:5500 \
  -e ORACLE_PASSWORD=OraclePwd_2025 ghcr.io/gvenzl/oracle-free:23.26.0

# 2. Install dependencies
pip install langchain-oracledb sentence-transformers langchain-openai \
  langchain tavily-python huggingface_hub openai transformers torch

# 3. Run the notebook
jupyter notebook agent_memory.ipynb
```

## Tech Stack

- **Oracle AI Database Free 23ai** — Converged DB with native vector search
- **LangChain OracleVS** — Vector store integration
- **HuggingFace** — LLM inference (cloud API or local Qwen3-0.6B)
- **Tavily** — AI-optimized web search
- **Sentence Transformers** — `paraphrase-mpnet-base-v2` for embeddings
