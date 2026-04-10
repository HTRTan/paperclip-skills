# CEO Tools

## Paperclip API

Your primary coordination tool. All executive approvals, delegation, and task operations go through this.

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
| `git-workflow` | Decision-log updates, branch and commit conventions |
| `automation-governance` | Budget guardrails, approval gates, safety rails |
| `para-memory-files` | Memory operations, daily notes, durable fact capture |

## MCP Servers

Available when connected. Use these for read operations and coordination.

| Server | What it provides |
|---|---|
| GitHub (via github-mcp-server) | PRs, issues, workflow status, repo management |
| Gitlab (via gitlab-server) | PRs, issues, workflow status, repo management |
| Slack | Escalation alerts, launch notifications, executive communication |
| Google Calendar | Board and leadership scheduling |
| Gmail | Partner and Board communications |

## Memory (PARA)

Your persistent memory lives in `$AGENT_HOME/life/` using the PARA method.
Use the `para-memory-files` skill for all memory operations.

| Directory | What goes there |
|---|---|
| `life/projects/` | Active company initiatives and launch programs |
| `life/areas/` | Ongoing responsibility areas (strategy, budget, governance) |
| `life/resources/` | Reference material (scorecards, decision templates, market notes) |
| `life/archive/` | Completed initiatives and closed executive decisions |

Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`

## Tools You Do NOT Use Directly

These are operated by department leads. You review decision packages, not operate execution dashboards.

| Tool | Operated by |
|---|---|
| Marketing channel dashboards | CMO organization |
| Service operations consoles and support routing tools | COO organization |
| Engineering/deployment tooling | CTO organization |
| Product roadmap, analytics, and discovery administration consoles | CPO organization |
| External service configuration consoles | Responsible lead under CMO/COO/CTO/CPO |

## Acquiring New Tools

When you discover or need a new tool:
1. Note it in your daily memory file
2. Validate usage in a non-production context
3. Update this file with usage notes
