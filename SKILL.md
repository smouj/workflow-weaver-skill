---
name: Workflow Weaver
description: Crafts and connects automated workflows across tools and services
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

Automation specialist for crafting, connecting, and orchestrating workflows across multiple tools and services. Integrates CI/CD pipelines, triggers actions based on events, and creates seamless toolchains.

## Purpose

Real-world use cases:
- Connect GitHub PR events to automatic testing and deployment pipelines
- Orchestrate multi-step data processing: fetch → transform → store → notify
- Create incident response workflows: alert → triage → create ticket → page on-call
- Sync content across platforms: commit → build → publish → social media announce
- Automate DevOps tasks: scale up on metrics, backup rotation, security scans

## Scope

Specific supported integrations and commands:
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

1. **Discovery** - Analyze existing automation setup
   ```bash
   workflow discover --type all --output yaml
   ```
   Scans for: .github/workflows/*.yml, .gitlab-ci.yml, Jenkinsfile, docker-compose.yml, k8s manifests, cron jobs

2. **Blueprint creation** - Define workflow structure
   ```bash
   workflow blueprint --name "ci-cd-pipeline" --trigger push --concurrency 3
   ```
   Generates template workflow file with specified triggers and constraints

3. **Node connection** - Link workflow steps with real tools
   ```bash
   workflow connect --from "github-push" --to "run-tests" --tool jest --timeout 300
   ```
   Creates dependency edge with timeout and retry policy

4. **Parameter injection** - Pass data between steps
   ```bash
   workflow params --step build --set IMAGE_TAG=$(git rev-parse --short HEAD)
   ```
   Uses shell command substitution to compute dynamic values

5. **Conditional logic** - Add branching
   ```bash
   workflow condition --step test --if-exit 0 --then "deploy" --else "notify-failure"
   ```
   Routes based on exit codes or custom expressions

6. **Secret management** - Inject credentials securely
   ```bash
   workflow secrets --add GITHUB_TOKEN --from-env GITHUB_TOKEN --mask true
   workflow secrets --add SLACK_WEBHOOK --from-file /path/to/webhook.secret
   ```

7. **Dry-run validation** - Test without executing
   ```bash
   workflow validate --dry-run --trace --output graphviz
   ```
   Generates dependency graph and simulates execution

8. **Deployment** - Install workflow
   ```bash
   workflow deploy --target .github/workflows/generated.yml --commit "Add CI/CD"
   ```
   Commits to repository and optionally opens PR

9. **Monitoring** - Track execution
   ```bash
   workflow monitor --watch --tail 100 --alert-on-failure
   ```
   Streams logs and sends alerts on errors

## Golden Rules

1. **Idempotency first**: Every workflow step must be safely repeatable. No side effects that accumulate (e.g., "create user" must check existence first).

2. **Explicit dependencies**: Never rely on implicit ordering. Use `workflow connect` for every dependency.

3. **Secrets never in logs**: Use `--mask` flag on all secret parameters. Verify with `workflow audit --check-secrets`.

4. **Timeouts on everything**: Network calls, subprocesses, file operations. Default 300s, adjust per step.

5. **Error handling mandatory**: Every step must have explicit failure path. No "let it crash" - use `--on-error continue|retry:N|abort`.

6. **Resource limits**: Set CPU/memory limits for heavy steps. `workflow resource --step build --cpu 2 --memory 4G`.

7. **Version pinning**: All external tools (docker images, npm packages) must use immutable tags (no `latest`).

8. **Observability**: Every step logs to structured JSON. Include `workflow_id`, `step_name`, `timestamp`, `duration_ms`, `exit_code`.

9. **Rollback ready**: For any state-changing step, provide corresponding undo step and test it quarterly.

10. **Test in staging first**: Production workflows must exist in staging environment for 48h before promotion.

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

## Troubleshooting

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