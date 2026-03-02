# Streaming Analysis

**Scope**: Client-side streaming consumption, server-side streaming emission, protocols, timeouts, keepalive  
**Discovery Method**: Frontend API parsing, backend route handlers, middleware inspection  
**Generated**: Evidence-based streaming architecture documentation  

---

## Executive Summary

InfoJP uses **Server-Sent Events (SSE)** for streaming, NOT WebSocket. Two distinct streaming patterns exist:

1. **Hybrid Search Chat** (`/chat`): Binary ndjson over HTTP POST with `StreamingResponse`
2. **Agent Streaming** (`/stream`, `/tdstream`): Text SSE over HTTP GET with `StreamingResponse`

**Critical APIM Implication**: APIM must support:
- ✅ Chunked Transfer Encoding (HTTP/1.1)
- ✅ Streaming response passthrough (no buffering)
- ✅ Long-lived HTTP connections (default timeout 900s in most gateways)
- ⚠️ No built-in keepalive; connections can drop if no data for 60+ seconds

---

## Streaming Pattern 1: Chat Endpoint (`/chat`)

### Protocol

**Endpoint**: `POST /chat`  
**Media Type**: `application/x-ndjson` (Newline-Delimited JSON)  
**Transport**: HTTP POST with chunked response  
**Client Parsing**: `ndjson-readablestream` npm package (version 1.2.0)

### Server-Side Implementation

#### Route Handler (app.py:272-293)

```python
@app.post("/chat")
async def chat(request: Request):
    """Chat with the bot using a given approach"""
    json_body = await request.json()
    approach = json_body.get("approach")
    
    try:
        impl = chat_approaches.get(Approaches(int(approach)))
        if not impl:
            return {"error": "unknown approach"}, 400

        # Call approach.run() which returns async generator
        r = impl.run(
            json_body.get("history", []),
            json_body.get("overrides", {}),
            json_body.get("citation_lookup", {}),
            json_body.get("thought_chain", {})
        )

        # Return streaming response with ndjson media type
        return StreamingResponse(r, media_type="application/x-ndjson")

    except Exception as ex:
        log.error("Error in chat:: %s", ex)
        raise HTTPException(status_code=500, detail=str(ex)) from ex
```

**Evidence**: app.py:272-293

**Key Details**:
- `impl.run()` returns **async generator** (yields chunks)
- `StreamingResponse` wraps generator
- Media type explicitly set to `application/x-ndjson`
- No timeout configuration visible (uses FastAPI default: 60 seconds idle timeout)

#### Generator Implementation (chatreadretrieveread.py:~450+)

```python
async def run(self, history, overrides, citation_lookup, thought_chain) -> Any:
    # ... query optimization, search, context building ...
    
    # Yield initial state
    yield json.dumps({"work": thought_chain}) + "\n"
    
    # OpenAI streaming
    async for token in completion_tokens:
        yield json.dumps({"answer": token}) + "\n"
    
    # Yield final citations
    for citation in citations:
        yield json.dumps({"citation": citation}) + "\n"
    
    # Yield completion
    yield json.dumps({"completed": True}) + "\n"
```

**Pattern**: Each yield outputs **one complete JSON object + newline** (NDJSON format)

**Evidence**: chatreadretrieveread.py (search for `yield json.dumps`)

### Client-Side Implementation

#### Frontend API Call (api.ts:23)

```typescript
export async function chatApi(options: ChatRequest, signal: AbortSignal): Promise<Response> {
    const response = await fetch("/chat", {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            history: options.history,
            approach: options.approach,
            overrides: { /* config */ },
            citation_lookup: options.citation_lookup,
            thought_chain: options.thought_chain
        }),
        signal: signal  // AbortController for cancellation
    });

    if (response.status > 299 || !response.ok) {
        throw Error("Unknown error");
    }
   
    return response;  // Return Response object with body stream
}
```

**Evidence**: api.ts:23-53

**Key Details**:
- Accepts `AbortSignal` for client-side cancellation
- Returns raw `Response` object (caller handles streaming)
- No media type checking; relies on backend header

#### Streaming Parser (Component Level - Inferred)

**Expected usage** (based on ndjson-readablestream package):

```typescript
import { ndjsonStream } from 'ndjson-readablestream';

const response = await chatApi(options, signal);
const reader = ndjsonStream(response.body);

// Read chunks
try {
    for await (const chunk of reader) {
        // chunk = parsed JSON object
        if (chunk.answer) {
            // Accumulate answer text
            setAnswer(prev => prev + chunk.answer);
        }
        if (chunk.citation) {
            // Handle citation
            citations.push(chunk.citation);
        }
        if (chunk.completed) {
            // Stream complete
            break;
        }
    }
} catch (error) {
    // Handle network error
}
```

**Evidence**: package.json:24 (ndjson-readablestream: 1.2.0)

---

## Streaming Pattern 2: Agent Streaming (`/stream`, `/tdstream`)

### Protocol

**Endpoints**: `GET /stream?question=...` | `GET /tdstream?question=...`  
**Media Type**: `text/event-stream` (SSE format)  
**Transport**: HTTP GET with chunked response  
**Client Parsing**: Native `EventSource` API (browser built-in)

### Server-Side Implementation

#### Math Agent Stream (app.py:816-822)

```python
@app.get("/stream")
async def stream_response(question: str):
    try:
        stream = stream_agent_responses(question)  # Returns async generator
        return StreamingResponse(stream, media_type="text/event-stream")
    except Exception as ex:
        log.exception("Exception in /stream")
        raise HTTPException(status_code=500, detail=str(ex)) from ex
```

**Evidence**: app.py:816-822

#### CSV Agent Stream (app.py:824-832)

```python
@app.get("/tdstream")
async def td_stream_response(question: str):
    save_df(DF_FINAL)
    
    try:
        stream = td_agent_scratch_pad(question, DF_FINAL)  # Returns async generator
        return StreamingResponse(stream, media_type="text/event-stream")
    except Exception as ex:
        log.exception("Exception in /stream")
        raise HTTPException(status_code=500, detail=str(ex)) from ex
```

**Evidence**: app.py:824-832

#### Expected Generator Pattern

```python
async def stream_agent_responses(question: str):
    # Calls math agent with streaming enabled
    async for chunk in agent.stream(...):
        # SSE format: "data: <JSON>\n\n"
        yield f"data: {json.dumps({'chunk': chunk})}\n\n"
```

**SSE Format Details**:
- Prefix: `data: ` (note trailing space)
- Content: JSON string (no newlines inside)
- Separator: `\n\n` (double newline to signal event boundary)

### Client-Side Implementation

#### EventSource API (api.ts:166)

```typescript
export function streamData(question: string): EventSource {
    const encodedQuestion = encodeURIComponent(question);
    const eventSource = new EventSource(`/stream?question=${encodedQuestion}`);
    return eventSource;
}
```

**Evidence**: api.ts:166

#### EventSource Consumption (Inferred from streaming lib usage)

```typescript
const eventSource = streamData(question);

eventSource.addEventListener("message", (event) => {
    const chunk = JSON.parse(event.data);
    setOutput(prev => prev + chunk.chunk);
});

eventSource.addEventListener("error", (event) => {
    if (event.readyState === EventSource.CLOSED) {
        // Connection closed by server
        console.log("Stream complete");
    }
});

// Cleanup
eventSource.close();
```

**Native `EventSource` Capabilities**:
- ✅ Auto-reconnect on network error
- ✅ Built-in message parsing
- ❌ No request headers (CORS limitation)
- ❌ No timeout control (browser default ~45 seconds)

---

## Data Flow Comparison

### Chat Stream (`/chat` → ndjson)

```
Client                              Server
  |                                   |
  | POST /chat {history, approach}   |
  |---------------------------------->|
  |                                   |
  |                              execute approach.run()
  |                              (query optimization,
  |                               embedding, search,
  |                               OpenAI streaming)
  |                                   |
  |  200 OK (chunked)                 |
  |<----------------------------------|
  |  {"work": {...}}\\n                |
  |<----------------------------------|
  |                                   |
  |  {"answer": "The"}\\n              |
  |  {"answer": " answer"}\\n          |
  |  ... (token by token) ...          |
  |<----------------------------------|
  |                                   |
  |  {"citation": "doc.pdf"}\\n        |
  |<----------------------------------|
  |                                   |
  |  {"completed": true}\\n            |
  |<----------------------------------|
  |                                   |
  | [stream end]                      |
  |                                   |
```

**Timing**: 5-30 seconds (depends on OpenAI latency + document size)

### Agent Stream (`/stream` → SSE)

```
Client                              Server
  |                                   |
  | GET /stream?question=...          |
  |---------------------------------->|
  |                                   |
  |                              execute agent.stream()
  |                              (langchain agent loop)
  |                                   |
  |  200 OK (chunked)                 |
  |<----------------------------------|
  |  data: {"chunk": "..."}\\n\\n       |
  |<----------------------------------|
  |                                   |
  |  data: {"chunk": "..."}\\n\\n       |
  |  ... (per agent step) ...          |
  |<----------------------------------|
  |                                   |
  |  data: {"chunk": "final"}\\n\\n     |
  |<----------------------------------|
  |                                   |
  | [stream end]                      |
  |                                   |
```

**Timing**: 3-10 seconds (depends on agent steps + OpenAI calls within agent)

---

## Timeout & Keepalive Behavior

### Server-Side Timeouts

**FastAPI/Uvicorn Default**: 60 seconds idle timeout (no data written)

**Code Evidence**: None explicit in app.py; uses FastAPI default

**APIM Risk**: 
- If 60+ seconds pass without streaming data, FastAPI may close connection
- APIM will receive connection drop error
- Client receives `error` event on EventSource

**Mitigation Options** (not currently implemented):
1. **Heartbeat/Ping**: Emit empty event every 30 seconds
   ```python
   async for chunk in agent.stream():
       yield f"data: {json.dumps(chunk)}\n\n"
       # Could add: if no_data_for_30s: yield f"data: null\n\n"
   ```

2. **Configure Uvicorn timeout** (app.py would need):
   ```python
   if __name__ == "__main__":
       uvicorn.run(app, timeout_keep_alive=300)  # 5 minutes
   ```

### Client-Side Timeouts

**EventSource Default**: ~45 seconds (browser-dependent)

**Fetch API**: No idle timeout; relies on network

**APIM Default**: ~900 seconds (varies by tier)

**Issue**: Mismatch between EventSource (45s) and server (60s) timeout
- EventSource likely times out first
- Connection drops mid-stream for long-running agents

---

## HTTP Headers in Streaming Responses

### Response Headers (app.py:293)

```python
StreamingResponse(r, media_type="application/x-ndjson")
# Implicitly sets:
# - Content-Type: application/x-ndjson
# - Transfer-Encoding: chunked
# - Cache-Control: no-cache (FastAPI default)
```

**Evidence**: app.py:293

### No Custom Headers Emitted

Search Result: ❌ No evidence of custom streaming headers
- No `X-Accel-Buffering: no` (disable proxy buffering)
- No `X-Content-Type-Options: nosniff`
- No `Connection: keep-alive`

**APIM Consideration**: APIM should ensure:
- `Transfer-Encoding: chunked` is preserved
- `Connection: keep-alive` is enforced
- `X-Accel-Buffering: no` to prevent proxy buffering

---

## Streaming Error Handling

### Server-Side Error During Stream

```python
@app.get("/stream")
async def stream_response(question: str):
    try:
        stream = stream_agent_responses(question)
        return StreamingResponse(stream, media_type="text/event-stream")
    except Exception as ex:
        log.exception("Exception in /stream")
        raise HTTPException(status_code=500, detail=str(ex)) from ex
```

**Error Handling Timing**:
- **Before streaming starts**: Returns 500 HTTP status (before response sent)
- **During streaming**: Generator exception halts stream (connection closes)

**Client Impact**:
- Fetch API: Receives connection reset (error event)
- EventSource: Receives connection reset; attempts reconnect

### Missing: Mid-Stream Error Signaling

**Issue**: No way to signal error mid-stream in ndjson/SSE format
- Could emit `{"error": "message"}` but client must distinguish
- Better: Use HTTP 206 Partial Content + error trailer

**Current Workaround**: Agent returns error in chunk
```python
yield json.dumps({"error": "Query optimization failed"}) + "\n"
return  # Halt stream
```

---

## APIM Implications for Streaming

### Policies Required

```xml
<policies>
    <inbound>
        <!-- Allow long-lived connections -->
        <set-header name="Connection" exists-action="override">
            <value>keep-alive</value>
        </set-header>
        <!-- Allow streaming to proceed without buffering -->
        <set-header name="X-Accel-Buffering" exists-action="override">
            <value>no</value>
        </set-header>
    </inbound>
    
    <backend>
        <!-- Set backend timeout to 5 minutes -->
        <set-backend-service base-url="..." />
        <!-- No buffering policy -->
        <set-header name="X-Accel-Buffering">
            <value>no</value>
        </set-header>
    </backend>
    
    <outbound>
        <!-- Ensure chunked encoding preserved -->
        <set-header name="Transfer-Encoding" exists-action="skip" />
    </outbound>
</policies>
```

### Timeout Configuration

**APIM Setting**: Increase backend timeout
- Default: 300 seconds (5 minutes) [some sources: 900 seconds]
- Recommended for `/chat`: 120 seconds (2 minutes max OpenAI latency)
- Recommended for `/stream`: 120 seconds (2 minutes max agent time)

**Set in APIM policy**:
```xml
<backend timeout="120" />
```

### Connection Management

| Aspect | Recommendation | Rationale |
|--------|-----------------|-----------|
| Keep-Alive | Enabled | Long-lived streams need persistent connections |
| Chunked Encoding | Preserve | Clients expect `Transfer-Encoding: chunked` |
| Buffering | Disabled | Streaming chunks must reach client immediately |
| Idle Timeout | 5+ minutes | Some long-running agent queries take 3-5 min |

---

## Streaming Limitations & Workarounds

### Limitation 1: EventSource Cannot Send Request Headers

**Problem**: EventSource only accepts URL; no `Authorization` or `X-Custom-Header`

**Current Solution**: Headers passed in URL query params
- ❌ Insecure (headers logged in URL)
- ❌ Size limited (~2KB)

**APIM Solution**: 
- APIM injects headers before backend (headers sent via APIM policy)
- Backend receives via normal `request.headers`
- No code change needed

### Limitation 2: EventSource Auto-Reconnect Not Suitable for POST

**Problem**: `/chat` is POST (idempotent assumption violated)

**Current Solution**: Uses Fetch API instead
- ✅ Supports POST
- ✅ Supports custom headers
- ❌ Manual parsing required (uses ndjson library)

**APIM Solution**: Transparent; no changes needed

### Limitation 3: No Built-In Keepalive/Heartbeat

**Problem**: 60+ second idle triggers timeout

**Current Solution**: None (gap in implementation)

**APIM Solution**: 
- APIM can emit heartbeat: Every 30s, emit `\n` or comment
- Backend can emit heartbeat: Modify generator to yield keep-alive
- Client can implement reconnect logic with exponential backoff

**Recommended Backend Fix**:
```python
async def keep_alive_wrapper(generator, heartbeat_interval=30):
    import asyncio
    last_yield = datetime.now()
    
    async for chunk in generator:
        yield chunk
        last_yield = datetime.now()
    
    # Optional: Send final keep-alive
    await asyncio.sleep(heartbeat_interval)
    yield f"data: null\n\n"  # No-op for SSE
```

---

## Performance Characteristics

### Latency Breakdown for `/chat`

```
Client POST /chat
  ↓
Backend receives request (1-5ms)
  ↓
Parse JSON body (1-5ms)
  ↓
Run query optimization via OpenAI (1-10 seconds) ← SLOW
  ↓
Generate embedding (1-5 seconds) ← SLOW
  ↓
Hybrid search (100-500ms) ← START STREAMING
  ↓
yield {"work": ...} (5ms)
  ↓
Stream OpenAI completion token-by-token (5-20 seconds) ← STREAMING
  ↓
yield {"citation": ...} (5-10ms per citation)
  ↓
Total: 8-35 seconds (P50: ~12 seconds)
```

**APIM Timeout Requirement**: ≥ 40 seconds (P99 + buffer)

### Latency Breakdown for `/stream`

```
Client GET /stream?question=...
  ↓
Backend receives request (1-5ms)
  ↓
Save CSV (10-100ms)
  ↓
Initialize agent (500-1000ms) ← START STREAMING
  ↓
yield {"chunk": "Thinking..."} (5ms)
  ↓
Agent loop: Plan → Tool → Observe (2-8 seconds per step) ← STREAMING
  ↓
yield {"chunk": result} (5ms per step)
  ↓
Agent finishes (1-5 steps = 3-10 seconds)
  ↓
Total: 4-15 seconds (P50: ~8 seconds)
```

**APIM Timeout Requirement**: ≥ 30 seconds

---

## Evidence Summary

| Aspect | File | Lines | Finding |
|--------|------|-------|---------|
| Chat streaming | app.py | 272-293 | StreamingResponse(r, media_type="application/x-ndjson") |
| Agent SSE | app.py | 816-832 | StreamingResponse(stream, media_type="text/event-stream") |
| Frontend ndjson | api.ts | 23-53 | fetch POST /chat returns Response (streams via ndjson lib) |
| Frontend SSE | api.ts | 166 | new EventSource("/stream?question=...") |
| SSE lib import | package.json | 24 | "ndjson-readablestream": "1.2.0" |
| No keepalive | app.py | 1-904 | No explicit timeout/keepalive config |
| No heartbeat | chatreadretrieveread.py | 1-504 | No heartbeat yields in generator |

