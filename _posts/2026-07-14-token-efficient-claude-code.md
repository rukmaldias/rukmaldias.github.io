---
layout: post
title: "Token-Efficient Claude Code: Graphify, Routing, and Persistent Memory"
date: 2026-07-14
description: "Claude Code burns tokens on context overhead, not bad prompting. A three-phase stack — graphify for code structure, a routing table in CLAUDE.md, and mem0 for persistent memory — cuts per-task token cost by 40-70%."
---

Claude Code is powerful for coding, analysis and design, and documentation work, but it can burn through tokens fast. The reason isn't bad prompting — it's context overhead.

Every session starts with a fixed baseline. Even a "hi" prompt consumes roughly 31,000 tokens before you type anything, because Claude Code loads the system prompt, global and project `CLAUDE.md` files, skill descriptions, every connected MCP server's full tool schema, and session state. That baseline is present in *every* message you send. Then on top of it, each question causes Claude to `grep`, `read`, and re-read files to answer, which compounds the cost.

This post covers a three-phase stack that pays back the highest ROI per hour of setup: a code knowledge graph, an explicit routing table, and persistent cross-session memory.

---

## What We're Solving

Three distinct classes of waste:

1. **Repeated code exploration.** Every question about the codebase makes Claude read the same files again. A pre-computed graph of the code answers structural questions without file reads.
2. **Repeated context re-explanation.** Across sessions, Claude forgets decisions, conventions, and preferences. You end up re-explaining "we use Postgres 14," "handlers follow this naming convention," and so on. A persistent memory store fixes this.
3. **Uncontrolled routing.** Without explicit instructions, Claude falls back to `grep + read` heuristics for every kind of question. A routing table in `CLAUDE.md` tells it to use structural tools first, file-reading only as fallback.

### What we're *not* solving

- Extended thinking overhead — turn it off for simple tasks, use `/model` to route down to Sonnet or Haiku
- Poor prompting — specific prompts with `@path/to/file` references beat vague ones; that's habit, not tooling
- Large output verbosity — add a `concise-output` skill separately if needed

These are worth addressing later. The stack below is what to fix first.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Claude Code session                                     │
│  (lean CLAUDE.md ~90 tokens, .claudeignore, Sonnet)      │
└────────────────────┬──────────────────────────────────────┘
                      │
       ┌──────────────┴──────────────┐
       │  Query routing (CLAUDE.md)  │
       │  Code? Decision? Fact?      │
       └──────┬────────────────┬─────┘
              │                │
      ┌───────▼──────┐  ┌──────▼──────────────────────┐
      │  graphify    │  │  mem0 MCP (self-hosted)      │
      │  code graph  │  │                              │
      │  (local)     │  │  ├── Qdrant (vector store)   │
      │              │  │  ├── Ollama + bge-m3         │
      │              │  │  │   (embeddings only)       │
      │              │  │  └── Claude OAT (extract,    │
      │              │  │      optional / disabled)    │
      └──────────────┘  └──────────────────────────────┘
```

| Component | Handles | Cost model |
|---|---|---|
| **graphify** | Code structure, call chains, impact | One-time build; local |
| **CLAUDE.md** | Routing decisions per query | ~90 tokens/session |
| **mem0 MCP** | Persistent cross-session facts | Local infra |
| **Qdrant** | Vector storage for memories | Docker container |
| **Ollama bge-m3** | Semantic embeddings for memory search | ~1.2 GB, idle CPU |

---

## Prerequisites

Verify these before starting:

```shell
brew --version      # macOS package manager
python3 --version   # 3.10 or higher
which uvx           # for running Python tools in isolated envs
docker --version    # for Qdrant
claude --version    # Claude Code CLI
```

If `uvx` is missing:

```shell
brew install uv
uv tool update-shell   # add uv's bin dir to PATH
# open a NEW terminal so the PATH change takes effect
```

---

## Phase 0: Baseline Hygiene

Before installing anything, do this. It costs zero tokens and delivers the biggest measurable savings.

### Measure your current baseline

Open Claude Code in your project and run `/context`. This shows a breakdown of what's consuming your context window at session start — system prompt, `CLAUDE.md`, MCP server schemas, skills. Write the number down; it's your "before" measurement.

Also run `/usage` to see recent spend attributed per skill, plugin, and MCP server. On Pro/Max plans the dollar figure is a local estimate, not your actual bill — treat it as a relative guide.

### Trim CLAUDE.md

A 5,000-token `CLAUDE.md` costs 5,000 tokens on *every* turn. Anthropic's guidance is under 200 lines. Ours ends up around 90 tokens total (see Phase 2). If you already have one, back it up first:

```shell
cp ~/.claude/CLAUDE.md ~/.claude/CLAUDE.md.backup
```

### Add .claudeignore

Same syntax as `.gitignore`. Keeps Claude out of build artifacts, generated files, and directories it shouldn't explore:

```shell
cat > .claudeignore <<'EOF'
node_modules/
dist/
build/
.next/
target/
vendor/
*.min.js
*.lock
.git/
coverage/
EOF
```

This quietly saves more tokens than trimming `CLAUDE.md` ever will.

### Model routing

Set the default subagent model to Haiku. Exploration agents run cheap; the main session stays on Sonnet for quality.

```shell
export CLAUDE_CODE_SUBAGENT_MODEL=haiku
echo 'export CLAUDE_CODE_SUBAGENT_MODEL=haiku' >> ~/.zshrc
```

Inside sessions, `/model sonnet` is the daily default. Only switch to `/model opus` for genuine architecture decisions or multi-file bugs.

### Defer MCP tool schemas

This matters a lot when adding MCP servers — we're about to add mem0. Every connected MCP server normally loads its full tool schema at session start, which can silently add 50,000–70,000 tokens.

```shell
export ENABLE_TOOL_SEARCH=1
echo 'export ENABLE_TOOL_SEARCH=1' >> ~/.zshrc
```

With this on, MCP tools load lazily, only when Claude actually needs them.

---

## Phase 1: Graphify — Code Knowledge Graph

Graphify scans your codebase and builds a queryable knowledge graph — every class, function, import, and call relationship becomes a node. When Claude asks "how does auth work," it queries the graph instead of grepping through files. Fewer file reads, fewer tokens, more accurate answers, because the graph captures structure that keyword search misses.

It also indexes Markdown, PDFs, Office docs, SQL schemas, and Terraform in the same graph — so a `docs/` folder is covered without a separate RAG stack.

### Install

```shell
# Package name is graphifyy (double-y), command is graphify
uv tool install graphifyy
uv tool update-shell
# open a NEW terminal
```

Verify:

```shell
which graphify
graphify --version
```

### Register the skill with Claude Code

```shell
graphify install --platform claude
```

This writes `~/.claude/skills/graphify/SKILL.md`. Claude Code discovers it at session start and exposes `/graphify` as a slash command.

**Known issue: macOS PATH resolution** — on some setups, `graphify install` itself fails with "command not found" even though the binary exists. Workaround:

```shell
uv tool run --from graphifyy graphify install --platform claude
```

### Build the graph

Inside your project directory, from within Claude Code:

```
/graphify .
```

This takes 30 seconds to a few minutes depending on repo size. Output lands in `graphify-out/`:

- `graph.html` — interactive visualization (open in browser)
- `graph.json` — the full graph
- `GRAPH_REPORT.md` — key concepts and suggested questions

For daily updates on changed files only:

```shell
graphify --update
```

Fast, minimal token cost. Consider installing a git post-commit hook so this runs automatically:

```shell
graphify hook install
```

### Verify graphify is being used

Ask Claude a structural question about your codebase, e.g. "Where is authentication handled?" If graphify is working, Claude answers via the graph without a burst of `grep` / `read` calls. If you see it grepping through files first, the CLAUDE.md routing rule from Phase 2 isn't in place yet — that's what tells Claude to prefer the graph.

---

## Phase 2: Intent Router in CLAUDE.md

Without explicit routing, Claude Code defaults to `grep + read` for every question. A terse routing table redirects structural queries to graphify and memory queries to mem0.

Critical rule: only add routing lines for tools that actually exist. Route to a tool that isn't installed, and Claude either wastes a turn discovering that or falls back silently — and you never notice the rule is broken.

### The final CLAUDE.md

After all phases, `~/.claude/CLAUDE.md` should be exactly this — around 90 tokens:

```markdown
# Query routing
- Code structure/flow/impact → graphify (MCP query_graph or `graphify query`); Grep/Read only as fallback
- Past decisions, preferences, conventions → mem0 `search_memories` FIRST. Do NOT read local memory files.

# mem0 rules
- Before `add_memory`: always `search_memories` first. If match score > 0.5, use `update_memory` instead — never duplicate.
- Always pass `infer=false` to `add_memory`. Store text verbatim.
- Session start: `search_memories` for context before asking me to re-explain.
- One fact per memory. Terse.
```

**Note:** add the mem0 lines only *after* Phase 3 is working. Add them first and Claude will try to call mem0 tools that don't exist yet.

### Design principles

- **Table format, not prose.** Bullets beat paragraphs on token count.
- **Never duplicate a rule.** Every duplicated line costs tokens per request.
- **No routing to non-existent tools.** An uninstalled tool reference is worse than no rule.
- **No verbose examples.** One-line rules are enough.
- **Verify what already loads.** Run `/memory` inside Claude Code to see every file that loaded at session start. If `graphify install` already wrote a SKILL.md describing its own trigger, don't duplicate that in CLAUDE.md.

---

## Phase 3: Persistent Memory (mem0)

mem0 stores durable facts about your work — architectural decisions, coding conventions, debugging insights — in a semantic vector store. Claude Code retrieves relevant memories at session start via a hook, and can add or search memories mid-session via MCP tools.

### Architecture choice: hybrid, not fully local

The `mem0-mcp-selfhosted` server supports three modes:

| Mode | Extraction LLM | Embeddings | Storage | Cost |
|---|---|---|---|---|
| Full cloud | Cloud mem0.ai | Cloud | Cloud | Paid, off-machine |
| Fully local | Ollama qwen3 | Ollama bge-m3 | Qdrant | Free, heavy on CPU |
| Hybrid | Claude OAT | Ollama bge-m3 | Qdrant | Free, light |

I picked **hybrid**: Ollama runs only the small `bge-m3` embedding model (~1.2 GB), Qdrant stores vectors locally, and fact extraction defaults to Claude via the existing Code session token. No Neo4j — graph memory is disabled, since it triples LLM calls per save.

Important: I ended up bypassing LLM extraction entirely using `infer=false`. See "Real issues encountered" below.

### Step 1: Run Qdrant

```shell
docker run -d --name qdrant \
  -p 6333:6333 \
  -v ~/qdrant_storage:/qdrant/storage \
  --restart unless-stopped \
  qdrant/qdrant

# verify
curl -s http://localhost:6333/collections
# should return: {"result":{"collections":[]},"status":"ok",...}
```

The `-v ~/qdrant_storage:/qdrant/storage` flag persists memories on disk. Without it, restarting the container wipes everything.

### Step 2: Install Ollama and pull bge-m3

```shell
brew install ollama
brew services start ollama    # keeps it running in background

ollama pull bge-m3

# verify
curl -s http://localhost:11434/api/tags | grep bge-m3
```

`bge-m3` is an *embedding* model, not a chat model — it uses ~1.2 GB of memory when loaded, ~0 CPU when idle. Don't pull a chat model just for mem0; Claude handles extraction, not Ollama.

### Step 3: Register the MCP server (with the version pin)

`mem0-mcp-selfhosted` has a version drift bug: it was built against `mem0ai < 2.0`, but `uvx` resolves the latest `mem0ai` (2.x) unless told otherwise. On 2.x, `get_memories` and `search_memories` both fail with a `user_id` parameter mismatch — reads are broken while writes silently succeed, which is worse than a hard failure. Pin the library at install:

```shell
claude mcp remove mem0 --scope user 2>/dev/null  # ok if not present

claude mcp add --scope user --transport stdio mem0 \
  --env MEM0_USER_ID=<your-username> \
  -- /path/to/uvx \
     --with 'mem0ai<2.0' \
     --from git+https://github.com/elvismdev/mem0-mcp-selfhosted.git \
     mem0-mcp-selfhosted
```

Adjust `MEM0_USER_ID` to your own identifier, and `/path/to/uvx` to the absolute path from `which uvx`. Using the absolute path avoids the common failure mode where Claude Code launches the MCP subprocess with a stripped PATH that doesn't include `~/.local/bin`.

### Step 4: Pre-warm the environment

The first `uvx` run downloads and builds ~125 packages. This can take 30-90 seconds, longer than Claude Code's MCP handshake timeout. Pre-warm so the first launch doesn't fail:

```shell
MEM0_USER_ID=<your-username> uvx \
  --with 'mem0ai<2.0' \
  --from git+https://github.com/elvismdev/mem0-mcp-selfhosted.git \
  mem0-mcp-selfhosted
```

Wait for "Installed N packages" to appear. Warnings about `extra 'graph'` or `google-cloud-aiplatform` are cosmetic — ignore them. When the server hangs silently (waiting for MCP stdio input), it works. Hit `Ctrl-C`.

### Step 5: Restart Claude Code and verify

Fully quit Claude Code (`Cmd-Q`), then relaunch, and run `/mcp`. It should show `mem0 · connected · 11 tools`. If it shows "failed," check "Real issues encountered" below.

### Step 6: End-to-end test

```
Use mem0 add_memory to store "Uses Emacs with org-roam on macOS" with infer=false
Use mem0 search_memories with query "editor"
Use mem0 get_memories to list everything
```

All three must succeed. Search should match "editor" against text containing "Emacs" — that's semantic embedding at work (similarity score around 0.35-0.5 is normal).

### Step 7: Whitelist mem0 tools (skip approval prompts)

By default Claude Code prompts you to approve every MCP tool call. Add to `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__mem0__*"
    ]
  }
}
```

Now all mem0 tool calls run without prompting. Same treatment for graphify tools if you want them silent too.

### Step 8: Install session hooks

Hooks make memory automatic — SessionStart injects relevant memories as context, Stop summarizes and saves the session.

```shell
uvx --with 'mem0ai<2.0' \
  --from git+https://github.com/elvismdev/mem0-mcp-selfhosted.git \
  mem0-install-hooks --global
```

This patches `~/.claude/settings.json` with the hook entries. Idempotent.

### Real issues encountered (and fixes)

**429 rate limit on add_memory.** mem0 defaults extraction to `claude-opus-4-6` — the most expensive option. Every `add_memory` call ate Opus tokens. Fixed by passing `infer=false`, which skips LLM extraction entirely and stores text verbatim. For a token-minimization goal, this is arguably the right default anyway.

**401 auth error on LLM calls.** Even after switching model, LLM extraction returned 401. Root cause not fully diagnosed — could be OAT token scope, could be a model string mismatch. Since `infer=false` sidesteps this entirely, I didn't chase it.

**get_memories / search_memories broken with mem0ai 2.x.** As documented above. Fixed by `--with 'mem0ai<2.0'` at install.

**Duplicate memories.** The first `add_memory` call created a duplicate of an existing fact because there was no "check first" rule. Fixed with the "search before add" line in CLAUDE.md. Delete duplicates as they accumulate with `mem0 delete_memory <id>`.

**Slow add_memory (~50 seconds).** Even with `infer=false`, the first save took ~50s — suspected cold-start overhead in the MCP server or Ollama loading `bge-m3`. Subsequent calls should be under 2 seconds. If persistent slowness continues, investigate.

---

## Measurement: Before vs After

Three metrics matter, in this order:

1. **Session-start baseline** (tokens loaded before you type) — Phase 0's `/context`. This is what CLAUDE.md size, MCP schemas, and skills cost on *every* turn.
2. **Per-task total** (tokens per completed task) — cost of a typical work unit, measured with `/usage` at task end minus start.
3. **Question type mix** — which *kinds* of questions eat the most tokens. `/usage` breaks this down per skill, plugin, and MCP server.

### Before-implementation baseline

Do this *before* Phase 0, or on a fresh install — otherwise there's nothing to compare against. Pick 5 representative questions from real work, for example:

1. "Where is user authentication handled in this repo?"
2. "What are the naming conventions for API handlers here?"
3. "Trace the request flow from HTTP entry to database call."
4. "What did we decide about caching last week?" (memory test)
5. "Summarize the API spec in `docs/api.md`." (doc test)

For each question, in a *fresh* Claude Code session (`/clear` or new launch): run `/context` and note the baseline, ask the question, then run `/usage` and note the total. Record it:

| # | Question type | Baseline | After answer | Delta | Wall time |
|---|---|---|---|---|---|
| 1 | code structure | 31000 | 78000 | 47000 | 42s |
| 2 | conventions | 31000 | 65000 | 34000 | 28s |
| 3 | request flow | 31000 | 92000 | 61000 | 55s |
| 4 | past decision | 31000 | 45000 | 14000 | 12s |
| 5 | doc summary | 31000 | 71000 | 40000 | 38s |
| **Σ** | | | | **196000** | **175s** |

### After-implementation measurement

Same 5 questions, same repo, fresh sessions between each. Expected pattern:

| # | Question type | Baseline | After answer | Delta | Wall time |
|---|---|---|---|---|---|
| 1 | code structure | ~28000 | ~40000 | ~12000 | ~15s |
| 2 | conventions | ~28000 | ~35000 | ~7000 | ~10s |
| 3 | request flow | ~28000 | ~48000 | ~20000 | ~22s |
| 4 | past decision | ~28000 | ~32000 | ~4000 | ~5s |
| 5 | doc summary | ~28000 | ~42000 | ~14000 | ~18s |

The wins come from:

- **Questions 1, 3** — graphify answers structurally, avoiding file reads
- **Question 4** — mem0 has the answer cached, no exploration
- **Question 5** — graphify indexes `docs/`, no rebuild-from-scratch
- **Question 2** — graphify surfaces patterns across handlers without reading all of them

If numbers don't improve as expected, check `/context` and `/usage` to see what's still costing tokens.

### Ongoing measurement

Once a week, review `/context` (is baseline creeping up?) and `/usage` (is a specific MCP eating a disproportionate share?). Check whether memories are actually being retrieved — if mem0 shows near-zero usage, the CLAUDE.md rule isn't firing.

Numbers are one measure. Also notice: do you still find yourself pasting file contents instead of using `@path/to/file`? Do you re-explain "we use X for Y" across sessions? Does Claude answer structural questions instantly? These are the qualitative tells that the stack is actually working.

---

## Ongoing Maintenance

**Weekly** — run `graphify --update` if you haven't installed the git hook, review `/context` and `/usage`, delete stale mem0 entries (memories going stale silently is a known failure mode; the `update_memory` rule helps but isn't perfect).

**Monthly** — reinstall the mem0 server with `uvx --refresh` to pick up upstream fixes (careful, this may reintroduce the mem0ai version drift issue — re-verify with the three end-to-end tests). Prune `~/qdrant_storage/` if it grows large. Review CLAUDE.md for rules you never see fire.

**When something breaks**, two failure modes are most likely:

1. **mem0 MCP fails to connect after a system update** — almost always PATH or Python version drift. Debug by verifying prerequisites (`curl` Qdrant and Ollama, `which uvx`), then running the server standalone to see the real error.
2. **Graphify gives stale answers** — you forgot to `--update` after a big refactor. Rebuild from scratch with `graphify . --force`.

---

## What I Deliberately Skipped

- **Doc RAG via LlamaIndex MCP.** Graphify already indexes `docs/` Markdown, PDFs, and Office docs alongside code. A second RAG pipeline would be redundant — revisit only if graphify's doc coverage is inadequate for real questions.
- **A unified router skill.** The routing table in CLAUDE.md already does what a "router skill" would. A separate skill file would duplicate the rules and add load-time cost.
- **Neo4j / graph memory in mem0.** Every graph-enabled `add_memory` triggers 3 extra LLM calls (entity extraction, relationships, conflicts). Semantic vector search over Qdrant is sufficient for "what did we decide about X" questions — graphify handles code-side structural relationships separately.
- **A fully local LLM for mem0 extraction.** Would need a 7B+ model with good tool-calling — heavy for a 2019 Intel MacBook. `infer=false` sidesteps the entire question with zero downside.

---

## Summary

The end state:

- graphify indexing code and docs into a queryable graph
- mem0 persisting durable facts, semantically searchable across sessions
- a ~90-token CLAUDE.md routing questions to the right tool
- `.claudeignore` keeping generated files out of exploration
- model routing — subagents to Haiku, default to Sonnet
- deferred MCP tool schema loading

Measured against the 5-question benchmark, expect a 40-70% reduction in per-task token cost, mostly from avoided file reads on structural questions.

Don't add more tooling until you can point at a specific measurement showing the current stack isn't enough.
