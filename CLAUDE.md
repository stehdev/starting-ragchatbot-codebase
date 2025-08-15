# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Setup Commands
```bash
# Install dependencies
uv sync

# Set up environment variables (required)
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a RAG (Retrieval-Augmented Generation) system for course materials built with FastAPI backend and vanilla JavaScript frontend.

### Core Architecture Pattern

The system follows a **tool-based RAG pattern** where:
1. **Claude AI autonomously decides** when to search course materials using tool calling
2. **Structured document processing** parses course documents into hierarchical chunks (course → lesson → content)
3. **Vector storage** enables semantic search through ChromaDB with sentence transformers
4. **Session management** maintains conversation context across queries

### Key Components

**Backend (`backend/`)**:
- `app.py` - FastAPI server serving both API endpoints and static frontend
- `rag_system.py` - Main orchestrator coordinating all components
- `ai_generator.py` - Anthropic Claude integration with tool execution handling
- `document_processor.py` - Parses structured course documents into searchable chunks
- `vector_store.py` - ChromaDB vector database with semantic search
- `search_tools.py` - Tool definitions for Claude's autonomous search capabilities
- `session_manager.py` - Conversation history management
- `config.py` - Centralized configuration with environment variable loading

**Document Processing Pipeline**:
Course documents must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [lesson_url]
[lesson content...]
```

Documents are processed into:
1. **Metadata extraction** (course title, instructor, links)
2. **Lesson structure parsing** (lesson numbers, titles, content)
3. **Sentence-based chunking** (800 chars with 100 char overlap)
4. **Context enhancement** (each chunk includes course/lesson metadata)

### Query Processing Flow

1. **Frontend** sends query via `/api/query` endpoint
2. **RAG System** builds prompt with conversation history
3. **AI Generator** calls Claude with available search tools
4. **Claude decides** whether to use `search_course_content` tool
5. **Search Tool** queries vector store with optional course/lesson filters
6. **Vector Store** performs semantic search and returns contextual results
7. **Claude synthesizes** final response from search results
8. **Frontend** displays answer with source attribution

### Configuration

Key settings in `config.py`:
- `CHUNK_SIZE: 800` - Text chunk size for vector storage
- `CHUNK_OVERLAP: 100` - Character overlap between chunks
- `MAX_RESULTS: 5` - Maximum search results returned
- `MAX_HISTORY: 2` - Conversation messages to remember
- `ANTHROPIC_MODEL: claude-sonnet-4-20250514`
- `EMBEDDING_MODEL: all-MiniLM-L6-v2`

### Database Storage

Uses ChromaDB with two collections:
- `course_catalog` - Course metadata for name resolution
- `course_content` - Searchable content chunks with course/lesson context

Course documents are auto-loaded from `docs/` folder on startup.

### Tool System

Claude has access to `search_course_content` tool with parameters:
- `query` (required) - What to search for
- `course_name` (optional) - Course filter with fuzzy matching
- `lesson_number` (optional) - Specific lesson filter

The tool system enables Claude to autonomously search when needed rather than always searching for every query.
- always to use uv to run the server do not use pip directly
- make sure to use uv to manage all dependencies