# Kiến trúc hệ thống

Tổng quan về cách AI Code Reviewer hoạt động.

## Overview

```
┌─────────────┐     Webhook      ┌─────────────┐
│   GitHub    │ ───────────────► │   FastAPI   │
│             │                  │   Server    │
└─────────────┘                  └──────┬──────┘
                                        │
                                        ▼
                                 ┌─────────────┐
                                 │    Redis    │
                                 │   (Queue)   │
                                 └──────┬──────┘
                                        │
                                        ▼
                                 ┌─────────────┐
                                 │   Celery    │
                                 │   Worker    │
                                 └──────┬──────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
             ┌───────────┐       ┌───────────┐       ┌───────────┐
             │ Security  │       │   Logic   │       │   Style   │
             │   Agent   │       │   Agent   │       │   Agent   │
             └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
                   │                   │                   │
                   └───────────────────┼───────────────────┘
                                       ▼
                                ┌─────────────┐
                                │  Aggregate  │
                                │  & Publish  │
                                └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │   GitHub    │
                                │  Comments   │
                                └─────────────┘
```

## Components

### 1. FastAPI Server

Nhận webhooks từ GitHub và validate requests.

```
src/app/
├── main.py              # App entry point
├── config.py            # Settings (Pydantic)
└── api/v1/
    └── webhooks.py      # Webhook handlers
```

**Responsibilities:**
- Verify webhook signature (HMAC-SHA256)
- Parse PR events
- Queue tasks to Celery

### 2. Celery Worker

Xử lý review tasks bất đồng bộ.

```
src/workers/
├── celery_app.py        # Celery config
└── tasks.py             # Task definitions
```

**Responsibilities:**
- Execute LangGraph pipeline
- Retry on failures
- Timeout handling

### 3. LangGraph Pipeline

Orchestrate AI agents.

```
src/agents/
├── graph.py             # Pipeline definition
├── state.py             # State schema
├── nodes/
│   ├── security.py      # Security agent
│   ├── logic.py         # Logic agent
│   └── style.py         # Style agent
└── prompts/
    └── *.py             # Prompt templates
```

**Pipeline steps:**
1. **Acknowledge** - Post "reviewing..." comment
2. **Extract** - Fetch PR files & diffs
3. **Analyze** - Run 3 agents in parallel
4. **Aggregate** - Dedupe & sort findings
5. **Publish** - Post comments to GitHub
6. **Notify** - Send Slack notification

### 4. Chat Handlers

Xử lý `@reviewer` commands.

```
src/chat/
├── router.py            # Command routing
└── handlers/
    ├── fix.py           # @reviewer fix
    ├── explain.py       # @reviewer explain
    └── tests.py         # @reviewer tests
```

## Data Flow

### PR Review Flow

```
1. Developer creates PR
2. GitHub sends webhook to /api/v1/webhooks/github
3. FastAPI validates signature & queues task
4. Celery worker picks up task
5. LangGraph pipeline runs:
   - Fetch PR data from GitHub API
   - Run Security, Logic, Style agents (parallel)
   - Aggregate results
   - Post comments via GitHub API
6. (Optional) Send Slack notification
```

### Chat Command Flow

```
1. User comments "@reviewer fix this"
2. GitHub sends issue_comment webhook
3. FastAPI routes to chat handler
4. Handler fetches context (original comment, code)
5. LLM generates response
6. Post reply via GitHub API
```

## Key Files

| File | Purpose |
|------|---------|
| `src/app/main.py` | FastAPI app |
| `src/app/config.py` | Environment settings |
| `src/agents/graph.py` | LangGraph pipeline |
| `src/agents/nodes/*.py` | AI agent implementations |
| `src/workers/tasks.py` | Celery tasks |
| `src/chat/router.py` | Chat command routing |
| `src/core/github.py` | GitHub API client |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GITHUB_APP_ID` | Yes | GitHub App ID |
| `GITHUB_PRIVATE_KEY` | Yes | App private key |
| `GITHUB_WEBHOOK_SECRET` | Yes | Webhook secret |
| `REDIS_URL` | Yes | Redis connection |
| `OPENROUTER_API_KEY` | Yes* | LLM API key |
| `SLACK_BOT_TOKEN` | No | Slack notifications |

*Or `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`

---

**Tiếp theo:** [Environment Variables](environment.md)
