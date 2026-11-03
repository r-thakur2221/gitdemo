# Task 1 — Security Analysis & Architecture Report

This report covers the security-focused analysis of the OpenClaw deployment. It includes the architecture narrative based on the provided diagram, the trust-boundary analysis, and the evaluation of three fundamental security principles.

## 1. Deployment Summary

| Item | Value |
|---|---|
| Host | Ubuntu VM on macOS (Apple Silicon), 8 GB RAM |
| Container image | `tenfyzhong/openclaw:latest` (OpenClaw 2026.10.9) |
| LLM | Ollama on the VM host, model `llama3.2:3b` (Q4_K_M, tools-capable) |
| Gateway bind | `lan` (0.0.0.0) inside container, published to `127.0.0.1:18789` only |
| Dashboard access | SSH tunnel from Mac to VM loopback, tokenized URL |

---

## 2. Architecture & Trust Boundaries

![Architecture Diagram](architecture-diagram.png)

The diagram above illustrates the flow of a user prompt through the OpenClaw system and highlights the critical security boundaries.

### 2.1 Components and Flow
1. **Ingress:** A Trusted Operator sends an authenticated request via the Web UI/CLI.
2. **Gateway:** The OpenClaw Gateway authenticates the token and routes the request to the Agent Runtime.
3. **Reasoning (LLM):** The Agent Runtime constructs the prompt context and sends it to the external Ollama LLM (port 11434) for reasoning and tool selection.
4. **Execution:** The Agent Runtime passes the planned action through the **Tool policy / Approvals** layer before executing Shell, Filesystem, or Network tools.

### 2.2 Trust Boundaries
* **Boundary 1 (Docker Container):** Encloses the Gateway, Agent Runtime, Tools, and Policy layer. While the operator is trusted, the *LLM reasoning* is probabilistic. If the LLM is manipulated by untrusted data, the Agent Runtime becomes an untrusted actor inside this boundary.
* **Boundary 2 (Host / External Resources):** Separates the container from the host machine and the internet. The tools (Shell, Filesystem, Network) act as bridges crossing this boundary, giving the agent access to host files, the unauthenticated Ollama service, and external web services.

---

## 3. Security Principle Analysis

### 3.1 Complete Mediation
* **Security-sensitive operation:** Every tool call (Shell/Exec, Filesystem, Network) initiated by the agent.
* **Protection that currently exists:** The Gateway completely mediates *client* connections via token authentication. The diagram shows a "Tool policy / Approvals" layer between the Agent Runtime and the tools.
* **Does the implementation follow the principle?** **Partially.** While the Gateway mediates user access, the "Tool policy / Approvals" layer in the default unhardened deployment (`tools.profile: "full"`) does not enforce mandatory per-call authorization. It acts as a routing mechanism rather than a strict security checkpoint. 
* **Potential security concern:** If a prompt injection tricks the LLM, the Agent Runtime will instruct the policy layer to execute a malicious tool. Because there is no strict mediation between the LLM's decision and the execution sink, the attack succeeds.

### 3.2 Least Privilege
* **Security-sensitive operation:** The capabilities granted to the containerized agent and its tools.
* **Protection that currently exists:** The container runs as the non-root `node` user (uid 1000) and uses isolated Docker named volumes rather than broad host bind mounts.
* **Does the implementation follow the principle?** **No.** The default onboarding sets `tools.profile: "full"`. As shown in the diagram, this enables Shell, Filesystem, and Network tools simultaneously. The agent has far more privileges than required for a single benign task.
* **Potential security concern:** Over-privilege amplifies the blast radius. A compromised agent can read/write files, execute system commands, and exfiltrate data over the network, rather than being restricted to a safe subset of tools.

### 3.3 Fail-Safe Defaults
* **Security-sensitive operation:** Gateway binding, authentication, and the exposure of the LLM endpoint.
* **Protection that currently exists:** OpenClaw refuses to bind the gateway to a non-loopback address without authentication. The compose file sets `OPENCLAW_GATEWAY_TOKEN` (fail-safe). However, the Ollama endpoint (Boundary 2) was configured with `OLLAMA_HOST=0.0.0.0:11434`, exposing it to all interfaces without authentication.
* **Does the implementation follow the principle?** **Partially.** The Gateway follows fail-safe defaults, but the Ollama configuration is fail-open. 
* **Potential security concern:** The unauthenticated Ollama API is exposed to the host network. Any process on the VM (or LAN, if not firewalled) can query the model, consume resources, or potentially manipulate the agent's reasoning context.

---

## 4. Evidence
The analysis is grounded in the running deployment. Evidence screenshots are located in `../environment/screenshots/`:
* `01-ollama-listen.png` (shows `*:11434`)
* `02-ollama-curl.png` (shows successful connection from container)
* `03-models-list.png` (shows `llama3.2:3b` registered)
* `04-gateway-status.png` (shows `Connectivity check: ok`)
* `05-compose-ps.png` (shows container Up)
* `06-dashboard.png` (shows UI access)

## 5. Known Caveats
* **Image Tag:** The deployment used `tenfyzhong/openclaw:latest`; this should be pinned to a digest for final submission.
* **Ollama Lifecycle:** Managed manually via CLI (`OLLAMA_HOST=0.0.0.0:11434 ollama serve`) due to systemd incompatibility with the Linuxbrew installation.
