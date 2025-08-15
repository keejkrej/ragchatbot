# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Setup and Dependencies
```bash
# Install dependencies
uv sync

# Create required .env file
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a **tool-based RAG (Retrieval-Augmented Generation) system** where Claude autonomously decides when to search course content based on query analysis.

### Core Architecture Pattern

The system follows a **coordinated component architecture** where `RAGSystem` acts as the central orchestrator:

```
Frontend (script.js) 
    ↓ POST /api/query
FastAPI (app.py) 
    ↓ rag_system.query()
RAGSystem (rag_system.py) [ORCHESTRATOR]
    ├── DocumentProcessor → processes course docs into chunks
    ├── VectorStore → ChromaDB semantic search
    ├── AIGenerator → Claude API with tool definitions
    ├── SessionManager → conversation history
    └── ToolManager → search tool coordination
```

### Key Architectural Decisions

**Tool-Based Search**: Unlike traditional RAG where search happens on every query, Claude decides whether to use the `search_course_content` tool based on query analysis. This enables both general knowledge responses and course-specific searches.

**Document Processing Pipeline**: Course documents must follow a specific format:
- Line 1: `Course Title: [title]`
- Line 2: `Course Link: [url]`
- Line 3: `Course Instructor: [instructor]`
- Remaining: `Lesson N: [title]` markers with content

**Session State Management**: Conversations are tracked via session IDs with configurable history limits (`MAX_HISTORY=2` in config).

**Context-Enhanced Chunking**: Text chunks include contextual prefixes like "Course X Lesson Y content:" to improve semantic search relevance.

## Component Responsibilities

### RAGSystem (`rag_system.py`)
- **Central coordinator** - all requests flow through here
- Manages component lifecycle and data flow
- Handles query → response orchestration
- Maintains conversation state via SessionManager

### AIGenerator (`ai_generator.py`)
- Claude API integration with tool definitions
- Contains system prompt defining tool usage rules
- Manages conversation history in API calls
- Temperature=0 for consistent responses

### VectorStore (`vector_store.py`)
- ChromaDB wrapper with semantic search
- Uses sentence-transformers (all-MiniLM-L6-v2)
- Supports course name and lesson number filtering
- Returns SearchResults with metadata and distances

### DocumentProcessor (`document_processor.py`)
- Parses structured course documents
- Sentence-based chunking with overlap (800 chars, 100 overlap)
- Creates Course/Lesson/CourseChunk models
- Handles lesson segmentation via regex patterns

### Search Tools (`search_tools.py`)
- Tool abstraction layer for Claude
- CourseSearchTool implements semantic search with filtering
- Tracks sources for response attribution
- Tool definitions in Anthropic's required format

## Configuration (`config.py`)

Key settings in Config dataclass:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters  
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation turns
- `CHROMA_PATH`: "./chroma_db"

## Data Flow Patterns

### Query Processing Flow
1. Frontend sends query + session_id to `/api/query`
2. RAGSystem retrieves conversation history
3. AIGenerator calls Claude with tool definitions
4. Claude autonomously decides whether to use search tool
5. If search needed: CourseSearchTool → VectorStore → ChromaDB
6. Claude synthesizes response from search results
7. Sources tracked and returned with response

### Document Loading (Startup)
1. FastAPI startup event loads `/docs/` folder
2. DocumentProcessor parses each file's structure
3. VectorStore adds course metadata and content chunks to ChromaDB
4. Duplicate courses skipped via title checking

## File Organization

```
backend/
├── app.py              # FastAPI endpoints and startup
├── rag_system.py       # Central orchestrator
├── ai_generator.py     # Claude API integration
├── vector_store.py     # ChromaDB wrapper
├── document_processor.py # Course document parsing
├── search_tools.py     # Tool definitions and execution
├── session_manager.py  # Conversation state
├── models.py          # Pydantic data models
└── config.py          # Configuration settings

frontend/
├── index.html         # Chat interface
├── script.js          # Frontend logic and API calls
└── style.css         # UI styling

docs/                  # Course documents (auto-loaded)
chroma_db/            # Vector database storage
```

## Important Implementation Notes

- **Tool Decision Logic**: Claude uses system prompt rules to determine when course search is needed vs. general knowledge responses
- **Vector Search Filtering**: Search supports both course name partial matching and specific lesson number filtering
- **Session Persistence**: Sessions are in-memory only; restart clears conversation history
- **Course Document Format**: Strict parsing requires proper metadata headers and lesson markers
- **No Testing Framework**: No test commands or test files present in codebase
- **Environment Dependencies**: Requires ANTHROPIC_API_KEY in .env file for functionality
- always use uv to manage packages and run app
- dont run app automatically i'm running it in separate terminal