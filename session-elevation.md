Created: 2025-09-13
Drafted by: Larry Maccherone (@lmaccherone)
Extention of: SEP-1364 (Shaun Smith @evalstate, Kurtis Van Gent @kurtisvg, Jonathan Hefner @jonathanhefner)
Status: Discussion-only pre-draft 

---

# MCP Session Elevation

## Abstract

This proposal builds on SEP-1364 Elevating MCP Sessions alternative 1, by adding a session resume capability. With this, the WebSocket SEP-1287/1288 can become as simple as the stdio transport specification.

It's also written to be compatible with SEP-1442 Make MCP Stateless (by default) which aims to remove initialization. It's try-and-error approach is similarly compatible if initialization remains required or if it becomes optional.

This proposal:

- Elevates sessions from the Streamable HTTP transport to the protocol layer and thus makes them available to all transports.
- Allows clients to maintain multiple sessions per server. For example, one session per context window.
- Introduces three new methods: `session/start`, `session/end`, and `session/resume`, along with standardized session identifiers.
- Enables clients to resume interrupted sessions on the same transport or on a different transport with catch-up and coalescing semantics.

---

## Specification

### What is a Session?
A session is a collection of related interactions between a client and server. Examples of things that might be scoped to a session include:
- Context for an ongoing chat or tool invocation thread
- Subscriptions initiated via `resources/subscribe` 
- Elicitation workflows

Sessions are **OPTIONAL**. In their absence, servers **MAY** continue to scope them as they did previously (per-client, per-connection, globally, etc.). When used, sessions provide a clear unit of scoping and resumption.

### New Methods

#### `session/start`
- A client **MAY** use this method to initiate a new session.
- If the server supports protocol-level sessions, it **MUST** respond with a unique `sessionId`.
- If the server does not support protocol-level sessions, it **MUST** respond with a method not found error.

**Session Event IDs**  
- If the server supports catch-up semantics (see below), it **MUST** begin numbering (or otherwise generating) `sessionEventId` values for all subsequent session-scoped notifications;
- This `sessionEventId` **MUST** be a unique (within that session or globally), monotonically increasing string or integer
- It **MAY** include the sessionId as a prefix for tracing and clarity.

#### `session/end`
- A client **MAY** use this command to explicitly terminate a session.
- Upon receiving this command, the server **SHOULD** release any associated resources.
- The server **MUST NOT** deliver further session-scoped notifications after `session/end`.
 
#### `session/resume`
- A client **MAY** use this method to resume a previously interrupted session by providing a known `sessionId`.
- If the session is unknown or has expired, the server **MUST** respond with an error.
- If the session is valid, the server **MUST** make this connection the only active connection for the session and reply indicating the outcome (see below).

#### Optional catch-up semantics
- Servers **MAY** support catch-up semantics on resume using a bounded replay window.
- To catch up, the client **MUST** provide a `lastSessionEventId` indicating the last event it successfully processed.
- If the session is within the server’s replay window, the server **MUST** start sending missed messages beginning immediately after `lastSessionEventId` and indicate `catchup: true` in the response.
- If the session is outside the replay window, the server **MUST** indicate `catchup: false` in the response; the client **MAY** use another mechanism to rehydrate state (e.g., `resources/get`).

#### Coalescing semantics on resume
- When resuming, servers **SHOULD** consider the semantics of missed communications and coalesce where appropriate. For example, multiple `notifications/resources/updated` for the same resource **SHOULD** be collapsed to a single notification.

### Cross-Transport Resumption
- A `session/resume` method **MAY** be used to resume a session across different transports (e.g., Streamable HTTP → WebSocket).

---

## JSON-RPC Examples

#### 1) Session Start
```json
Request:
{ "id": 1, "method": "session/start", "params": {} }
```

Response:
```json
{ "id": 1, "result": { "sessionId": "s_abc123" } }
```

#### 2) Session Resume
Request (with catch-up):
```json
{
  "id": 2,
  "method": "session/resume",
  "params": {
    "sessionId": "s_abc123",
    "lastSessionEventId": "se_305"
  }
}
```

Response (resumed and within replay window):
```json
{
  "id": 2,
  "result": {
    "resumed": true,
    "catchup": true
  }
}
```

Coalesced notification after resume (instead of replaying se_306, se_307, se_308 individually for the same resource, the server coalesces):
```json
{
  "method": "notifications/resources/updated",
  "params": {
    "sessionEventId": "se_310",
    "uri": "doc:/foo"
  }
}
```

Response (resumed but outside replay window):
```json
{
  "id": 2,
  "result": {
    "resumed": true,
    "catchup": false
  }
}
```
Client next step: rehydrate state out-of-band (e.g., call `resources/get` for relevant URIs), then continue processing live events.

#### 3) Session End
Request:
```json
{ "id": 3, "method": "session/end", "params": { "sessionId": "s_abc123" } }
```

Response:
```json
{ "id": 3, "result": {} }
```

---

## Rationale and Alternatives Considered

### 1. Location of Session Management (Transport vs. Protocol)
Alternative: Rely on transport-layer features (e.g., Streamable HTTP headers)  
- Streamable HTTP currently supports resumability via headers like Mcp-Session-Id and Last-Event-ID.  
- However, many transports (WebSockets, stdio) lack a per-message side-channel which means they cannot utilize the same capabilities 
- The transport layer should be blind to protocol semantics but aspects of sessions and catch-up semantics are inherently intertwined with protocol-level concepts.
- For example: The transport cannot recognize that three notifications/resources/updated could be collapse to a single notification. Transport-level replay is potentially redundant, especially for idempotent or state-overwriting notifications.  
**Decision: Elevate sessions to protocol-level.**

### 2. Sessions in Initialization (SEP-1287 and SEP-1364 alternative 2)
Alternative: Bundle session creation into initialize  
 - The Transport WG is pursuing removing initialization or making it optional in order to make MCP stateless by default (SEP-1442).
**Decision: Avoid tying sessions to initialize.**

### 3. Structure/Placement of Session Fields
Alternative: Message envelope to carry session fields on every message  
- Unnecessary overhead
Alternative: Include session fields inside _meta
- Too hidden for an interop-critical feature
**Decision: Make sessions a first-class concept and do not use an envelope.**

### 4. Handling Lack of Notification Ids
Problem: JSON-RPC notifications have no id, complicating precise replay tracking.    
Alternative: Upgrade notifications to calls
- Too disruptive to change the methods to not start with `notifications/`
- Too confusing to leave them starting with `notifications/` if they are no longer JSON-RPC notifications.
**Decision: Keep notifications as JSON-RPC notifications and add a stable sessionEventId field for ordering and resumption.**

---

## Security Considerations

This proposal introduces no new security risks. Clients must already be authenticated to invoke session/* methods; resumption therefore inherits existing authentication and authorization. Using sessionEventId as a replay position does not expose additional attack surface beyond normal session identifiers.

On the contrary, sessions may serve as a foundation for least-privilege capability scoping--a server may bind limited permissions to a session thereby reducing blast radius relative to transport-scoped state.

---

## Backward Compatibility

The new methods are optional. If a client invokes `session/*` on a server that does not implement them, the server **MUST** return an error. Clients that do not use sessions continue to operate unchanged.

### Migration Path (Streamable HTTP ↔ Protocol-Level Sessions)

This proposal is an alternative to Streamable HTTP’s Mcp-Session-Id and Last-Event-ID. Servers **MAY** support both, but a client **MUST** choose exactly one mode per connection.

Existing Streamable HTTP deployments can adopt this proposal incrementally:

- Dual support: Continue honoring Mcp-Session-Id / Last-Event-ID while also implementing session/* methods.
- Coalescing upgrade: Use MCP-level semantics (e.g., notifications/resources/updated coalescing) when resuming via sessionEventId, even if legacy clients still depend on transport headers.
- Eventual deprecation (optional): Over time, prefer protocol-level sessions for all transports; potentially deprecate transport-specific session support in a later version of MCP.

#### Mode Selection and Conflict Handling

- Modes (mutually exclusive per connection):
  - Transport mode: Client uses Mcp-Session-Id and/or Last-Event-ID; client does not invoke `session/*`.
  - Protocol mode: Client invokes `session/*` and uses `sessionEventId`.
  - No-session mode: Neither mechanism is used.

- Mode selection (first-wins):
  - If the first session-scoped action is a `session/start`, the connection enters protocol mode. Any subsequent request carrying Mcp-Session-Id and/or Last-Event-ID on this connection **MUST** be rejected with an error.
  - If the first request arrives with Mcp-Session-Id and/or Last-Event-ID, the connection enters transport mode. The server **MUST NOT** accept `session/*` on that connection and **MUST** return an error if attempted.

- Conflicts:
  - If both Last-Event-ID and `lastSessionEventId` are supplied (e.g., via mixed client behavior), the server **MUST** reject with an error rather than guess precedence.
