# Getting Started

Welcome to the project. This guide covers local setup and your first contribution.

## Prerequisites

- Node.js 20 or later
- npm 10 or later
- A GitHub personal access token with `repo` scope

## Installation

```bash
npm install
```

## Running locally

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser.

## Running tests

```bash
# Unit tests
npm test

# End-to-end tests
npx playwright test
```

## Project layout

| Directory | Purpose |
|-----------|----------|
| `src/` | React frontend |
| `server/` | Local Hono server |
| `e2e/` | Playwright specs |
| `docs/` | Architecture and guides |
| `rfcs/` | Request-for-Comment proposals |

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-change`
3. Commit your changes using Conventional Commits
4. Open a pull request
