# System Architecture

## Overview

The system is split into three layers: a React single-page application, a local Hono
server for filesystem access, and the GitHub REST API for remote collaboration.

```mermaid
flowchart LR
    subgraph Browser
        UI[React SPA]
    end
    subgraph Local
        CLI[redraft CLI]
        FS[Filesystem]
        WS[WebSocket hub]
    end
    subgraph Remote
        GH[GitHub REST API]
        Repo[(Git Repository)]
    end

    UI -- local mode --> CLI
    CLI --> FS
    CLI --> WS
    WS -- live events --> UI
    UI -- remote mode --> GH
    GH --> Repo
```

## Local Mode

In local mode the CLI (`npx redraft [directory]`) starts a Hono HTTP server that:

1. Serves the pre-built React SPA
2. Exposes `/api/local/*` routes that mirror the GitHub Contents API shape
3. Pushes filesystem change events to the browser via WebSocket

The browser detects local mode from `window.__REDRAFT_LOCAL__` injected into `index.html`
and skips the GitHub PAT auth gate.

## Remote Mode

In remote mode the browser talks directly to `https://api.github.com` using a
personal access token supplied by the user. No server component is needed.

## Comment Storage

```mermaid
flowchart TD
    A[User selects text] --> B[Comment form opens]
    B --> C[User submits thread]
    C --> D{Mode?}
    D -- local --> E[PUT /api/local/contents/.redraft/comments/path.comments.json]
    D -- remote --> F[PUT https://api.github.com/.../.redraft/comments/path.comments.json]
    E --> G[(Disk: .redraft/comments/)]
    F --> H[(GitHub: .redraft/comments/)]
```

Comment sidecars are stored at `.redraft/comments/<mirrored-path>.comments.json`
relative to the repository root. This keeps review state co-located with the repo
but separate from the document content.

## Data Flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Local Server
    participant D as Disk

    B->>S: GET /api/local/tree
    S->>D: walkMarkdownFiles(root)
    D-->>S: [path, ...]
    S-->>B: { documents, underReview }

    B->>S: GET /api/local/contents/docs/arch.md
    S->>D: readFile(path)
    D-->>S: content + sha
    S-->>B: { content, sha }

    D->>S: chokidar event (file changed)
    S->>B: WebSocket message { type: "change", path }
    B->>S: GET /api/local/contents/docs/arch.md
```
