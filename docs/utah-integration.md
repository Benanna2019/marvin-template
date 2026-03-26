# Utah Integration — Ben's Automation Layer

This document explains the Utah agent, how it relates to Marvin, and how the two systems work together. Read this to understand the full picture of Ben's personal agent stack.

---

## What Utah Is

Utah (Universally Triggered Agent Harness) is a durable AI agent built on Inngest. It lives at github.com/Benanna2019/utah.

Utah handles everything that needs to run in the background — scheduled tasks, webhook responses, and automated check-ins. It is the automation layer. Marvin is the conversational layer. They are complementary, not duplicates.

The key distinction:
- **Marvin** = you talk to it. It thinks, drafts, gathers context, and works through problems with Ben interactively.
- **Utah** = it runs on its own. It fires on a schedule or in response to external events, does its job, and notifies Ben.

---

## Utah Architecture

Utah uses a think/act/observe loop where every LLM call and tool execution is an Inngest step. This means:

- Every action is durable — if it fails, Inngest retries it automatically
- Every run is observable in the Inngest dashboard
- Singleton concurrency prevents race conditions — one conversation at a time per session
- New messages cancel the current run and start fresh

### Core files

```
src/
├── agent-loop.ts        # The think/act/observe loop
├── client.ts            # Inngest client
├── config.ts            # Configuration
├── worker.ts            # Worker that connects to Inngest Cloud via WebSocket
├── setup.ts             # Channel setup (webhooks, transforms)
├── functions/
│   ├── message.ts       # Main agent function — handles incoming messages
│   ├── heartbeat.ts     # Memory maintenance cron (runs every 30 min)
│   ├── acknowledge-message.ts
│   ├── failure-handler.ts
│   ├── send-reply.ts
│   └── sub-agent.ts     # Delegates tasks to isolated sub-agents
├── channels/
│   ├── slack/           # Slack channel integration
│   └── telegram/        # Telegram channel integration
└── lib/
    ├── tools.ts         # All tools available to the agent
    ├── memory.ts        # MEMORY.md and daily log management
    ├── session.ts       # Session management
    ├── llm.ts           # LLM interface (Anthropic/OpenAI/Google via pi-ai)
    ├── context.ts       # Context building
    └── compaction.ts    # Context compaction
```

---

## Memory System

Utah maintains persistent memory in two files in the workspace directory:

**MEMORY.md** — The agent's long-term memory. Distilled facts, preferences, decisions, and context that persist across sessions. Read automatically at the start of every session.

**Daily logs** — Append-only log files (one per day) that capture what happened during sessions. The heartbeat cron distills these into MEMORY.md automatically.

### How memory works

The heartbeat runs every 30 minutes but only triggers LLM distillation when:
- The daily log exceeds 4KB, OR
- More than 8 hours have passed since the last distillation

Most heartbeat runs are just a file check — no LLM cost. When distillation does run, it reviews the last 7 days of logs and updates MEMORY.md. Logs older than 30 days are pruned automatically.

This means **Ben never has to manually maintain memory**. The system keeps itself current.

---

## Tools Available to Utah

Utah has access to two categories of tools:

### File system tools (from pi-coding-agent)
- `read` — read files with smart truncation
- `edit` — surgical text replacement (not full rewrites)
- `write` — create or overwrite files
- `bash` — shell execution
- `grep` — regex search respecting .gitignore
- `find` — glob-based file discovery
- `ls` — directory listing

### Custom Utah tools
- `remember` — saves a note to today's daily log
- `web_fetch` — fetches a URL and returns the body as text
- `delegate_task` — spawns a sub-agent for complex multi-step tasks

---

## Planned Additions to Utah (Ben's Automation Layer)

The following functions need to be added to `src/functions/` to complete Ben's automation system:

### morning-digest.ts
**Trigger:** Cron at 8:30am EST daily

Steps:
1. Fetch Linear DevRel team tickets (in progress, in review, backlog)
2. Fetch yesterday's messages from #dev-rel-docs Slack (channel ID: C068B3TL06L)
3. Search Notion for context on the highest priority ticket
4. Call Anthropic API to generate a conversational digest
5. DM the digest to Ben's Slack (user ID: U09CL3BC4AF)

Digest tone: a colleague catching Ben up over coffee. Short paragraphs, no bullet lists, no em dashes. Lead with the one thing most worth attention. End with one small specific suggestion.

### pr-watcher.ts
**Trigger:** GitHub webhook event when a PR is opened or updated

Steps:
1. Check if the PR is on a `ben/*` branch in inngest/website
2. Format a brief status update (PR title, status, reviewer if assigned)
3. DM Ben on Slack

### linear-watcher.ts
**Trigger:** Linear webhook event

Steps:
1. Filter for DevRel team tickets only
2. Fire when a ticket moves to In Review or Done
3. DM Ben a brief update on Slack

### context-gatherer.ts
**Trigger:** Callable function (invoked by morning-digest or manually)

Input: Linear ticket ID
Steps:
1. Fetch full ticket details from Linear
2. Search Notion for pages related to the ticket title
3. Search #dev-rel-docs Slack for threads mentioning the ticket or related topic
4. Return structured context object

Used by morning-digest to pre-gather context before briefing Ben.

---

## Connector Abstraction Layer

Utah needs a `src/lib/connectors.ts` file that wraps external services as tools the agent can call without caring about the underlying service. This is the tool-agnostic layer.

Planned tools:

```typescript
get_my_tickets        // wraps Linear API — returns DevRel issues assigned to Ben
get_channel_messages  // wraps Slack API — returns messages from a channel by ID
get_context_for       // wraps Notion search — returns relevant pages for a topic
send_dm               // wraps Slack — sends a DM to a user ID
create_pr_spec        // wraps GitHub — creates a branch-ready spec from a ticket
```

The agent calls these tools generically. If the underlying service changes (e.g. Linear to Jira), only the connector wrapper changes — the agent behavior stays identical.

---

## Deployment

Utah is intended to be deployed on Fly.io, not Vercel. The worker connects to Inngest Cloud via WebSocket — no public endpoint required, no ngrok, no VPS needed for the conversational interface.

For the scheduled functions (morning-digest, pr-watcher, linear-watcher), deployment to Fly ensures they run reliably regardless of whether any local machine is awake.

---

## How Marvin and Utah Work Together

| Layer | Tool | What It Does |
|-------|------|--------------|
| Conversational | Marvin (this repo) | Ben talks to it in Cursor. Drafts docs, gathers context, works through tickets, updates Linear, assigns Cursor as delegate. |
| Automation | Utah | Runs in background. Morning digest, PR status updates, Linear status changes. DMs Ben on Slack. |
| Execution | Cursor | Receives Linear tickets with Cursor as delegate, checks out the git branch, implements changes, opens PRs. |
| Handoff | Linear | The bridge between Marvin and Cursor. Marvin writes the spec into the ticket, assigns Cursor as delegate, Cursor picks it up. |

### The daily workflow in practice

1. Utah fires at 8:30am — generates digest, DMs Ben on Slack
2. Ben opens Marvin in Cursor, says `/start`
3. Marvin reads CLAUDE.md, pulls Linear and Slack, briefs Ben
4. Ben says "let's work on DEV-276"
5. Marvin fetches the ticket, searches Notion and Slack for context, asks Ben what he knows
6. They work through the doc changes conversationally
7. Marvin updates the Linear ticket with the finalized spec and assigns Cursor as delegate
8. Cursor checks out `ben/dev-276-add-public-docs-for-step-metadata`, implements the changes, opens a PR on inngest/website with DEV-276 in the PR description
9. Utah fires a webhook when the PR opens — DMs Ben the status
10. Utah fires again when the ticket moves to In Review — DMs Ben

Ben's job throughout: judgment, direction, and conversation. Everything else is handled.

---

## Key IDs and References

| Thing | Value |
|-------|-------|
| Ben's Slack user ID | U09CL3BC4AF |
| #dev-rel-docs channel ID | C068B3TL06L |
| Linear team | DevRel |
| GitHub repo (docs) | inngest/website |
| GitHub repo (utah) | Benanna2019/utah |
| GitHub repo (marvin) | Benanna2019/marvin-template |
| Notion operating manual | https://www.notion.so/32eb64753bbd81ad8993f735ad54d9b4 |

---

*Last updated: March 25, 2026*
