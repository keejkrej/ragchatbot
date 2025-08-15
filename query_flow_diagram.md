# User Query Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend<br/>(script.js)
    participant FastAPI as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant Session as Session Manager<br/>(session_manager.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant Claude as Claude API<br/>(Anthropic)
    participant Tools as Search Tools<br/>(search_tools.py)
    participant Vector as Vector Store<br/>(vector_store.py)
    participant Chroma as ChromaDB<br/>(Database)

    User->>Frontend: Types query & clicks send
    Frontend->>Frontend: Disable input, show loading
    Frontend->>Frontend: Add user message to chat
    
    Frontend->>FastAPI: POST /api/query<br/>{"query": "...", "session_id": "..."}
    
    FastAPI->>FastAPI: Validate QueryRequest
    FastAPI->>FastAPI: Create session if needed
    
    FastAPI->>RAG: query(query, session_id)
    
    RAG->>RAG: Format prompt:<br/>"Answer this question about course materials: {query}"
    
    RAG->>Session: get_conversation_history(session_id)
    Session-->>RAG: Return chat history
    
    RAG->>AI: generate_response(<br/>query, history, tools, tool_manager)
    
    AI->>Claude: API call with:<br/>- System prompt<br/>- User query<br/>- Tool definitions<br/>- Conversation history
    
    Note over Claude: Claude decides if search needed<br/>based on query type
    
    alt Claude needs to search
        Claude->>AI: Tool call: search_course_content
        AI->>Tools: execute(query, course_name?, lesson_number?)
        Tools->>Vector: search(query, filters)
        Vector->>Chroma: Semantic search in embeddings
        Chroma-->>Vector: Return relevant chunks
        Vector-->>Tools: SearchResults with chunks
        Tools->>Tools: Track sources for attribution
        Tools-->>AI: Return formatted search results
        AI-->>Claude: Provide search results
    end
    
    Claude-->>AI: Generated response text
    
    AI->>Tools: get_last_sources()
    Tools-->>AI: Return sources list
    
    AI-->>RAG: Return (response, sources)
    
    RAG->>Session: add_exchange(session_id, query, response)
    
    RAG-->>FastAPI: Return (response, sources)
    
    FastAPI->>FastAPI: Create QueryResponse model
    FastAPI-->>Frontend: JSON response:<br/>{"answer": "...", "sources": [...], "session_id": "..."}
    
    Frontend->>Frontend: Remove loading animation
    Frontend->>Frontend: Parse markdown & render response
    Frontend->>Frontend: Display collapsible sources
    Frontend->>Frontend: Re-enable input
    Frontend->>Frontend: Update session_id if new
    
    Frontend-->>User: Display AI response with sources
```

## Key Components Flow

### 1. **Frontend Layer** (`script.js`)
- User interaction handling
- API communication
- UI state management
- Response rendering

### 2. **API Layer** (`app.py`)
- Request validation
- Session management
- Error handling
- Response formatting

### 3. **RAG Orchestration** (`rag_system.py`)
- Coordinates all components
- Manages prompt formatting
- Handles conversation flow

### 4. **AI Processing** (`ai_generator.py`)
- Claude API integration
- Tool-based search decisions
- Response generation

### 5. **Search & Retrieval** (`search_tools.py` + `vector_store.py`)
- Semantic search execution
- Source attribution
- Content filtering

### 6. **Data Storage** (ChromaDB)
- Vector embeddings storage
- Similarity search
- Course content retrieval

## Tool-Based Search Decision Tree

```mermaid
flowchart TD
    A[User Query] --> B{Claude Analyzes Query}
    B -->|General Knowledge| C[Direct Response<br/>No Search]
    B -->|Course-Specific| D[Execute search_course_content]
    D --> E[Vector Search in ChromaDB]
    E --> F[Return Relevant Chunks]
    F --> G[Claude Synthesizes Response]
    C --> H[Final Response]
    G --> H[Final Response]
    H --> I[Display to User with Sources]
```