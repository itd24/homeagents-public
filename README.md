# HomeAgents

HomeAgents is a self-hosted AI platform for one household, running entirely on a
single home server. Chat, coding agents, document memory, web search, personal
finance and a no-code app builder each run as their own Docker Compose stack.
The stacks share one model gateway, one Docker network and one data directory.

This repository is the map for the rest of the project. It explains what the
parts are and how they fit together. Every component lives in its own repository
so it can be read, installed and updated on its own.

## Repositories

| Repository | What it is |
|---|---|
| [`homeagents-core`](https://github.com/itd24/homeagents-core) | The platform itself: LiteLLM gateway, Qdrant and TEI memory, SearXNG web search, Firefly III MCP, OpenHands, plus the shared `.env` and `VOLUMES/` data root |
| [`homeagents-cli`](https://github.com/itd24/homeagents-cli) | A thin PowerShell runner. `homeagents <stage> <command>` starts and stops each stack |
| [`librechat-deployment`](https://github.com/itd24/librechat-deployment) | The chat front end, wired to LiteLLM with profiles, personas and MCP tools |
| [`dify-deployment`](https://github.com/itd24/dify-deployment) | Dify, for visual LLM app and workflow building |
| [`homeagents-mcp-servers`](https://github.com/itd24/homeagents-mcp-servers) | Two custom Model Context Protocol servers: `qdrant-mcp` for memory and `web-search-mcp` for SearXNG and Tavily |
| [`paperless-qdrant-ingestion`](https://github.com/itd24/paperless-qdrant-ingestion) | A one-shot worker that indexes OCR'd Paperless-ngx documents into the shared vector memory |

Clone them side by side. The CLI and the compose files expect that layout:

```
HomeAgents/
  homeagents-core/
  homeagents-cli/
  librechat-deployment/
  dify-deployment/
  homeagents-mcp-servers/
  paperless-qdrant-ingestion/
```

## Architecture

```
        ┌──────────────┐        ┌──────────────────┐
        │  LibreChat   │        │      Dify        │
        │  :3080       │        │  :8085           │
        └──────┬───────┘        └────────┬─────────┘
               │                         │
               │ OpenAI-compatible API   │ (own networks,
               ▼                         ▼  host.docker.internal)
        ┌──────────────────────────────────────────┐
        │   LiteLLM gateway  :4000                 │
        │   virtual models: fast / coder /         │
        │   architect / premium / openai-fallback  │
        └──────┬───────────────────────────────────┘
               │ provider APIs (DeepSeek, Z.ai, Anthropic, OpenAI)
               │
   ai-platform-net (shared Docker network)
   ├── Qdrant :6333 ── TEI embeddings :8081  (memory)
   ├── qdrant-mcp :8765                      (memory tools)
   ├── SearXNG :8082 ── web-search-mcp        (search tools)
   ├── firefly-mcp :3002                     (finance tools)
   └── OpenHands :3000                       (coding agent)

   paperless-qdrant-ingestion  -> reads Paperless-ngx, writes to Qdrant
```

Two ideas hold the platform together:

* **All model traffic goes through LiteLLM.** Everything that needs a model talks
  to the gateway. Applications only ever see virtual model names, so switching
  providers or changing budgets is a one-file edit and never a UI change.
* **One memory, many domains.** Long-term memory is a single Qdrant collection
  with a `domain` payload field. Each profile reads and writes only its own
  domain, so a personal assistant and a work assistant share the infrastructure
  without sharing context.

## Ports

Every published port binds to `127.0.0.1`, so nothing is reachable from the LAN
by default. Containers talk to each other by service name on `ai-platform-net`,
which is why the host ports exist only for a browser on this machine. The one
exception is Dify's optional store profiles, which are off by default and keep
their upstream port mappings.

| Port | Service | Repository |
|---|---|---|
| 3000 | OpenHands | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 3002 | Firefly III MCP | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 3080 | LibreChat | [librechat-deployment](https://github.com/itd24/librechat-deployment) |
| 4000 | LiteLLM gateway | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 6333 | Qdrant | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 8081 | TEI embeddings | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 8082 | SearXNG | [homeagents-core](https://github.com/itd24/homeagents-core) |
| 8085 | Dify console | [dify-deployment](https://github.com/itd24/dify-deployment) |
| 8765 | qdrant-mcp | [homeagents-core](https://github.com/itd24/homeagents-core) |

## Getting started

```powershell
# Clone the six repositories side by side.
foreach ($repo in 'homeagents-core', 'homeagents-cli', 'homeagents-mcp-servers',
                  'librechat-deployment', 'dify-deployment',
                  'paperless-qdrant-ingestion') {
    git clone "https://github.com/itd24/$repo.git"
}

# Provider keys and host paths go into one file.
Copy-Item homeagents-core/.env-example homeagents-core/.env

# The two custom MCP servers are built locally.
docker build -t qdrant-mcp:latest     homeagents-mcp-servers/qdrant
docker build -t web-search-mcp:latest homeagents-mcp-servers/websearch

# The CLI creates the ai-platform-net network on the first run.
./homeagents-cli/homeagents.ps1 core      run
./homeagents-cli/homeagents.ps1 websearch run
./homeagents-cli/homeagents.ps1 chat      run
```

The `homeagents-core/README.md` file has the details and the per-stack
commands. The `homeagents-cli/README.md` file documents the stage names.

## What it looks like in use

* **Ask the chat about your own notes.** The `fast` profile in LibreChat has the
  memory tools attached, so a question about last month's decision reads the
  vectors that an earlier conversation wrote, not the open web.
* **Drop a document, find it later.** Anything OCR'd by Paperless-ngx gets
  indexed by the ingestion worker and becomes recallable from the same
  collection.
* **Hand a task to the coding agent.** `homeagents programming run` starts
  OpenHands with a project folder mounted and it plans and edits inside it.
* **Build a small app without code.** The Dify console assembles workflows with a
  local knowledge base against the same gateway the chat profiles use.

## Where the AI actually runs

Two models do the work. Hosted language models (DeepSeek, Z.ai, Anthropic,
OpenAI) sit behind LiteLLM and answer chat and coding requests. One local model,
TEI serving `BAAI/bge-m3`, turns text into vectors for memory search, so indexing
runs on the home server's CPU and costs no API credit. OpenHands is the coding
agent, and the two MCP servers are the tool layer: memory and web search.

## Design principles

* **Reproducible by default.** Most images are pinned to an explicit release tag
  and the data root is a plain directory of bind mounts, so a rebuild never
  silently changes behaviour and a backup is a directory copy. Four services
  ship only a moving tag upstream (Qdrant, TEI `cpu-latest`, SearXNG, OpenHands);
  pin them yourself if you need the same guarantee everywhere.
* **Secrets stay in one place.** A single git-ignored `.env` lives in
  `homeagents-core`. Every other repository ships an `.env-example` or documents
  which variables it reads.
* **Local first.** Embeddings run on local hardware (TEI with `bge-m3`) and the
  search engine is self-hosted. The only outbound calls go to the model providers
  you enable yourself.
* **One concern per repository.** Each stack can be read, forked and deployed
  without the others, which is also why each one here has its own git history.

## Requirements

* Docker with Compose v2
* PowerShell 5.1 or newer (for the CLI only)
* A machine that can keep Qdrant, Postgres, Redis and the embedding model
  resident. 16 GB of RAM is a comfortable floor.
* API keys for the model providers you want to use

## Licence

MIT, see [LICENSE](LICENSE). Third-party components keep their own licences:
Dify, LibreChat, LiteLLM, SearXNG and the Firefly III MCP server are separate
upstream projects, and the Dify deployment repository vendors some upstream
files (Apache-2.0) with local edits.
