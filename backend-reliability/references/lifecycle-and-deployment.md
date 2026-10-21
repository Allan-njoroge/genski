# Lifecycle and Deployment

A service is started, checked, restarted, and replaced many times over its life. Handling these moments well prevents a large share of production incidents.

## Contents

1. Startup
2. Health checks
3. Graceful shutdown
4. Configuration
5. Statelessness and disposability
6. Safe deployments
7. Rollbacks
8. Feature flags

## 1. Startup

- **Validate all configuration first.** Required settings missing or invalid: log a clear message and exit with an error. Never start with silent defaults for things like database URLs or secrets.
- **Connect to essential dependencies,** with retries and a limit, so a service starting slightly before its database doesn't fail permanently.
- **Report readiness only when ready to serve,** after connections are established and any warm-up is done.
- **Start quickly.** Slow startup makes scaling and recovery slow. Move heavy initialisation out of the startup path where possible.
- **Don't run database migrations from every instance on startup,** unless the project's tooling handles concurrent runs safely. Run them as a separate deployment step.

## 2. Health checks

Platforms such as Kubernetes and load balancers use health checks to decide whether to send traffic to an instance and whether to restart it. Different checks answer different questions:

| Check | Question | If it fails | Should check |
|---|---|---|---|
| **Liveness** | Is the process alive and not stuck? | The instance is restarted | Only the process itself: can it respond? |
| **Readiness** | Can it serve traffic right now? | Traffic is routed elsewhere until it recovers | Essential dependencies, warm-up complete, not shutting down |
| **Startup** | Has it finished starting? | Other checks wait | Initialisation complete |

The most common mistake: **a liveness check that depends on the database or other services.** When the database has a brief problem, every instance fails liveness, and the platform restarts them all at once, turning a small problem into a full outage. Keep liveness checks shallow.

Guidelines:

- Make health checks fast and cheap; they run frequently.
- Don't require authentication for internal health endpoints, but don't expose detailed health information publicly.
- Readiness should fail during graceful shutdown, so traffic stops arriving.
- Don't make readiness depend on non-essential dependencies, or a minor service outage takes yours offline too.

## 3. Graceful shutdown

Instances are stopped for deployments, scaling, and maintenance all the time. Each stop should be invisible to users.

On receiving a termination signal (such as `SIGTERM`):

1. **Fail readiness** and stop accepting new requests and jobs.
2. **Wait briefly** for the load balancer to stop routing new traffic, where the platform needs this.
3. **Finish in-flight work**: complete current requests, and finish or safely return current jobs to the queue.
4. **Close resources**: database pools, message consumers, file handles; flush logs and metrics.
5. **Exit** within the platform's grace period. If work can't finish in time, stop it cleanly so it can be retried.

Background workers must be able to stop mid-job safely: jobs should be idempotent and resumable, or returned to the queue unacknowledged so another worker picks them up.

## 4. Configuration

- Store configuration in the environment, separate from code (Twelve-Factor App).
- Validate it at startup, with clear errors.
- Use safe defaults only for settings where a default is genuinely safe, such as a page size. Never default secrets, URLs of external systems, or security settings.
- Keep timeouts, retry counts, pool sizes, and limits configurable, so they can be tuned during incidents.
- Log the effective configuration at startup, excluding secrets.

## 5. Statelessness and disposability

- **Keep instances stateless:** no user sessions, uploaded files, or job state held only in one instance's memory or local disk. Store them in a database, cache, or object storage, so any instance can serve any request and instances can be replaced at any time.
- **Assume any instance can stop at any moment,** including crashes without a graceful shutdown. Work must be recoverable: committed, queued, or retried.
- **Avoid in-memory scheduled jobs that must run exactly once.** With several instances, each would run them. Use a scheduler with locking or a single job runner.

## 6. Safe deployments

- **Small, frequent deployments** are safer than large, rare ones: less changes at once, and problems are easier to find.
- **Backward-compatible changes:** every deployment must work alongside the previous version, because both run at the same time during rollout. This applies to database schemas (see `backend-data`), APIs (see `backend-api-design`), message formats, and cache formats.
- **Gradual rollout:** where the platform supports it, send a small share of traffic to the new version first (canary releases), or roll out instance by instance, watching error rates and latency.
- **Automatic checks:** stop or roll back automatically if error rates or latency rise beyond thresholds, where the platform supports it.
- **Deploy when people can respond:** avoid risky deployments when nobody is available to watch them.

## 7. Rollbacks

- Every deployment must be reversible quickly, ideally by redeploying the previous version.
- Rollbacks only work if the previous version can run against the current database schema. This is why destructive migrations are separated into later releases (expand and contract).
- Know the rollback procedure before you need it, and practise it.
- If a change can't be rolled back (such as a data migration that discards information), treat it as an irreversible decision (see `decisions`) and plan it accordingly.

## 8. Feature flags

Feature flags separate deploying code from releasing behaviour.

- **Release gradually:** enable a feature for internal users, then a percentage of users, then everyone.
- **Turn off quickly:** disable a misbehaving feature without a deployment.
- **Default safely:** if the flag system is unavailable, the code should fall back to the safe, known behaviour.
- **Remove flags** once the feature is fully released. Old flags accumulate into confusing branches of untested code.
- **Test both states** of a flag while it exists.