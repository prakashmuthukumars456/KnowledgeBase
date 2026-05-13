You can use these official Anthropic documentation links directly with your team:

# Official Anthropic Docs

* [Claude Tool Use Overview](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview?utm_source=chatgpt.com)
* [Claude Tool Use Documentation](https://docs.claude.com/en/docs/agents-and-tools/tool-use/implement-tool-use?utm_source=chatgpt.com)
* [Token Counting Documentation](https://docs.claude.com/en/docs/build-with-claude/token-counting?utm_source=chatgpt.com)
* [Context Windows Explained](https://docs.claude.com/en/docs/build-with-claude/context-windows?utm_source=chatgpt.com)
* [MCP (Model Context Protocol) Documentation](https://docs.claude.com/en/docs/agents-and-tools/mcp?utm_source=chatgpt.com)
* [Prompt Caching Documentation](https://docs.claude.com/en/docs/build-with-claude/prompt-caching?utm_source=chatgpt.com)
* [Extended Thinking / Reasoning Docs](https://docs.claude.com/en/docs/build-with-claude/extended-thinking?utm_source=chatgpt.com)

---

Below is a concise `.md` document you can directly add to your repo/wiki/team docs.

# Understanding Tool Use, MCP, and Token Consumption in Claude/AI Agents

## Overview

Modern AI agents such as Claude Code, Cursor, GitHub Copilot Agents, Roo Code, and MCP-enabled systems use **tool calling** to interact with external systems.

Examples:

* File system access
* GitHub
* Azure DevOps
* Kubernetes
* Databases
* Browser automation
* Web search

The model itself does not execute tools directly. Instead:

```text
User Prompt
   ↓
LLM decides tool is needed
   ↓
Tool call JSON emitted
   ↓
Client/Application executes tool
   ↓
Tool result returned to model
   ↓
LLM generates response
```

---

# Important Mental Model

## Everything visible to the LLM becomes tokens

This includes:

* System prompts
* Tool definitions
* Tool descriptions
* Tool schemas
* Tool outputs
* File contents
* Logs
* Previous conversation history

The external API/tool execution itself does NOT consume tokens.

The context sent to the LLM does.

---

# Why Token Usage Suddenly Increases

Usually caused by:

| Cause                 | Description                          |
| --------------------- | ------------------------------------ |
| Tools increased       | More MCP tools loaded                |
| Context increased     | Longer conversation history          |
| Schemas bloated       | Large JSON/OpenAPI schemas           |
| Tool outputs exploded | Huge logs/files returned             |
| Recursive loops       | Agent repeatedly calling tools       |
| Parallel tools        | Multiple tools called simultaneously |

---

# 1. Tool Definitions (Hidden Cost)

When MCP or agents start, many tools may already be loaded automatically.

Example tools:

```text
read_file
write_file
search_code
bash
git_diff
create_pr
ado_work_items
kubernetes_logs
docker_exec
browser_search
```

Each tool includes:

* name
* description
* parameters/schema

All of this is included in the request context.

## Example

Even if user prompt is:

```text
Fix login bug
```

Actual hidden request may already contain thousands of tokens of tool metadata.

---

# 2. Hidden System Prompts

Anthropic/OpenAI internally inject additional instructions for tool handling.

Example hidden instructions:

```text
You may use tools.
Validate tool arguments.
Return structured JSON.
Avoid hallucinating tool outputs.
```

These hidden prompts consume tokens even if tools are never used.

## Why Larger Models Consume More

More advanced models typically receive:

* additional reasoning instructions
* safety guidance
* orchestration policies

Example:

* Claude Opus → larger hidden prompts
* Claude Sonnet → smaller hidden prompts

---

# 3. Large Tool Descriptions

Bad example:

```json
{
  "description": "This advanced semantic repository traversal system..."
}
```

Good example:

```json
{
  "description": "Search repository files"
}
```

Large descriptions increase prompt size unnecessarily.

---

# 4. Recursive Agent Loops

A recursive agent loop happens when agents repeatedly call tools.

Example:

```text
search_code
↓
read_file
↓
run_tests
↓
read_logs
↓
search_again
```

Every iteration:

* adds tokens
* increases context
* increases cost

This is common in coding agents.

---

# 5. Parallel Tool Calls

Instead of sequential execution:

```text
search repo
then search logs
then read tests
```

Agent frameworks may execute:

```text
search repo + search logs + read tests
```

simultaneously.

Benefits:

* faster execution

Tradeoff:

* more context returned immediately
* higher token usage

---

# 6. Context Explosion

The biggest hidden problem.

## Small output

```text
3 errors found
```

Cheap.

## Large output

```text
Entire source file
Entire PR diff
Entire build logs
Entire Kubernetes manifests
```

Can become tens of thousands of tokens from a single tool result.

---

# Real Example

User asks:

```text
Fix typo
```

Agent workflow:

```text
read_repository
read_file
run_tests
collect_logs
search_imports
```

Result:

* 200k+ context tokens possible
* despite tiny user request

---

# MCP and Tool Ecosystem

Common MCP integrations:

* GitHub
* Azure DevOps
* Kubernetes
* Docker
* Slack
* Jira
* PostgreSQL
* AWS

Loading many MCP servers increases:

* tool count
* schema size
* hidden prompt overhead

---

# Best Practices

## Keep tool descriptions short

Good:

```text
Search repository files
```

Bad:

```text
Advanced semantic multi-index traversal system...
```

---

## Minimize schemas

Avoid deeply nested OpenAPI definitions where possible.

---

## Limit tool outputs

Prefer summaries:

Good:

```text
5 errors found
```

Bad:

```text
20,000-line logs
```

---

## Trim context

Avoid keeping unnecessary:

* logs
* file contents
* previous outputs

---

## Use summarization

Compress previous reasoning/results before continuing long sessions.

---

# Key Takeaway

Most token costs in agentic systems come from:

* context size
* tools
* schemas
* outputs
* recursive workflows

—not from the final generated answer itself.

---

# References

* [https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview)
* [https://docs.claude.com/en/docs/build-with-claude/token-counting](https://docs.claude.com/en/docs/build-with-claude/token-counting)
* [https://docs.claude.com/en/docs/agents-and-tools/mcp](https://docs.claude.com/en/docs/agents-and-tools/mcp)
