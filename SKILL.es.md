name: Workflow Weaver
description: Crea y conecta flujos de trabajo automatizados entre herramientas y servicios
version: 1.2.0
author: OpenClaw Team
tags: [automation, workflows, integration, productivity, orchestration]
dependencies:
  - python3.9+
  - nodejs:16+
  - jq
  - curl
  - git
  - docker (optional)
  - kubectl (optional)
required_env_vars:
  - optional: WORKFLOW_WEAVER_CONFIG
  - optional: WEBHOOK_SECRET
  - optional: SLACK_WEBHOOK_URL
  - optional: GITHUB_TOKEN
compatible_agents: [flickclaw-bot, flick-devops, code-reviewer, test-engineer]
---

# Workflow Weaver

Especialista en automatización para crear, conectar y orquestar flujos de trabajo entre múltiples herramientas y servicios. Integra pipelines CI/CD, activa acciones basadas en eventos y crea cadenas de herramientas sin fisuras.

## Purpose

Casos de uso reales:
- Conectar eventos de GitHub PR a pipelines de testing y despliegue automático
- Orquestar procesamiento de datos multi-paso: fetch → transform → store → notify
- Crear flujos de trabajo de respuesta a incidentes: alert → triage → create ticket → page on-call
- Sincronizar contenido entre plataformas: commit → build → publish → social media announce
- Automatizar tareas DevOps: scale up on metrics, backup rotation, security scans

## Alcance

Integraciones y comandos específicos soportados:
- GitHub Actions: `workflow trigger --repo owner/repo --event push --branch main`
- GitLab CI: `workflow trigger --project 12345 --event merge_request`
- Jenkins: `workflow jenkins --job build-app --params BUILD_NUMBER=42`
- CircleCI: `workflow circleci --pipeline deploy --branch staging`
- Docker Compose: `workflow docker-compose --file docker-compose.prod.yml --action up`
- Kubernetes: `workflow k8s --namespace prod --deploy deployment/app --image tag:v2.1`
- AWS: `workflow aws --service lambda --invoke function-arn --payload file.json`
- Slack notifications: `workflow notify --slack "#alerts" --message "Build failed" --severity high`
- Email via SendGrid/Mailgun: `workflow email --to team@company.com --template incident-alert`
- Webhooks: `workflow webhook --url https://api.example.com/hook --data '{"status":"success"}'`
- Cron scheduling: `workflow schedule --cron "0 2 * * *" --name daily-backup`
- Conditional branching: `workflow condition --if "[ $STATUS -eq 0 ]" --then "notify-success" --else "rollback"`
- Environment variable injection: `workflow env --set NODE_ENV=production --secret DATABASE_URL`
- File operations: `workflow file --copy src/ dest/ --exclude "*.tmp" --chmod 755`

## Detailed Work Process

1. **Discovery** - Analizar configuración de automatización existente
   ```bash
   workflow discover --type all --output yaml
   ```
   Escanea: .github/workflows/*.yml, .gitlab-ci.yml, Jenkinsfile, docker-compose.yml, k8s manifests, cron jobs

2. **Blueprint creation** - Definir estructura del workflow
   ```bash
   workflow blueprint --name "ci-cd-pipeline" --trigger push --concurrency 3
   ```
   Genera archivo de workflow de plantilla con triggers y restricciones especificados

3. **Node connection** - Conectar pasos del workflow con herramientas reales
   ```bash
   workflow connect --from "github-push" --to "run-tests" --tool jest --timeout 300
   ```
   Crea edge de dependencia con política de timeout y retry

4. **Parameter injection** - Pasar datos entre pasos
   ```bash
   workflow params --step build --set IMAGE_TAG=$(git rev-parse --short HEAD)
   ```
   Usa sustitución de comando shell para calcular valores dinámicos

5. **Conditional logic** - Añadir branching
   ```bash
   workflow condition --step test --if-exit 0 --then "deploy" --else "notify-failure"
   ```
   Enruta basado en códigos de salida o expresiones personalizadas

6. **Secret management** - Inyectar credenciales de forma segura
   ```bash
   workflow secrets --add GITHUB_TOKEN --from-env GITHUB_TOKEN --mask true
   workflow secrets --add SLACK_WEBHOOK --from-file /path/to/webhook.secret
   ```

7. **Dry-run validation** - Probar sin ejecutar
   ```bash
   workflow validate --dry-run --trace --output graphviz
   ```
   Genera grafo de dependencias y simula ejecución

8. **Deployment** - Instalar workflow
   ```bash
   workflow deploy --target .github/workflows/generated.yml --commit "Add CI/CD"
   ```
   Commits al repositorio y opcionalmente abre PR

9. **Monitoring** - Rastrear ejecución
   ```bash
   workflow monitor --watch --tail 100 --alert-on-failure
   ```
   Streams logs y envía alertas en errores

## Reglas de Oro

1. **Idempotency first**: Cada paso del workflow debe ser seguro de repetir. Sin efectos secundarios que se acumulen (ej. "create user" debe verificar existencia primero).

2. **Explicit dependencies**: Nunca confiar en ordering implícito. Usar `workflow connect` para cada dependencia.

3. **Secrets never in logs**: Usar flag `--mask` en todos los parámetros secretos. Verificar con `workflow audit --check-secrets`.

4. **Timeouts on everything**: Llamadas de red, subprocesos, operaciones de archivo. Default 300s, ajustar por paso.

5. **Error handling mandatory**: Cada paso debe tener ruta de fallo explícita. No "let it crash" - usar `--on-error continue|retry:N|abort`.

6. **Resource limits**: Setear límites CPU/memory para pasos pesados. `workflow resource --step build --cpu 2 --memory 4G`.

7. **Version pinning**: Todas las herramientas externas (imágenes docker, paquetes npm) deben usar tags inmutables (no `latest`).

8. **Observability**: Cada paso loguea a JSON estructurado. Incluir `workflow_id`, `step_name`, `timestamp`, `duration_ms`, `exit_code`.

9. **Rollback ready**: Para cualquier paso que cambie estado, proveer paso undo correspondiente y probarlo trimestralmente.

10. **Test in staging first**: Workflows de producción deben existir en ambiente staging por 48h antes de promoción.

## Examples

### Example 1: GitHub PR to automated deployment

User input:
```
Create a workflow that when a PR is opened to main branch, it should:
1. Run npm test with coverage
2. Run npm build
3. If tests pass, deploy to staging via Docker Compose
4. Post results to Slack
```

Generated commands:
```bash
# Initialize workflow
workflow blueprint --name "pr-to-staging" --trigger "pull_request" --branch main

# Define test step
workflow add-step --name test --command "npm ci && npm test -- --coverage --ci"
workflow resource --step test --timeout 600 --memory 2G

# Define build step with dependency
workflow add-step --name build --command "npm run build"
workflow connect --from test --to build --if-exit 0

# Define deploy step
workflow add-step --name deploy --command "docker-compose -f docker-compose.staging.yml up -d --build app"
workflow connect --from build --to deploy --if-exit 0

# Add Slack notification on success
workflow add-step --name notify-success --command "workflow notify --slack '#devops' --message 'PR #${{ github.event.number }} deployed to staging'"
workflow connect --from deploy --to notify-success --if-exit 0

# Add notification on failure
workflow add-step --name notify-failure --command "workflow notify --slack '#alerts' --message 'PR failed: ${{ steps.test.outcome }}' --severity high"
workflow connect --from test --to notify-failure --if-exit 1
workflow connect --from build --to notify-failure --if-exit 1

# Validate and deploy
workflow validate --strict
workflow deploy --target .github/workflows/pr-staging.yml
```

### Example 2: Scheduled data pipeline

User input:
```
Every day at 2 AM, fetch data from API, transform with jq, upload to S3, and send email report.
```

Generated commands:
```bash
workflow blueprint --name daily-data-pipeline" --schedule "0 2 * * *"

# Fetch step with retry
workflow add-step --name fetch --command "curl -s https://api.example.com/data | tee /tmp/raw.json"
workflow retry --step fetch --attempts 3 --delay 30

# Transform step
workflow add-step --name transform --command "jq '.items[] | select(.active==true)' /tmp/raw.json > /tmp/processed.json"
workflow connect --from fetch --to transform --if-exit 0

# Upload to S3
workflow add-step --name upload --command "aws s3 cp /tmp/processed.json s3://bucket/data/$(date +%Y%m%d).json --storage-class STANDARD_IA"
workflow connect --from transform --to upload --if-exit 0

# Email report
workflow add-step --name report --command "workflow email --to analytics@company.com --template daily-report --attachments /tmp/processed.json"
workflow connect --from upload --to report --if-exit 0

# Secrets setup
workflow secrets --add AWS_ACCESS_KEY_ID --from-env AWS_ACCESS_KEY_ID
workflow secrets --add AWS_SECRET_ACCESS_KEY --from-env AWS_SECRET_ACCESS_KEY

# Validate cron
workflow validate --check-schedule-overlap

# Install
workflow install --cron-system user
```

### Example 3: Kubernetes rolling update with health checks

User input:
```
Deploy new image to production with zero downtime, rollback on failure.
```

Generated commands:
```bash
workflow blueprint --name "k8s-rolling-update" --manual-approval

# Set image
workflow add-step --name set-image --command "kubectl set image deployment/app app=myapp:$IMAGE_TAG -n prod"
workflow param --step set-image --name IMAGE_TAG --required --prompt "Enter image tag"

# Wait for rollout
workflow add-step --name wait-rollout --command "kubectl rollout status deployment/app -n prod --timeout=600s"
workflow connect --from set-image --to wait-rollout

# Health check
workflow add-step --name health-check --command "curl -f https://app.example.com/health || exit 1"
workflow resource --step health-check --timeout 30 --retries 10 --delay 5
workflow connect --from wait-rollout --to health-check

# Rollback on failure
workflow add-step --name rollback --command "kubectl rollout undo deployment/app -n prod"
workflow connect --from health-check --to rollback --if-exit 1 --on-failure continue

# Notify
workflow add-step --name notify --command "workflow slack --channel '#prod' --message 'Deployment $IMAGE_TAG: ${{ steps.health-check.result }}'"
workflow connect --from health-check --to notify --if-exit 0
workflow connect --from rollback --to notify

# Secrets
workflow secrets --add KUBECONFIG --from-file ~/.kube/config --mask true

workflow validate --dry-run
workflow deploy --target workflows/k8s-deploy.yml
```

## Rollback Commands

- **Workflow deletion**: `workflow undeploy --name <workflow-name> --backup`
- **Step-level reversion**: `workflow rollback --step <step-name> --to-commit <commit-hash>`
- **File restore**: `workflow file restore --backup /var/backups/workflow-pre-deploy-$(date +%s).tar.gz`
- **Configuration rollback**: `workflow config revert --revision HEAD~1 --target .github/workflows/*.yml`
- **Secret cleanup**: `workflow secrets remove --all --added-after <timestamp>`
- **Resource scaling down**: `workflow resource scale-down --all --grace-period 300`
- **Container cleanup**: `workflow docker prune --orphans --volumes --force`
- **Database migrations**: `workflow db migrate down --steps ALL --backup-path /backups/WorkflowWeaver_$(date +%Y%m%d_%H%M%S).sql`
- **Rollback verification**: `workflow verify-rollback --simulate --dry-run --report format=json`

## Verification Steps

After any workflow change:
```bash
# 1. Syntax validation
workflow validate --strict --schema-version latest

# 2. Dry-run simulation
workflow simulate --trace-all --output timeline.html

# 3. Secret exposure check
workflow audit --check-secrets --exclude-masks

# 4. Dependency resolution
workflow graph --format dot | dot -Tpng -o deps.png

# 5. Resource estimation
workflow estimate --cost --duration

# 6. Security scan
workflow scan --vulnerabilities --config-misconfigurations

# 7. Integration test against staging
workflow test-run --environment staging --exit-on-failure

# 8. Performance baseline
workflow benchmark --baseline baseline.json --compare
```

## Solución de Problemas

**Issue**: Workflow step times out
```bash
workflow debug --step <step-name> --trace-timeout
workflow resource increase --step <step-name> --timeout +300
```

**Issue**: Secrets not masking in logs
```bash
workflow secrets audit --unmasked
workflow rewrite-logs --mask-secrets --in-place
```

**Issue**: Deadlock in workflow (circular dependency)
```bash
workflow graph --detect-cycles --format json
workflow break-cycle --edge from:stepA to:stepB
```

**Issue**: Workflow not triggering on schedule
```bash
workflow schedule verify --cron-expression "0 2 * * *" --timezone UTC
workflow logs --system cron --tail 50
```

**Issue**: High resource usage
```bash
workflow top --sort memory
workflow resource limit --global --cpu 4 --memory 8G
```

**Issue**: Failed deploys in production
```bash
workflow history --status failed --last 10
workflow rollback --to-previous-successful --verify-health
```

## Dependencies & Requirements

System packages:
- `jq` (1.6+) - JSON processing
- `curl` (7.68+) - HTTP requests
- `git` (2.30+) - repository operations
- `docker` (20.10+) optional - container workflows
- `kubectl` (1.25+) optional - Kubernetes workflows
- `awscli` (2.0+) optional - AWS workflows
- `nodejs` (16+) - JavaScript/TypeScript workflow nodes
- `python3.9+` with packages: `pyyaml`, `requests`, `click`, `jinja2`

Environment setup:
```bash
# Install Workflow Weaver core
pip install workflow-weaver

# Configure global settings
workflow config set --workspace ~/.workflow-weaver
workflow config set --log-level INFO
workflow config set --default-timeout 300

# Initialize project
workflow init --project-name "my-automation"
```

Configuration file (`~/.workflow-weaver/config.yaml`):
```yaml
workspace: ~/.workflow-weaver
default_timeout: 300
max_concurrent_workflows: 10
alert_channels:
  slack: ${SLACK_WEBHOOK_URL}
  email: alerts@company.com
secrets_backend: env|vault|aws-secretsmanager
docker_registry: registry.example.com
kubernetes_context: production
notification_template: |
  Workflow {{ workflow_name }} {{ status }}
  Step: {{ step_name }}
  Duration: {{ duration_ms }}ms
```

## Security Considerations

- Never commit secrets to repository. Use `workflow secrets` management only.
- Rotate all credentials quarterly via `workflow secrets rotate --all`.
- Restrict workflow execution to designated runner environments via IAM/SSH keys.
- Enable audit logging: `workflow audit enable --log-file /var/log/workflow-weaver/audit.log`.
- Use read-only tokens for pull-based triggers.
- Validate all webhook payload signatures: `workflow webhook verify-signature --secret ${WEBHOOK_SECRET}`.
```