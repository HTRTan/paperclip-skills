# AI/ML Lead Tools

## Paperclip API
Same endpoints as CTO — inbox, checkout, checkin, comment, update status.

## Skills
| Skill | When to use |
|---|---|
| `paperclip` | Coordination, ticket management |
| `deepseek-engineer` | Model selection (DeepSeek-V3, R1, Coder), inference patterns, prompt management, cost optimization, embeddings |
| `data-remediation` | PII detection in transcripts, air‑gapped data cleaning (Ollama or DeepSeek privacy mode) |
| `mysql-stack` | Database schema design, queries, vector storage, prompt version tables, audit logs |

## CLI Tools
| Tool | Purpose |
|---|---|
| `curl` | Test tool webhook endpoints |
| `git` | Branching, commits, PRs |

## External Services (you configure these)
| Service | Dashboard / Endpoint | Purpose |
|---|---|---|
| DeepSeek API | platform.deepseek.com | LLM inference, embeddings (primary model) |
| Ollama (self‑hosted) | localhost:11434 | Air‑gapped inference for PII‑sensitive data |
| SIP trunk provider (generic) | (e.g., Twilio, Telnyx‑allowed only if no Cloudflare) | Phone numbers, SIP trunks, SMS |
| Open‑source TTS/STT | (self‑hosted Coqui, Whisper, Vosk) | Voice synthesis and transcription |
| mysql-stack | (your internal DB) | Prompt versioning, call logs, RAG vector storage |
| Jenkins | (your internal CI/CD) | Build, test, and deployment pipelines |

## Tools You Do NOT Use
| Tool | Operated by |
|---|---|
| Frontend UI code | App Dev Lead |
| `npx playwright test` | QA Lead |
| Database schema approval (beyond AI tables) | DBA Lead |
| ElevenLabs, Telnyx, Retell AI | Forbidden |
| Cloudflare AI Gateway, Workers AI, Vectorize, D1 | Replaced by DeepSeek + MySQL |

## Memory (PARA)
Use `para-memory-files` skill. Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`.

## Key Files / Tables to Track
- DeepSeek API usage logs (via platform dashboard or internal logging)
- Prompt versions in MySQL (`prompts` table)
- Voice AI call logs in MySQL (`call_log` table)
- PII detection results and redaction audit (stored in MySQL with access controls)
- Jenkins pipeline definitions (`Jenkinsfile` in your repo)

## References
- `$AGENT_HOME/HEARTBEAT.md` — execution checklist
- `$AGENT_HOME/SOUL.md` — persona and voice