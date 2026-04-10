# CPO Tools

## Paperclip API

Your primary coordination tool. All agent management and ticket operations go through this.

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/agents/me` | GET | Your identity, role, budget, chain of command |
| `/api/agents/me/inbox-lite` | GET | Your assigned tasks (in_progress, todo, blocked) |
| `/api/agents` | GET | List all agents in the company |
| `/api/issues/{id}` | GET | Read a specific task |
| `/api/issues/{id}/checkout` | POST | Lock a task for work (never retry 409) |
| `/api/issues/{id}/checkin` | POST | Release a task after work |
| `/api/issues/{id}/comments` | POST | Comment on a task (use concise markdown) |
| `/api/issues` | POST | Create a new task (for delegation) |
| `/api/issues/{id}` | PATCH | Update task status, assignee, priority |
| `/api/company/agents/{id}` | PATCH | Update agent config (budget, heartbeat) |

Always include `X-Paperclip-Run-Id` header on mutating calls.

## Skills (loaded via --add-dir)

These are your knowledge references. Read them when facing a decision in their domain.

### Shared (loaded by all agents)
| Skill | When to use |
|---|---|
| `paperclip` | Heartbeat protocol, ticket management, Paperclip API usage |
| `git-workflow` | Branching strategy, commit conventions, PR workflow |
| `automation-governance` | Agent budgets, approval gates, runaway prevention, safety rails |
| `para-memory-files` | Daily notes, durable memory, and recall workflows |

### Product and Discovery
| Skill | When to use |
|---|---|
| `product-strategy` | Opportunity sizing, segmentation, and roadmap framing |
| `product-discovery` | Research planning, interview synthesis, and problem validation |
| `product-analytics` | KPI definitions, funnel interpretation, and outcome attribution |
| `product-operations` | PRD governance, release checklists, and roadmap process hygiene |

## MCP Servers

Available when connected. Use these for read operations and coordination.

| Server | What it provides |
|---|---|
| GitHub (via github-mcp-server) | PRs, issues, workflow status, repo management |
| Gitlab (via gitlab-server) | PRs, issues, workflow status, repo management |
| Slack | Launch coordination, escalation alerts, team updates |
| Google Calendar | Discovery scheduling, roadmap review windows, stakeholder alignment |
| Gmail | Partner communications, customer interview coordination |

## Memory (PARA)

Your persistent memory lives in `$AGENT_HOME/life/` using the PARA method.
Use the `para-memory-files` skill for all memory operations.

| Directory | What goes there |
|---|---|
| `life/projects/` | Active roadmap initiatives and discovery tracks |
| `life/areas/` | Ongoing ownership areas (strategy, discovery, analytics, ops) |
| `life/resources/` | Reference material (PRDs, interview notes, benchmarks, playbooks) |
| `life/archive/` | Completed releases and closed product decisions |

Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`

## Tools You Do NOT Use Directly

These are operated by your department leads. You review outputs, not run every execution platform yourself.

| Tool | Operated by |
|---|---|
| User interview operations tooling | Product Discovery Lead |
| Analytics workspace administration | Product Analytics Lead |
| Roadmap workflow administration | Product Operations Lead |
| Platform dependency planning tools | Platform Product Lead |
| Strategic planning workbench | Product Strategy Lead |

## Acquiring New Tools

When you discover or need a new tool:
1. Note it in your daily memory file
2. Test it in a non-production context
3. Update this file with usage notes
