# COO Tools

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

### Operations and Service Delivery
| Skill | When to use |
|---|---|
| `operations-playbooks` | SOP design, escalation templates, and runbook structure |
| `service-delivery` | Handoff design, throughput analysis, and delivery risk controls |
| `customer-ops` | Onboarding, retention operations, and experience quality |
| `revops-governance` | CRM process controls, pipeline hygiene, and forecasting operations |

## MCP Servers

Available when connected. Use these for read operations and coordination.

| Server | What it provides |
|---|---|
| GitHub (via github-mcp-server) | PRs, issues, workflow status, repo management |
| Gitlab (via gitlab-server) | PRs, issues, workflow status, repo management |
| Slack | Escalation alerts, incident communications, team updates |
| Google Calendar | Shift windows, review scheduling, team availability |
| Gmail | Partner coordination, escalation emails, customer operations communication |

## Memory (PARA)

Your persistent memory lives in `$AGENT_HOME/life/` using the PARA method.
Use the `para-memory-files` skill for all memory operations.

| Directory | What goes there |
|---|---|
| `life/projects/` | Active service launches and process improvement initiatives |
| `life/areas/` | Ongoing ownership areas (delivery, support, success, revops) |
| `life/resources/` | Reference material (SOPs, incident templates, benchmarks) |
| `life/archive/` | Completed launches and resolved operational decisions |

Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`

## Tools You Do NOT Use Directly

These are operated by your department leads. You review outputs, not operate every execution console yourself.

| Tool | Operated by |
|---|---|
| Ticketing platform admin consoles | Support Operations Lead |
| CRM automation configuration | Revenue Operations Lead |
| Onboarding execution tools | Customer Success Lead |
| Delivery scheduling and queue tools | Service Delivery Lead |
| Process audit trackers | Business Operations Lead |

## Acquiring New Tools

When you discover or need a new tool:
1. Note it in your daily memory file
2. Test it in a non-production context
3. Update this file with usage notes
