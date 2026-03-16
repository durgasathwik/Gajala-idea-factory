PROJECT: Terminus Flow
SPEC_ID: spec_9632926a
ARCHITECT_VERSION: 1.0

# 1. EXECUTIVE SUMMARY
Terminus Flow is a precision CLI tool designed for Platform Engineers managing "Rainbow Deployments" in durable execution environments (Temporal, DBOS). In these environments, workers often stick around to finish long-running workflows after a new version is deployed. Currently, these "legacy" workers are often left running indefinitely or killed manually, leading to resource waste or interrupted workflows. 

Terminus Flow automates the lifecycle of legacy workers by monitoring active job counts for specific version tags and triggering a graceful shutdown sequence only when the active job count reaches zero.

# 2. TECH STACK DECISION
*   **Language:** TypeScript (Strict Mode) - Ensures type safety for complex workflow metadata.
*   **Runtime:** Node.js (v20+) - Native support for Temporal/DBOS SDKs.
*   **CLI Framework:** `oclif` - Standard, robust framework for building developer tools.
*   **Workflow Integration:** `@temporalio/client` (Initial focus).
*   **Configuration:** `cosmiconfig` - Supports `.terminusrc`, `terminus.config.js`, or JSON.
*   **Logging:** `pino` - High-performance structured logging.
*   **Tables/UI:** `cli-ux` (built into oclif) and `chalk`.

# 3. FEATURE SCOPE

### In-Scope (V1)
*   **Version Discovery:** Identify active Task Queues and Versioning Sets in a Temporal Namespace.
*   **Job Monitoring:** Poll for active (Open) workflow executions associated with specific Build IDs or Task Queues.
*   **Graceful Termination:** Execute a two-stage shutdown (SIGTERM -> Wait -> SIGKILL) via remote SSH or Container API (configurable).
*   **Dry Run Mode:** Report what *would* be killed without taking action.
*   **Idle Grace Period:** Wait `X` minutes after job count hits zero before terminating to handle "bursty" workflow starts.

### Out-of-Scope (V1)
*   **Web Dashboard:** All interactions remain in the CLI.
*   **Kubernetes Operator:** We are building a tool *used* by scripts/CI, not a resident CRD controller (yet).
*   **Auto-Scaling Integration:** We kill workers; we don't spin them up.
*   **Support for non-durable systems:** Strictly for Temporal/DBOS-like stateful systems.

# 4. FILE & FOLDER STRUCTURE
```text
terminus-flow/
├── bin/                # CLI entry points
├── src/
│   ├── commands/       # Oclif command definitions (watch, list, kill)
│   ├── core/           # Business logic (Evaluator, State Machine)
│   ├── providers/      # Integration logic (TemporalProvider, DBOSProvider)
│   ├── transport/      # Logic for killing processes (SSH, K8s-exec, Docker)
│   ├── utils/          # Config loader, Logger, Formatters
│   └── index.ts
├── test/               # Integration & Unit tests
├── README.md
├── package.json
└── tsconfig.json
```

# 5. DATA MODELS

### Config Schema
```typescript
interface Config {
  provider: 'temporal' | 'dbos';
  connection: {
    address: string;
    namespace: string;
    tls?: { cert: string; key: string; };
  };
  rules: {
    version_pattern: string; // e.g. "v1.*"
    idle_grace_seconds: number;
    termination_signal: 'SIGTERM' | 'SIGQUIT';
    max_wait_seconds: number;
  };
  targets: Array<{
    version_id: string;
    remote_exec: 'ssh' | 'kubectl' | 'local';
    command: string; // The command to run to stop the worker
  }>;
}
```

# 6. API & INTEGRATION CONTRACTS

### Temporal Provider
*   `listTaskQueues()`: Returns list of active queues.
*   `getActiveCount(versionId: string)`: Returns count of workflows where `ExecutionStatus === RUNNING` and `BuildID === versionId`.

### Transport Layer (The "Killer")
*   `terminate(target: TargetConfig)`: Dispatches the shutdown command.
*   Interface: `ITransport { exec(cmd: string): Promise<void>; }`

# 7. COMPONENT / MODULE BREAKDOWN

1.  **Discovery Engine:** Queries the provider (Temporal) to find all "Legacy" versions (those not marked as 'current' or 'default').
2.  **Health Monitor:** For each legacy version, it checks the active job count.
3.  **Decision Logic (The "Brain"):** 
    *   If Jobs > 0: Mark as `ACTIVE_BUSY`.
    *   If Jobs == 0: Check `grace_period`. If expired, mark as `READY_FOR_TERMINATION`.
4.  **Executioner:** Orchestrates the actual process killing based on the `transport` configuration.

# 8. UI/UX SPECIFICATION
The CLI must prioritize clarity to prevent accidental production outages.

**Command: `terminus status`**
Displays a table:
```text
VERSION    | JOBS | STATUS  | IDLE TIME | ACTION
v1.0.2     | 12   | ACTIVE  | -         | KEEP
v1.0.1     | 0    | GRACE   | 04:20s    | WAIT (60s)
v1.0.0     | 0    | READY   | 15:00s    | TERMINATE
```

**Command: `terminus reap --dry-run`**
*   Color-coded output: Green for kept, Red for targeted kills.
*   Requires `--confirm` flag for actual execution.

# 9. TESTING STRATEGY
*   **Unit Tests:** Mock the `TemporalClient` to return varying job counts and ensure the Decision Logic hits the correct states (Grace vs. Reap).
*   **Integration Tests:** A Docker Compose setup with a local Temporal server and a dummy "Worker" script that responds to SIGTERM.
*   **Mock Transport:** A test transport that records "kill" commands in an array rather than executing them.

# 10. DEPLOYMENT & PUBLISHING PLAN
1.  **Package Manager:** Publish to NPM as `@rainbowops/terminus-flow`.
2.  **Binary:** Use `pkg` or Oclif's standalone binary export for users who don't want a Node.js dependency in their toolchain.
3.  **CI:** GitHub Actions for automated testing and semantic release.

# 11. RISK FLAGS FOR DEVELOPER
*   **The "Start-up Race":** A worker might have 0 jobs for 5 seconds before a new one is assigned. **Mitigation:** Force a minimum `idle_grace_period` of 60 seconds.
*   **API Rate Limiting:** Polling Temporal every second might trigger throttling. **Mitigation:** Default poll interval to 30-60 seconds.
*   **Permission Hell:** SSH/Kubectl access from the runner to the worker nodes. **Mitigation:** Clear documentation on required `kubeconfig` or `SSH_AUTH_SOCK` setups.

# 12. DEVELOPMENT CHECKLIST

- [ ] **Phase 1: Setup**
    - [ ] Initialize Oclif project.
    - [ ] Define Config schema and validator (Zod).
- [ ] **Phase 2: Provider Integration**
    - [ ] Implement Temporal Client wrapper.
    - [ ] Implement `list` logic for workflow versions.
- [ ] **Phase 3: Core Logic**
    - [ ] Build the "Reap Evaluator" (State machine for Jobs -> Grace -> Kill).
- [ ] **Phase 4: Transport**
    - [ ] Implement Local and SSH transport layers.
- [ ] **Phase 5: CLI Surface**
    - [ ] Build `status` and `reap` commands.
    - [ ] Add `dry-run` and interactive confirmation.
- [ ] **Phase 6: Hardening**
    - [ ] Add verbose logging and error handling for network timeouts.
    - [ ] Final documentation and README.
