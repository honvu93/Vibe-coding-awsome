# Deployment Safety

## The Core Principle

Every deployment is a potential outage. Treat it with respect.

## Pre-Deployment Checklist

Before any deploy:

- [ ] All tests pass locally
- [ ] Build succeeds locally
- [ ] Database migrations tested (if any)
- [ ] CHANGELOG updated
- [ ] Recent commits reviewed — no accidental debug code, no credential files
- [ ] Backup exists (database at minimum)

## Scoped Commands

Never use global process management commands on shared infrastructure. One project's deploy must not restart another project's processes.

```bash
# DANGEROUS — affects ALL projects on the server
pm2 restart all
systemctl restart nginx
docker restart $(docker ps -q)

# SAFE — targets only your project
pm2 reload ecosystem.config.cjs   # scoped to config file
systemctl restart myapp.service   # scoped to named service
docker-compose restart web        # scoped to service name
```

If your deployment tooling does not support scoped commands, that is a gap to fix before the next deploy — not something to work around.

## Health Checks After Deploy

After every deployment, verify:

1. Your application responds on its expected port
2. Health endpoint returns 200
3. Key functionality works — not just "the page loads"
4. Other co-hosted services still respond (if on shared infrastructure)

```bash
# Check your app
curl -f http://localhost:3000/health || echo "HEALTH CHECK FAILED"

# Check co-hosted services on shared servers
curl -f http://localhost:4000/health || echo "SERVICE B DOWN"
curl -f http://localhost:5000/health || echo "SERVICE C DOWN"
```

This takes 30 seconds. Do it every time.

## Zero-Downtime Deployment

For production systems, use rolling restarts:

1. Start new process with updated code
2. New process passes health check
3. Old process receives graceful shutdown signal
4. Old process drains in-flight requests (with timeout)
5. Old process exits

The invariant: at least one healthy process is always serving traffic. Never stop the old process before the new one is confirmed healthy.

## Database Migration Safety

- Always backup before running migrations
- Test migrations against a copy of production data, not just an empty development database
- Never lock tables in production — use non-blocking index creation and multi-step strategies for large tables
- Make migrations reversible — include rollback scripts or DOWN migrations
- Separate schema changes from data migrations — run them in distinct deploy steps

Schema changes that require coordinated deploys (e.g., removing a column still read by old code) follow a three-step pattern:

1. Deploy code that tolerates both old and new schema
2. Apply schema migration
3. Deploy code that only works with new schema

## Rollback Plan

Before deploying, know the answers to:

- How do you roll back the code?
- How do you roll back the database?
- How long does rollback take?
- Have you practiced it?
- Who needs to be notified if rollback happens?

If you cannot answer these, you are not ready to deploy.

## Environment-Specific Rules

### Shared Servers (Multiple Projects)

- Always use scoped commands
- Health-check ALL services after deploy — not just yours
- Monitor resource usage during deploy (builds can consume enough memory to OOM other services)
- Coordinate deploy windows when changes are high-risk

### Dedicated Servers or Containers

- Still use health checks
- Still backup databases
- Global commands are safer here — there is nothing else to break

### CI/CD Pipelines

- Make health check the final pipeline step — pipeline is not done until health check passes
- Implement automatic rollback on failed health check
- Never deploy on a red CI build

## Anti-Patterns

- **Deploy and pray** — Always verify. Checking takes 30 seconds; an undetected outage costs hours.
- **Friday deploys** — Avoid unless you have on-call coverage and full rollback automation.
- **"It's a small change, no backup needed"** — Small changes break things too. The backup exists for exactly this case.
- **Global process commands on shared servers** — This is the single most common cause of taking down unrelated services during a routine deploy.
- **Skipping migration testing** — The leading source of deploy failures is a migration that behaves differently on production data than on development data.
- **No rollback plan** — You will need one eventually. Write it before the incident, not during it.

## Incident Response Template

When something goes wrong after a deploy:

1. **Assess** — Is the service fully down or degraded? What is the blast radius?
2. **Communicate** — Notify affected parties immediately, even before you have a fix
3. **Decide** — Roll back or fix forward? Roll back is usually faster
4. **Act** — Execute the decision quickly and without hesitation
5. **Verify** — Confirm the fix or rollback actually worked (re-run health checks)
6. **Document** — Write a short post-mortem: what happened, root cause, and what changes prevent recurrence

Keep the post-mortem blameless. The goal is system improvement, not individual fault-finding.
