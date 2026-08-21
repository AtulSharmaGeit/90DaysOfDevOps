# Multi-Tool Agents, MCP, and CI/CD Analyzer
>Yesterday we built a Docker-only agent. Today we extend it to handle both Docker AND Kubernetes, learn the Model Context Protocol (MCP) — the emerging standard for connecting AI to tools — and build a CI/CD Failure Analyzer that diagnoses broken GitHub Actions pipelines.

By the end of today, our agent can troubleshoot across three domains (Docker, Kubernetes, CI/CD) and our tools can be used by any MCP-compatible AI client — not just our Python script.

Reference:<br> 
https://github.com/TrainWithShubham/agentic-ai-for-devops -- modules 3, 6<br>
https://github.com/TrainWithShubham/AI-BankApp-DevOps.git -- AI-BankApp-DevOps Repo

## Table of Contents
1. [Build the Multi-Tool DevOps Agent](#1-build-the-multi-tool-devops-agent-module-3)
2. [Understanding MCP](#2-understand-the-model-context-protocol-mcp)
3. [Build and Use the MCP Server](#3-build-and-use-the-mcp-server-module-3)
4. [Build the CI/CD Failure Analyzer](#4-build-the-cicd-failure-analyzer-module-6)
5. [Build Your Own Tool](#5-build-your-own-tool)
6. [Clean Up and Review](#6-clean-up)

## 1. Build the Multi-Tool DevOps Agent (Module 3)
The Day 87 Docker agent had 3 tools. This module adds 3 Kubernetes tools to the same agent — one LLM, six tools, two infrastructure domains.
### Set up a Kind Cluster with a Broken Pod:
```bash
kind create cluster --name devops-demo
kubectl cluster-info
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1921).png)

```bash
kubectl apply -f module-3/broken_pod.yaml
kubectl get pods
```
- Pod named **broken-pod** will enter CrashLoopBackOff

The `broken_pod.yaml` deploys a pod that crashes immediately after starting:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    command: ["sh", "-c", "echo 'app starting...' && sleep 2 && exit 1"]
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1926).png)

### Create Broken Docker Container
Run a container that exits immediately to simulate failure.
```bash
docker run -d --name broken-container nginx:alpine \
  sh -c "echo 'container starting...' && sleep 2 && exit 1"

docker ps -a | grep broken-container
# Expected: Restarting state
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1928).png)

### Study the New Kubernetes Tools in `module-3/agent.py`
It has 6 tools now:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1933).png)

Docker tools (from Day 87):
- `list_containers()` -- `docker ps -a`
- `get_logs(container_name)` -- `docker logs`
- `inspect_container(container_name)` -- `docker inspect`

Kubernetes tools (new):
```python
@tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def describe_pod(pod_name: str, namespace: str = "default") -> str:
    """Get detailed info about a Kubernetes pod including events and conditions."""
    result = subprocess.run(
        ["kubectl", "describe", "pod", pod_name, "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def get_events(namespace: str = "default") -> str:
    """Get recent Kubernetes events in a namespace (useful for troubleshooting)."""
    result = subprocess.run(
        ["kubectl", "get", "events", "-n", namespace, "--sort-by=.lastTimestamp"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr
```
The tool selection principle is the same as Day 87: the LLM reads each docstring to decide which tool to call. A question about pods triggers `list_pods` or `describe_pod`. A question about Docker triggers the Docker tools. A question spanning both domains triggers tools from both.

### Run the Multi-Domain Agent
Execute the agent script to test multi-domain troubleshooting.
```bash
python3 module-3/agent.py
```
Ask questions that span both domains:
```
> What's broken across Docker and Kubernetes?
> Why is broken-pod crashing?
> Are there any unhealthy containers on Docker?
> Describe the events in the default namespace
```
The agent decides which tools to use based on the question. Ask about Docker -- it uses Docker tools. Ask about pods -- it switches to Kubernetes tools. Ask about both -- it uses all of them.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1960).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1961).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1965).png)

**This is the power of the ReAct pattern:** One agent, many tools, one brain that decides what to use.

## 2. Understand the Model Context Protocol (MCP)
MCP (Model Context Protocol) is an open standard created by Anthropic for connecting AI models to external tools and data sources. Instead of writing tools inside our agent code, we expose them as an MCP server and any compatible client can discover and call them.

**Why MCP matters for DevOps:**
| Without MCP | With MCP |
|---|---|
| Tools are locked to one framework (LangChain) | Tools work with any MCP-compatible client |
| Every AI client re-implements the same Docker/K8s tools | Write once, expose as a service |
| Tool access is tied to the agent's codebase | Tools are discoverable at runtime via the MCP protocol |
| New AI client = rewrite all tools | New AI client = point it at the existing MCP server |

**MCP-Compatible Clients:**
- Claude Desktop
- VS Code (GitHub Copilot)
- Cursor
- Claude Code (the CLI you might already be using)
- Any LangChain agent via `langchain-mcp-adapters`

### Architecture
```
[MCP Server]                         [MCP Clients]
  │
  ├── list_pods()                     ┌── Claude Desktop
  ├── describe_pod()      ◄──────►    ├── VS Code Copilot
  ├── get_events()                    ├── Your Python agent
  │                                   └── Any MCP client
  │
  (exposes tools via stdio or HTTP)
```
The server exposes tools. Clients discover and call them. The tool implementation lives in one place.

## 3. Build and Use the MCP Server (Module 3)
### Create the MCP Server
Navigate to our project folder:
```bash
cd agentic-ai-for-devops/module-3
```
Open `mcp_server.py` and add Kubernetes tools:
```python
from fastmcp import FastMCP

mcp = FastMCP("Kubernetes Tools")

@mcp.tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@mcp.tool
def describe_pod(pod_name: str, namespace: str = "default") -> str:
    """Get detailed info about a Kubernetes pod including events and conditions."""
    # ...

@mcp.tool
def get_events(namespace: str = "default") -> str:
    """Get recent Kubernetes events in a namespace."""
    # ...

if __name__ == "__main__":
    mcp.run()
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1971).png)

Key difference from LangChain tools:
- `@mcp.tool` instead of `@tool` -- registered with the MCP server
- `FastMCP("Kubernetes Tools")` -- creates a named MCP server
- `mcp.run()` -- starts the server (stdio transport by default)
- Any MCP client can discover and call these tools

### Run the MCP Server
Start the server:
```bash
cd agentic-ai-for-devops/module-3
python3 mcp_server.py
```
- By default, it uses stdio transport (local machine).
- The server now exposes our Kubernetes tools (list_pods, describe_pod, get_events) to any MCP client.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1973).png)

### Create the MCP Client Agent
Open `agent_with_mcp.py`:
```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async def main():
    client = MultiServerMCPClient({
        "docker-mcp": {
            "transport": "stdio",
            "command": "python",
            "args": ["mcp_server.py"]
        }
    })

    tools = await client.get_tools()    # Dynamically discovers tools from MCP
    llm = ChatOllama(model="gemma4", temperature=0.8)
    agent = create_agent(llm, tools)    # Same ReAct agent, but tools come from MCP
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1975).png)

The agent does not define tools locally. It connects to the MCP server and discovers them at runtime.

### Run the MCP Agent
```bash
cd module-3
python3 agent_with_mcp.py
```
Ask the same Kubernetes questions:
```
> List the pods in my cluster
> Why is broken-pod crashing?
> What events happened recently?
```
Same result as before, but the tools are served via MCP - instead of being hardcoded in the agent.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1979).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1982).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1989).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(1998).png)

### Integrate with Claude Desktop (Optional)
Configure Claude Desktop with your MCP server (if you have Claude Desktop installed):

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):
```json
{
  "mcpServers": {
    "kubernetes-tools": {
      "command": "python3",
      "args": ["/full/path/to/agentic-ai-for-devops/module-3/mcp_server.py"]
    }
  }
}
```
Restart Claude Desktop. Now you can ask Claude: "List the pods in my cluster" and it will call your MCP server's `list_pods()` tool.

## 4. Build the CI/CD Failure Analyzer (Module 6)
The same ReAct pattern works for CI/CD. This agent uses the `gh` CLI to diagnose GitHub Actions failures.
### Authenticate GitHub CLI
Before anything else, make sure the GitHub CLI (gh) is authenticated:
```bash
gh auth login
```
- Choose **GitHub.com**.
- Select **HTTPS**.
- Paste personal access token (PAT) if prompted.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2002).png)

- Verify with:
    ```bash
    gh auth status
    ```

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2006).png)

### Study `ci_analyzer.py`
Open `module-6/ci_analyzer.py`. It defines three tools:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2009).png)

```python
@tool
def list_workflow_runs(status: str = "failure") -> str:
    """List recent GitHub Actions workflow runs. Use status='failure' for failed runs."""
    result = subprocess.run(
        ["gh", "run", "list", "--status", status, "--limit", "5"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def get_failed_logs(run_id: str) -> str:
    """Get the failed step logs from a GitHub Actions run. Pass the run ID."""
    result = subprocess.run(
        ["gh", "run", "view", run_id, "--log-failed"],
        capture_output=True, text=True,
    )
    output = result.stdout + result.stderr
    if len(output) > 5000:
        output = output[:5000] + "\n\n[...truncated, showing first 5000 chars]"
    return output

@tool
def get_workflow_file(workflow_name: str) -> str:
    """Read a GitHub Actions workflow YAML file. Pass the filename like 'ci.yml'."""
    import pathlib
    path = pathlib.Path(f".github/workflows/{workflow_name}")
    if path.exists():
        return path.read_text()
    return f"File not found: {path}"
```
**Note the log truncation** in `get_failed_logs` -- LLMs have token limits. We cannot send 100KB of CI logs. Truncating to 5000 characters keeps it within bounds while preserving the most important information (the failed step output).

### Run the CI/CD Analyzer
Run it from inside the AI-BankApp repo (which has GitHub Actions workflows):
```bash
cd AI-BankApp-DevOps
python3 ../agentic-ai-for-devops/module-6/ci_analyzer.py
```
Ask:
```
> What failed in my last CI run?
> Show me the recent workflow runs
> Read the gitops-ci.yml workflow file and explain what it does
```
The agent lists failed runs, fetches their logs, reads the workflow file, and explains the root cause.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2019).png)

### Test with a Broken Workflow
Create a deliberately broken workflow in a test repository:

`.github/workflows/broken-ci.yml`
```yaml
name: Broken CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test    # Will fail -- no package.json!
```
Push it:
```bash
git add .github/workflows/broken-ci.yml
git commit -m "Add broken CI workflow"
git push origin main
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2023).png)

Wait for the workflow to fail, then ask:
```Code
Why did broken-ci fail?
```
The agent will list the failed run, fetch its logs, identify that `npm test` failed because there is no `package.json`, and explain the fix.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2026).png)

## 5. Build Your Own Tool
The pattern is now clear. Any CLI command can become an agent tool. Choose one of these extensions:

**Option A — Terraform Plan Analyzer:**
```python
@tool
def terraform_plan() -> str:
    """Run terraform plan and return the output showing what would change."""
    result = subprocess.run(
        ["terraform", "plan", "-no-color"],
        capture_output=True, text=True,
        cwd="/path/to/your/terraform/project"
    )
    output = result.stdout + result.stderr
    if len(output) > 5000:
        output = output[:5000] + "\n[...truncated]"
    return output
```
Test query: `What changes would terraform plan show?`

**Option B — AWS Resource Checker:**
```python
@tool
def list_ec2_instances() -> str:
    """List all EC2 instances with their state, type, and name."""
    result = subprocess.run(
        ["aws", "ec2", "describe-instances",
         "--query", "Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,Tags[?Key=='Name'].Value|[0]]",
         "--output", "table"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr
```
Test query: `List all my EC2 instances with their state.`

**Option C — Log Searcher:**
```python
@tool
def search_logs(keyword: str, namespace: str = "default") -> str:
    """Search for a keyword in the logs of all pods in a namespace."""
    pods = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace, "-o", "name"],
        capture_output=True, text=True,
    )
    results = []
    for pod in pods.stdout.strip().split("\n"):
        if not pod:
            continue
        logs = subprocess.run(
            ["kubectl", "logs", pod, "-n", namespace, "--tail=100"],
            capture_output=True, text=True,
        )
        if keyword.lower() in logs.stdout.lower():
            results.append(f"{pod}: found '{keyword}'")
    return "\n".join(results) if results else f"No pods contain '{keyword}' in their logs"
```
Test query: `Search for 'error' in my pod logs.`

### Add the Tool to Your Agent
Open your agent file (e.g., `module-3/agent.py` or `module-6/ci_analyzer.py`) and add the new tool. Example for Log Searcher:
```python
@tool
def search_logs(keyword: str, namespace: str = "default") -> str:
    """Search for a keyword in the logs of all pods in a namespace."""
    pods = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace, "-o", "name"],
        capture_output=True, text=True,
    )
    results = []
    for pod in pods.stdout.strip().split("\n"):
        if not pod:
            continue
        logs = subprocess.run(
            ["kubectl", "logs", pod, "-n", namespace, "--tail=100"],
            capture_output=True, text=True,
        )
        if keyword.lower() in logs.stdout.lower():
            results.append(f"{pod}: found '{keyword}'")
    return "\n".join(results) if results else f"No pods contain '{keyword}' in their logs"
```
- Place it alongside your other tools.
- Make sure the **docstring is clear** — this is how the agent decides when to use it.

### Run the Agent
Start our agent again:
```bash
python3 module-3/agent.py
```

### Ask a Triggering Question
Now test the tool by asking something that matches its docstring:
- For Terraform:<br>
`What changes would terraform plan show?`

- For AWS:<br>
`List all my EC2 instances with their state.`

- For Log Searcher:<br>
`Search for 'error' in my pod logs.`

The agent will:
1. Parse our question.
2. Match it to the tool’s docstring.
3. Run the CLI command.
4. Return the output (truncated if too long).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2030).png)

### How did the agent decide when to use it?
- The agent selects the new tool when the question matches its docstring. If the tool is never called, check the docstring — it may be too vague or use terminology the LLM does not associate with the question.


## 6. Clean Up
```bash
# Delete Kind cluster
kind delete cluster --name devops-demo
kind get clusters

# Remove broken container
docker rm -f broken-container 2>/dev/null
docker ps -a

# Deactivate Python venv (if needed later)
deactivate
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b948c83fc26d161b256d7c758d6ef08c1d999f16/2026/day-88/Screenshots/Screenshot%20(2032).png)

###  Map What we Built Today:
| Module | What | Tools | Pattern |
|--------|------|-------|---------|
| 3 (agent.py) | Multi-tool agent | 3 Docker + 3 K8s | LangChain ReAct |
| 3 (mcp_server.py) | MCP server | 3 K8s tools via MCP | FastMCP |
| 3 (agent_with_mcp.py) | MCP client agent | Tools from MCP server | LangChain + MCP adapter |
| 6 (ci_analyzer.py) | CI/CD analyzer | 3 GitHub Actions tools | LangChain ReAct |

### The Core Pattern Across All Modules
 
Every agent follows the same structure regardless of domain:
 
```python
# 1. Define tools that wrap CLI commands
@tool
def my_tool(arg: str) -> str:
    """Clear docstring — the LLM reads this to decide when to call the tool."""
    result = subprocess.run(["cli", "command", arg], capture_output=True, text=True)
    return result.stdout or result.stderr
 
# 2. Create the LLM
llm = ChatOllama(model="gemma4", temperature=0)
 
# 3. Create the ReAct agent
agent = create_react_agent(llm, [my_tool, ...])
 
# 4. Ask a question — the agent reasons, selects tools, and answers
agent.invoke({"messages": [("human", "your question here")]})
```
 
Replace Docker tools with Kubernetes tools, GitHub tools, Terraform tools, or AWS CLI tools — the architecture is identical. The domain changes; the pattern stays the same.