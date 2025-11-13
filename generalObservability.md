# General Agentic Observability Framework - Design Proposal

## Executive Summary

This document proposes a comprehensive design for transforming the Claude Code Hooks Multi-Agent Observability system into a generic, extensible framework that can observe and monitor any agentic AI system. The current implementation is tightly coupled to Claude Code's hook system, but the core concepts of event capture, centralized storage, and real-time visualization are universally applicable to AI agent observability.

## Current State Analysis

### Strengths
- **Proven Architecture**: HTTP + WebSocket event streaming works well
- **Robust Storage**: SQLite with WAL mode handles concurrent writes
- **Rich Visualization**: Real-time Vue.js dashboard with filtering and charts
- **Event Model**: Comprehensive event structure capturing tool usage, sessions, and conversations
- **Hook Integration**: Deep integration with Claude Code lifecycle events

### Limitations
- **Claude Code Specific**: Relies entirely on Claude Code's proprietary hook system
- **Tight Coupling**: Event schemas and hook scripts are Claude-specific
- **Single Tool Focus**: No abstraction layer for other agentic frameworks
- **Python-Only Hooks**: Hook scripts require Python + uv runtime
- **Hardcoded Event Types**: Event types are specific to Claude Code lifecycle

## Design Principles

1. **Platform Agnostic**: Support any agentic framework (LangChain, AutoGPT, CrewAI, etc.)
2. **Protocol Over Implementation**: Define standard protocols rather than specific implementations
3. **Backward Compatible**: Maintain support for existing Claude Code integration
4. **Extensible**: Easy to add new agent platforms and event types
5. **Language Neutral**: Support instrumentation in any programming language
6. **Minimal Dependencies**: Lightweight clients with optional features
7. **Developer Experience**: Simple to integrate, debug, and extend

## Proposed Architecture

### 1. Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Visualization Layer                       │
│  (Web UI, CLI, Plugins - platform agnostic dashboards)      │
└─────────────────────────────────────────────────────────────┘
                            ↑
┌─────────────────────────────────────────────────────────────┐
│                   Observability Server                       │
│  (Event ingestion, storage, streaming, query API)           │
└─────────────────────────────────────────────────────────────┘
                            ↑
┌─────────────────────────────────────────────────────────────┐
│                   Protocol Layer (HTTP/gRPC)                 │
│  (Standard event format, versioned APIs)                    │
└─────────────────────────────────────────────────────────────┘
                            ↑
┌─────────────────────────────────────────────────────────────┐
│                    Adapter Layer                             │
│  (Platform-specific adapters translating to standard events) │
│  Claude Code | LangChain | AutoGPT | CrewAI | Custom         │
└─────────────────────────────────────────────────────────────┘
                            ↑
┌─────────────────────────────────────────────────────────────┐
│                   Instrumentation Layer                      │
│  (SDK libraries, decorators, middleware for each platform)  │
└─────────────────────────────────────────────────────────────┘
                            ↑
┌─────────────────────────────────────────────────────────────┐
│                     Agent Systems                            │
│  (Running AI agents in various frameworks)                  │
└─────────────────────────────────────────────────────────────┘
```

### 2. Universal Event Schema

Define a standardized event schema that can represent any agent action:

```typescript
interface UniversalAgentEvent {
  // Core identification
  id?: string;
  platform: string;          // "claude-code", "langchain", "autogpt", etc.
  platform_version?: string;
  
  // Source tracking
  source_app: string;
  source_type: string;       // "agent", "tool", "human", "system"
  agent_id: string;          // Unique agent identifier
  session_id: string;        // Session/conversation ID
  parent_event_id?: string;  // For nested/sub-agent events
  
  // Event metadata
  event_type: string;        // Standardized event type
  event_category: string;    // "lifecycle", "tool", "decision", "error", etc.
  timestamp: number;
  duration_ms?: number;
  
  // Event payload (platform-specific)
  payload: {
    action?: string;         // What action was performed
    input?: any;            // Input to the action
    output?: any;           // Output from the action
    metadata?: Record<string, any>;
  };
  
  // Context
  model?: string;           // LLM model used
  tokens?: {
    input?: number;
    output?: number;
    total?: number;
  };
  
  // Optional enrichments
  summary?: string;         // AI-generated summary
  tags?: string[];
  severity?: "debug" | "info" | "warning" | "error" | "critical";
  
  // Conversation/chat context
  chat_history?: ConversationMessage[];
  
  // Human-in-the-loop
  requires_human?: boolean;
  human_response?: any;
  
  // Custom extensions
  extensions?: Record<string, any>;
}

interface ConversationMessage {
  role: "user" | "assistant" | "system" | "tool";
  content: string;
  timestamp?: number;
  metadata?: Record<string, any>;
}
```

### 3. Standard Event Types

Define a taxonomy of standard event types that map across platforms:

**Lifecycle Events:**
- `agent.started`
- `agent.stopped`
- `agent.paused`
- `agent.resumed`
- `session.created`
- `session.ended`

**Tool/Action Events:**
- `tool.invoked`
- `tool.completed`
- `tool.failed`
- `action.planned`
- `action.executed`

**Decision Events:**
- `decision.made`
- `planning.started`
- `planning.completed`
- `reasoning.trace`

**Communication Events:**
- `message.received` (from user)
- `message.sent` (to user)
- `agent.communication` (inter-agent)

**Resource Events:**
- `memory.read`
- `memory.write`
- `context.compacted`
- `token.limit.approaching`

**Error/Warning Events:**
- `error.occurred`
- `warning.issued`
- `retry.attempted`

**Human-in-the-Loop Events:**
- `human.approval.requested`
- `human.input.requested`
- `human.feedback.received`

### 4. Adapter Pattern Implementation

Create platform-specific adapters that translate native events to universal format:

**Claude Code Adapter:**
```python
class ClaudeCodeAdapter:
    def __init__(self, config: AdapterConfig):
        self.platform = "claude-code"
        self.client = ObservabilityClient(config.server_url)
    
    def handle_pre_tool_use(self, hook_data: dict) -> UniversalAgentEvent:
        return UniversalAgentEvent(
            platform="claude-code",
            source_app=self.config.source_app,
            agent_id=hook_data.get("session_id"),
            session_id=hook_data.get("session_id"),
            event_type="tool.invoked",
            event_category="tool",
            timestamp=int(time.time() * 1000),
            payload={
                "action": hook_data.get("tool_name"),
                "input": hook_data.get("tool_input"),
                "metadata": {"hook_type": "PreToolUse"}
            }
        )
    
    def send_event(self, event: UniversalAgentEvent):
        self.client.send(event)
```

**LangChain Adapter:**
```python
from langchain.callbacks.base import BaseCallbackHandler

class LangChainObservabilityCallback(BaseCallbackHandler):
    def __init__(self, config: AdapterConfig):
        self.platform = "langchain"
        self.client = ObservabilityClient(config.server_url)
        self.session_id = str(uuid.uuid4())
    
    def on_chain_start(self, serialized, inputs, **kwargs):
        event = UniversalAgentEvent(
            platform="langchain",
            source_app=self.config.source_app,
            agent_id=kwargs.get("run_id"),
            session_id=self.session_id,
            event_type="action.planned",
            event_category="lifecycle",
            timestamp=int(time.time() * 1000),
            payload={
                "action": serialized.get("name"),
                "input": inputs,
                "metadata": {"chain_type": serialized.get("_type")}
            }
        )
        self.client.send(event)
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        event = UniversalAgentEvent(
            platform="langchain",
            event_type="tool.invoked",
            event_category="tool",
            payload={"action": serialized.get("name"), "input": input_str}
        )
        self.client.send(event)
```

**AutoGPT/CrewAI Adapter:**
```python
class CrewAIAdapter:
    def instrument_agent(self, agent):
        """Monkey-patch or wrap agent methods"""
        original_execute = agent.execute_task
        
        def wrapped_execute(task, *args, **kwargs):
            start_time = time.time()
            event_id = str(uuid.uuid4())
            
            # Send start event
            self.client.send(UniversalAgentEvent(
                platform="crewai",
                event_type="action.executed",
                event_category="tool",
                payload={"action": "execute_task", "input": task}
            ))
            
            try:
                result = original_execute(task, *args, **kwargs)
                # Send completion event
                self.client.send(UniversalAgentEvent(
                    event_type="action.completed",
                    duration_ms=int((time.time() - start_time) * 1000),
                    payload={"output": result}
                ))
                return result
            except Exception as e:
                # Send error event
                self.client.send(UniversalAgentEvent(
                    event_type="error.occurred",
                    severity="error",
                    payload={"error": str(e)}
                ))
                raise
        
        agent.execute_task = wrapped_execute
```

### 5. Multi-Language SDK Support

**Python SDK:**
```python
# observability_sdk/python/client.py
class ObservabilityClient:
    def __init__(self, server_url: str, source_app: str, platform: str):
        self.server_url = server_url
        self.source_app = source_app
        self.platform = platform
        self.session_id = str(uuid.uuid4())
    
    def send_event(self, event_type: str, **kwargs):
        event = {
            "platform": self.platform,
            "source_app": self.source_app,
            "session_id": self.session_id,
            "event_type": event_type,
            "timestamp": int(time.time() * 1000),
            **kwargs
        }
        requests.post(f"{self.server_url}/events", json=event)
    
    @contextmanager
    def track_action(self, action_name: str, **metadata):
        """Context manager for tracking actions"""
        start = time.time()
        event_id = str(uuid.uuid4())
        
        self.send_event("action.started", 
                       event_id=event_id,
                       payload={"action": action_name, "metadata": metadata})
        
        try:
            yield
            self.send_event("action.completed",
                          parent_event_id=event_id,
                          duration_ms=int((time.time() - start) * 1000))
        except Exception as e:
            self.send_event("error.occurred",
                          parent_event_id=event_id,
                          severity="error",
                          payload={"error": str(e)})
            raise
```

**TypeScript/JavaScript SDK:**
```typescript
// observability-sdk/typescript/client.ts
export class ObservabilityClient {
  constructor(
    private serverUrl: string,
    private sourceApp: string,
    private platform: string
  ) {
    this.sessionId = uuidv4();
  }
  
  async sendEvent(eventType: string, options: Partial<UniversalAgentEvent> = {}) {
    const event: UniversalAgentEvent = {
      platform: this.platform,
      source_app: this.sourceApp,
      session_id: this.sessionId,
      event_type: eventType,
      timestamp: Date.now(),
      ...options
    };
    
    await fetch(`${this.serverUrl}/events`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(event)
    });
  }
  
  // Decorator for tracking functions
  trackAction(actionName: string) {
    return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
      const original = descriptor.value;
      
      descriptor.value = async function(...args: any[]) {
        const start = Date.now();
        const eventId = uuidv4();
        
        await this.client.sendEvent('action.started', {
          event_id: eventId,
          payload: { action: actionName, input: args }
        });
        
        try {
          const result = await original.apply(this, args);
          await this.client.sendEvent('action.completed', {
            parent_event_id: eventId,
            duration_ms: Date.now() - start,
            payload: { output: result }
          });
          return result;
        } catch (error) {
          await this.client.sendEvent('error.occurred', {
            parent_event_id: eventId,
            severity: 'error',
            payload: { error: error.message }
          });
          throw error;
        }
      };
      
      return descriptor;
    };
  }
}
```

**Go SDK:**
```go
// observability-sdk/go/client.go
package observability

type Client struct {
    serverURL string
    sourceApp string
    platform  string
    sessionID string
}

func NewClient(serverURL, sourceApp, platform string) *Client {
    return &Client{
        serverURL: serverURL,
        sourceApp: sourceApp,
        platform:  platform,
        sessionID: uuid.New().String(),
    }
}

func (c *Client) SendEvent(eventType string, opts ...EventOption) error {
    event := &UniversalAgentEvent{
        Platform:    c.platform,
        SourceApp:   c.sourceApp,
        SessionID:   c.sessionID,
        EventType:   eventType,
        Timestamp:   time.Now().UnixMilli(),
    }
    
    for _, opt := range opts {
        opt(event)
    }
    
    body, _ := json.Marshal(event)
    resp, err := http.Post(
        c.serverURL+"/events",
        "application/json",
        bytes.NewBuffer(body),
    )
    return err
}

// Middleware for tracking HTTP handlers
func (c *Client) TrackHandler(name string, handler http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        eventID := uuid.New().String()
        
        c.SendEvent("action.started", 
            WithEventID(eventID),
            WithPayload(map[string]interface{}{
                "action": name,
                "method": r.Method,
                "path": r.URL.Path,
            }))
        
        handler(w, r)
        
        c.SendEvent("action.completed",
            WithParentEventID(eventID),
            WithDuration(time.Since(start).Milliseconds()))
    }
}
```

### 6. Integration Methods

**Method 1: Hook-Based (Like Claude Code)**
- Requires platform support for lifecycle hooks
- Deepest integration, most comprehensive data
- Example: Claude Code, some VS Code extensions

**Method 2: Callback-Based**
- Platform provides callback/event system
- Good integration, moderate effort
- Example: LangChain callbacks, OpenAI function calling

**Method 3: Wrapper/Proxy Pattern**
- Wrap platform functions/classes
- Works with most platforms, moderate data capture
- Example: Monkey-patching, decorators, middleware

**Method 4: Instrumentation/Tracing**
- Manual instrumentation in agent code
- Most flexible, requires developer effort
- Example: Custom agents, production systems

**Method 5: Log Parsing/Scraping**
- Parse logs/stdout from agent systems
- Least intrusive, limited data
- Example: Legacy systems, closed-source agents

### 7. Configuration System

**Universal Configuration Format:**
```yaml
# observability.config.yaml
observability:
  # Server connection
  server:
    url: "http://localhost:4000"
    protocol: "http"  # http, grpc, websocket
    api_version: "v1"
    
  # Source identification
  source:
    app_name: "my-agent-system"
    platform: "langchain"
    platform_version: "0.1.0"
    environment: "development"  # development, staging, production
    
  # Event configuration
  events:
    enabled: true
    sampling_rate: 1.0  # 0.0-1.0, for high-volume systems
    batch_size: 10
    flush_interval_ms: 1000
    include_chat_history: true
    max_payload_size_kb: 100
    
  # Event types to capture
  capture:
    lifecycle: true
    tools: true
    decisions: true
    errors: true
    warnings: true
    human_in_loop: true
    
  # Performance
  async: true
  timeout_ms: 5000
  retry_attempts: 3
  
  # Privacy/Security
  redact_patterns:
    - "api[_-]?key"
    - "password"
    - "secret"
    - "token"
  exclude_fields:
    - "payload.raw_credentials"
    
  # Extensions
  plugins:
    - name: "summarizer"
      enabled: true
      config:
        model: "claude-3-haiku"
    - name: "cost-tracker"
      enabled: true
```

### 8. Server Enhancements

**Enhanced API Endpoints:**
```
POST   /v1/events                    # Ingest single event
POST   /v1/events/batch               # Batch ingest
GET    /v1/events                     # Query events
GET    /v1/events/:id                 # Get specific event
GET    /v1/sessions/:id/events        # Get session events
GET    /v1/sessions/:id/summary       # Get session summary
GET    /v1/agents/:id/events          # Get agent events
WS     /v1/stream                     # Real-time event stream
WS     /v1/stream/filtered            # Filtered event stream

# Platform management
GET    /v1/platforms                  # List supported platforms
GET    /v1/platforms/:name/schema     # Get platform event schema

# Analytics
GET    /v1/analytics/sessions         # Session statistics
GET    /v1/analytics/agents           # Agent performance
GET    /v1/analytics/costs            # Token/cost tracking
GET    /v1/analytics/errors           # Error rates

# Export
GET    /v1/export/session/:id         # Export session data
POST   /v1/export/query               # Export query results
```

**Platform registry**
CREATE TABLE platforms (
  name TEXT PRIMARY KEY,
  display_name TEXT,
  version TEXT,
  schema_version TEXT,
  adapter_type TEXT,
  enabled BOOLEAN DEFAULT 1,
  config JSON,
  created_at INTEGER,
  updated_at INTEGER
);

## Conclusion

This design transforms the Claude Code Hooks observability system into a universal framework capable of monitoring any agentic AI system. The key innovations are:

1. **Universal Event Schema**: Platform-agnostic event model
2. **Adapter Pattern**: Clean separation of concerns
3. **Multi-Language SDKs**: Accessibility across ecosystems
4. **Flexible Integration**: Multiple integration methods
5. **Backward Compatible**: Existing Claude Code support maintained

The phased migration approach ensures the project remains functional while evolving into a more powerful, generic solution. The framework will enable developers to gain visibility into AI agents regardless of the underlying platform, making it an essential tool for the emerging agentic AI ecosystem.

## Next Steps

1. **Validate Design**: Review with stakeholders and community
2. **Prototype**: Build proof-of-concept for LangChain adapter
3. **Refactor Server**: Implement universal event schema support
4. **Build SDK**: Create Python SDK with adapter interface
5. **Documentation**: Create getting started guide
6. **Community**: Open source and gather feedback

---

**Document Version**: 1.0  
**Date**: 2025-01-13  
**Status**: Proposal  
**Authors**: AI Assistant + Community
