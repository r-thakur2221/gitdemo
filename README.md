# Task 1: Security Analysis & Architecture

## 1. Deployment Summary

We deployed OpenClaw in a Docker container and connected it to a local Ollama instance on the host machine. 

* **Host:** macOS Apple Silicon (Docker Desktop with Compose v2.24+)
* **Container:** `tenfyzhong/openclaw:latest` (OpenClaw 2026.10.9)
* **LLM:** `qwen3:8b` running on the host
* **Network:** The container reaches the LLM via `http://host.docker.internal:11434`.
* **Gateway:** Published only to `127.0.0.1:18789` on the host. Access requires a token.
* **Note:** The template is built for Apple Silicon, but actual reproduction in the grading environment is still to be verified.

---

## 2. Architecture & Boundaries

![Architecture Diagram](architecture-diagram.png)

The diagram shows how a user prompt moves through the system and where the security boundaries are.

**How it flows:**
1. **User to Gateway:** The operator sends an authenticated request to the OpenClaw Gateway.
2. **Reasoning:** The Gateway passes the prompt to the Agent Runtime, which sends it to the `qwen3:8b` LLM on the host for reasoning.
3. **Tool Execution:** Based on the LLM's response, the Agent Runtime selects a tool (Shell, Filesystem, or Network) and checks the "Tool policy" before running it.

**Trust Boundaries:**
* **Boundary 1 (Container):** This separates the trusted user from the container. However, because the LLM is easily influenced by untrusted data (like web pages), the agent's reasoning inside this boundary is considered a risk.
* **Boundary 2 (Host/External):** This separates the container from the host machine and the internet. The tools act as bridges across this boundary, allowing the agent to touch host files, the LLM API, and the web.

---

## 3. Security Principle Analysis

We evaluated three principles against our deployment.

### 1. Complete Mediation
* **Operation:** Agent tool calls (Shell, Filesystem, Network).
* **Current Protection:** The Gateway authenticates the user. The diagram shows a "Tool policy" layer before tools execute.
* **Does it follow the principle?** **Partially.** User access is mediated, but the Tool Policy layer does not strictly mediate *every* tool call in the default setup (`tools.profile: "full"`). It is more of a router than a security checkpoint.
* **Concern:** If a prompt injection tricks the LLM, it can command the Agent Runtime to run a malicious tool. Because there is no strict check between the LLM's decision and execution, the attack works.

### 2. Least Privilege
* **Operation:** The permissions given to the agent and container.
* **Current Protection:** The container runs as a non-root user and uses isolated Docker volumes.
* **Does it follow the principle?** **No.** The default setup enables Shell, Filesystem, and Network tools all at once. Additionally, the `qwen3:8b` model is large and computationally heavy.
* **Concern:** Over-privilege increases the damage an attacker can do. A compromised agent can read files, run commands, and exfiltrate data. Also, a heavy prompt could cause a Denial of Service (DoS) by exhausting the host's RAM.

### 3. Fail-Safe Defaults
* **Operation:** Gateway authentication and LLM exposure.
* **Current Protection:** The Gateway requires a token to bind to the network (fail-safe). However, the Ollama API is exposed via `host.docker.internal` without any authentication.
* **Does it follow the principle?** **Partially.** The Gateway is secure by default, but the LLM endpoint is fail-open. 
* **Concern:** Because Ollama has no authentication, any compromised process inside the container can query the LLM, waste host resources, or manipulate the agent's context.

---

## 4. Evidence
Proof of the setup is documented in `../environment/`:
* **Ollama Connectivity:** `curl http://host.docker.internal:11434/api/tags` returns the `qwen3:8b` model.
* **Container Status:** `openclaw gateway status` shows `Connectivity check: ok`.
* **Model Registration:** `openclaw models list` shows `qwen3:8b` is active.

## 5. Caveats
* **Image Tag:** We used `latest`; this should be pinned to a digest for the final submission.
* **Ollama Setup:** Ollama is started manually via CLI (`OLLAMA_HOST=0.0.0.0:11434 ollama serve`) to allow the Docker container to connect.
