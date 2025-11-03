# Ready To Review Systems Architecture

## Overview

Event-driven PR review system running on **Google Cloud Run**. GitHub webhooks flow through Sprinkler's WebSocket broadcast to notification services (Goose, Slacker) and Review-Bot for intelligent reviewer assignment. TurnServer provides turn tracking and PR metadata cache. Dashboard served via Cloudflare.

Built with [ko](https://ko.build/) and [Chainguard Images](https://www.chainguard.dev/chainguard-images) for supply chain security.

**Security:** Never store source code. Cache only PR metadata (21 days max). Zero Trust architecture.

## System Architecture

```mermaid
flowchart TB
    subgraph External["External Systems"]
        direction LR
        GH["GitHub API"]
        WHooks["GitHub Webhooks"]
        Slack["Slack"]
    end

    subgraph CF["Cloudflare"]
        CFProxy["DDoS Protection"]
    end

    subgraph GCR["Google Cloud Run"]
        direction LR
        Sprinkler["Sprinkler<br/><i>Broadcaster</i>"]
        ReviewBot["Review-Bot<br/><i>Assignment</i>"]
        Slacker["Slacker<br/><i>Slack Bot</i>"]
        TurnServer["TurnServer<br/><i>Cache & Turn Tracking</i>"]
        Dashboard["Dashboard<br/><i>Static UI + OAuth</i>"]
    end

    subgraph Clients["Clients"]
        direction LR
        Browser["Browser"]
        Goose["Goose"]
    end

    WHooks -->|Webhook| Sprinkler

    Sprinkler -.->|WSS| ReviewBot
    Sprinkler -.->|WSS| Goose
    Sprinkler -.->|WSS| Slacker

    ReviewBot -->|API| GH
    TurnServer -->|API| GH

    Goose -->|API| TurnServer
    Slacker -->|API| TurnServer
    Browser -->|API| TurnServer

    Browser -->|HTTPS| CFProxy
    CFProxy -->|HTTPS| Dashboard
    Slacker -->|Post| Slack

    Browser -.->|Search| GH
    Goose -.->|Search| GH
    Slacker -.->|Search| GH

    classDef external fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef cloudrun fill:#4285f4,stroke:#1565c0,stroke-width:2px,color:#fff
    classDef cloudflare fill:#f4511e,stroke:#bf360c,stroke-width:2px,color:#fff
    classDef client fill:#f1f8e9,stroke:#558b2f,stroke-width:2px

    class GH,WHooks,Slack external
    class Sprinkler,ReviewBot,Slacker,TurnServer,Dashboard cloudrun
    class CFProxy cloudflare
    class Browser,Goose client
```

## Components

- **Sprinkler:** Webhook receiver with HMAC-SHA256 verification. Broadcasts via WSS to subscribers. Auth: GitHub PAT.
- **Review-Bot:** PR analysis and reviewer assignment. WebSocket subscriber. Scores candidates by file expertise and workload. Auth: GitHub App (JWT + installation tokens).
- **TurnServer:** Turn calculator + PR metadata cache. Two-tier cache (memory + disk, 21 days). Auth: GitHub PAT.
- **Goose/Slacker:** Notification services (desktop/Slack). Query TurnServer for data.
- **Dashboard:** Static web UI running on Cloud Run, protected by Cloudflare. Handles GitHub OAuth. Serves JavaScript that queries TurnServer from the browser.

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
| Review-Bot cache | 3 days (metadata), 6h (lists) | Memory |
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
- Sprinkler: Org membership (read) for token validation

**Network:**
- All traffic: HTTPS/WSS (TLS 1.2+)
- Cloud Run: Public endpoints with Google-managed TLS
- Dashboard: Cloudflare DDoS protection

**Trust Boundaries & Authentication:**

```mermaid
flowchart LR
    subgraph UC["User Controlled"]
        direction TB
        Browser["Browser<br/><i>PAT: GitHub OAuth or user-provided</i>"]
        Goose["Goose<br/><i>PAT: from 'gh' CLI</i>"]
    end

    subgraph CF["Cloudflare"]
        CFProxy["DDoS Protection"]
    end

    subgraph OI["Our Infrastructure (Cloud Run)"]
        direction TB
        Dashboard["Dashboard<br/><i>Static UI (OAuth provider)</i>"]
        TurnServer["TurnServer<br/><i>Passthrough (validates nothing)</i>"]
        Sprinkler["Sprinkler<br/><i>Validates client PATs</i>"]
        ReviewBot["Review-Bot<br/><i>GitHub App JWT</i>"]
        Slacker["Slacker<br/><i>GitHub App JWT</i>"]
    end

    subgraph TP["Third Party Services"]
        direction TB
        GitHub["GitHub API"]
        Slack["Slack API"]
    end

    Browser -->|"HTTPS<br/>loads UI"| CFProxy
    CFProxy -->|HTTPS| Dashboard

    Browser -->|"HTTPS + PAT<br/>(user's GitHub token)"| TurnServer
    Goose -->|"HTTPS + PAT<br/>(gh CLI token)"| TurnServer
    Slacker -->|"HTTPS + PAT<br/>(client passes through)"| TurnServer

    GitHub -->|"Webhook<br/>(HMAC-SHA256 verified)"| Sprinkler

    Sprinkler -->|"WSS + PAT<br/>(validates org membership)"| ReviewBot
    Sprinkler -->|"WSS + PAT<br/>(validates org membership)"| Goose
    Sprinkler -->|"WSS + PAT<br/>(validates org membership)"| Slacker

    ReviewBot -->|"GitHub App<br/>(JWT + installation token)"| GitHub
    Slacker -->|"GitHub App<br/>(JWT + installation token)"| GitHub
    TurnServer -->|"PAT<br/>(passes through from clients)"| GitHub

    Slacker -->|"Slack OAuth"| Slack

    classDef user fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef third fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef ours fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef cloudflare fill:#f4511e,stroke:#bf360c,stroke-width:2px,color:#fff

    class Browser,Goose user
    class GitHub,Slack third
    class Dashboard,Sprinkler,ReviewBot,Slacker,TurnServer ours
    class CFProxy cloudflare
```

**Key Security Properties:**
- **Zero-trust design**: TurnServer forwards client PATs to GitHub without validation. Per-PAT caching (SHA256-hashed) isolates data by user.
- **Webhook integrity**: All GitHub webhooks HMAC-SHA256 verified before processing.
- **Authentication**: Review-Bot/Slacker use GitHub Apps (JWT + installation tokens). User clients provide their own PATs (OAuth or gh CLI).
- **Access control**: Sprinkler validates org membership on WebSocket connections. Connection rate limiting per IP.
- **Data minimization**: Never cache source code. PR metadata only (21 days max). Sprinkler broadcasts URLs only.
- **DoS protection**: Request body size limits, CORS restrictions, Cloud Run rate limiting, Cloudflare DDoS protection.
