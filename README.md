# ZT-RL Prototype

A runnable prototype of a system and method for AI-powered autonomous cloud hardening using reinforcement learning (RL) within a Zero Trust architecture, with an OPA policy gate and a digital twin safety simulator that validates changes before enforcement.

## Services
- `control-plane` (Node.js + Express): ingest telemetry, compute state, call RL, call OPA, simulate, execute (sandbox), compute reward
- `rl-engine` (Python + FastAPI): epsilon-greedy contextual bandit
- `policy` (OPA/Rego): Zero Trust policy decisions (allow/deny + reason)
- `ui-dashboard` (React + Vite): timeline + decisions view
- `postgres`: stores telemetry, states, decisions, simulations, executions, rewards


## Quick start
Prereqs: Docker Desktop.

```bash
cd zt-rl-prototype
docker compose up --build
```

Open:
- UI: http://localhost:5173
- Control plane API: http://localhost:8080
- RL engine: http://localhost:8000/docs
- OPA: http://localhost:8181

## Demo flow
1. In the UI, click **Generate Sample Telemetry**
2. Click **Run Hardening Cycle**
3. Watch: telemetry → RL decision → OPA gate → digital twin sim → execution → reward

## Evaluation
This prototype is designed to be **measurable**. For a strong demo (or a patent/proof-of-concept appendix), track:

### Core metrics
- **Gate pass rate (safety):** how often a proposed action passes both gates.
  - Proxy: `simulations.pass_fail = true` count / total.
- **Predicted breakage risk:** average / p95 of `simulations.predicted_impact.breakage_risk`.
- **Reward trend:** mean reward over time should improve as the RL policy adapts.
- **Action distribution:** shows exploration vs convergence (what actions the agent favors).
- **Time-to-decision:** optional; track end-to-end cycle time (API latency).

### How to run a quick evaluation
Run multiple cycles (you can re-run this as many times as you want):

```bash
# insert sample telemetry once
curl -sS -X POST http://localhost:8080/telemetry/sample -H 'Content-Type: application/json' -d '{}'

# run 50 cycles
for i in $(seq 1 50); do
  curl -sS -X POST http://localhost:8080/cycle/run -H 'Content-Type: application/json' -d '{"environment":"sandbox"}' >/dev/null
done

# fetch latest records (quick look)
curl -sS 'http://localhost:8080/timeline?limit=50' | head
```

### Example SQL queries (recommended)
```bash
docker compose exec -T postgres psql -U ztrl -d ztrl
```

Then:
```sql
-- Gate pass rate
SELECT
  COUNT(*) FILTER (WHERE pass_fail) * 1.0 / NULLIF(COUNT(*), 0) AS gate_pass_rate
FROM simulations;

-- Predicted breakage risk (avg + p95)
SELECT
  AVG((predicted_impact->>'breakage_risk')::float) AS avg_breakage_risk,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY (predicted_impact->>'breakage_risk')::float) AS p95_breakage_risk
FROM simulations;

-- Reward trend (per minute)
SELECT
  date_trunc('minute', ts) AS t,
  AVG(reward_value) AS avg_reward
FROM rewards
GROUP BY 1
ORDER BY 1;

-- Action distribution
SELECT action_id, COUNT(*) AS c
FROM decisions
GROUP BY 1
ORDER BY c DESC;
```

## Improvements / roadmap
If you want to move from “prototype” to “research-grade” or “enterprise-ish”, these are the highest leverage upgrades:

1. **Persist pre/post risk** per cycle (store `beforeRisk` and `afterRisk` in DB) so you can plot posture improvement directly.
2. **Store policy decision explicitly** (allowed/denied + reason) rather than inferring from `simulations.pass_fail`.
3. **Richer digital twin**
   - dependency graph + blast-radius
   - SLO checks (latency/availability budgets)
   - drift detection vs desired state
4. **Better RL**
   - contextual bandit with regularization
   - constrained RL / safe exploration (action masking)
   - per-domain agents (IAM agent, network agent, workload agent)
5. **Real executor (hybrid mode)**
   - apply changes to a real sandbox (K8s NetworkPolicy/RBAC, Terraform in a test AWS account)
   - idempotency + rollback on regression
6. **Explainability**
   - log “why this action” with top features and gate reasoning
   - generate an audit report per action for compliance
## Use cases (practical scenarios)

The ZT-RL prototype is meant to model **closed-loop security hardening** where:
1) telemetry describes risk, 2) an RL agent proposes an action, 3) OPA enforces policy guardrails, 4) a digital twin simulates impact, 5) only safe actions execute, and 6) rewards drive improvement over time.

Below are concrete scenarios you can demo with this architecture (control-plane + RL engine + OPA + digital twin).  

### 1) Auto-remediation for risky network exposure
**Problem:** Services drift to overly permissive inbound rules (0.0.0.0/0, wide ports).  
**Telemetry signals:** exposed_ports_count, public_ingress=true, failed_scans, unusual inbound spikes.  
**Candidate actions:** restrict ingress CIDRs, close ports, enforce service-to-service allowlists.  
**OPA constraints:** never block approved admin CIDRs; require change ticket tag; deny changes to “protected” namespaces.  
**Digital twin checks:** “would this break health checks / dependencies?”; simulated request success rate.  
**Reward idea:** reduce exposure score without increasing simulated outage.

### 2) Kubernetes posture hardening (baseline enforcement)
**Problem:** Workloads run with privileged settings or weak pod security.  
**Telemetry signals:** privileged_pods, hostNetwork, hostPath mounts, missing resource limits.  
**Candidate actions:** apply restrictive PodSecurity settings, drop capabilities, enforce resource limits.  
**OPA constraints:** forbid changes that violate exception list for system components.  
**Digital twin checks:** simulate scheduling failures + startup failures.  
**Reward idea:** improve posture score while keeping “pods_ready” stable.

### 3) IAM privilege minimization (least privilege drift control)
**Problem:** Roles accumulate permissions over time; blast radius grows.  
**Telemetry signals:** unused_permissions %, anomalous API calls, new principals, high-privilege role usage.  
**Candidate actions:** remove unused permissions, enforce MFA, rotate credentials, scope roles to resources.  
**OPA constraints:** never remove break-glass role; require owner approval tag for permission removals.  
**Digital twin checks:** simulate access for critical workflows; test minimal permission set.  
**Reward idea:** lower privilege risk without increasing “access_denied” for critical paths.

### 4) “Safe” firewall/WAF rule tuning with guardrails
**Problem:** Security teams want to tighten rules but fear false positives and outages.  
**Telemetry signals:** attack signatures, error rate, bot traffic, suspicious geos, request anomalies.  
**Candidate actions:** increase rule strictness, add rate limits, block patterns, adjust thresholds.  
**OPA constraints:** cap max strictness per service tier; never block partner allowlisted routes.  
**Digital twin checks:** replay synthetic traffic; check latency/error budgets.  
**Reward idea:** reduce attack score while meeting SLO constraints.

### 5) Incident response autopilot (containment-first, reversible)
**Problem:** During an incident you need fast containment, but with controlled blast radius.  
**Telemetry signals:** credential anomaly, unusual egress, privilege escalation attempts, high-severity alerts.  
**Candidate actions:** isolate workload segment, disable suspect credentials, tighten egress.  
**OPA constraints:** containment allowed only above severity threshold; require audit logging always on.  
**Digital twin checks:** simulate essential business flows; verify rollback plan exists.  
**Reward idea:** stop “threat propagation score” while minimizing impacted dependencies.

### 6) Continuous compliance controls (audit-friendly enforcement)
**Problem:** Maintain compliance posture continuously instead of point-in-time checks.  
**Telemetry signals:** encryption_at_rest=false, weak TLS configs, missing logging, noncompliant configs.  
**Candidate actions:** enforce TLS versions, enable logging, enforce encryption, tighten config baselines.  
**OPA constraints:** encode “must-have” controls; deny any action that weakens compliance baseline.  
**Digital twin checks:** validate config compatibility; ensure no downtime regressions.  
**Reward idea:** compliance score increases; time-to-compliance decreases.

### 7) Supply chain hardening for CI/CD and build pipelines
**Problem:** CI/CD runners and pipelines are common targets; secrets leak risk.  
**Telemetry signals:** unsigned artifacts, unpinned dependencies, secret scan hits, risky build steps.  
**Candidate actions:** enforce signed artifacts, pin dependencies, restrict runner permissions, rotate build secrets.  
**OPA constraints:** always require approvals for pipeline permission changes; deny disabling scanning steps.  
**Digital twin checks:** simulate pipeline success; verify build reproducibility.  
**Reward idea:** reduce supply-chain risk without increasing build failures.

### 8) Adaptive “risk budget” enforcement across environments
**Problem:** Dev/stage/prod need different risk tolerance but consistent policy.  
**Telemetry signals:** environment tag, risk score, change frequency, recent incident history.  
**Candidate actions:** allow more exploration in sandbox, conservative changes in prod.  
**OPA constraints:** prod requires stricter thresholds; sandbox allows broader action set.  
**Digital twin checks:** environment-specific SLO tests.  
**Reward idea:** faster posture improvement in sandbox; fewer regressions in prod.

### 9) Tenant isolation for multi-tenant platforms
**Problem:** One tenant’s compromise shouldn’t affect others (blast-radius control).  
**Telemetry signals:** cross-tenant access attempts, shared resource hotspots, unusual inter-tenant calls.  
**Candidate actions:** tighten network segmentation, per-tenant identity policies, isolate shared resources.  
**OPA constraints:** enforce strict tenant boundary rules; deny cross-tenant access.  
**Digital twin checks:** simulate tenant workflows; verify no cross-tenant leakage.  
**Reward idea:** reduce cross-tenant risk while keeping tenant SLOs.

### 10) “Autonomous hardening suggestions” mode (human-in-the-loop)
**Problem:** Teams want recommendations + rationale before auto-enforcement.  
**Flow:** RL proposes actions, OPA + digital twin validate, but executor runs in “recommend-only”.  
**Outputs:** ranked actions, reasons for allow/deny, predicted impact, rollback hints.  
**Reward idea:** measure acceptance rate by humans + reduced incident rate over time.

## Local development (optional)
Each service can also be run standalone; see each subfolder README.

## Git push
After you download this repo:
```bash
git init
git add .
git commit -m "Initial prototype"

git branch -M main
git remote add origin git@github.com:<your-username>/<your-repo>.git
git push -u origin main
```

