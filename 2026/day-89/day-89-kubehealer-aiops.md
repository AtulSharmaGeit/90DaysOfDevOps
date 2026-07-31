# Production AI Agents: KubeHealer and AIOps
>We have built agents that diagnose problems. Today we build one that fixes them — autonomously. KubeHealer is a production-grade AI agent that scans our Kubernetes cluster for broken pods, diagnoses the root cause using Claude, proposes fixes, and applies them with your approval. It uses Temporal for durable execution — meaning if the agent crashes mid-repair, it picks up exactly where it left off. 

This is AIOps: AI-powered operations. Not a chatbot. A system that watches, reasons, and acts.

Reference: https://github.com/TrainWithShubham/agentic-ai-for-devops -- modules 4, 5
KubeHealer repo: https://github.com/TrainWithShubham/kubehealer

## Table of Contents
1. [AIOps and Production Guardrails](#1-understand-aiops-and-production-guardrails-module-4)
2. [Set Up KubeHealer](#2-set-up-kubehealer)
3. [Deploy Broken Applications](#3-deploy-broken-applications)
4. [Run KubeHealer](#4-run-kubehealer)
5. [Test Crash Recovery with Temporal](#5-test-crash-recovery-temporal-durability)
6. [Reflect on the Agentic AI Journey](#6-reflect-on-the-agentic-ai-journey)

## 1. Understand AIOps and Production Guardrails (Module 4)
### What is AIOps?
**Ans.** AIOps uses AI to automate IT operations: monitoring, diagnosis, and remediation. The goal is not to replace human operators but to augment them — the agent handles routine, well-understood issues (image typos, resource limit violations) while escalating complex or ambiguous ones to a human.
 
### Why Durable Execution (Temporal) Matters?
**Ans.** Without durability (e.g., a plain Python script): if the agent crashes mid-diagnosis, all progress is lost. You restart from scratch, potentially leaving infrastructure in a half-repaired state.
 
With Temporal: every step is recorded as an event in the workflow history. If the worker process crashes and restarts, Temporal replays completed steps from history and resumes from the exact point of failure. For agents that modify infrastructure, this is not optional — partial fixes are often worse than no fix at all.
 
### When to Use AI Agents vs. Traditional Automation
| Use AI Agents When | Use Traditional Automation When |
|---|---|
| Problem requires reasoning over unknown errors | Problem has a known, fixed solution |
| Multiple possible causes and fixes exist | One cause, one fix (if X then Y) |
| Natural language output helps humans understand | No human needs to interpret the output |
| Examples: troubleshooting, root cause analysis | Examples: scaling, restarts, scheduled deploys |
 
### Production Guardrails
Every production AI agent needs these safeguards:
| Guardrail | Why It Matters | Example |
|---|---|---|
| **Human approval** | Agents must not make destructive changes without permission | "Found 3 broken pods. Here are the fixes. Approve?" |
| **Scope limits** | Agents should only operate in allowed namespaces | Cannot touch `kube-system` or production databases |
| **Audit trail** | Every action must be recorded and attributable | Temporal workflow history: every tool call, every decision, every Claude response |
| **Rollback capability** | Every fix must be reversible | Agent creates patches, not in-place replacements |
| **Timeout and retry limits** | Agents must not loop indefinitely | Max 3 retries per pod, 5-minute total timeout |
| **Escalation path** | When the agent cannot fix it, alert a human | "config-app needs a ConfigMap I cannot create. Escalating." |

## 2. Set Up KubeHealer
### Clone the Repository
This gives you the KubeHealer source code.
```bash
git clone https://github.com/TrainWithShubham/kubehealer.git
cd kubehealer
```

### Check Prerequisites
Make sure you have:
- **Docker** (for Temporal) → run `docker --version`
- **Kind** (for Kubernetes cluster) → run `kind version`
- **Python 3.10+** → run `python3 --version`
- We also need an Anthropic API key (Claude Sonnet) — sign up at `https://console.anthropic.com`.

### Start the Infrastructure
#### 1. Create a Kind Cluster
This simulates Kubernetes locally.
```bash
kind create cluster --name kubehealer-demo
```
Verify:
```bash
kubectl get nodes
```
We should see one control-plane node running.

#### 2. Start Temporal (Durable Execution Engine)
Temporal ensures crash recovery and audit trails.
```bash
temporal server start-dev
``` 
Open `http://localhost:8233` in our browser — the Temporal UI should load. Temporal is the durable execution engine that records every workflow step and enables crash recovery.

#### 3. Set Up Python Environment
Create a virtual environment and install dependencies.
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
Run `pip list` — we should see packages like `temporalio`, `anthropic`, etc.

#### 4. Configure Anthropic API Key
This lets KubeHealer talk to Claude.
```bash
export ANTHROPIC_API_KEY="your-api-key-here"
echo $ANTHROPIC_API_KEY
# Expected: your key printed
```

### At This Point
- You have **KubeHealer cloned**.
- A **Kind cluster running**.
- **Temporal server running** with UI.
- A **Python environment ready**.
- **Claude API key configured**.

## 3. Deploy Broken Applications
KubeHealer needs something to fix. Deploy three intentionally broken pods that represent real failure modes:

### Deploy App 1 — Image Typo (Fixable by the Agent)
This pod has a typo in the image name (`ngnix` instead of `nginx`).
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: default
spec:
  containers:
  - name: web
    image: ngnix:latest
    ports:
    - containerPort: 80
EOF
```
Verify:
```bash
kubectl get pods
kubectl describe pod web-app
```
- Status should show **ImagePullBackOff**.
- Events will mention `Failed to pull image "ngnix:latest"`.
- Correctable — agent patches the image field to `nginx:latest`.

### Deploy App 2 — OOM Crash (Fixable by the Agent)
This pod has an unrealistically low memory limit (`1Mi`) causes the container to be killed by the OOM killer.
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: memory-app
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      limits:
        memory: "1Mi"
    command: ["sh", "-c", "echo 'starting' && sleep 3600"]
EOF
```
Verify:
```bash
kubectl get pods
kubectl describe pod memory-app
```
- Status should show **CrashLoopBackOff**.
- Events will mention `OOMKilled` (out of memory).
- Correctable — agent patches `resources.limits.memory` to `128Mi`.

### Deploy App 3 — Missing ConfigMap (NOT Fixable by the Agent)
This pod references a ConfigMap that doesn’t exist.
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-app
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    envFrom:
    - configMapRef:
        name: app-config
EOF
```
The agent can diagnose this but should escalate it -- creating arbitrary ConfigMaps requires human decision.

Verify:
```bash
kubectl get pods
kubectl describe pod config-app
```
- Status should show **CreateContainerConfigError**.
- Events will mention `configmap "app-config" not found`.
- Not correctable — creating a ConfigMap requires knowing what data it should contain, which the agent cannot infer. This case demonstrates the escalation path.

### Confirm All Three Are Broken
```bash
kubectl get pods
```
Expected output:
```
NAME         READY   STATUS              RESTARTS
web-app      0/1     ImagePullBackOff    0
memory-app   0/1     CrashLoopBackOff    3
config-app   0/1     CreateContainerConfigError   0
```
You now have **three broken pods**:
- `web-app` → Image typo
- `memory-app` → OOMKilled
- `config-app` → Missing ConfigMap

These are the test cases KubeHealer will scan, diagnose, and attempt to fix in **Task 4**.

## 4. Run KubeHealer
### Start the Temporal Worker
This worker is the **agent** that runs inside Temporal.
```bash
python3 worker.py
# Expected: Worker started — listening for workflows
```
- This means the agent is now waiting for workflows to be triggered.

### Trigger a Healing Run
Open a second terminal (keep the worker running in the first one) and start the workflow:
```bash
python3 starter.py
```
- You should see output like `Workflow started` or `Scanning pods…`.

### Watch the Agent Work
The agent will go through these phases:<br>
**Phase 1 — Scan**: Lists all pods, identifies those in non-Running states.<br>
**Phase 2 — Diagnose**: For each broken pod, calls `kubectl describe`, reads events, and sends the output to Claude with a structured prompt asking for root cause and fix recommendations.<br>
**Phase 3 — Propose**: Presents all findings and proposed fixes:
   - `web-app`: "Image typo. Fix: change `ngnix:latest` to `nginx:latest`"
   - `memory-app`: "OOMKilled. Fix: increase memory limit to 128Mi"
   - `config-app`: "Missing ConfigMap `app-config`. Cannot fix automatically -- requires manual ConfigMap creation"

**Phase 4 — Apply**: On approval, patches the correctable pods and reports the escalation for `config-app`.<br>
Type `yes` to approve.

In the terminal, we will see:
```
Found 3 broken pods.

Proposed fixes:
1. web-app: Fix image typo (ngnix -> nginx)
2. memory-app: Increase memory limit (1Mi -> 128Mi)
3. config-app: CANNOT FIX - needs manual ConfigMap creation

Approve all fixes? [yes/no]:
```

Type `yes`. The agent:
- Patches `web-app` with the correct image
- Patches `memory-app` with increased memory
- Skips `config-app` and reports it needs human attention

### Confirm Results
```bash
kubectl get pods
```
Expected output:
```Code
NAME         READY   STATUS    RESTARTS   AGE
web-app      1/1     Running   0          <minutes>
memory-app   1/1     Running   0          <minutes>
config-app   0/1     Error     0          <minutes>
```
- `web-app` and `memory-app` should now be Running. `config-app` still broken (as expected).

All three guardrails operated as designed: human approval before changes, scope-limited patches, and escalation when the fix exceeds the agent's authority.

### At This Point
- You’ve seen the ***agent scan, diagnose, propose fixes, and apply them with approval***.
- You’ve confirmed that **two pods were healed automatically** and one was escalated.
- This demonstrates the **guardrails**: human approval, scope limits, audit trail, rollback capability, and escalation.

## 5. Test Crash Recovery (Temporal Durability)
This section demonstrates the production-grade durability that makes KubeHealer safe to run against real infrastructure.

### Redeploy Broken Apps
First, clean up the pods from Task 4 and redeploy the broken ones.
```bash
kubectl delete pod web-app memory-app config-app
```
Then re-apply the manifests from **Task 3** (the three broken apps: `web-app`, `memory-app`, `config-app`).

Verify:
```bash
kubectl get pods
```
We should again see:
```Code
web-app      0/1   ImagePullBackOff
memory-app   0/1   CrashLoopBackOff
config-app   0/1   CreateContainerConfigError
```

### Start Worker + Trigger Healing
In terminal 1, start the worker:
 
```bash
python3 worker.py &
```
 
In terminal 2, trigger the workflow:
 
```bash
python3 starter.py
```
 
Wait until the agent is actively diagnosing — you will see log output indicating it is calling `kubectl describe` and sending results to Claude. Before it reaches the approval prompt, kill the worker:
 
```bash
kill %1
# Or Ctrl+C in terminal 1
```
 
The agent is dead mid-diagnosis. In a traditional script, all progress is lost.

### Restart the Worker
Bring the worker back:
```bash
python3 worker.py
```
Watch the logs: Temporal replays the completed activities (scan, diagnose) from its event history and resumes the workflow from the exact point of interruption. The agent proceeds to the approval prompt as if nothing happened.
 
### Inspect the Workflow in the Temporal UI
Open `http://localhost:8233` and navigate to the running workflow. You will see:
- Every activity execution with its inputs and outputs
- The timestamp and duration of each Claude API call
- The crash event and the recovery point
- The complete timeline from start to finish

This is your audit trail — every tool call, every diagnostic decision, every fix applied, all recorded automatically and permanently.

## 6. Reflect on the Agentic AI Journey
### Map the 3-day progression:
| Day | Module | What You Built | Pattern |
|-----|--------|---------------|---------|
| 87 | 0-2 | Docker Error Explainer + Docker Agent | Basic LLM -> ReAct Agent |
| 88 | 3, 6 | Multi-tool Agent + MCP Server + CI/CD Analyzer | Multi-domain tools, MCP protocol |
| 89 | 4-5 | KubeHealer -- production self-healing agent | Temporal durability, human approval, guardrails |

### Understand the Evolution
```
Day 87: LLM explains errors             (passive — read only)
        │
Day 88: Agent diagnoses across Docker,
        Kubernetes, and CI/CD           (autonomous investigation)
        │
Day 89: Agent diagnoses AND fixes
        with human approval             (autonomous action with guardrails)
```
This shows your journey from explainer → investigator → fixer.

### Key Principles for Production AI Agents
| Principle | What It Means |
|---|---|
| **Tools are CLI wrappers** | Any command you run daily can become an agent tool |
| **ReAct is universal** | The Think → Act → Observe loop works for any domain |
| **MCP standardises tool access** | Write tools once, expose them to any AI client |
| **Guardrails are not optional** | Approval gates, scope limits, and audit trails are table stakes |
| **Durability matters** | Temporal prevents lost state during infrastructure changes |
| **Know when not to use AI** | Simple if/then automation is better for known, fixed problems |

### Connect to the Rest of the Challenge
| Day Range | Topic | Connection to Agentic AI |
|---|---|---|
| 29–37 | Docker | Docker tools in Module 2 wrap the exact same `docker` commands you learned |
| 40–49 | GitHub Actions | The CI/CD Analyzer in Module 6 diagnoses the pipelines you built |
| 50–67 | Kubernetes | KubeHealer uses `kubectl` — the same tool, now wielded autonomously |
| 73–77 | Observability | Agents could query Prometheus and Loki to drive metric-based diagnosis |
| 84–86 | ArgoCD / GitOps | An agent could trigger ArgoCD syncs, rollbacks, or flag drift |
 
Every block of this challenge is a potential tool in an AI agent. The agent is the integration layer.

### Clean Up
Once you’re done experimenting:
```bash
kind delete cluster --name kubehealer-demo
# Stop Temporal: Ctrl+C in the terminal running temporal server start-dev
deactivate
```

### At This Point
You’ve:
- Mapped the **3-day agentic AI journey**.
- Understood the **principles and guardrails** for production agents.
- Connected KubeHealer back to your earlier Docker, CI/CD, Kubernetes, Observability, and ArgoCD work.

This reflection makes your portfolio entry for Day 89 complete and professional.