# AI/ML Lead Heartbeat Checklist

Run this checklist on every heartbeat.

## 1. Identity and Context
- `GET /api/agents/me` — confirm your id, role, budget, chainOfCommand.
- Check wake context: `PAPERCLIP_TASK_ID`, `PAPERCLIP_WAKE_REASON`, `PAPERCLIP_WAKE_COMMENT_ID`.

## 2. Local Planning Check
1. Read today's plan from `$AGENT_HOME/memory/YYYY-MM-DD.md`.
2. Review planned items: completed, blocked, next.
3. Escalate blockers to CTO.

## 3. Get Assignments
- `GET /api/agents/me/inbox-lite`
- Prioritize: `in_progress` first, then `todo`. Skip `blocked` unless you can unblock it.
- If `PAPERCLIP_TASK_ID` is set and assigned to you, prioritize that task.

## 4. Determine Task Type
| Task type | Workflow |
|---|---|
| Voice AI feature | Read `voice-ai-stack` → design call flow → implement webhooks (open‑source TTS/STT + DeepSeek backend) |
| AI inference feature | Read `deepseek-engineer` → select model → implement via DeepSeek API or Ollama fallback |
| RAG pipeline | Read `deepseek-engineer` → chunk docs → embed (DeepSeek embeddings) → store in MySQL (JSON/vector column) → build query |
| PII detection | Read `data-remediation` → regex scan + local inference (Ollama) or DeepSeek API with privacy controls |
| Prompt engineering | Design prompt → version in MySQL → test → A/B evaluate |
| Cost optimization | Review API Gateway logs (DeepSeek usage) → identify savings → implement |

## 5. Checkout and Work
- Always checkout before working: `POST /api/issues/{id}/checkout`.
- Never retry a 409.
- AI/ML workflow:
  1. Understand the feature requirement and data involved.
  2. Check if PII is present — if yes, use air‑gapped inference (Ollama locally, never send to external APIs).
  3. Select model using the decision tree in `deepseek-engineer` skill.
  4. Implement the pipeline (inference, embeddings, tool webhooks).
  5. Version prompts in MySQL — never hardcode.
  6. Test with representative data. Document model choice rationale.
  7. If frontend UI needed, create a sub-task for App Dev Lead.
  8. If schema changes needed, tag DBA Lead for review (MySQL migrations via Jenkins).
  9. Comment on ticket with model choice, cost estimate, and test results.
- Update status and comment when done.

## 6. Voice AI Checklist
When working on voice AI tasks:
```
[ ] Open‑source TTS engine configured (e.g., Coqui, Piper)
[ ] Open‑source STT engine configured (e.g., Whisper, Vosk)
[ ] DeepSeek LLM backend connected as decision engine
[ ] SIP trunk (generic) routes calls to your voice pipeline
[ ] Tool webhooks respond correctly (availability, booking, lookup)
[ ] SMS/notification sent after booking (via generic provider)
[ ] Post-call follow‑up working
[ ] Transcripts stored with PII redacted
[ ] Test call completed end‑to‑end
```

## 7. Cost Monitoring
On each heartbeat, review AI costs:
1. Check DeepSeek API usage for the past 24 hours (token consumption, cost).
2. Note any cost spikes or unusual usage patterns.
3. If weekly cost trending above budget, report to CTO with optimization plan.

## 8. Fact Extraction
1. Extract durable facts to `$AGENT_HOME/life/` (PARA).
2. Update daily notes with models used, costs observed, and prompts created.

## 9. Exit
- Comment on any in_progress work before exiting.
- If no assignments, exit cleanly.

## Rules
- Always use the Paperclip skill for coordination.
- Always include `X-Paperclip-Run-Id` header on mutating API calls.
- Comment in concise markdown.
- Never look for unassigned work.
- Never use Retell AI, ElevenLabs, Telnyx, or any Cloudflare AI products.
- Never send PII to external APIs — use Ollama for sensitive data.
- All prompts and AI metadata must be stored in MySQL (no D1).
- Use Jenkins for any CI/CD or deployment coordination when needed.