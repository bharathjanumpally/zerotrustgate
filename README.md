# ZT-RL Prototype

A runnable prototype for **RL-driven autonomous security hardening** with a **Zero Trust gate** and a **digital twin** safety simulator.

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

