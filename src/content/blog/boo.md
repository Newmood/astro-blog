---
title: 'Engineering Boo: An AI-powered Discord bot'
description: 'Here is a sample of some basic Markdown syntax that can be used when writing Markdown content in Astro.'
pubDate: '30 August, 2025'
lastUpdated: '30 August, 2025'
heroImage: '/boo.png'
---

Discord bots are usually one of 2 things: A tiny weekend project or a bloated monolith that tries to do everything in a single process. [**Boo**](https://github.com/VVIP-Kitchen/boo) takes a different route.

Boo is an AI-powered Discord bot that doesn't just answer commands; it chats with context, analyzes images, summarizes conversations, runs code safely, adapts to each server with its own system prompt. Under the hood, it's designed like a real distributed system: multiple services, clean separation of concerns, and safety boundaries between untrusted code and and critical state.

The motivation was simple: bring modern LLM workflows into Discord, but engineer them with the same rigor you would expect from production microservices.

### Architecture

![Mermaid diagram of Boo architecture](/boo-arch.png)

Boo runs as a [Docker Compose](https://docs.docker.com/compose) stack with 5 services, each with a clear job:

- `discord-bot`: A Python bot built using [`discord.py`](https://github.com/Rapptz/discord.py)
  - Handles events, slash commands, embeds and all user interaction.
  - Talks to LLMs, external APIs and the manager service.
- `manager`: A Go microservice that acts as a backend API for Boo.
  - It owns the Postgres schema.
  - It provides REST endpoints for prompts, messages and token usage.
  - This isolates the persistence logic from the bot
- `postgres`: The database of record.
  - Stores long-term data: system prompts, message logs and LLM token usage stats
- `redis`: A short-term cache of recent channel activity (15 minutes).
  - Keeps ephemeral state for summaries and lightweight context without bloating Postgres.
- `sandbox`: A locked-down FastAPI service that executes Python snippets in a restricted environment.
  - It enforces CPU/memory limits, runs as a non-root user, and only writes to `tmpfs`.
  - This lets Boo safely support "run code" tool calls.

All of these are stitches together with [`compose.yml`](https://github.com/VVIP-Kitchen/boo/blob/main/compose.yml). The bot and sandbox sit on a private internal network, denying any form of network access to the sandbox. Postgres and Redis expose ports only on localhost. Manager service waits for Postgres healthchecks before boot. Each service has restart policies, minimal privileges, and resource caps.

**The result?** Boo behaves like a tiny cluster of cooperating services rather one single fat process. Discord users only see the playful ghost; behind the curtain it's a carefully separated architecture tuned for resilience and safety.

### The Manager Service (Go)

Boo doesn't let the Discord bot talk directly to Postgres. Instead, persistence runs through a small Go REST API called [**manager**](https://github.com/VVIP-Kitchen/boo/tree/main/manager). The keeps the database logic contained, makes schema migrations explicit and lets the bot remain stateless.

**Why Go?**
- Small, fast binaries → Perfect for containerized microservices.
- Strong typing around DB interactions, fewer runtime surprises.
- Simple to execute HTTP endpoints with [`gin`](https://github.com/gin-gonic/gin)

**Responsibilities**
> The manager owns 3 core domains.

1. Guild Prompts
  - Stored in a table `boo_prompts (guild_id, system_prompt)`.
  - Each Discord server (guild) can override Boo's behavior with their own system prompt.
  - Endpoints:
    - `GET /prompts` → List all prompts.
    - `GET /prompt?guild_id=` → Fetch prompt for a given guild.
    - `POST /prompt` → Add a prompt.
    - `PUT /prompt?guild_id=` → Update prompt for a given guild.
2. Messages
  - Logs Discord messages into `discord_messages`.
  - Captures author info, channel IDs, and content for both persistence, analysis and later fine-tuning an LLM.
  - Endpoint:
    - `POST /message` → Add a message
3. Token Usage
  - Tracks LLM input/output tokens per message in `token_usage`.
  - Useful for cost monitoring and usage stats.
  - Endpoints:
    - `POST /token` → Add token usage.
    - `GET /token/stats?guild=&author_id=&period=` → Fetch usage (filter by: guild, user, time period)

Manager is small but essential. It formalizes persistence into an API boundary and prevents the Python bot from ever touching the DB directly.

### The Discord Bot (Python)

The Python service is Boo’s face: it talks to Discord, calls into LLMs, and stitches tools together. It’s built on discord.py with an extension-based design.

**Structure**
  - [`bot.py`](https://github.com/VVIP-Kitchen/boo/blob/main/src/bot/bot.py) → Initializes the bot with custom intents (message content, presences, members) and loads extensions.
  - Cogs (modular classes)
    - [`events.py`](https://github.com/VVIP-Kitchen/boo/blob/main/src/bot/events.py) → core event loop
      - listens for messages, runs AI completions, handles image attachments, resets context
    - [`commands/general.py`](https://github.com/VVIP-Kitchen/boo/blob/main/src/commands/general.py) → slash commands like `/weather`, `/token_stats`, `/summary`
    - [`commands/admin.py`](https://github.com/VVIP-Kitchen/boo/blob/main/src/commands/admin.py) → privileged commands like `sync`

**Key Features**
1. Message Handling
  - Every incoming message is logged (`log_message`) → stored in Postgres via Manager, cached in Redis for 15 mins.
  - Bot checks whether to reply or not (Only replies if it's a mention, reply or a DM)
  - Builds context prompt
    - Server lore (time, online members, custom emojis)
    - Recent context (from Redis + in-memory buffer)
    - User message (+ images as base64, if present)
2. LLM Completion
  - Routes through [OpenRouter](https://openrouter.ai)
  - Supports text + vision (image attachments)
  - Can call tools (hackernews, web search, sandbox code execution)
  - Handles rate limits gracefully → tells user when to retry.
3. Server customization
  - Each server can set/update it's own system prompt (`/update_prompt`, requires the role "boo manager")
4. Fun/Utility commands
  - `/bonk` → Bonk a server member, with a GIF from Tenor.
  - `/weather` → Fetch weather report from Tomorrow.IO
  - `/token_stats` → Who's consuming the most amount of tokens.
  - `/summary` → A summary of what happened in the last 15 mins, when the chat moves too fast and you can't keep up 😩

**Philosophy**
The bot itself is deliberately de-coupled from persistence, It just:
- Logs to Redis and Manager.
- Call APIs, be it LLM, Tenor, TomorrowIO, Tavily or Sandbox.
- Formats responses, especially for messages, embeds, stickers, etc.

### **LLM Integration**
At the heart of Boo, it isn't just an API call and dump the result back; it treats the LLM as a collaborator.
- **Provider:** Boo uses OpenRouter as the gateway. That means it can swap models (GPT, LLaMa, Mistral, etc.) without changing the bot logic.
- **Inputs:** Each request bundles together:
  - server-specific system prompt (from Postgres)
  - ephemeral "server-lore" (time, online members, available emojis)
  - sliding window of past conversations (from Redis + in-memory)
  - user message (and attached images, converted into base64 `image_url`)
- **Outputs:** The LLM reply, sometimes with tool calls (like hackernews, web search or sandbox code execution)
  - Boo continues the tool, feeds the result back into the conversation and asks the LLM to continue.

Errors are caught; if OpenRouter throws a 429. Boo calculates the retry window from headers and tells the user exactly how long to wait. This feedback loop is crucial for interactivity.

### Context Management

Coherence is tricky: you don’t want Boo to forget what you said two minutes ago, but you also don’t want infinite history (memory is limited). Boo solves this with layered context!

1. Redis (15 mins)
  - Every message in a channel is pushed into Redis with a TTL cutoff of ~15 minutes.
  - Useful for quick summaries.
  - Cleans itself via periodic sweeps.
2. Sliding Window in Memory
  - For ongoing conversations, Boo appends user/assistant messages into an in-memory list keyed by server.
  - Once this grows past the configured `CONTEXT_LIMIT` (say, 25 turns), it trims older entries—keeping only the latest slice.
  - Result: Boo always has enough backscroll to stay coherent, but never floods the LLM with unnecessary tokens.
3. Postgres (Long-term)
  - Stores raw messages and system prompts, but isn’t used for live completions.
  - More like a ledger than a memory. Will be used later to fine-tune an LLM on Discord chats!

This architecture balances short-term recall (Redis + memory window) with long-term persistence (Postgres). Boo won’t hallucinate whole backstories, but it will remember what you said a few messages ago.

### Safe Code Execution (Sandbox)

One of Boo’s most unusual features: it can run Python code snippets suggested by the LLM, but only inside a hardened sandbox.

The sandbox is a separate FastAPI service running in its own container. Safety is layered!

- Resource limits!
  - CPU time capped (`RLIMIT_CPU`)
  - Memory capped at 512MB (`RLIMIT_AS`)
  - Process/file descriptor caps (`pids_limit`, `nofile`)
- User isolation: Runs as a non-root Linux user with no extra capabilities (`cap_drop: ALL`)
- Filesystem isolation: Only `/tmp` is writable, mounted as `tmpfs` (RAM-backed, auto-cleaned)
- Network isolation: Container sits on an internal-only network—no outbound internet access.
- Timeouts: All executions hard-stop after a max of 10s.

Execution flow:
1. User asks a question (e.g. “write me a quick Python script to calculate primes”).
2. LLM decides to call the `run_code` tool.
3. Bot sends the code to `sandbox:8081/run`
4. Sandbox writes it to a temp file, runs it with resource limits, and streams back `stdout`/`stderr`

If it hangs, crashes, or tries something nasty, it just dies in isolation. The Discord bot never sees the blast radius.

That keeps Boo playful but safe; you can let it execute code on demand without worrying about someone mining crypto through your Discord server *heh*.

### DevOps and Deployment

![Boo Docker diagram](/boo-deploy.png)

Boo isn’t deployed on some elaborate Kubernetes cluster. It runs on a Hetzner VM with plain old Docker. Simple, predictable, and good enough for a side project that needs to be reliable but not very over-engineered.

**Deployment Flow**
Right now, deployment looks like this:
1. Pull the latest code from GitHub onto the VM.
2. Build images locally via Docker Compose:
  ```shell
  docker compose up --build -d
  ```
3. Services start in dependency order, mentioned in docker compose yaml!

**Container Boundaries**
- **Public ports:** Postgres and Redis are bound only to `127.0.0.1`. Manager listens on `127.0.0.1:8080`
  - Only the bot needs Discord’s outbound internet access.
- **Private network:** Bot and sandbox share an `internal_net` so tool calls never leave the box.
- **Restart policies:** Every service is configured with `restart: always`
  - If Boo crashes or Hetzner reboots, it comes back without manual intervention.
- **Resource limits:** Sandbox has explicit caps on memory, CPU, processes, and file descriptors so a runaway script can’t starve the machine.

**Observability (basic but functional)**
- Logs are streamed to stdout/stderr of each container and tailed when debugging.
- Postgres healthchecks keep the stack honest.
- No Prometheus/Grafana yet, but the token usage stats tracked by Boo itself are a decent proxy for workload.

**What Could Be Next?**
- **CI/CD:** Automating rebuilds with GitHub Actions instead of manual pulls.
- **Secrets management:** Right now env vars are passed via `.env`
  - Moving to Vault or SOPS would be safer.
- **Metrics:** Hooking into Hetzner monitoring or Prometheus for deeper visibility.
- **Blue-green deploys:** Keeping old containers alive until new ones are healthy to avoid downtime during updates.

For now, the trade-off is speed of iteration vs polish. The rudimentary pipeline works because Compose gives you just enough orchestration without the full weight of Kubernetes.

### Lessons Learned
1. **Separation pays off**
  - Splitting the system into small services (bot, manager, sandbox, DB, cache) makes everything clearer. Bugs stay contained, and you don’t fear one piece taking down the whole bot.
2. **Go and Python; a match made in binary**
  - Go is excellent for a tiny, predictable API surface (the manager). Python shines in Discord + AI ecosystems. Marrying the two gave a nice balance.
3. **Context is a constant trade-off**
  - Sliding windows, Redis, Postgres; each adds overhead, but without them Boo either forgets too quickly or gets too expensive.
  - Tuning the layers was half the challenge.
4. **Sandboxing is non-negotiable**
  - Letting an LLM execute arbitrary code in your Discord server sounds like a terrible idea; unless you isolate it.
  - Docker limits, non-root users, tmpfs, and hard caps are what make the feature viable.
5. **Ops don’t need to be fancy**
  - A single Hetzner VM, Docker Compose, and some discipline go a long way.
  - No need to haul in Kubernetes until the scale truly demands it.

### Closing

Boo started as a Discord bot experiment and ended up a miniature distributed system. It can chat with context, analyze images, run code safely, summarize conversations, and even nudge people toward inclusive language; all while staying resilient across container boundaries.

Running it feels like operating a tiny production service: Postgres migrations, Redis caches, REST APIs, rate limits, sandbox security. But at the same time, it’s just a playful ghost in a Discord channel.

That’s the sweet spot; engineering with rigor, but in service of something fun.
