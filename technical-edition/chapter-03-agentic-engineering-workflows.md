# Chapter 3: Agentic Engineering Workflows

**Learning Objectives**:
- Master effective prompting strategies for infrastructure development
- Understand the human-AI collaboration model for systems engineering
- Learn validation and testing techniques for AI-generated configurations
- Develop troubleshooting skills for AI-assisted workflows
- Recognize when to trust AI and when to verify independently
- Build muscle memory for iterative refinement with AI assistants

---

## The Agentic Engineering Model

Agentic engineering is not "AI writes code, human accepts it." It's a structured collaboration where:

1. **Human provides systems thinking**: Architecture, requirements, constraints, priorities
2. **AI provides implementation**: Configuration files, scripts, syntax, best practices
3. **Human validates**: Tests, reviews, understands, approves
4. **AI adapts**: Refines based on feedback, corrects errors, improves
5. **Iterate until correct**: Repeat the cycle until the system works as intended

**This is not delegation—it's augmentation.**

You remain the engineer. The AI is your highly capable assistant that knows syntax, documentation, and patterns but lacks your judgment about what should be built and why.

---

## Fundamental Principles

### Principle 1: Context is King

AI assistants perform dramatically better with proper context.

**Poor context** (vague, no environment details):
```
"Set up monitoring"
```

**Good context** (specific, environment-aware):
```
"I need Prometheus monitoring for my infrastructure. Environment:
- Docker Compose on Ubuntu 22.04
- 15 application containers
- PostgreSQL database container
- Traefik reverse proxy
- 4GB RAM, 40GB disk

I want to monitor:
- System metrics (CPU, memory, disk, network)
- Docker container metrics
- PostgreSQL metrics
- Traefik access logs

Requirements:
- 30-day retention
- Alerts when CPU > 80% or memory > 90%
- Grafana dashboards for visualization
- Runs on the same server (no external dependencies)

Create a docker-compose.yml with all necessary components."
```

**Why the second is better**:
- Specifies exact environment (Docker Compose, Ubuntu 22.04)
- Lists current infrastructure (15 containers, PostgreSQL, Traefik)
- Defines monitoring scope (what to monitor)
- States requirements (retention, alerts, dashboards)
- Clarifies constraints (same server, no external services)

**Result**: AI can generate a complete, working solution on the first attempt.

### Principle 2: Iterative Refinement

First generation is rarely perfect. Expect 2-4 iterations.

**Iteration pattern**:

```
Iteration 1: Generate initial solution
→ Review, test, identify gaps

Iteration 2: Refine based on issues found
→ Review, test, verify improvements

Iteration 3: Add missing features or edge cases
→ Review, test, validate completeness

Iteration 4: Optimize and finalize
→ Final review, production-ready
```

**Example workflow**:

```markdown
Human (Iteration 1):
"Create a docker-compose.yml for Prometheus, Grafana, and node_exporter."

AI Response:
[Generates basic docker-compose.yml]

Human (reviews, finds gaps):
"This looks good, but:
1. Prometheus has no persistent storage
2. Grafana has no datasource pre-configured
3. node_exporter isn't collecting Docker metrics

Can you:
1. Add volumes for Prometheus and Grafana data persistence
2. Add a datasource provisioning file for Grafana to auto-configure Prometheus
3. Add cAdvisor for Docker container metrics"

AI Response:
[Generates improved version with all requested changes]

Human (tests, validates):
"Perfect! Now add alerting rules for:
- CPU > 80% for 5 minutes
- Memory > 90% for 5 minutes
- Disk > 85%

And configure Alertmanager to send alerts to Slack webhook: https://hooks.slack.com/..."

AI Response:
[Adds alert rules and Alertmanager configuration]

Human (final validation):
"Excellent. One last thing: add comments explaining each service's purpose and
important configuration options for future reference."

AI Response:
[Adds comprehensive comments]
```

**Result**: Production-ready configuration in 4 iterations, ~20 minutes.

### Principle 3: Validation Before Trust

Never deploy AI-generated infrastructure without validation.

**Validation checklist**:

1. **Syntax validation**: Does it parse correctly?
   ```bash
   docker-compose config  # Validates docker-compose.yml syntax
   terraform validate     # Validates Terraform syntax
   ansible-playbook --syntax-check playbook.yml
   ```

2. **Security review**: Are there exposed secrets or insecure defaults?
   ```bash
   # Check for hardcoded secrets
   grep -r "password\|secret\|key" .

   # Check for exposed ports
   grep -r "0.0.0.0:" docker-compose.yml
   ```

3. **Resource limits**: Are resources bounded?
   ```yaml
   # Good: Has limits
   resources:
     limits:
       memory: 512M
       cpus: '0.5'

   # Bad: Unbounded
   resources: {}
   ```

4. **Test in isolation**: Deploy in a test environment first
   ```bash
   # Test environment
   docker-compose -f docker-compose.yml -f docker-compose.test.yml up

   # Validate it works
   curl http://localhost:9090/api/v1/targets
   ```

5. **Understand what was generated**: Read through the configuration
   - Can you explain what each component does?
   - Do you understand the relationships?
   - Could you troubleshoot if something breaks?

**If you can't explain it, don't deploy it.**

### Principle 4: Semantic Understanding Over Blind Copying

The goal is not just "working infrastructure" but **infrastructure you understand**.

**Bad workflow**:
```
1. Ask AI for config
2. Copy-paste
3. Deploy
4. Hope it works
5. Panic when it breaks (because you don't understand it)
```

**Good workflow**:
```
1. Ask AI for config
2. Review and understand each section
3. Ask AI to explain anything unclear
4. Test in isolation
5. Document your understanding
6. Deploy with confidence
7. Troubleshoot effectively (because you understand how it works)
```

**Example of seeking understanding**:

```markdown
Human:
"I see this in the Prometheus config you generated:

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

Can you explain what scrape_configs, job_name, and static_configs mean?
Why is Prometheus scraping itself?"

AI:
"Explanation of Prometheus configuration architecture, scrape intervals,
service discovery, and why self-monitoring is valuable..."
```

**Outcome**: You now understand the configuration and can troubleshoot it independently.

---

## Effective Prompting Strategies

The quality of AI output is directly proportional to the quality of your prompts.

### The CEDAR Framework for Infrastructure Prompts

**C**ontext: What environment are you working in?
**E**xpectation: What do you want the AI to create?
**D**etails: What specific requirements must be met?
**A**ssumptions: What constraints or limitations exist?
**R**esult format: How should the output be structured?

**Example using CEDAR**:

```markdown
**Context**:
I'm running a VPS with Ubuntu 22.04, Docker, and Docker Compose. Current services:
- PostgreSQL 15 (port 5432)
- Redis 7 (port 6379)
- Nginx reverse proxy (ports 80, 443)
- 3 application containers

**Expectation**:
Create a complete backup solution for this infrastructure.

**Details**:
- Daily automated backups at 2 AM UTC
- PostgreSQL database dumps (all databases)
- Redis RDB snapshots
- Application container volumes
- Retain last 7 daily backups, last 4 weekly backups, last 3 monthly backups
- Upload to S3-compatible object storage (MinIO)
- Send notification to Slack on backup completion or failure

**Assumptions**:
- MinIO credentials available as environment variables
- Slack webhook URL available as environment variable
- Backups should be encrypted before upload
- Total storage budget: 100GB
- Backup window: 2 AM - 4 AM (low traffic period)

**Result format**:
- Docker Compose service definition for backup container
- Backup script (bash or Python)
- Cron configuration
- Example environment variables file
- Documentation on how to restore from backup
```

**This prompt will generate a comprehensive, working backup solution.**

### Prompt Patterns for Common Infrastructure Tasks

#### Pattern 1: Initial Setup

```markdown
"I need to set up [SYSTEM] for [ENVIRONMENT].

Environment details:
- OS: [Operating System and version]
- Current infrastructure: [List existing components]
- Resources: [CPU, RAM, Disk]
- Access: [How services are accessed]

Requirements:
- [Requirement 1]
- [Requirement 2]
- [Requirement 3]

Constraints:
- [Constraint 1]
- [Constraint 2]

Provide:
1. Complete configuration files
2. Step-by-step deployment instructions
3. Health check commands
4. Troubleshooting tips"
```

#### Pattern 2: Debugging Assistance

```markdown
"I'm experiencing [PROBLEM] in my [SYSTEM].

Environment:
[Paste relevant configuration]

Error message:
```
[Paste complete error message with stack trace]
```

What I've tried:
1. [Action 1] - Result: [What happened]
2. [Action 2] - Result: [What happened]

Current behavior:
[Describe what's happening]

Expected behavior:
[Describe what should happen]

Can you:
1. Explain what's causing this error
2. Provide a solution
3. Explain why your solution works"
```

#### Pattern 3: Optimization Request

```markdown
"I need to optimize [SYSTEM] for [GOAL].

Current setup:
[Paste current configuration]

Performance metrics:
- Current: [Metric] = [Value]
- Target: [Metric] = [Value]
- Bottleneck: [Where the problem is]

Constraints:
- [Cannot change X]
- [Must maintain Y]
- [Budget/resource limits]

Optimize for:
1. [Priority 1]
2. [Priority 2]

Provide:
1. Analysis of current bottlenecks
2. Recommended changes with justification
3. Expected performance improvements
4. Risks or trade-offs"
```

#### Pattern 4: Migration Assistance

```markdown
"I need to migrate from [CURRENT_SYSTEM] to [NEW_SYSTEM].

Current setup:
[Details of existing system]

Target setup:
[Details of desired system]

Migration requirements:
- Zero downtime: [Yes/No]
- Rollback plan: [Required/Optional]
- Data migration: [Size and type]
- Testing phase: [Duration]

Risks I'm concerned about:
1. [Risk 1]
2. [Risk 2]

Provide:
1. Step-by-step migration plan
2. Rollback procedure
3. Testing checklist
4. Common pitfalls to avoid"
```

### Incremental Prompting for Complex Tasks

Break complex infrastructure into smaller, manageable pieces.

**Bad approach** (trying to do everything at once):
```
"Set up complete production infrastructure with monitoring, security,
CI/CD, load balancing, databases, caching, logging, and backups."
```

**Good approach** (incremental):

```markdown
Phase 1: Foundation
"Set up basic Docker Compose infrastructure with:
- PostgreSQL
- Redis
- Nginx reverse proxy
Include health checks and basic resource limits."

[Review, test, validate Phase 1]

Phase 2: Monitoring
"Add Prometheus and Grafana to the existing docker-compose.yml.
Monitor all existing services (PostgreSQL, Redis, Nginx).
[Paste current docker-compose.yml]"

[Review, test, validate Phase 2]

Phase 3: Security
"Add CrowdSec to the existing setup for intrusion detection.
Integrate with Traefik (replacing Nginx) for better security.
[Paste current docker-compose.yml]"

[Continue incrementally...]
```

**Benefits**:
- Each phase is testable independently
- Easier to debug when something breaks
- Can roll back individual phases
- Builds understanding progressively

---

## Testing and Validation Workflows

How do you verify that AI-generated infrastructure is correct?

### Three-Layer Validation

**Layer 1: Syntax Validation** (Does it parse?)

```bash
# Docker Compose
docker-compose config

# Terraform
terraform fmt -check
terraform validate

# Ansible
ansible-playbook --syntax-check playbook.yml

# Kubernetes
kubectl apply --dry-run=client -f manifest.yml
```

**Layer 2: Static Analysis** (Is it secure and well-formed?)

```bash
# Docker security scanning
docker scan image:tag

# Infrastructure as Code security
checkov -f docker-compose.yml
tfsec .

# Kubernetes security
kubesec scan manifest.yml

# General secret detection
gitleaks detect --source=.
```

**Layer 3: Functional Testing** (Does it work?)

```bash
# Deploy to test environment
docker-compose -f docker-compose.yml up -d

# Health checks
curl http://localhost:9090/-/healthy  # Prometheus
curl http://localhost:3000/api/health  # Grafana

# Functional tests
pytest tests/integration/
./test-backup.sh

# Load testing (if applicable)
ab -n 1000 -c 10 http://localhost:8080/

# Monitoring validation
curl http://localhost:9090/api/v1/query?query=up
# Expect: All targets showing "up"
```

### Test Environment Strategy

Never test directly in production.

**Minimum test environment**:
```yaml
# docker-compose.test.yml
version: '3.8'

services:
  # Override production settings for testing
  postgres:
    environment:
      - POSTGRES_DB=testdb
    volumes:
      - ./test-data:/docker-entrypoint-initdb.d

  app:
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://postgres:5432/testdb
```

**Test workflow**:
```bash
# Start test environment
docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d

# Run tests
npm test  # or pytest, or go test, etc.

# Verify health
./health-check.sh

# Tear down test environment
docker-compose -f docker-compose.yml -f docker-compose.test.yml down -v
```

### Validation Checklist

Before deploying AI-generated infrastructure to production:

- [ ] **Syntax validation passed** (config, validate, syntax-check)
- [ ] **Security scan passed** (no critical vulnerabilities)
- [ ] **No hardcoded secrets** (all secrets use env vars or secret management)
- [ ] **Resource limits defined** (CPU, memory limits on all services)
- [ ] **Health checks configured** (liveness and readiness probes)
- [ ] **Logging configured** (structured logging to stdout/stderr)
- [ ] **Monitoring configured** (metrics exposed, alerts defined)
- [ ] **Backup tested** (can restore from backup successfully)
- [ ] **Documentation updated** (README, runbooks, architecture diagrams)
- [ ] **Rollback plan documented** (how to revert if deployment fails)
- [ ] **Test environment validated** (works in staging/test)
- [ ] **Peer reviewed** (another engineer reviewed the changes)

**If any checkbox is unchecked, don't deploy.**

---

## Troubleshooting AI-Assisted Workflows

What to do when AI-generated infrastructure doesn't work as expected?

### Common Failure Modes

#### Failure Mode 1: Syntax Errors

**Symptom**: Configuration doesn't parse or validate

**Diagnosis**:
```bash
docker-compose config
# Error: services.prometheus.image: invalid reference format
```

**Resolution workflow**:
```markdown
Human to AI:
"I'm getting this error when running docker-compose config:

```
Error: services.prometheus.image: invalid reference format
```

Here's my docker-compose.yml:
[Paste the generated configuration]

Can you identify and fix the syntax error?"

AI Response:
[Explains the error and provides corrected configuration]
```

#### Failure Mode 2: Runtime Failures

**Symptom**: Services start but crash immediately

**Diagnosis**:
```bash
docker-compose up
# Services start, then exit with error code 1

docker-compose logs prometheus
# Error: opening storage failed: create storage/wal/00000000: permission denied
```

**Resolution workflow**:
```markdown
Human to AI:
"My Prometheus container is crashing with this error:

```
Error: opening storage failed: create storage/wal/00000000: permission denied
```

Here's the relevant docker-compose.yml section:
[Paste Prometheus service definition]

This seems like a permissions issue. Can you:
1. Explain why this is happening
2. Provide the fix
3. Explain how to prevent this in the future"

AI Response:
[Explains that Prometheus runs as non-root user 65534, volume needs proper ownership]
[Provides fix: add user: "65534:65534" or chown volume]
[Explains best practices for container user management]
```

#### Failure Mode 3: Integration Issues

**Symptom**: Individual services work, but can't communicate

**Diagnosis**:
```bash
# Prometheus UI shows all targets as "DOWN"
curl http://localhost:9090/api/v1/targets
# Shows: state: "down", last_error: "connection refused"
```

**Resolution workflow**:
```markdown
Human to AI:
"Prometheus is running but can't scrape targets. All targets show 'connection refused'.

Environment:
- Prometheus in one Docker Compose stack
- Targets in another Docker Compose stack
- Both on same host

Prometheus targets:
```yaml
scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:8080']
```

Error: Connection refused when trying to scrape app:8080

What's wrong and how do I fix it?"

AI Response:
[Explains that services in different Compose stacks are on different networks]
[Provides solution: shared external network or use host IP]
[Shows corrected configuration]
```

#### Failure Mode 4: Performance Issues

**Symptom**: System works but is slow or unstable

**Diagnosis**:
```bash
# High CPU usage
docker stats
# prometheus: 180% CPU usage

# Check Prometheus logs
docker-compose logs prometheus | grep -i "warn\|error"
# Warning: Query exceeded timeout
```

**Resolution workflow**:
```markdown
Human to AI:
"Prometheus is using 180% CPU and queries are timing out.

Current configuration:
[Paste prometheus.yml]

Metrics being collected:
- 50 targets
- ~10,000 time series per target
- 15-second scrape interval
- 30-day retention

Hardware:
- 2 CPU cores
- 4GB RAM

The system worked fine with 20 targets, but scaling to 50 caused this issue.

Can you:
1. Identify the bottleneck
2. Suggest configuration optimizations
3. Recommend if I need more resources"

AI Response:
[Analysis of query load, cardinality, and resource requirements]
[Suggests increasing scrape interval, reducing retention, or adding resources]
[Provides optimized configuration]
```

### Debugging Decision Tree

```
Problem occurs
│
├─ Does it parse/validate?
│  ├─ No → Syntax error → Ask AI to fix syntax
│  └─ Yes → Continue
│
├─ Do services start?
│  ├─ No → Startup failure → Check logs → Ask AI about error message
│  └─ Yes → Continue
│
├─ Can services communicate?
│  ├─ No → Network/connectivity → Check networking → Ask AI about integration
│  └─ Yes → Continue
│
├─ Are services performing well?
│  ├─ No → Performance issue → Check metrics → Ask AI for optimization
│  └─ Yes → Working correctly!
│
└─ Is behavior correct?
   ├─ No → Logic error → Review requirements → Ask AI to adjust logic
   └─ Yes → Success!
```

### When AI Can't Solve the Problem

Sometimes AI gets stuck or provides incorrect solutions repeatedly.

**Signs AI is stuck**:
- Suggests the same solution 3+ times
- Solutions become progressively worse
- AI says "this should work" but it doesn't
- AI suggests workarounds instead of root cause fixes

**When this happens**:

1. **Switch AI assistants**: Try a different tool (Claude vs ChatGPT vs Copilot)
2. **Simplify the problem**: Break it into smaller pieces
3. **Search documentation**: AI might not have context for edge cases
4. **Ask the community**: StackOverflow, Reddit, Discord communities
5. **Consult official docs**: Sometimes the source of truth is needed

**Example**:
```markdown
"I've been working with AI on this Prometheus issue for 30 minutes and we're stuck.
The problem is: [detailed problem description]

I'm going to:
1. Check the official Prometheus docs for this specific error
2. Search GitHub issues for similar problems
3. Come back with more context for the AI to work with"
```

---

## AI Assistant Selection and Usage

Different AI assistants excel at different tasks.

### Capability Matrix

| Task | Claude Code | Cursor | GitHub Copilot | OpenAI Codex | ChatGPT |
|------|-------------|--------|----------------|--------------|---------|
| Complete project setup | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| File editing | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| Debugging | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Explanations | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Infrastructure as Code | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Syntax completion | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Multi-file operations | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

### Usage Recommendations

**For initial infrastructure setup**:
- **Primary**: Claude Code (understands full project context)
- **Secondary**: OpenAI Codex (excellent for natural language → infrastructure code)

**For specific file editing**:
- **Primary**: Cursor (best in-editor experience)
- **Secondary**: GitHub Copilot (great autocomplete)

**For debugging and troubleshooting**:
- **Primary**: Claude Code (deep analysis capabilities)
- **Secondary**: ChatGPT (great explanations)

**For learning and explanations**:
- **Primary**: ChatGPT (best at teaching)
- **Secondary**: Claude Code (technical depth)

### Prompt Adaptation by Tool

Different tools have different strengths. Adapt your prompts accordingly.

**Claude Code** (best for comprehensive tasks):
```markdown
"Create a complete monitoring stack with Prometheus, Grafana, and Loki.

[Provide full context with CEDAR framework]

I want a production-ready setup with HA, persistent storage, and complete configuration."
```

**Cursor** (best for targeted edits):
```markdown
"In this docker-compose.yml file, add health checks to all services.

For each service, add appropriate health check commands based on the service type."
```

**GitHub Copilot** (best for inline completion):
```yaml
# Type a comment, let Copilot autocomplete
# Add Prometheus service with node_exporter
prometheus:
  # Copilot will suggest the rest
```

**ChatGPT** (best for explanations):
```markdown
"Explain how Prometheus service discovery works and why I would use it instead of
static_configs. Use examples relevant to a Docker Compose environment."
```

---

## Real-World Example: Complete Workflow

Let's walk through building a complete monitoring stack using agentic engineering.

### Scenario

You need to add comprehensive monitoring to an existing Docker Compose infrastructure running:
- 3 Node.js applications
- PostgreSQL database
- Redis cache
- Nginx reverse proxy

**Goal**: Production-ready monitoring with alerts.

### Iteration 1: Initial Setup

**Prompt to Claude Code**:
```markdown
I need to add monitoring to my existing Docker Compose infrastructure.

Current setup:
- 3 Node.js apps (app1, app2, app3) on ports 3001-3003
- PostgreSQL 15 on port 5432
- Redis 7 on port 6379
- Nginx reverse proxy on ports 80/443

Environment:
- Ubuntu 22.04
- 4 CPU cores, 8GB RAM, 100GB disk
- Docker Compose v2

Requirements:
- Monitor all services (apps, PostgreSQL, Redis, Nginx, and system)
- 30-day retention
- Grafana dashboards
- Alerts for CPU > 80%, memory > 90%, disk > 85%
- Alert to Slack webhook: https://hooks.slack.com/services/xxx

Provide:
1. Updated docker-compose.yml (add monitoring services)
2. Prometheus configuration with scrape configs for all services
3. Alertmanager configuration for Slack notifications
4. Grafana datasource provisioning
```

**AI Response (Claude Code)**:

```yaml
# docker-compose.yml - Monitoring Stack Addition
version: '3.8'

services:
  # Existing services (app1, app2, app3, postgres, redis, nginx not shown)

  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.1.2
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-clock-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - monitoring

  node_exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node_exporter
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
    ports:
      - "9100:9100"
    volumes:
      - /:/host:ro,rslave
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:v0.14.0
    container_name: postgres_exporter
    restart: unless-stopped
    environment:
      - DATA_SOURCE_NAME=postgresql://postgres:password@postgres:5432/postgres?sslmode=disable
    ports:
      - "9187:9187"
    networks:
      - monitoring

  redis_exporter:
    image: oliver006/redis_exporter:v1.54.0
    container_name: redis_exporter
    restart: unless-stopped
    environment:
      - REDIS_ADDR=redis:6379
    ports:
      - "9121:9121"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

**prometheus/prometheus.yml**:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    environment: 'vps'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres_exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis_exporter:9121']

  - job_name: 'applications'
    static_configs:
      - targets: ['app1:3001', 'app2:3002', 'app3:3003']
```

**prometheus/alerts.yml**:
```yaml
groups:
  - name: infrastructure_alerts
    interval: 30s
    rules:
      - alert: HighCpuUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}% for 5 minutes"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"

      - alert: HighDiskUsage
        expr: (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"} * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High disk usage on {{ $labels.instance }}"
          description: "Disk usage is {{ $value }}%"
```

**alertmanager/alertmanager.yml**:
```yaml
global:
  slack_api_url: 'https://hooks.slack.com/services/xxx'

route:
  receiver: 'slack-notifications'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

**grafana/provisioning/datasources/prometheus.yml**:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

**Human reviews Iteration 1**:

✅ **What works**:
- All exporters configured correctly
- Prometheus scraping all services
- Alert rules defined for CPU, memory, disk
- Grafana datasource provisioning included

⚠️ **Issues found**:
1. No resource limits (services can consume unlimited CPU/memory)
2. Grafana default password is hardcoded (insecure)
3. PostgreSQL password exposed in docker-compose.yml (should use env vars)
4. No health checks defined
5. Missing Nginx exporter (Nginx not being monitored)

### Iteration 2: Security and Resource Hardening

**Prompt to Claude Code**:
```markdown
Good start! I need to address these security and reliability issues:

1. Add resource limits to all monitoring services (reasonable limits for 8GB RAM system)
2. Move all sensitive credentials to environment variables
3. Add health checks to all services
4. Add nginx-prometheus-exporter for Nginx monitoring
5. Set Prometheus and Grafana to run as non-root users

Current docker-compose.yml:
[Paste Iteration 1 docker-compose.yml]

Provide:
1. Updated docker-compose.yml with all fixes
2. Example .env file with all required variables
3. Brief explanation of resource limit choices
```

**AI Response (Claude Code)**:

```yaml
# docker-compose.yml - Iteration 2 (Hardened)
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    user: "65534:65534"  # nobody user
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    user: "65534:65534"
    restart: unless-stopped
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'
        reservations:
          memory: 128M
          cpus: '0.1'
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  grafana:
    image: grafana/grafana:10.1.2
    container_name: grafana
    user: "472:472"  # grafana user
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_INSTALL_PLUGINS=grafana-clock-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  node_exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node_exporter
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
    ports:
      - "9100:9100"
    volumes:
      - /:/host:ro,rslave
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.1'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:v0.14.0
    container_name: postgres_exporter
    restart: unless-stopped
    environment:
      - DATA_SOURCE_NAME=${POSTGRES_EXPORTER_DSN}
    ports:
      - "9187:9187"
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.1'

  redis_exporter:
    image: oliver006/redis_exporter:v1.54.0
    container_name: redis_exporter
    restart: unless-stopped
    environment:
      - REDIS_ADDR=${REDIS_ADDR}
    ports:
      - "9121:9121"
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.1'

  nginx_exporter:
    image: nginx/nginx-prometheus-exporter:0.11.0
    container_name: nginx_exporter
    restart: unless-stopped
    command:
      - '-nginx.scrape-uri=http://nginx:80/stub_status'
    ports:
      - "9113:9113"
    networks:
      - monitoring
    deploy:
      resources:
        limits:
          memory: 64M
          cpus: '0.1'

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

**.env file**:
```bash
# Grafana
GRAFANA_ADMIN_PASSWORD=<strong-password-here>

# PostgreSQL Exporter
POSTGRES_EXPORTER_DSN=postgresql://postgres:${DB_PASSWORD}@postgres:5432/postgres?sslmode=disable

# Redis
REDIS_ADDR=redis:6379

# Alertmanager Slack Webhook
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
```

**Resource Limit Rationale**:
- **Prometheus**: 2GB limit (can hold ~30 days of metrics from this scale)
- **Grafana**: 512MB limit (UI and dashboard rendering)
- **Alertmanager**: 256MB limit (lightweight alert routing)
- **Exporters**: 64-128MB limits (minimal overhead)
- **Total**: ~3.5GB allocated, leaving 4.5GB for applications and system

**Human validation of Iteration 2**:

✅ **Improvements**:
- All services now have resource limits
- Credentials moved to environment variables
- Health checks added to all critical services
- Non-root users configured
- Nginx now monitored

✅ **Tests pass**:
```bash
docker-compose config  # Syntax valid
docker-compose up -d   # All services start
curl http://localhost:9090/-/healthy  # Prometheus healthy
curl http://localhost:3000/api/health  # Grafana healthy
```

⚠️ **Minor issues**:
1. Need to add Nginx stub_status location to Nginx config for exporter
2. Would be nice to have pre-built Grafana dashboards
3. Alert routing could be more sophisticated (different channels for different severities)

### Iteration 3: Production Polish

**Prompt to Claude Code**:
```markdown
Almost perfect! Final improvements needed:

1. Add Nginx stub_status configuration
2. Include pre-built Grafana dashboards (Node Exporter, cAdvisor, PostgreSQL, Redis)
3. Improve alert routing: warnings to #alerts-warn, critical to #alerts-critical

Also, can you provide:
- Deployment instructions
- Health check script
- Troubleshooting guide for common issues

Current setup:
[Configuration is good, just need additions]
```

**AI Response (Claude Code)**:

**nginx/conf.d/stub_status.conf**:
```nginx
server {
    listen 80;
    server_name localhost;

    location /stub_status {
        stub_status;
        access_log off;
        allow 172.16.0.0/12;  # Docker networks
        deny all;
    }
}
```

**Updated alertmanager/alertmanager.yml**:
```yaml
global:
  slack_api_url: 'https://hooks.slack.com/services/xxx'

route:
  receiver: 'slack-warnings'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h
  routes:
    - match:
        severity: critical
      receiver: 'slack-critical'
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warn'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
```

**grafana/provisioning/dashboards/dashboards.yml**:
```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/provisioning/dashboards/json
```

**deploy.sh** (deployment script):
```bash
#!/bin/bash
set -e

echo "🚀 Deploying monitoring stack..."

# Validate configuration
echo "Validating configuration..."
docker-compose config > /dev/null || exit 1

# Check environment variables
if [ ! -f .env ]; then
    echo "❌ .env file not found!"
    exit 1
fi

# Create necessary directories
mkdir -p prometheus grafana/provisioning/datasources grafana/provisioning/dashboards/json alertmanager

# Set correct permissions for Prometheus
echo "Setting permissions..."
sudo chown -R 65534:65534 prometheus/
sudo chown -R 472:472 grafana/

# Deploy
echo "Starting services..."
docker-compose up -d

# Wait for services to become healthy
echo "Waiting for services to become healthy..."
sleep 30

# Health checks
echo "Running health checks..."
./health-check.sh

echo "✅ Deployment complete!"
echo "Grafana: http://localhost:3000 (admin / ${GRAFANA_ADMIN_PASSWORD})"
echo "Prometheus: http://localhost:9090"
echo "Alertmanager: http://localhost:9093"
```

**health-check.sh** (validation script):
```bash
#!/bin/bash

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

check_service() {
    local name=$1
    local url=$2

    if curl -sf "$url" > /dev/null; then
        echo -e "${GREEN}✓${NC} $name is healthy"
        return 0
    else
        echo -e "${RED}✗${NC} $name is unhealthy"
        return 1
    fi
}

echo "🔍 Checking monitoring stack health..."
echo ""

check_service "Prometheus" "http://localhost:9090/-/healthy"
check_service "Grafana" "http://localhost:3000/api/health"
check_service "Alertmanager" "http://localhost:9093/-/healthy"
check_service "Node Exporter" "http://localhost:9100/metrics"
check_service "cAdvisor" "http://localhost:8080/healthz"
check_service "PostgreSQL Exporter" "http://localhost:9187/metrics"
check_service "Redis Exporter" "http://localhost:9121/metrics"
check_service "Nginx Exporter" "http://localhost:9113/metrics"

echo ""
echo "Checking Prometheus targets..."
targets=$(curl -s http://localhost:9090/api/v1/targets | jq -r '.data.activeTargets[] | select(.health != "up") | .labels.job')

if [ -z "$targets" ]; then
    echo -e "${GREEN}✓${NC} All Prometheus targets are up"
else
    echo -e "${RED}✗${NC} Down targets: $targets"
fi

echo ""
echo "✅ Health check complete!"
```

**TROUBLESHOOTING.md**:
```markdown
# Monitoring Stack Troubleshooting

## Prometheus won't start

**Error**: `permission denied` when creating storage

**Solution**:
```bash
sudo chown -R 65534:65534 prometheus/
docker-compose restart prometheus
```

## Grafana shows "No data"

**Check**:
1. Is Prometheus healthy? `curl http://localhost:9090/-/healthy`
2. Are targets being scraped? Visit http://localhost:9090/targets
3. Is Grafana datasource configured? Check Grafana → Configuration → Datasources

**Solution**:
```bash
# Verify Prometheus is reachable from Grafana
docker exec grafana wget -O- http://prometheus:9090/-/healthy
```

## PostgreSQL exporter showing "connection refused"

**Check**: Is the DATA_SOURCE_NAME correct in .env?

**Solution**:
```bash
# Test connection from exporter container
docker exec postgres_exporter psql ${DATA_SOURCE_NAME} -c "SELECT 1"
```

## Alerts not firing

**Check**:
1. Are alert rules loaded? Visit http://localhost:9090/alerts
2. Is Alertmanager configured? Visit http://localhost:9093
3. Is Slack webhook correct?

**Test Slack webhook**:
```bash
curl -X POST ${SLACK_WEBHOOK_URL} \
  -H 'Content-Type: application/json' \
  -d '{"text":"Test alert from monitoring stack"}'
```

## High CPU usage from cAdvisor

**Symptom**: cAdvisor using 50%+ CPU

**Solution**: Reduce collection frequency
```yaml
# In docker-compose.yml
cadvisor:
  command:
    - '--housekeeping_interval=30s'  # Default is 10s
    - '--docker_only=true'
```
```

**Human validation of Iteration 3**:

✅ **Production ready**:
- All dashboards imported successfully
- Alert routing working correctly (tested with test alerts)
- Health check script validates entire stack
- Deployment script handles permissions correctly
- Troubleshooting guide covers all common issues encountered during testing

✅ **Final tests**:
```bash
./deploy.sh  # Deploys successfully
./health-check.sh  # All checks pass
# Triggered test alert → Correctly routed to #alerts-warn
# Simulated critical condition → Correctly routed to #alerts-critical
```

### Iteration 4: Documentation and Handoff

**Prompt to Claude Code**:
```markdown
Perfect! Last step: create comprehensive README.md for the monitoring stack.

Include:
- Architecture overview
- Quick start guide
- Configuration reference
- Maintenance procedures
- Scaling considerations

This will be used by other engineers, so make it thorough but accessible.
```

**AI Response (Claude Code)**:

**README.md**:
```markdown
# Production Monitoring Stack

Comprehensive monitoring solution for Docker Compose infrastructure using Prometheus, Grafana, and Alertmanager.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Monitored Services                    │
│  Node.js Apps  │  PostgreSQL  │  Redis  │  Nginx       │
└────────┬─────────────┬────────────┬──────────┬──────────┘
         │             │            │          │
         │             │            │          │
    ┌────▼─────┐  ┌───▼──────┐ ┌──▼────┐ ┌──▼──────┐
    │   App    │  │ Postgres │ │ Redis │ │  Nginx  │
    │ Metrics  │  │ Exporter │ │Export.│ │ Export. │
    └────┬─────┘  └───┬──────┘ └──┬────┘ └──┬──────┘
         │            │            │          │
         └────────────┴────────────┴──────────┘
                      │
                 ┌────▼─────────┐
                 │  Prometheus  │ ← Scrapes metrics every 15s
                 │  (Storage)   │ ← Stores 30 days of data
                 └────┬─────────┘ ← Evaluates alert rules
                      │
          ┌───────────┴───────────┐
          │                       │
     ┌────▼────────┐      ┌──────▼──────┐
     │  Grafana    │      │Alertmanager │
     │(Dashboards) │      │   (Alerts)  │
     └─────────────┘      └──────┬──────┘
                                 │
                           ┌─────▼──────┐
                           │   Slack    │
                           │(#alerts-*) │
                           └────────────┘
```

## Quick Start

```bash
# 1. Clone configuration
git clone <repo> && cd monitoring

# 2. Configure environment
cp .env.example .env
nano .env  # Set GRAFANA_ADMIN_PASSWORD, POSTGRES credentials, SLACK_WEBHOOK_URL

# 3. Deploy
./deploy.sh

# 4. Verify
./health-check.sh

# 5. Access
# Grafana: http://localhost:3000 (admin / <your-password>)
# Prometheus: http://localhost:9090
# Alertmanager: http://localhost:9093
```

## Configuration Reference

### Resource Allocation

| Service | Memory Limit | CPU Limit | Purpose |
|---------|--------------|-----------|---------|
| Prometheus | 2GB | 1.0 | Metrics storage and querying |
| Grafana | 512MB | 0.5 | Dashboard visualization |
| Alertmanager | 256MB | 0.25 | Alert routing |
| cAdvisor | 256MB | 0.25 | Container metrics |
| Node Exporter | 128MB | 0.1 | System metrics |
| Exporters (each) | 64-128MB | 0.1 | Service-specific metrics |

**Total**: ~3.5GB allocated (leaves 4.5GB for applications on 8GB system)

### Alert Rules

| Alert | Threshold | Duration | Severity | Channel |
|-------|-----------|----------|----------|---------|
| HighCpuUsage | >80% | 5min | warning | #alerts-warn |
| HighMemoryUsage | >90% | 5min | critical | #alerts-critical |
| HighDiskUsage | >85% | 5min | critical | #alerts-critical |

### Metrics Retention

- **Storage**: 30 days
- **Scrape interval**: 15 seconds
- **Estimated size**: ~500MB per day for this scale

## Maintenance

### Daily

- Check Slack channels for alerts
- Verify all Grafana dashboards are updating

### Weekly

- Review Prometheus storage usage: `du -sh /var/lib/docker/volumes/prometheus_data`
- Check for dead targets: http://localhost:9090/targets

### Monthly

- Review and tune alert thresholds based on false positive rate
- Update dashboard filters based on team feedback
- Backup Grafana dashboards: `./backup-dashboards.sh`

### Updates

```bash
# Update monitoring stack images
docker-compose pull
docker-compose up -d
./health-check.sh
```

## Scaling Considerations

### When to scale UP (more resources)

- Prometheus memory usage consistently >75%
- Query timeouts in Grafana
- Scrape interval falling behind

**Solution**: Increase Prometheus memory limit:
```yaml
prometheus:
  deploy:
    resources:
      limits:
        memory: 4G  # Increased from 2G
```

### When to scale OUT (more infrastructure)

- Monitoring >100 targets
- Need multi-region monitoring
- Need HA for Prometheus

**Solution**: Consider:
- Thanos for multi-cluster Prometheus
- Cortex for HA metrics
- VictoriaMetrics for better performance

## Common Tasks

### Add new service to monitoring

1. Add scrape config to `prometheus/prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'new-service'
    static_configs:
      - targets: ['new-service:8080']
```

2. Reload Prometheus:
```bash
curl -X POST http://localhost:9090/-/reload
```

### Create custom dashboard

1. Access Grafana: http://localhost:3000
2. Create → Dashboard → Add Panel
3. Query: `rate(http_requests_total[5m])`
4. Save dashboard
5. Export JSON: Dashboard Settings → JSON Model
6. Save to `grafana/provisioning/dashboards/json/custom.json`

### Silence alerts temporarily

```bash
# Silence for 2 hours
curl -X POST http://localhost:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [{"name": "alertname", "value": "HighCpuUsage"}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%S)'",
    "endsAt": "'$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%S)'",
    "createdBy": "ops-team",
    "comment": "Planned maintenance"
  }'
```

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for detailed troubleshooting guide.

## Support

- **Slack**: #infrastructure
- **On-call**: ops-team@company.com
- **Documentation**: https://wiki.company.com/monitoring
```

**Final Result**:

✅ **Complete monitoring stack delivered**:
- 4 iterations, ~2 hours total time
- Production-ready configuration
- Full documentation
- Deployment automation
- Health validation
- Troubleshooting guide

**Human's final validation**:
```bash
# Deploy to production
./deploy.sh  # Success

# Monitor for 48 hours
# - All services stable
# - Alerts firing correctly
# - No performance issues
# - Team feedback positive

✅ Monitoring stack approved for production use
```

---

## Summary: Agentic Engineering in Practice

This real-world example demonstrated the agentic engineering workflow:

1. **Iteration 1**: AI generated initial solution → Human identified gaps
2. **Iteration 2**: AI hardened security and reliability → Human validated improvements
3. **Iteration 3**: AI added production polish → Human confirmed production-readiness
4. **Iteration 4**: AI created comprehensive documentation → Human approved for deployment

**Key lessons**:
- **Start with good context**: CEDAR framework → 80% correct first iteration
- **Expect refinement**: 4 iterations normal for production infrastructure
- **Validate constantly**: Never deploy without testing
- **Understand the output**: You can explain every configuration choice
- **Document everything**: Future you (and your team) will thank you

**Time saved**: ~20 hours of manual configuration and documentation compressed into 2 hours of human-AI collaboration.

---

## Best Practices Summary

### Do These Things

✅ **Provide comprehensive context** (environment, constraints, requirements)
✅ **Break complex tasks into phases** (foundation → features → polish)
✅ **Validate at every iteration** (syntax → security → functionality)
✅ **Understand what AI generates** (read and comprehend before deploying)
✅ **Test in isolation first** (dev environment → staging → production)
✅ **Document your infrastructure** (future you needs to understand it)
✅ **Use resource limits** (prevent runaway processes)
✅ **Implement health checks** (automated validation of service health)
✅ **Set up monitoring** (observe your infrastructure)
✅ **Plan for failure** (rollback procedures, disaster recovery)

### Don't Do These Things

❌ **Don't blindly copy-paste** (understand before deploying)
❌ **Don't skip validation** (syntax, security, functionality all matter)
❌ **Don't hardcode secrets** (use environment variables or secret management)
❌ **Don't ignore resource limits** (unbounded services will consume everything)
❌ **Don't test in production first** (use dev/staging environments)
❌ **Don't deploy without documentation** (you'll forget how it works)
❌ **Don't trust AI blindly** (verify critical configurations independently)
❌ **Don't skip health checks** (silent failures are the worst failures)
❌ **Don't over-engineer** (start simple, add complexity as needed)
❌ **Don't forget about maintenance** (infrastructure needs care)

---

## Chapter Summary

In this chapter, you learned the **agentic engineering workflow** for infrastructure development:

**Core Concepts**:
- Agentic engineering is **human-AI collaboration**, not delegation
- You provide **systems thinking and judgment**
- AI provides **implementation and syntax**
- Validation is **mandatory, not optional**

**Practical Skills**:
- **CEDAR framework** for effective prompting (Context, Expectation, Details, Assumptions, Result)
- **Three-layer validation** (syntax → security → functionality)
- **Iterative refinement** (expect 2-4 iterations for production-ready)
- **Debugging workflows** (systematic problem-solving with AI assistance)
- **Tool selection** (match AI assistant to task type)

**Mindset Shifts**:
- From "AI writes code" → "AI and I build infrastructure together"
- From "first draft is perfect" → "iterative improvement is normal"
- From "trust blindly" → "validate everything"
- From "copy-paste" → "understand deeply"

**Key Takeaway**: Agentic engineering is a **skill** that improves with practice. The more you collaborate with AI assistants, the better you become at:
- Writing effective prompts
- Identifying what to validate
- Knowing when AI is correct vs. when to verify independently
- Iterating efficiently toward production-ready solutions

You are not being replaced by AI. You are being **augmented** by it. The engineer who can effectively collaborate with AI is 10x more productive than the engineer who can't—and 100x more productive than the engineer who refuses to try.

---

## What's Next: Chapter 4

In Chapter 4, we'll apply agentic engineering workflows to build **The Nervous System (Observability)**: Prometheus, Grafana, Loki, and Jaeger.

You'll learn:
- How to design observability architectures with AI assistance
- Implementing the Golden Signals (latency, traffic, errors, saturation)
- Building custom dashboards and alert rules
- Log aggregation and distributed tracing
- Troubleshooting production incidents with observability data

**Preview**: We'll build a complete observability stack from scratch using the agentic engineering techniques from this chapter, showing you exactly how human systems thinking combines with AI implementation capabilities to create production-grade monitoring infrastructure.

Ready to see your infrastructure's nervous system in action? Turn to Chapter 4.

---

## Apex Agentic Engineering Stack

To operate at the leading edge of human-AI collaboration, anchor your workflows in these apex tools and rituals:

- **Prompt Mastery** — Use the CEDAR and SPEC frameworks alongside architecture decision records so every AI request carries precise context, constraints, and success criteria.
- **IDE Copilots** — Pair Cursor or GitHub Copilot Enterprise with Anthropic Claude, gpt-5-codex, or OpenAI o1 preview backends to blend code-generation speed with reasoning-grade reviews.
- **Pre-flight Validation** — Wire tfsec/Checkov for IaC scanning, kubeconform + Polaris for Kubernetes policy, Hadolint/Trivy for container supply chain, and Conftest/OPA for bespoke guardrails before human review ever starts.
- **Continuous Verification Pipelines** — Automate `terraform plan`, `helm template`, `docker compose config`, and `kubeval` steps in CI via Atlantis, Spacelift, or GitHub Actions reusable workflows, with policy gates enforced by OPA or Cedar.
- **Change Intelligence** — Use Infracost for cost deltas, DORA Four Keys dashboards for deployment health, and Slack/Linear integrations that attach AI-generated diffs with natural-language rationales for faster approvals.
- **Operational Memory** — Archive successful prompts, failure analyses, and red-team findings in Backstage TechDocs or Notion playbooks so every engineer (human or AI) benefits from collective experience.

These practices turn AI from a code generator into a disciplined engineering partner that reliably ships production infrastructure.