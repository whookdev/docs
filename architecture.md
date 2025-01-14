# Proposed Architecture

## Component Overview

```mermaid
flowchart TD
    subgraph external[External]
        CLIs[Whook CLIs]
        WS[Webhook Senders]
    end

    subgraph cloud[Cloud Infrastructure]
        DNS[DNS *.whook.dev]
        LB[Load Balancer]

        subgraph auth[Auth Service]
            Auth[Authentication]
            AuthDB[(Auth Database)]
        end

        subgraph conductor_service[Conductor Service]
            C1[Conductor Instance 1]
            C2[Conductor Instance 2]
            C3[Conductor Instance 3]
        end

        subgraph data[Data Layer]
            Redis[(Redis)]
            PG[(PostgreSQL)]
            MQ[(Message Queue)]
        end

        subgraph relay_servers[Relay Servers]
            R1[Relay Server 1]
            R2[Relay Server 2]
            R3[Relay Server 3]
        end

        subgraph observability[Observability]
            Prom[Prometheus]
            Graf[Grafana]
            Logs[Logging System]
            Errors[Error Tracking]
        end
    end

    subgraph local[Local Development]
        DEV[Development Server]
    end

    %% DNS routing
    WS --> DNS
    CLIs --> DNS
    DNS --> LB

    %% Authentication flow
    LB --> Auth
    Auth --> AuthDB

    %% Load balancer to conductor
    LB --> C1
    LB --> C2
    LB --> C3

    %% Conductor to data stores
    C1 & C2 & C3 --> Redis
    C1 & C2 & C3 --> PG
    C1 & C2 & C3 --> MQ

    %% Relay servers registration
    R1 & R2 & R3 --> Redis

    %% Conductor to relay servers
    C1 & C2 & C3 --> R1 & R2 & R3

    %% Relay to local development
    R1 & R2 & R3 -.->|Optional HTTP| DEV

    %% Monitoring
    C1 & C2 & C3 & R1 & R2 & R3 --> Prom
    C1 & C2 & C3 & R1 & R2 & R3 --> Logs
    C1 & C2 & C3 & R1 & R2 & R3 --> Errors
    Prom --> Graf

    classDef service fill:#f9f,stroke:#333,stroke-width:2px;
    classDef database fill:#fff,stroke:#333,stroke-width:2px;
    classDef external fill:#bbf,stroke:#333,stroke-width:2px;
    classDef monitoring fill:#bfb,stroke:#333,stroke-width:2px;

    class C1,C2,C3,R1,R2,R3,Auth service;
    class Redis,PG,MQ,AuthDB database;
    class CLIs,WS,DEV external;
    class Prom,Graf,Logs,Errors monitoring;
```

### External Layer

- **Webhook Senders**: External services (GitHub, Stripe, Chargebee, etc) that send webhooks to `whook`
- **whook CLI**: Command-line interfaces running on developers' machines that display webhooks and optionally forward them to local development servers

### Cloud Infrastructure

#### DNS Layer

- Handles routing of all \*.whook.dev domains to the appropriate services
- Ensures webhook requests reach the correct project endpoints

#### Load Balancer

- Distributes incoming traffic across multiple conductor instances
- Provides the first layer of reliability and scaling

#### Auth Service

- Manages user authentication and project ownership
- Maintains its own database for auth-specific data

#### Conductor Service

- Multiple instances for high availability
- Coordinates tunnel assignments between clients and relay servers
- Manages project configurations and routing logic

#### Relay servers

- Handles WebSocket connections from whook CLIs
- Forward incoming webhooks to connected clients
- Scale independently based on connection load

#### Data Layer

- **Redis**: Fast in-memory storage, contains relay servers with latest heartbeat and current load
- **PostgreSQL**: Persistent storage for project data and configuration
- **Message Queue**: Handles async operations and request buffering

#### Observability

- **Prometheus/Grafana**: Metrics collection and visualisation
- **Logging System**: Centralised log aggregation (logstash?)
- **Error Tracking**: Real-time error monitoring and reporting (sentry, but something self hosted?)

### Local Development

- Optional connection to local development server
- Enables testing webhooks against local environments while still having observability over the body of the request, optionally it could also log the response from the local development server and act as a complete HTTP proxy
