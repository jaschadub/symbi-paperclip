---
name: symbiont-governed-agents
description: >
  Build and run governed AI agents using Symbiont's zero-trust runtime.
  Use when a Paperclip agent needs policy enforcement, cryptographic audit trails,
  sandbox isolation, or compliance guarantees (HIPAA, SOC2, FedRAMP).
  Produces agents that run through the ORGA reasoning loop with Cedar policy
  authorization at the Gate phase -- actions that aren't explicitly permitted
  are structurally impossible, not just blocked.
version: 1.0.0
author: ThirdKey AI
tags: [security, governance, compliance, agent-runtime, zero-trust]
---

# Symbiont Governed Agents

## When to Use This Skill

Use Symbiont instead of bare Claude Code / Codex / bash when any of these apply:

- The agent handles **sensitive data** (PII, PHI, financial records, credentials)
- The task requires a **tamper-proof audit trail** (regulatory, legal, compliance)
- You need **policy enforcement** that the LLM cannot bypass or prompt-inject around
- The agent runs **untrusted code or external tools** and needs sandbox isolation
- You want **cryptographic proof** of what the agent did, when, and under what authority

If the task is simple coding with no compliance requirements, use your normal coding agent. Symbiont adds value when the cost of an ungoverned mistake is high.

## How It Works with Paperclip

Paperclip orchestrates the org chart, goals, budgets, and task assignment.
Symbiont governs what each agent can actually do at execution time.

```
Paperclip heartbeat
    |
    v
symbi run <agent> --input '<task-context-json>'
    |
    v
ORGA Loop: Observe -> Reason -> Gate (Cedar policy) -> Act
    |
    v
Results + signed audit journal returned to Paperclip ticket
```

The Gate phase runs **outside LLM influence**. The model cannot convince the Gate
to allow an action that Cedar policy does not permit. This is the architectural
difference from approval-gate governance.

## Prerequisites

Install the Symbiont CLI on the machine running the Paperclip agent:

```bash
# macOS
brew tap thirdkeyai/tap && brew install symbi

# Linux / macOS (install script)
curl -fsSL https://raw.githubusercontent.com/thirdkeyai/symbiont/main/scripts/install.sh | bash

# Initialize a governed project (one-time)
symbi init --profile minimal
```

## Creating a Governed Agent

### Step 1: Write the Agent DSL

Create `agents/<name>.dsl` in the Paperclip project workspace:

```dsl
metadata {
    version = "1.0.0"
    author = "Paperclip CEO Agent"
    description = "Describe what this agent does and why"
    tags = ["paperclip", "governed"]
}

agent task_worker(input: String) -> String {
    capabilities = ["read_data", "write_output", "analyze"]

    policy worker_policy {
        allow: ["read_data", "write_output"]
            if input.length() < 100000

        deny: ["network_access", "spawn_process", "file_system"]
            if true

        require: {
            input_validation: true,
            output_sanitization: true,
            timeout: "30000ms"
        }

        audit: {
            log_level: "info",
            include_input: false,
            include_output: true,
            compliance_tags: ["SOC2"]
        }
    }

    with
        memory = "ephemeral",
        security = "high",
        sandbox = "Tier1",
        timeout = 30000
    {
        try {
            if input.is_empty() {
                return error("No task input provided");
            }

            let result = process(input);
            return result;

        } catch (error) {
            log("ERROR", "Task failed: " + error.message);
            return error("Task execution failed");
        }
    }
}
```

### Step 2: Write the Cedar Policy

Create `policies/<name>.cedar` for formal authorization:

```cedar
// Allow the agent to respond and use approved tools
permit(
    principal == Agent::"task_worker",
    action == Action::"respond",
    resource
);

permit(
    principal == Agent::"task_worker",
    action == Action::"tool_call::read_data",
    resource
);

permit(
    principal == Agent::"task_worker",
    action == Action::"tool_call::write_output",
    resource
);

// Deny everything else by default (Cedar is deny-by-default)
forbid(
    principal == Agent::"task_worker",
    action,
    resource
) unless {
    action == Action::"respond" ||
    action == Action::"tool_call::read_data" ||
    action == Action::"tool_call::write_output"
};
```

### Step 3: Run the Agent

```bash
# Single execution (maps to Paperclip heartbeat)
symbi run task_worker --input '{"task": "...", "context": "..."}'

# With specific features enabled
symbi run task_worker \
    --features "cloud-llm,cedar" \
    --input '{"task": "...", "goal_ancestry": "..."}'
```

## Paperclip Adapter Configuration

Configure the Paperclip agent to use Symbiont as its execution backend.

### Agent Config (in Paperclip)

```json
{
    "name": "Governed Coder",
    "role": "engineer",
    "adapterType": "bash",
    "adapterConfig": {
        "command": "symbi",
        "args": ["run", "task_worker", "--input"],
        "inputMode": "argument",
        "workingDirectory": "/path/to/project"
    }
}
```

### Heartbeat Adapter (bash type)

Paperclip sends a heartbeat, the adapter runs `symbi run` with the task
context as input, and the agent returns structured results. The adapter
should pass the full Paperclip task context including goal ancestry:

```bash
#!/bin/bash
# heartbeat-symbiont.sh
# Called by Paperclip on each heartbeat tick

TASK_CONTEXT="$1"

# Run governed agent, capture output and exit code
OUTPUT=$(symbi run task_worker --input "$TASK_CONTEXT" 2>&1)
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
    echo "$OUTPUT"
else
    echo "{\"error\": \"Agent execution failed\", \"details\": \"$OUTPUT\"}"
    exit 1
fi
```

### HTTP Adapter (webhook type)

If Symbiont is running with the `http-input` feature, Paperclip can send
heartbeats as HTTP POST requests:

```json
{
    "name": "Governed Analyst",
    "role": "analyst",
    "adapterType": "http",
    "adapterConfig": {
        "url": "http://localhost:8081/webhook",
        "method": "POST",
        "headers": {
            "Authorization": "Bearer ${SYMBI_WEBHOOK_TOKEN}",
            "Content-Type": "application/json"
        }
    }
}
```

## Common Agent Patterns for Paperclip Roles

### Governed Code Reviewer

```dsl
metadata {
    version = "1.0.0"
    description = "Reviews code changes with policy-enforced read-only access"
    tags = ["paperclip", "code-review", "governed"]
}

agent code_reviewer(input: String) -> String {
    capabilities = ["read_files", "analyze_code", "write_review"]

    policy review_policy {
        allow: ["read_files", "analyze_code", "write_review"]
        deny: ["write_files", "execute_code", "network_access"]
            if true

        require: {
            sandbox_tier: "Tier1",
            timeout: "120000ms"
        }

        audit: {
            log_level: "info",
            include_findings: true,
            compliance_tags: ["SOC2"]
        }
    }

    with memory = "ephemeral", sandbox = "Tier1", timeout = 120000 {
        let review = analyze_code(input);
        return review;
    }
}
```

### Governed Data Analyst (HIPAA)

```dsl
metadata {
    version = "1.0.0"
    description = "Analyzes data with HIPAA-compliant audit trails"
    tags = ["paperclip", "analyst", "hipaa"]
}

agent data_analyst(input: String) -> String {
    capabilities = ["read_data", "statistical_analysis", "generate_report"]

    policy hipaa_policy {
        allow: ["read_data", "statistical_analysis", "generate_report"]
            if input.length() < 50000000

        deny: [
            "network_access",
            "file_system",
            "execute_code",
            "export_raw_data"
        ] if true

        require: {
            encryption: "AES-256-GCM",
            input_validation: true,
            output_sanitization: true,
            sandbox_tier: "Tier2"
        }

        audit: {
            log_level: "warning",
            include_input: false,
            include_output: false,
            include_metadata: true,
            retention_days: 2190,
            compliance_tags: ["HIPAA", "BAA"]
        }
    }

    with
        memory = "ephemeral",
        privacy = "strict",
        security = "high",
        sandbox = "Tier2",
        timeout = 300000
    {
        try {
            let analysis = run_analysis(input);
            let report = generate_report(analysis);
            return report;
        } catch (error) {
            log("ERROR", "Analysis failed: " + error.message);
            return error("Analysis failed");
        }
    }
}
```

### Governed Security Scanner

```dsl
metadata {
    version = "1.0.0"
    description = "Scans code and dependencies for vulnerabilities"
    tags = ["paperclip", "security", "scanner"]
}

agent security_scanner(input: String) -> String {
    capabilities = [
        "read_files",
        "analyze_dependencies",
        "check_vulnerabilities"
    ]

    policy scanner_policy {
        allow: [
            "read_files",
            "analyze_dependencies",
            "check_vulnerabilities"
        ] if true

        deny: [
            "write_files",
            "execute_code",
            "modify_permissions"
        ] if true

        require: {
            sandbox_tier: "Tier2",
            signature_verification: true,
            max_scan_time: "300000ms"
        }

        audit: {
            log_level: "warning",
            include_findings: true,
            include_risk_scores: true,
            alert_on_critical: true,
            compliance_tags: ["OWASP", "CWE"]
        }
    }

    with memory = "ephemeral", security = "high", sandbox = "Tier2" {
        let findings = scan(input);
        return format_findings(findings);
    }
}
```

## Sandbox Tier Selection

| Tier | Isolation | Use When | Startup |
|------|-----------|----------|---------|
| **Tier1** (Docker) | Container | General governed tasks | ~100ms |
| **Tier2** (gVisor) | Kernel-level | Untrusted input, external data | ~500ms |
| **Tier3** (Firecracker) | MicroVM | Multi-tenant, regulatory compliance | ~2s |

Default to **Tier1** for Paperclip coding agents. Use **Tier2** for agents
processing external data or user-provided content. Use **Tier3** only for
regulated workloads requiring full isolation guarantees.

## Passing Paperclip Goal Context

Paperclip provides goal ancestry (mission > project > agent goal > task).
Pass this to Symbiont so the agent's reasoning has full context:

```bash
symbi run task_worker --input '{
    "task": "Implement WebSocket handler for document sync",
    "goal_ancestry": {
        "mission": "Build the #1 AI note-taking app to $1M MRR",
        "project": "Ship collaboration features",
        "agent_goal": "Implement real-time sync",
        "task_id": "CLIP-1042"
    },
    "budget_remaining_tokens": 50000
}'
```

The agent receives this as its input parameter and can reference the goal
ancestry during its Reason phase for aligned decision-making.

## Audit Trail Integration

Symbiont produces Ed25519-signed, hash-chained audit journals. To surface
these in Paperclip's ticket trace:

```bash
# Export journal entries as JSON for the most recent run
symbi journal export --agent task_worker --format json --last-run

# Example output (each entry is signed and hash-chained):
# {"event":"Started","timestamp":"...","signature":"...","prev_hash":"..."}
# {"event":"ReasoningComplete","proposed_actions":3,"signature":"..."}
# {"event":"PolicyEvaluated","allowed":2,"denied":1,"signature":"..."}
# {"event":"ToolsDispatched","tools":2,"duration_ms":847,"signature":"..."}
# {"event":"Terminated","reason":"complete","iterations":3,"signature":"..."}
```

Post journal entries back to the Paperclip ticket as a comment or attachment
so the full governed execution trace is visible in the Paperclip dashboard.

## What the Gate Provides That Paperclip Governance Does Not

Paperclip governance operates at the **organizational** level: approve hires,
review strategy, pause agents, enforce budgets. This is necessary but not
sufficient for regulated workloads.

Symbiont's Gate operates at the **execution** level:

| Paperclip Governance | Symbiont Gate |
|---------------------|---------------|
| Board approves agent hiring | Cedar policy defines what the agent can do |
| Budget limits spending | Policy blocks unauthorized tool calls |
| Tickets trace conversations | Signed journal proves tamper-proof execution |
| Approval gates are human-triggered | Gate runs on every action, automatically |
| Agent can do anything its runtime allows | Agent can only do what policy permits |

They are complementary. Paperclip manages the organization. Symbiont governs
the execution. Together, you get both.

## Reference

For the complete Symbiont DSL specification, policy patterns, anti-patterns,
security scanner, reasoning loop details, and all built-in functions, see:

- `references/symbiont-skill.md` (full 1,500-line agent development guide)
- [DSL Guide](https://github.com/thirdkeyai/symbiont/blob/main/docs/dsl-guide.md)
- [DSL Specification](https://github.com/thirdkeyai/symbiont/blob/main/docs/dsl-specification.md)
- [Example Agents](https://github.com/thirdkeyai/symbiont/blob/main/agents/README.md)
- [Symbiont GitHub](https://github.com/thirdkeyai/symbiont)
