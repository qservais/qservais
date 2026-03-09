# CLAUDE.md

This file provides guidance to AI assistants (Claude and others) working in this repository.

> **Note:** This repository is currently empty. This CLAUDE.md serves as a starting template. Update it as the codebase grows.

---

## Repository Overview

**Repository:** qservais/qservais
**Purpose:** _(describe the project purpose here)_
**Status:** Initial setup — no source code committed yet.

---

## Development Setup

### Prerequisites

_(List required tools, runtimes, and their versions once known, e.g.:)_

```
Node.js >= 20
Python >= 3.11
Docker
```

### Installation

```bash
# Clone the repository
git clone <repo-url>
cd qservais

# Install dependencies (update once package manager is chosen)
# npm install      # for Node.js
# pip install -r requirements.txt  # for Python
# go mod download  # for Go
```

### Environment Variables

Copy `.env.example` to `.env` and fill in values:

```bash
cp .env.example .env
```

_(Document required environment variables here once defined.)_

---

## Common Commands

> Update these once the tech stack and scripts are established.

| Task | Command |
|------|---------|
| Install dependencies | `<command>` |
| Run dev server | `<command>` |
| Run tests | `<command>` |
| Run linter | `<command>` |
| Format code | `<command>` |
| Build for production | `<command>` |
| Run type checker | `<command>` |

---

## Project Structure

_(Update this section once the codebase is scaffolded.)_

```
qservais/
├── CLAUDE.md           # This file
├── README.md           # Human-facing documentation
├── .env.example        # Environment variable template
├── src/                # Source code
└── tests/              # Test files
```

---

## Architecture & Key Conventions

### Code Style

_(Document linting tools, formatter config, and style preferences here.)_

### Testing Strategy

_(Describe test framework, coverage requirements, and test organization.)_

### Git Workflow

- Branch naming: `feature/<description>`, `fix/<description>`, `chore/<description>`
- Commit messages: Use conventional commits format — `type(scope): message`
  - Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `style`
- Keep PRs focused and small

### Key Patterns

_(Document recurring patterns, abstractions, or idioms once the codebase exists.)_

---

## AI Assistant Guidelines

### When Making Changes

1. **Read before editing** — always read the relevant files before modifying them.
2. **Follow existing conventions** — match the code style, naming, and structure already in place.
3. **Minimal changes** — only change what is necessary for the task at hand.
4. **No unnecessary abstractions** — don't create helpers or utilities for single-use operations.
5. **No unsolicited refactors** — don't clean up code that wasn't part of the request.
6. **No speculative features** — don't add error handling, flags, or config for hypothetical scenarios.

### When Adding Dependencies

- Prefer well-maintained packages with active communities.
- Check for security advisories before adding new dependencies.
- Keep the dependency list lean.

### Security

- Never commit secrets, API keys, or credentials.
- Validate all external input at system boundaries.
- Avoid common vulnerabilities (SQL injection, XSS, command injection, etc.).

---

## CI/CD

_(Document CI/CD pipelines, deployment targets, and environment promotion strategy once configured.)_

---

## Contributing

_(Add contribution guidelines, code review expectations, and PR checklist once established.)_
