# Introduction to Agentic AI for DevOps
>We have built the entire DevOps pipeline -- Linux, Docker, CI/CD, Kubernetes, Terraform, Ansible, observability, Helm, EKS, and GitOps. Now we add AI to it. Agentic AI is not about chatbots. It is about building autonomous agents that can inspect our infrastructure, diagnose problems, and fix them using the same CLI tools we use every day (docker, kubectl, gh).

Today we set up our environment, make our first LLM call, and build our first AI agent that troubleshoots Docker containers on its own.

Reference: https://github.com/TrainWithShubham/agentic-ai-for-devops

## Table of Contents
1. [Understanding Agentic AI for DevOps](#1-understand-agentic-ai-for-devops)
2. [Set Up the Environment](#2-set-up-the-environment)
3. [Build the Docker Error Explainer](#3-build-the-docker-error-explainer-module-1)
4. [Build the Docker Troubleshooter Agent](#4-build-the-docker-troubleshooter-agent-module-2)
5. [Understanding the Agent Architecture](#5-understand-the-agent-architecture)
6. [Experiment and Extend](#6-experiment-and-extend)

## 1. Understand Agentic AI for DevOps
### What is an AI Agent?
An AI agent is an LLM that can use **tools** to interact with the real world. Unlike a chatbot that only generates text, an agent can run commands, read files, and call APIs. The LLM decides which tool to use and with what arguments, based on the task at hand.
 
### Why Agents for DevOps?
DevOps is tool-heavy: `docker`, `kubectl`, `terraform`, `gh`, `ansible` ,  all CLI-based, all producing structured text output that an LLM can read and reason about. An agent wraps these CLIs as tools and lets the LLM reason over their output.
 
Example flow:
```
User: "Why is my pod crashing?"
 
Agent calls:  kubectl get pods
Agent sees:   broken-app  CrashLoopBackOff
 
Agent calls:  kubectl describe pod broken-app
Agent sees:   OOMKilled (exit code 137)
 
Agent answers: "The pod is being killed by the OOM killer —
                it is exceeding its memory limit. Increase
                resources.limits.memory in your Deployment."
```
 
### The ReAct Pattern (Reason + Act)
The reasoning loop that drives all modern agents:
```
User: "Why is broken-app crashing?"
 
THINK:   I should check which containers are running
ACT:     list_containers()
OBSERVE: broken-app is in Restarting state
 
THINK:   I should check the logs to see what happens before it crashes
ACT:     get_logs("broken-app")
OBSERVE: "app starting..." → exit code 1 after 2 seconds
 
THINK:   The container exits immediately — I have enough to explain this
ANSWER:  "The container crashes because its entrypoint command
          exits with code 1 after approximately 2 seconds..."
```
The LLM decides which tools to call and in what order. We never prescribe the investigation steps.
 
### Key Components
| Component | Role | Technology |
|---|---|---|
| **LLM** | The brain - reasoning and decision-making | Ollama + Gemma 4 (local), or Claude / GPT for production |
| **Tools** | The hands - Python functions wrapping CLI commands | `subprocess.run(["docker", ...])` |
| **Agent framework** | Orchestrates the ReAct reasoning loop | LangChain's `create_react_agent` |
| **MCP** | Standard protocol for exposing tools to any AI client | FastMCP (covered on Day 88) |

## 2. Set Up the Environment
### Clone the Reference Repository
This repo contains all the modules we’ll need.
```bash
sudo apt update && sudo apt upgrade -y
git clone https://github.com/TrainWithShubham/agentic-ai-for-devops.git
cd agentic-ai-for-devops
```

### Install Ollama (Local LLM Runtime)
Ollama runs LLMs locally - no API keys, no cloud cost, no data leaving our machine.
```bash
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh

# Verify
ollama --version
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1789).png)

### Start Ollama and Pull the Gemma 4 Model
```bash
# Starts Ollama in the background
ollama serve &

# Downloads the Gemma 4 model (~5GB RAM needed)
ollama pull gemma4
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1791).png)

Verify:
```bash
ollama list
# Should show gemma4 in the list
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1794).png)

### Set Up Python Environment
Create a virtual environment so dependencies don’t pollute our system Python.
```bash
python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt

# Verify the venv is active
which python
# Expected: .../agentic-ai-for-devops/.venv/bin/python
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1796).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1797).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1799).png)

### Understand Installed Dependencies
The `requirements.txt` installs:
| Package | Purpose |
|---|---|
| `ollama` | Python client for the Ollama API |
| `langchain` + `langchain-ollama` | Agent framework with Ollama integration |
| `langgraph` | Graph-based agent execution engine used by `create_react_agent` |
| `fastmcp` | Model Context Protocol server framework |
| `langchain-mcp-adapters` | Bridges MCP tools into LangChain agents |

### Run Pre-flight Check
This script ensures our environment is ready.
```bash
python3 module-0/verify_setup.py
```
We should see:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1801).png)

**If any FAIL**: Fix before proceeding (e.g., install Docker, configure kubectl, ensure Kind cluster works, restart Ollama).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1804).png)

## 3. Build the Docker Error Explainer (Module 1)
This is the simplest possible LLM usage - no agents, no tools, no reasoning loop. We paste a Docker error and the LLM explains it in plain English.

### Open the Explainer Script
Navigate into the repo:
```bash
cd module-1
cat explainer.py
```
We’ll see the code that defines the **system prompt** and makes a call to Ollama.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1806).png)

### Understand the System Prompt
The script sets:
```python
SYSTEM_PROMPT = """You are a Docker expert. When given a Docker error, explain:
1. What went wrong (plain English)
2. Most likely cause
3. How to fix it (with commands)
Keep it short."""
```
- **System prompt** = tells the LLM what persona to adopt and how to format responses.
- Here, it makes the model act like a **Docker expert**.

### Understand the LLM Call
The script runs:
```python
response = ollama.chat(
    model="gemma4",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": error},
    ],
    options={"temperature": 0.3},
)
```
- **Model**: Gemma 4 (local via Ollama).
- **Messages**: System prompt + user error.
- `Temperature: 0.3` → keeps responses deterministic and technical - higher values produce more creative but less reliable answers. For diagnostic tooling, keep it low.
- No tools, no agent loop -- just a single LLM call

### Run the Explainer
Execute:
```bash
python3 module-1/explainer.py
```
Paste one of these errors when prompted:
```
docker: Error response from daemon: Conflict. The container name "/myapp" is already in use.
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1815).png)

Or:
```
Error response from daemon: driver failed programming external connectivity on endpoint myapp:
Bind for 0.0.0.0:8080 failed: port is already allocated.
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1819).png)

Or:
```
Error response from daemon: pull access denied for mycompany/private-app, repository does not
exist or may require 'docker login'.
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1821).png)

The LLM explains the root cause and provides the exact commands to fix it -- no Googling required.

### How does the system prompt affect the quality of the response? Try changing it and see what happens.
**Ans.** The system prompt defines the LLM's persona and output format. Changing it dramatically changes the quality and style of responses - try replacing "Docker expert" with "site reliability engineer" or removing the numbered format to see the difference.

## 4. Build the Docker Troubleshooter Agent (Module 2)
Now the real thing - an agent that autonomously uses tools to diagnose Docker issues without being told which commands to run.

### Create a Broken Container
We need something for the agent to diagnose.
```bash
docker run -d --name broken-app nginx:alpine \
  sh -c "echo 'app starting...' && sleep 2 && exit 1"

# Verify it is in Restarting state
docker ps -a | grep broken-app
```
- This container starts, prints “app starting…”, waits 2 seconds, then exits with code 1.
- Docker will keep restarting it → similar to **CrashLoopBackOff** in Kubernetes.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1822).png)

### Study the Agent Tools
Open `module-2/agent.py`. Three tools are defined:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1825).png)

```python
@tool
def list_containers() -> str:
    """List all Docker containers (running and stopped)."""
    result = subprocess.run(["docker", "ps", "-a"], capture_output=True, text=True)
    return result.stdout or result.stderr

@tool
def get_logs(container_name: str) -> str:
    """Get the last 50 lines of logs from a Docker container."""
    result = subprocess.run(
        ["docker", "logs", "--tail", "50", container_name],
        capture_output=True, text=True,
    )
    return result.stdout + result.stderr

@tool
def inspect_container(container_name: str) -> str:
    """Get detailed info about a Docker container (state, config, network)."""
    result = subprocess.run(
        ["docker", "inspect", container_name],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr
```
- **list_containers** → wraps `docker ps -a`
- **get_logs** → wraps `docker logs`
- **inspect_container** → wraps `docker inspect`

### How each element works:
| Element | Purpose |
|---|---|
| `@tool` decorator | Registers the function with LangChain as an available tool |
| Docstring | **Critical** - the LLM reads this to decide when to call the tool. Poor docstrings lead to incorrect tool selection |
| `subprocess.run` | Executes the actual CLI command in a subprocess |
| Return value | stdout/stderr as a string - what the LLM reads in the Observe step |

### Create the Agent
The agent is defined like this:
```python
llm = ChatOllama(model="gemma4", temperature=0)
tools = [list_containers, get_logs, inspect_container]
agent = create_react_agent(llm, tools)
```
- `temperature=0` is appropriate here - we want reproducible, deterministic diagnostic reasoning, not creative variation.
- `create_react_agent` builds the ReAct loop: the LLM reasons about the problem, picks a tool, calls it, reads the result, and repeats until it has an answer.

### Run the Agent
Execute:
```bash
python3 module-2/agent.py
```
Ask it:
```
> Why is broken-app crashing?
```
Watch the agent's reasoning:
1. It calls `list_containers()` → sees `broken-app` in "Restarting" state
2. It calls `get_logs("broken-app")` → sees "app starting..." then exit
3. It calls `inspect_container("broken-app")` → sees exit code 1
4. It answers: "The container crashes because the command exits with code 1..."

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1831).png)

**The LLM decided which tools to call and in what order.** We never told it to check logs - it reasoned that logs are the right next step after confirming the restart state.

### Try More Questions
Experiment with queries:
```
> List all my running containers
> What image is broken-app using?
> Is any container using port 8080?
```
The agent will autonomously pick the right tool and respond.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1837).png)

### Clean Up
When finished:
```bash
docker rm -f broken-app
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1840).png)

## 5. Understand the Agent Architecture

```
[User Question]
       │
       ▼
[LLM: Gemma 4 via Ollama]
       │
       │  THINK: which tool to use?
       ▼
[Tool Selection]
       ├──► list_containers()    →  docker ps -a
       ├──► get_logs()           →  docker logs --tail 50
       └──► inspect_container()  →  docker inspect
       │
       │  OBSERVE: read tool output
       ▼
[LLM reasons over output]
       │
       │  (repeat Think → Act → Observe until ready)
       ▼
[Final Answer to User]
```
 
### Why This Pattern Matters for DevOps
**Ans.** The architecture is domain-agnostic. Replace Docker tools with Kubernetes tools (`kubectl get pods`, `kubectl describe`, `kubectl logs`) and the same ReAct loop diagnoses Kubernetes failures. Replace them with Terraform or AWS CLI tools and it analyses infrastructure state. The reasoning engine stays the same - only the tools change.
 
On Day 88, we will add Kubernetes tools to this same agent. On Day 89, we will add production guardrails and build an agent that automatically fixes broken pods.
 
### The Tool Template
Every agent tool follows the same structure:
```python
@tool
def my_tool(argument: str) -> str:
    """Description the LLM reads to decide when to call this tool.
    Be specific — vague docstrings lead to incorrect tool selection."""
    result = subprocess.run(
        ["some-cli", "command", argument],
        capture_output=True, text=True
    )
    return result.stdout or result.stderr
```
Any CLI command can become an agent tool. Any DevOps workflow can be automated this way.

## 6. Experiment and Extend
Try adding a new tool to the agent.

### Add `list_images` tool
Create a tool to list Docker images and their sizes.
- Edit `module-2/agent.py` and Insert the function:
    ```python
    @tool
    def list_images() -> str:
        """List all Docker images on this machine with their sizes."""
        result = subprocess.run(["docker", "images"], capture_output=True, text=True)
        return result.stdout or result.stderr
    ```

### Update tools list
Include the new tool in the agent’s toolset.
- Modify tools array:
    ```python
    tools = [list_containers, get_logs, inspect_container, list_images]
    ```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1842).png)

### Run agent with new tool
Test the agent’s ability to call the new tool.
- Execute:
    ```bash
    python3 module-2/agent.py
    ```
- Ask:
    ```code
    “What images do I have and how much space are they using?”
    ```
- Observe agent calling `list_images()`, our new tool.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1844).png)

### Add `restart_container` tool
Create a tool to restart containers, but note safety implications.
- Insert function:
    ```python
    @tool
    def restart_container(container_name: str) -> str:
        """Restart a Docker container."""
        result = subprocess.run(["docker", "restart", container_name], capture_output=True, text=True)
        return result.stdout or result.stderr
    ```
- Add to tools list:
    ```python
    tools = [list_containers, get_logs, inspect_container, list_images, restart_container]
    ```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1847).png)

### Test restart functionality
Verify the agent can restart a container.
- Run agent again
- Ask: “broken-app keeps crashing, can you restart it?”
- Observe agent calling `restart_container()`

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1852).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ce84fdce898a79384f270f6aa0739ee8d8673d17/2026/day-87/Screenshots/Screenshot%20(1872).png)

### Safety Considerations
The `restart_container` tool can restart any container on the system, including system-critical ones. In production, we need explicit guardrails:
| Guardrail | Implementation |
|---|---|
| **Allowlist** | Only restart containers matching a known prefix or label |
| **Confirmation prompt** | Require human approval before executing write actions |
| **Dry-run mode** | Log what would happen without executing |
| **Audit logging** | Record every tool call with timestamp and arguments |
 
Giving an agent write access to any system requires explicit safety boundaries - the same principle as IAM least privilege, applied to AI tool use. Day 89 covers production-grade agent guardrails in depth.