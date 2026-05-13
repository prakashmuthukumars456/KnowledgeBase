# Prompt Caching in AI Agents (Claude / MCP / GHCP)

# Overview

Prompt caching is an optimization feature that allows AI systems to reuse previously processed prompt content instead of recomputing the entire context repeatedly.

This is especially important for:

* GitHub Copilot Agent (GHCP)
* Claude Code
* Cursor
* MCP-enabled systems
* Long-running coding agents

Prompt caching helps reduce:

* token cost
* latency
* repeated processing

---

# Core Mental Model

Without caching:

```text
Every request reprocesses:
- system prompts
- MCP tools
- repo context
- instructions
- examples
```

With caching:

```text
Reusable prompt sections are processed once
and reused across future requests.
```

---

# What Gets Cached

The following content can be cached:

* system prompts
* tool definitions
* MCP schemas
* repository summaries
* examples
* previous assistant/user blocks
* tool outputs

Anthropic officially supports caching for:

* `tools`
* `system`
* `messages`

using `cache_control`.

---

# What is a System Prompt?

A system prompt is hidden instruction/context provided to the model.

Example:

```text
You are an enterprise coding assistant.
Follow clean architecture.
Use MCP tools safely.
Prefer minimal changes.
```

Users usually do not see these prompts directly in:

* GHCP
* Cursor
* Claude Code
* Copilot Agent

But they consume tokens.

---

# What is MCP Context?

MCP context usually includes:

```text
Tool schemas
Tool descriptions
Repository summaries
Workspace structure
Agent instructions
```

Example MCP tools:

```text
search_repo
read_file
create_pr
query_work_item
get_kubernetes_logs
```

These tools and schemas are sent to the LLM as context.

---

# Why Prompt Caching Matters

Without caching:

```text
System Prompt = 20k
Tools = 15k
Repo Context = 60k

Total = 95k tokens/request
```

Every request repeats processing.

With caching:

```text
Request 1:
95k cache write

Later requests:
reuse cached context
process only new/dynamic content
```

This significantly reduces:

* latency
* token processing cost

---

# Cache Write vs Cache Read

## Cache Write

First request:

* processes prompt
* stores reusable inference state

Cost:

* slightly more expensive than normal input

---

## Cache Read

Subsequent requests:

* reuse cached prompt prefix
* much cheaper

Anthropic pricing:

* cache writes ≈ 1.25x input
* cache reads ≈ 0.1x input

---

# Default Cache Duration (TTL)

Anthropic prompt caching uses:

```text
5-minute TTL by default
```

Officially:

* cache type = `ephemeral`
* default lifetime = 5 minutes

Optional:

```json
{
  "cache_control": {
    "type": "ephemeral",
    "ttl": "1h"
  }
}
```

---

# IMPORTANT:

# Caching is NOT Automatically Enabled in APIs

For raw Anthropic API usage:

* developers must explicitly enable caching
* applications/frameworks decide whether to use it

Example:

* Claude UI → managed internally
* Cursor → internally optimized
* GHCP → internal GitHub handling
* custom MCP apps → developer responsibility

---

# Does GHCP Expose Cache Controls?

Currently:

* GitHub Copilot does NOT expose direct cache configuration
* No visible:

  * cache hit rate
  * cache statistics
  * cache TTL settings

GitHub likely manages optimizations internally.

---

# What Causes Cache Misses?

Caching depends on identical prompt prefixes.

Good:

```text
[Same system prompt]
[Same tools]
[Same repo context]
[New user request]
```

Cache hit.

Bad:

```text
[Different tool ordering]
[Modified instructions]
[Dynamic logs inserted early]
```

Cache miss.

---

# Static vs Dynamic Content

This is the MOST important practical design principle.

---

# Static Content (Cache-Friendly)

Rarely changes.

Examples:

* coding standards
* architecture guidelines
* repo instructions
* MCP schemas
* reusable examples
* agent instructions

Place these at the beginning.

---

# Dynamic Content (Non-Cacheable)

Changes frequently.

Examples:

* current bug
* latest logs
* stack traces
* active file
* latest tool output

Place these at the end.

---

# GOOD Prompt Structure

```text
[STATIC]
system prompt
repo instructions
MCP tools
architecture summary

[DYNAMIC]
current bug
latest logs
current diff
```

This maximizes cache reuse.

---

# BAD Prompt Structure

```text
logs
stack traces
system instructions
repo summary
more logs
```

Dynamic content near the beginning breaks caching.

---

# GHCP Practical Example

## Static Context

Store in:

* `.github/copilot-instructions.md`
* repository skills
* agent configuration

Example:

```text
Project uses:
- Clean Architecture
- CQRS
- MediatR
- xUnit

Coding standards:
- async everywhere
- unit tests mandatory
```

---

# Dynamic Prompt

Only send current task:

```text
Fix guest login failure in AuthService.cs

Stack trace:
...
```

This is significantly more efficient.

---

# Important Anthropic Cache Rules

## Minimum cache size

Anthropic cache requires minimum token thresholds:

* ~1024 tokens for Sonnet/Opus variants
* larger for some Haiku models

Smaller prompts may silently skip caching.

---

# Parallel Request Nuance

Cache becomes available AFTER first response begins.

Good:

```text
Request 1 → seeds cache
Requests 2-10 → reuse cache
```

Bad:

```text
10 parallel requests immediately
```

Many may miss cache.

---

# Best Practices

## Keep static content reusable

Move reusable knowledge into:

* repo instructions
* agent skills
* MCP configuration

---

## Avoid reposting architecture repeatedly

Do not resend:

* entire repo structure
* coding standards
* architecture docs
* huge examples

every prompt.

---

## Limit tool outputs

Avoid returning:

* huge logs
* massive JSON
* entire repositories

Prefer summarized outputs.

---

# Key Takeaway

Most enterprise AI cost problems come from:

* prompt structure
* context size
* MCP schemas
* large tool outputs
* cache inefficiency

—not the final generated answer itself.

Efficient agent systems focus heavily on:

* prompt layering
* cache-friendly design
* context compression
* static/dynamic separation
* tool output management

---

# Official References

* https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
* https://docs.anthropic.com/en/docs/build-with-claude/token-counting
* https://docs.anthropic.com/en/docs/agents-and-tools/mcp

# Additional Research

* Prompt Cache Research Paper:
  https://arxiv.org/abs/2311.04934

* Agentic Cache Optimization Research:
  https://arxiv.org/abs/2601.06007
