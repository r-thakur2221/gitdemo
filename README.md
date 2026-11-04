# Task 1: Security Analysis & Architecture

## 1. Deployment Setup
We deployed OpenClaw in a Docker container connected to a local LLM.
* **Host:** macOS Apple Silicon (Docker Desktop/Compose v2.24+)
* **Container:** OpenClaw (`tenfyzhong/openclaw:latest`)
* **LLM:** `qwen3:8b` running via Ollama on the host.
* **Network:** The container accesses the LLM via `http://host.docker.internal:11434/api/tags`.

## 2. Architecture & Boundaries
![Architecture Diagram](architecture-diagram.png)

**Flow of Operations:**
1. **User:** Authenticates via the Web UI/CLI to reach the Gateway.
2. **Gateway:** Routes the request to the Agent Runtime.
3. **Agent Runtime:** Sends the prompt to the external LLM (Ollama) for reasoning, then selects a tool based on the response.
4. **Tools:** The request passes through the "Tool policy" before executing on the Shell, Filesystem, or Network.

**Trust Boundaries:**
* **Boundary 1 (Container):** The Agent Runtime is untrusted because LLM reasoning is probabilistic and can be manipulated by prompt injection.
* **Boundary 2 (Host/External):** The tools cross this boundary to access host files, the unauthenticated LLM API, and the internet.

## 3. Security Principle Analysis

### 1. Complete Mediation
* **Operation:** Agent tool calls (Shell, Filesystem, Network).
* **Current Protection:** The Gateway authenticates the user. A "Tool policy" layer exists before tools execute.
* **Follows Principle?** **Partially.** The Gateway mediates user access, but the Tool Policy layer does not strictly enforce authorization for *every* tool call (default is `tools.profile: "full"`).
* **Concern:** A prompt injection can trick the LLM into executing a tool without a strict security checkpoint between the LLM's decision and the execution.

### 2. Least Privilege
* **Operation:** Permissions granted to the agent and container.
* **Current Protection:** The container runs as a non-root user and uses Docker volumes.
* **Follows Principle?** **No.** The default setup enables Shell, Filesystem, and Network tools all at once. The `qwen3:8b` model also uses significant host resources.
* **Concern:** If compromised, the agent has full access to read files, run commands, and exfiltrate data. Heavy inference requests could also cause a Denial of Service (DoS) on the host's RAM.

### 3. Fail-Safe Defaults
* **Operation:** Gateway binding and LLM exposure.
* **Current Protection:** The Gateway requires a token to bind to the network (fail-safe). However, the Ollama API is exposed without authentication.
* **Follows Principle?** **Partially.** The Gateway is secure by default, but the LLM endpoint is fail-open.
* **Concern:** Any process inside the container (or a compromised container) can freely query the unauthenticated LLM, wasting host resources or manipulating the agent's context.

## 4. Evidence
* **Connectivity:** `curl http://host.docker.internal:11434/api/tags` returns `qwen3:8b`.
* **Status:** `openclaw gateway status` shows `Connectivity check: ok`.
