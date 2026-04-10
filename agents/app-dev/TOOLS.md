# App Dev Lead Tools (SvelteKit + Spring Boot + MySQL)

## Paperclip API
Same endpoints as CTO — see CTO TOOLS.md for full reference.
Key operations: inbox, checkout, checkin, comment, update status.

## Skills
| Skill | When to use |
|-------|--------------|
| `paperclip` | Coordination, ticket management |
| `sveltekit-frontend` | UI components, pages, layouts, load functions, form actions (SvelteKit + TypeScript) |
| `springboot-mybatis` | Backend REST APIs (Spring Boot 3.x), MyBatis mappers, services, DTOs |
| `database-design` | Schema proposals, migration scripts (Flyway/Liquibase) – coordinate with DBA Lead |
| `code-reviewer` | PR review checklists, anti-patterns (both frontend and backend) |
| `git-workflow` | Branching, commits, PR workflow |
| `software-architect` | ADRs, architecture decisions (full-stack) |
| `mysql-stack` | MySQL conventions, test data setup, migration safety |

## CLI Tools (Frontend)
| Tool | Purpose |
|------|---------|
| `npm run dev` | Start SvelteKit dev server (localhost:5173) |
| `npm run build` | Build frontend for production |
| `npx biome check .` | Lint and format (or ESLint + Prettier) |
| `npx vitest run` | Run frontend unit/component tests |
| `npx vitest run --coverage` | Frontend coverage report |
| `npm run preview` | Preview production build locally |

## CLI Tools (Backend)
| Tool | Purpose |
|------|---------|
| `./mvnw spring-boot:run` | Run Spring Boot app (dev mode) |
| `./mvnw compile` | Compile backend |
| `./mvnw test` | Run JUnit 5 tests |
| `./mvnw verify -Pintegration` | Run integration tests with Testcontainers |
| `./mvnw jacoco:report` | Generate coverage report (JaCoCo) |
| `./mvnw mybatis-generator:generate` | Generate MyBatis mappers from schema (optional) |
| `mysql -h localhost -u root -p` | Connect to MySQL dev database |
| `flyway migrate -url=jdbc:mysql://localhost:3306/db` | Apply migrations locally (dev only) |
| `flyway info` | Check migration status |
| `docker-compose up -d mysql` | Start MySQL container for local dev |

## Database & Migration Tools (Coordination)
| Tool | Purpose | Operated by |
|------|---------|--------------|
| `flyway migrate` (local dev) | Apply migrations to dev database | You (App Dev Lead) |
| `flyway migrate` (prod/staging) | Apply migrations | DevOps Lead (after DBA review) |
| `./mvnw generate-migrations` | Create Flyway SQL script from schema diff | You, but DBA reviews |

## Tools You Do NOT Use
| Tool | Operated by |
|------|--------------|
| `flyway migrate` on production | DevOps Lead (you advise, DBA reviews) |
| `docker push` / `docker build` (prod) | DevOps Lead |
| `kubectl apply` / `docker stack deploy` | DevOps Lead |
| `npx playwright test` | QA Lead |
| ElevenLabs / Telnyx dashboards | AI/ML Lead |
| Cloudflare Wrangler, D1, R2, KV | Forbidden – you use Spring Boot + MySQL |

## Memory (PARA)
Use `para-memory-files` skill. Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`.

## Key Files to Track (App Dev Lead)
- `frontend/src/` — all SvelteKit components, routes, stores
- `backend/src/main/java/` — Spring Boot controllers, services, MyBatis mappers
- `backend/src/main/resources/mapper/*.xml` — MyBatis SQL maps
- `db/migration/*.sql` — Flyway migration scripts (reviewed by DBA)
- `backend/src/test/java/` — JUnit tests
- `frontend/tests/` — Vitest/component tests

## References
- `$AGENT_HOME/HEARTBEAT.md` — execution checklist
- `$AGENT_HOME/SOUL.md` — persona and voice