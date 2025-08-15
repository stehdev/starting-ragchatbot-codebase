# RAG Chatbot Query Processing Flow

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend (script.js)
    participant API as FastAPI (app.py)
    participant RAG as RAG System
    participant SM as Session Manager
    participant AI as AI Generator
    participant TM as Tool Manager
    participant ST as Search Tool
    participant VS as Vector Store
    participant DB as ChromaDB

    %% User initiates query
    U->>FE: Types query / clicks suggestion
    FE->>FE: Disable input, show loading
    FE->>FE: Add user message to chat

    %% Frontend to API
    FE->>API: POST /api/query<br/>{query, session_id}
    
    %% API to RAG System
    API->>RAG: rag_system.query(query, session_id)
    
    %% RAG System orchestration
    RAG->>SM: get_conversation_history(session_id)
    SM-->>RAG: Previous messages context
    
    RAG->>AI: generate_response(query, history, tools, tool_manager)
    
    %% AI Processing with Claude
    AI->>AI: Build system prompt + context
    AI->>AI: Call Claude API with tools
    
    %% Claude decides to use search tool
    Note over AI: Claude returns tool_use request
    AI->>TM: execute_tool("search_course_content", params)
    TM->>ST: execute(query, course_name, lesson_number)
    
    %% Search execution
    ST->>VS: search(query, course_name, lesson_number)
    VS->>VS: _resolve_course_name() if needed
    VS->>VS: _build_filter() for course/lesson
    VS->>DB: query with semantic search
    DB-->>VS: Raw results with embeddings
    VS-->>ST: SearchResults with documents/metadata
    
    ST->>ST: _format_results() with context headers
    ST->>ST: Store sources for UI
    ST-->>TM: Formatted search results
    TM-->>AI: Tool execution results
    
    %% Final response generation
    AI->>AI: Send tool results back to Claude
    AI->>AI: Claude generates final answer
    AI-->>RAG: Generated response text
    
    %% Source handling and session update
    RAG->>TM: get_last_sources()
    TM-->>RAG: Sources list for UI
    RAG->>TM: reset_sources()
    RAG->>SM: add_exchange(session_id, query, response)
    
    RAG-->>API: (response_text, sources_list)
    API-->>FE: JSON: {answer, sources, session_id}
    
    %% Frontend display
    FE->>FE: Remove loading animation
    FE->>FE: Add assistant message with sources
    FE->>FE: Re-enable input, scroll to bottom
    FE-->>U: Display answer with sources

    %% Error handling paths (dotted)
    Note over VS,DB: If search fails
    VS-->>ST: SearchResults.empty(error_msg)
    ST-->>AI: "No results found" message
    
    Note over API: If any error occurs
    API-->>FE: HTTP 500 with error detail
    FE->>FE: Show error message to user
```

## Key Components Breakdown

### Frontend Layer (script.js)
- **User Interface**: Input handling, message display, loading states
- **API Communication**: HTTP POST to `/api/query` endpoint
- **Session Management**: Tracks `currentSessionId` for conversation continuity

### API Layer (app.py) 
- **Request Handling**: FastAPI endpoint with Pydantic models
- **Session Creation**: Generates new session if none provided
- **Response Formatting**: Structures data for frontend consumption

### RAG Orchestration (rag_system.py)
- **Component Coordination**: Manages AI, search, and session components
- **Context Building**: Combines query with conversation history
- **Source Management**: Collects and resets search sources

### AI Generation (ai_generator.py)
- **Claude Integration**: Anthropic API calls with system prompts
- **Tool Orchestration**: Handles tool use requests from Claude
- **Response Synthesis**: Combines search results into coherent answers

### Search System (search_tools.py + vector_store.py)
- **Semantic Search**: ChromaDB vector similarity search
- **Smart Filtering**: Course name resolution and lesson filtering
- **Context Enhancement**: Adds course/lesson metadata to results

### Data Flow Characteristics
- **Asynchronous**: Frontend uses async/await for non-blocking UI
- **Stateful**: Session manager maintains conversation context
- **Tool-Driven**: Claude autonomously decides when to search
- **Error Resilient**: Multiple fallback layers for graceful degradation