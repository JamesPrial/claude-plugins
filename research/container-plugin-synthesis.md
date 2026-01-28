# Container Sandboxing + Claude Code Plugins: Synthesis Research

## Executive Summary

This document explores how Unix namespaces, Docker containers, and MicroVM technologies can be integrated with Claude Code's plugin architecture to create secure, isolated execution environments for AI agents. The goal is to enable plugins to run arbitrary code (tests, builds, deployments) while containing blast radius and preventing credential exfiltration.

---

## Part 1: Container & Sandbox Technologies

### 1.1 Linux Kernel Isolation Primitives

Linux provides three fundamental mechanisms for isolation:

| Mechanism | Purpose | What It Isolates |
|-----------|---------|------------------|
| **Namespaces** | Virtualize system resources | PIDs, network, mounts, users, IPC, UTS, cgroups |
| **Cgroups** | Resource limits | CPU, memory, I/O, device access |
| **Seccomp-BPF** | Syscall filtering | Dangerous system calls |

**Key Namespaces for AI Agent Sandboxing:**

```
┌─────────────────────────────────────────────────────────────┐
│  PID Namespace     - Agent sees only its own processes      │
│  Mount Namespace   - Custom filesystem view, no host access │
│  Network Namespace - Isolated network stack, proxy-only     │
│  User Namespace    - Unprivileged container root            │
│  IPC Namespace     - No shared memory with host             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Bubblewrap: Lightweight Userspace Sandboxing

[Bubblewrap](https://github.com/containers/bubblewrap) provides unprivileged sandboxing using namespaces:

```bash
# Example: Sandbox with read-only /usr, isolated PID/network
bwrap \
  --ro-bind /usr /usr \
  --ro-bind /lib /lib \
  --ro-bind /lib64 /lib64 \
  --tmpfs /tmp \
  --proc /proc \
  --dev /dev \
  --unshare-pid \
  --unshare-net \
  --unshare-ipc \
  --new-session \
  --die-with-parent \
  /path/to/agent-script
```

**Advantages:**
- No root required (uses user namespaces)
- Minimal overhead (~1ms startup)
- Fine-grained mount control
- Used by Flatpak for desktop app sandboxing

**Limitations:**
- Shared kernel (container escape via kernel exploits possible)
- Complex to configure correctly
- No built-in network proxying

### 1.3 Docker Containers

Standard Docker provides process-level isolation:

```dockerfile
FROM python:3.12-slim
RUN useradd -m agent
USER agent
WORKDIR /workspace
# Agent runs as unprivileged user in isolated container
```

**Security Hardening Options:**
- `--security-opt=no-new-privileges` - Prevent privilege escalation
- `--cap-drop=ALL` - Drop all Linux capabilities
- `--read-only` - Read-only root filesystem
- `--network=none` or custom network - Network isolation
- `--pids-limit=100` - Prevent fork bombs

### 1.4 Docker Sandboxes (MicroVMs)

[Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) use lightweight VMs for stronger isolation:

```bash
# Launch Claude Code in isolated microVM
docker sandbox run claude ~/my-project
```

**Architecture:**
```
┌─────────────────────────────────────────┐
│           Host System                   │
│  ┌───────────────────────────────────┐  │
│  │      MicroVM (separate kernel)    │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  Private Docker Daemon      │  │  │
│  │  │  ┌───────────────────────┐  │  │  │
│  │  │  │   AI Agent Process    │  │  │  │
│  │  │  │   (Claude Code)       │  │  │  │
│  │  │  └───────────────────────┘  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Key Benefits:**
- Separate kernel = no kernel exploit escape
- Private Docker daemon inside VM
- Workspace sync at same absolute paths
- Agent can spin up test containers freely

### 1.5 Isolation Strength Comparison

```
Weakest ──────────────────────────────────────────► Strongest

  Process     Namespace    Container    gVisor      MicroVM
  Isolation   (unshare)    (Docker)     (sandbox)   (Firecracker)
     │            │            │            │            │
  Shared      Shared       Shared      User-space   Separate
  everything  kernel       kernel      kernel       kernel
              + namespace  + cgroups   intercept    + hypervisor
```

---

## Part 2: Claude Code Plugin Architecture

### 2.1 Plugin Types

The Claude Code plugin system supports three distinct types:

| Type | Purpose | Execution Model |
|------|---------|-----------------|
| **Hook Plugins** | Intercept tool calls | Synchronous, can block |
| **Agent Plugins** | Autonomous task execution | Subprocess with tool access |
| **Skill Plugins** | Knowledge & helpers | Reference docs + scripts |

### 2.2 Hook Plugin Architecture

Hooks intercept Claude Code tool calls at two points:

```
User Request
     │
     ▼
┌─────────────┐
│ PreToolUse  │◄── Hook can BLOCK (exit 2) or ALLOW (exit 0)
└─────────────┘
     │
     ▼
┌─────────────┐
│ Tool Exec   │    (Bash, Edit, Write, etc.)
└─────────────┘
     │
     ▼
┌─────────────┐
│ PostToolUse │◄── Hook for logging/auditing (never blocks)
└─────────────┘
```

**Hook Configuration (hooks.json):**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/scripts/sandbox_exec.py",
        "timeout": 300
      }]
    }]
  }
}
```

**Hook Script Protocol:**
- Input: JSON via stdin with `tool_name`, `tool_input`, `session_id`
- Output: JSON via stdout with modified parameters or block decision
- Exit 0 = allow, Exit 2 = block

### 2.3 Agent Plugin Architecture

Agents are subprocesses launched via the Task tool:

```markdown
---
name: sandboxed-builder
description: "Build and test code in isolated container"
tools: Bash, Read, Write
model: sonnet
---

Execute builds in isolated Docker containers...
```

**Agent Execution Flow:**
```
Claude Code Main Process
         │
         ▼
    Task Tool
         │
         ▼
┌─────────────────────┐
│  Agent Subprocess   │
│  - Has tool access  │
│  - Runs commands    │
│  - Returns results  │
└─────────────────────┘
```

---

## Part 3: Synthesis - Container-Isolated Plugin Execution

### 3.1 Architecture Options

#### Option A: Hook-Based Container Interception

Use a PreToolUse hook to redirect Bash commands into containers:

```
┌──────────────────────────────────────────────────────────────┐
│                    Claude Code                                │
│  ┌────────────┐    ┌─────────────────────┐    ┌───────────┐  │
│  │ User Req   │───►│ PreToolUse Hook     │───►│ Container │  │
│  │ "run tests"│    │ (sandbox_exec.py)   │    │ Executor  │  │
│  └────────────┘    └─────────────────────┘    └───────────┘  │
│                              │                       │        │
│                              ▼                       ▼        │
│                    ┌─────────────────────────────────────┐   │
│                    │  Docker Container / MicroVM          │   │
│                    │  ┌─────────────────────────────────┐ │   │
│                    │  │  pytest /workspace/tests        │ │   │
│                    │  └─────────────────────────────────┘ │   │
│                    └─────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**Implementation (sandbox_exec.py):**
```python
#!/usr/bin/env python3
"""PreToolUse hook that redirects commands to containers."""
import json
import subprocess
import sys
import os

def main():
    hook_input = json.load(sys.stdin)

    if hook_input.get("tool_name") != "Bash":
        return 0  # Pass through non-Bash tools

    command = hook_input["tool_input"].get("command", "")
    project_dir = os.environ.get("CLAUDE_PROJECT_DIR", "/workspace")

    # Commands that should run in sandbox
    sandbox_patterns = ["pytest", "npm test", "go test", "make", "cargo"]

    if any(p in command for p in sandbox_patterns):
        # Redirect to Docker container
        docker_cmd = [
            "docker", "run", "--rm",
            "-v", f"{project_dir}:/workspace:rw",
            "-w", "/workspace",
            "--network=none",  # No network access
            "--cap-drop=ALL",
            "--security-opt=no-new-privileges",
            "agent-sandbox:latest",
            "sh", "-c", command
        ]
        result = subprocess.run(docker_cmd, capture_output=True, text=True)

        # Return result without executing original command
        output = {
            "hookSpecificOutput": {
                "sandboxed": True,
                "stdout": result.stdout,
                "stderr": result.stderr,
                "returncode": result.returncode
            },
            "decision": "block"  # Block original, we handled it
        }
        json.dump(output, sys.stdout)
        return 0

    return 0  # Allow non-sandboxed commands

if __name__ == "__main__":
    sys.exit(main())
```

#### Option B: Sandboxed Agent Plugin

Create a dedicated agent that runs entirely within a container:

```markdown
---
name: container-executor
description: "Execute builds, tests, and scripts in isolated containers"
tools: Bash, Read, Write
model: haiku
---

You are a sandboxed execution agent. All commands you run execute
inside an isolated Docker container with:
- No network access (--network=none)
- Read-only root filesystem
- Dropped capabilities
- Resource limits (1 CPU, 2GB RAM)

When given a task:
1. Start the container with workspace mounted
2. Execute the requested commands
3. Capture and return output
4. Clean up the container
```

**Orchestration Flow:**
```
User: "Run the test suite"
         │
         ▼
   Main Claude Code
         │
         ▼
┌─────────────────────────┐
│ container-executor      │
│ (agent subprocess)      │
│                         │
│ 1. docker run ...       │◄─── Runs in isolated container
│ 2. Capture output       │
│ 3. Return summary       │
└─────────────────────────┘
```

#### Option C: MCP Sandbox Server

Use Model Context Protocol for container management:

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │              MCP Client                          │    │
│  └──────────────────────┬──────────────────────────┘    │
└─────────────────────────┼───────────────────────────────┘
                          │ JSON-RPC
                          ▼
┌─────────────────────────────────────────────────────────┐
│              MCP Sandbox Server                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Tools:                                           │    │
│  │  - sandbox_create(image, workspace)              │    │
│  │  - sandbox_exec(container_id, command)           │    │
│  │  - sandbox_copy_file(container_id, path, content)│    │
│  │  - sandbox_destroy(container_id)                 │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         │                                │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │           Docker / Podman / MicroVM              │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐ │    │
│  │  │ Container1 │  │ Container2 │  │ Container3 │ │    │
│  │  └────────────┘  └────────────┘  └────────────┘ │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Security Architecture

#### Network Isolation Strategy

```
┌─────────────────────────────────────────────────────────────┐
│  Host Network                                                │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Container (--network=none)                             │ │
│  │                                                          │ │
│  │  ┌──────────────────┐     ┌────────────────────────┐   │ │
│  │  │  Agent Process   │────►│  Unix Domain Socket    │   │ │
│  │  └──────────────────┘     │  (HTTP/SOCKS5 Proxy)   │   │ │
│  │                           └───────────┬────────────┘   │ │
│  └───────────────────────────────────────┼────────────────┘ │
│                                          │                   │
│  ┌───────────────────────────────────────▼────────────────┐ │
│  │  Proxy Process (outside container)                      │ │
│  │  - Whitelist: npm, pypi, github.com, api.anthropic.com │ │
│  │  - Log all requests                                     │ │
│  │  - Block exfiltration attempts                          │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Whitelisted Domains (Example):**
```python
ALLOWED_DOMAINS = [
    "registry.npmjs.org",
    "pypi.org", "files.pythonhosted.org",
    "proxy.golang.org",
    "github.com", "api.github.com",
    "api.anthropic.com",
]
```

#### Filesystem Isolation

```python
# Mount configuration for sandboxed execution
MOUNT_CONFIG = {
    # Workspace: read-write access to project only
    f"{project_dir}": "/workspace:rw",

    # System: read-only, minimal
    "/usr/bin": "/usr/bin:ro",
    "/usr/lib": "/usr/lib:ro",

    # Blocked: no access to sensitive paths
    # ~/.ssh, ~/.aws, ~/.config, /etc/passwd, etc.
    # (not mounted = not accessible)

    # Temp: container-local tmpfs
    "/tmp": "tmpfs",
}
```

### 3.3 Implementation: sandbox-executor Plugin

**Directory Structure:**
```
sandbox-executor/
├── .claude-plugin/
│   └── plugin.json
├── hooks/
│   └── hooks.json
├── agents/
│   └── container-executor.md
├── scripts/
│   ├── sandbox_manager.py     # Container lifecycle
│   ├── proxy_server.py        # Network proxy
│   └── hook_intercept.py      # PreToolUse hook
├── images/
│   ├── Dockerfile.python      # Python sandbox image
│   ├── Dockerfile.node        # Node.js sandbox image
│   └── Dockerfile.go          # Go sandbox image
└── CLAUDE.md
```

**plugin.json:**
```json
{
  "name": "sandbox-executor",
  "version": "1.0.0",
  "description": "Execute code in isolated containers with network/filesystem restrictions",
  "author": {"name": "James Prial"},
  "license": "MIT",
  "keywords": ["sandbox", "container", "security", "isolation"]
}
```

**hooks.json:**
```json
{
  "description": "Intercept risky commands and run them in containers",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/hook_intercept.py",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

### 3.4 Usage Patterns

#### Pattern 1: Automatic Test Sandboxing

```bash
# User types: "Run the tests"
# Hook intercepts `pytest` command
# Executes in container with:
#   - Project mounted read-write
#   - No network access
#   - 2GB RAM limit
#   - 5 minute timeout
```

#### Pattern 2: Build Isolation

```bash
# User types: "Build the Docker image"
# Agent spins up Docker-in-Docker container
# Build happens inside nested container
# Output image available in workspace
```

#### Pattern 3: Untrusted Code Review

```bash
# User types: "Clone and analyze this repo for security issues"
# Clone happens in isolated container
# Static analysis runs sandboxed
# Results returned without executing untrusted code
```

---

## Part 4: Advanced Considerations

### 4.1 Credential Management

**Problem:** Agents need some credentials (git push, npm publish) but shouldn't access others (~/.aws).

**Solution: Scoped Credential Injection:**
```python
# Only inject credentials needed for current task
if task_type == "npm_publish":
    env_vars = {"NPM_TOKEN": get_secret("npm_token")}
elif task_type == "git_push":
    env_vars = {"GIT_TOKEN": get_secret("git_token")}
else:
    env_vars = {}  # No credentials

docker_run(..., environment=env_vars)
```

### 4.2 Stateful Containers

**Problem:** Some workflows need persistent state (node_modules, venv).

**Solution: Named Volumes:**
```python
# Cache dependencies between runs
volumes = {
    f"{project_dir}": "/workspace:rw",
    "node_modules_cache": "/workspace/node_modules:rw",
    "pip_cache": "/root/.cache/pip:rw",
}
```

### 4.3 GPU Access for ML Workloads

```python
# Controlled GPU passthrough for ML tasks
if task_requires_gpu:
    docker_run(
        ...,
        device_requests=[
            docker.types.DeviceRequest(
                count=1,
                capabilities=[["gpu"]]
            )
        ]
    )
```

### 4.4 Monitoring & Audit

```python
# Log all sandboxed executions
audit_log = {
    "timestamp": datetime.utcnow().isoformat(),
    "session_id": session_id,
    "command": command,
    "container_id": container.id,
    "exit_code": result.returncode,
    "stdout_hash": hashlib.sha256(result.stdout.encode()).hexdigest(),
    "network_requests": proxy.get_request_log(),
}
```

---

## Part 5: Comparison with Existing Solutions

| Feature | DevContainer | Docker Sandbox | This Plugin |
|---------|--------------|----------------|-------------|
| Isolation level | Container | MicroVM | Container/MicroVM |
| Network control | Whitelist | Full isolation | Proxy + whitelist |
| Integration | Manual setup | CLI tool | Automatic via hooks |
| Credential scoping | All or nothing | Manual | Per-task injection |
| Multi-language | Via Dockerfile | Via images | Preconfigured images |
| Audit logging | None | None | Built-in |

---

## Part 6: Recommended Implementation Path

### Phase 1: Hook-Based Interception (MVP)
1. Create `sandbox-executor` plugin with PreToolUse hook
2. Intercept `pytest`, `npm test`, `make` commands
3. Execute in Docker container with `--network=none`
4. Return output to Claude Code

### Phase 2: Network Proxy
1. Implement SOCKS5/HTTP proxy outside container
2. Mount proxy socket into container
3. Whitelist package registries
4. Log all network requests

### Phase 3: Agent Integration
1. Create `container-executor` agent definition
2. Support explicit sandboxed execution requests
3. Add container lifecycle management (create, exec, destroy)

### Phase 4: MCP Server (Optional)
1. Implement MCP sandbox server
2. Support multiple concurrent containers
3. Add resource quotas and monitoring

---

## Sources

- [Docker Sandboxes Documentation](https://docs.docker.com/ai/sandboxes/)
- [Bubblewrap - Low-level Sandboxing](https://github.com/containers/bubblewrap)
- [Agent Sandbox for Kubernetes](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox)
- [AIO Sandbox - All-in-One Agent Sandbox](https://github.com/agent-infra/sandbox)
- [Code Sandbox MCP Server](https://github.com/Automata-Labs-team/code-sandbox-mcp)
- [Claude Code DevContainer Documentation](https://code.claude.com/docs/en/devcontainer)
- [DevContainers for AI Agent Isolation](https://codewithandrea.com/articles/run-ai-agents-inside-devcontainer/)
- [Best Code Execution Sandbox for AI Agents 2026](https://northflank.com/blog/best-code-execution-sandbox-for-ai-agents)
