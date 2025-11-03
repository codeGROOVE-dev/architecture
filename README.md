# Ready To Review Systems Architecture

## Overview

Event-driven PR review system running on **Google Cloud Run**. GitHub webhooks flow through Sprinkler's WebSocket broadcast to notification services (Goose, Slacker) and Review-Bot for intelligent reviewer assignment. TurnServer provides turn tracking and PR metadata cache. Dashboard served via Cloudflare.

Built with [ko](https://ko.build/) and [Chainguard Images](https://www.chainguard.dev/chainguard-images) for supply chain security.

**Security:** Never store source code. Cache only PR metadata (21 days max). Zero Trust architecture.

## System Architecture

```mermaid
flowchart TB
    subgraph External["External Systems"]
        GH["GitHub API"]
        WHooks["GitHub Webhooks"]
    end

    subgraph GCR["Google Cloud Run - Public Services"]
        Sprinkler["Sprinkler<br/><i>Webhook Receiver & Broadcaster</i>"]
        ReviewBot["Review-Bot<br/><i>PR Analysis & Assignment</i>"]
        Goose["Goose<br/><i>Local Notifications</i>"]
        Slacker["Slacker<br/><i>Slack Notifications</i>"]
        TurnServer["TurnServer<br/><i>Turn Calculator & PR Metadata Cache</i><br/><small>21-day cache</small>"]
    end

    subgraph CF["Cloudflare Protected"]
        Dashboard["Web Dashboard<br/><i>User Interface</i>"]
    end

    subgraph Clients["Client Endpoints"]
        Browser["Web Browser"]
        Desktop["Desktop Client"]
        Slack["Slack Workspace"]
    end

    WHooks -->|HTTPS| Sprinkler

    Sprinkler -.->|WebSocket Stream| ReviewBot
    Sprinkler -.->|WebSocket Stream| Goose
    Sprinkler -.->|WebSocket Stream| Slacker

    ReviewBot -->|GitHub API| GH

    Goose -->|HTTPS API| TurnServer
    Slacker -->|HTTPS API| TurnServer

    TurnServer -->|GitHub API<br/>on cache miss| GH

    Browser -->|HTTPS| Dashboard
    Dashboard -->|HTTPS API| TurnServer

    Browser -->|Search Queries<br/>Direct| GH

    Goose -->|Notify| Desktop
    Slacker -->|Post| Slack

    classDef external fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef cloudrun fill:#4285f4,stroke:#1565c0,stroke-width:2px,color:#fff
    classDef cloudflare fill:#f4511e,stroke:#bf360c,stroke-width:2px,color:#fff
    classDef client fill:#f1f8e9,stroke:#558b2f,stroke-width:2px

    class GH,WHooks external
    class Sprinkler,ReviewBot,Goose,Slacker,TurnServer cloudrun
    class Dashboard cloudflare
    class Browser,Desktop,Slack client
```

## Components

- **Sprinkler:** Webhook receiver with HMAC-SHA256 verification. Broadcasts via WSS to subscribers. Auth via GitHub PAT.
- **Review-Bot:** PR analysis and reviewer assignment. WebSocket subscriber. Scores candidates by file expertise and workload. Uses GitHub App (JWT + installation tokens).
- **TurnServer:** Turn calculator + PR metadata cache. Two-tier cache (memory + disk, 21 days). Auth via GitHub PAT.
- **Goose/Slacker:** Notification services (desktop/Slack). Query TurnServer for data.
- **Dashboard:** Web UI behind Cloudflare. Queries TurnServer.

## Data Flow

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant SP as Sprinkler
    participant RB as Review-Bot
    participant GO as Goose/Slacker
    participant TS as TurnServer

    GH->>SP: Webhook (HMAC verified)
    par Broadcast
        SP-->>RB: WSS
        SP-->>GO: WSS
    end
    RB->>GH: Assign reviewer
    GO->>TS: Get PR metadata
    alt Cache Hit
        TS-->>GO: Cached (21d)
    else Miss
        TS->>GH: Fetch
    end
```

## Data Storage

| Data Type | Retention | Location |
|-----------|-----------|----------|
| PR metadata | 21 days | TurnServer (memory + disk) |
| Review-Bot cache | 20 days (metadata), 6h (lists) | Memory |
| **Source code/diffs** | **Never** | **N/A** |
| Webhook events | Real-time only | Ephemeral |
| Auth tokens | Persistent | Encrypted at rest |

## Technology Stack

- **Language:** Go
- **Build:** [ko](https://ko.build/) - builds from source without Dockerfiles
- **Base images:** [Chainguard Images](https://www.chainguard.dev/chainguard-images) - distroless, zero CVEs
- **Runtime:** Google Cloud Run
- **TLS:** 1.2+ minimum, managed by Google/Cloudflare

## Security

**Authentication:**
- Review-Bot: GitHub App (JWT + installation tokens, auto-refresh)
- TurnServer: GitHub PAT
- Sprinkler: GitHub PAT
- Dashboard: GitHub OAuth

**GitHub Permissions:**
- Review-Bot: Repo (read/write), PRs (read/write), Org (read)
- TurnServer: Repo/PRs/Checks (read)

**Network:**
- All traffic: HTTPS/WSS (TLS 1.2+)
- Cloud Run: Public endpoints with Google-managed TLS
- Dashboard: Cloudflare DDoS protection

**Trust Boundaries:**

```mermaid
graph TB
    subgraph "User Controlled"
        Browser["Browser"]
        GooseClient["Goose Client"]
    end

    subgraph "Third Party"
        GitHub["GitHub"]
        Slack["Slack"]
        CF["Cloudflare"]
    end

    subgraph "Our Infrastructure (Cloud Run)"
        Dashboard["Dashboard"]
        Sprinkler["Sprinkler"]
        ReviewBot["Review-Bot"]
        Goose["Goose"]
        Slacker["Slacker"]
        TurnServer["TurnServer"]
    end

    Browser -->|HTTPS| CF
    CF -->|HTTPS| Dashboard
    Dashboard -->|Auth| TurnServer

    GitHub -->|Webhook| Sprinkler
    Sprinkler -->|WSS+Auth| ReviewBot
    Sprinkler -->|WSS+Auth| Goose
    Sprinkler -->|WSS+Auth| Slacker

    ReviewBot -->|OAuth| GitHub
    TurnServer -->|OAuth| GitHub

    Goose -->|Auth| TurnServer
    Goose -->|Notify| GooseClient

    Slacker -->|Auth| TurnServer
    Slacker -->|OAuth| Slack

    classDef user fill:#f9f9f9,stroke:#333,stroke-width:2px
    classDef third fill:#fff3cd,stroke:#856404,stroke-width:2px
    classDef ours fill:#d1ecf1,stroke:#0c5460,stroke-width:2px

    class Browser,GooseClient user
    class GitHub,Slack,CF third
    class Dashboard,Sprinkler,ReviewBot,Goose,Slacker,TurnServer ours
```
