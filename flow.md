# Request Flow

## Sequence Diagram

```mermaid
sequenceDiagram
    actor U as User
    actor C as Webhook Sender
    participant CLI as Whook CLI
    participant Con as Conductor
    participant R as Relay Server

    Note over U,R: Phase 1: Setup

    U->>CLI: whook start
    Note over CLI: Create new project if needed


    CLI->>Con: POST api.whook.dev/tunnel<br/>{"project": "my-project"}
    activate Con
    Note over Con: Find available relay server
    Con-->>CLI: {"websocket_url": "wss://relay-1.whook.dev/tunnel",<br/>"token": "abc123", "project_id": "proj_123"}
    deactivate Con

    CLI->>R: Connect to WebSocket<br/>wss://relay-1.whook.dev/tunnel?token=abc123
    activate R
    R-->>CLI: WebSocket connection established

    CLI->>U: ✓ Tunnel ready!<br/>https://my-project.whook.dev

    Note over U,R: Phase 2: Request Handling

    C->>Con: POST my-project.whook.dev
    activate Con
    Con->>Con: Look up project's relay server
    Con->>R: Forward request
    R->>CLI: Send request over WebSocket

    CLI-->>R: Send response over WebSocket
    R-->>Con: Forward response
    Con-->>C: Return response
    CLI-->>U: Display request
    deactivate Con
    deactivate R
```

### Phase 1: Initial Setup

1. User starts whook CLI
2. CLI requests a tunnel from the conductor:

```
POST api.whook.dev/tunnel
{"project": "my-project"}
```

3. Conductor assigns a relay server and returns connection details

```json
{
  "websocket_url": "wss://relay-1.whook.dev/tunnel",
  "token": "abcd123",
  "project_id": "my-project"
}
```

4. CLI establishes WebSocket connection with the assigned relay server
5. System is ready to receive webhooks

### Phase 2: Webhook Handling

1. External service sends webhook to `my-project.whook.dev`
2. Request flows through system:
   - Conductor recevies request and identfiies appropriate relay server
   - Relay server forwards request through established WebSocket
   - CLI recevies webhook data and:
     - Displays it in the terminal interface
     - Optionally forwards it to a local development server
3. Response flows back through the same path to the original sender
