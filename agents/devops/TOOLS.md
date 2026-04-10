# DevOps Lead Tools

## Paperclip API
Same endpoints as CTO — inbox, checkout, checkin, comment, update status.

## Skills
| Skill | When to use |
|-------|--------------|
| `paperclip` | Coordination, ticket management |
| `cicd-pipelines` | Jenkins/GitHub Actions workflows, pipeline templates (build, test, deploy, rollback) |
| `docker-automator` | Docker CLI, image builds, registry operations, container lifecycle |
| `git-workflow` | CI-friendly branching, PR conventions, tagging strategies |

## CLI Tools (Primary — you use these directly)
| Tool | Purpose |
|------|---------|
| `docker build -t myapp:tag .` | Build container image |
| `docker push registry/myapp:tag` | Push image to registry (Docker Hub / ECR / private) |
| `docker pull registry/myapp:tag` | Pull image for deployment |
| `docker run --rm -d -p 8080:8080 myapp:tag` | Test container locally |
| `docker container ls / stop / rm` | Manage running containers |
| `docker system prune -f` | Clean up unused images/containers |
| `docker-compose -f docker-compose.prod.yml up -d` | Deploy stack with Compose (or Swarm) |
| `docker stack deploy -c docker-compose.yml myapp` | Docker Swarm deployment |
| `kubectl apply -f k8s/` | Kubernetes deployment (optional) |
| `jenkins build my-pipeline -s` | Trigger Jenkins pipeline (CLI) |
| `gh workflow run deploy.yml --ref main` | Trigger GitHub Actions workflow (alternative) |
| `mysql -h dbhost -u user -p dbname < migration.sql` | Run MySQL migration manually (emergency only) |
| `flyway migrate -url=jdbc:mysql://... -user=... -password=...` | Apply versioned migrations (recommended) |
| `flyway info` | Check migration status |
| `liquibase update --changelog-file=changelog.xml` | Alternative migration tool |
| `mysqldump --single-transaction dbname > backup.sql` | Create logical backup |
| `mysqlbinlog binlog.000001 --start-datetime=...` | Point-in-time recovery inspection |
| `trivy image myapp:tag` | Container vulnerability scan |
| `gitleaks detect --source .` | Secret leak detection in repo |
| `git tag -a v1.2.3 -m "release" && git push --tags` | Tag releases for rollback |
| `git diff --name-only HEAD~1` | Detect changed files for selective CI |

## Container Registry & Orchestration Commands
| Command | Purpose |
|---------|---------|
| `aws ecr get-login-password --region ... \| docker login` | Login to ECR |
| `docker tag myapp:tag registry/myapp:tag && docker push` | Tag and push |
| `docker manifest inspect registry/myapp:tag` | Check multi-arch image |
| `docker service update --image registry/myapp:newtag myservice` | Swarm rolling update |
| `kubectl set image deployment/myapp myapp=registry/myapp:newtag` | K8s rolling update |
| `kubectl rollout undo deployment/myapp` | K8s rollback |

## Jenkins Pipeline Snippets (for reference)
```groovy
stage('Build') {
    steps {
        sh 'docker build -t myapp:${BUILD_NUMBER} .'
        sh 'docker tag myapp:${BUILD_NUMBER} registry/myapp:latest'
    }
}
stage('Test') {
    steps {
        sh 'docker run --rm myapp:${BUILD_NUMBER} npm test'
        sh 'trivy image --exit-code 1 --severity HIGH registry/myapp:${BUILD_NUMBER}'
    }
}
stage('Migrate Database') {
    steps {
        sh 'flyway -url=jdbc:mysql://prod-db:3306/mydb -user=$DB_USER -password=$DB_PASS migrate'
    }
}
stage('Deploy') {
    steps {
        sh 'docker push registry/myapp:${BUILD_NUMBER}'
        sh 'ssh deploy@prod-server "docker pull registry/myapp:${BUILD_NUMBER} && docker-compose up -d"'
    }
}
```

## MCP Servers
| Server | Purpose |
|--------|---------|
| Jenkins (REST API) | Get pipeline status, trigger builds, fetch logs |
| GitHub | Workflow status, PR management, issue creation |
| Slack | Deploy notifications, incident alerts |
| Docker Registry API | List images, tags, delete old images |

## Tools You Do NOT Use
| Tool | Operated by |
|------|--------------|
| Application code (src/) | App Dev Lead |
| Database schema design (DDL outside migrations) | DBA Lead |
| `npx playwright test` | QA Lead |
| Cloudflare Wrangler / Workers / D1 / R2 / KV | Forbidden – you use Docker + MySQL + Jenkins |
| Manual `mysql` to alter schema in production | DBA Lead (via migration scripts only) |

## Memory (PARA)
Use `para-memory-files` skill. Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`.

## Container & Database Stack
- **Container runtime**: Docker (Compose for dev, Swarm/K8s for prod)
- **Database**: MySQL 8.0+
- **Migration tool**: Flyway (primary) or Liquibase
- **CI/CD**: Jenkins (primary), GitHub Actions (optional for PR checks)
- **Registry**: Docker Hub / AWS ECR / private registry
- **Secrets**: Jenkins Credentials Binding / HashiCorp Vault

## References
- `$AGENT_HOME/HEARTBEAT.md` — execution checklist
- `$AGENT_HOME/SOUL.md` — persona and voice