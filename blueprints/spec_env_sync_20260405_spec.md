PROJECT: EnvSync — CI/CD Environment Configuration Drift Detector
SPEC_ID: spec_env_sync_20260405
ARCHITECT_VERSION: 1.0

# 1. EXECUTIVE SUMMARY

EnvSync is a lightweight CLI + GitHub Action that detects configuration drift across deployment environments (dev → staging → prod). It scans environment variable definitions across CI/CD stages, produces a human-readable drift report, and optionally generates sync manifests — all without touching secrets.

**The wedge:** A free, zero-friction audit tool. Point it at your GitHub repo, get a report showing which env vars differ between environments and how much manual fix-up time that costs. Report creates urgency → paid tier unlocks alerts + one-click sync.

**Validation status:** REJECTED by Validator (Agent 02) due to insufficient signal count (3 of 5 required), scoring 5.9/10. This blueprint is produced anyway per mission directive.

# 2. TECH STACK DECISION

Staying with "boring, proven" technology for single-developer velocity:

| Layer           | Choice                                      | Why |
|-----------------|---------------------------------------------|-----|
| Language        | TypeScript (strict mode)                    | Shared types across CLI and Action |
| Runtime         | Node.js v20+                                | GitHub Actions default runtime |
| CLI Framework   | oclif                                       | Battle-tested, same stack as existing blueprints |
| Config Parsing  | `yaml` + `dotenv`                           | Covers .yml, .env, GitHub Actions variable format |
| Diff Engine     | `json-diff` (for structured config), custom | Lightweight, no heavy deps |
| GitHub Action   | Node.js composite action                    | Runs on `ubuntu-latest` with Node.js pre-installed |
| Package         | `conf` for local state                      | Simple, no DB needed |
| Testing         | Vitest + GitHub Actions test workflow        | Fast, integrates with repo CI |
| Packaging       | oclif standalone binaries + NPM             | Covers both install patterns |

**What's NOT in the stack (V1):** No database, no web dashboard, no Kubernetes operator, no secrets manager integration (intentional — secrets never sync).

# 3. FEATURE SCOPE

### In-Scope (V1 — 14-day MVP)

- **Drift Scan (CLI core):** Parse env config files (`.env`, YAML vars, JSON configs) from multiple environment sources and produce a structured diff report.
- **GitHub Action:** Reusable action (`env-sync-scan`) that runs drift detection on PR or schedule. Outputs a markdown comment with drift summary.
- **Diff Report Generator:** Table showing per-variable drift status: `ALIGNED`, `MISSING`, `VALUE_DIFFERS`, `TYPE_DIFFERS`.
- **Dry-Run Sync Manifest:** Generate a patch file showing what *would* change to align environments — but does NOT apply anything.
- **Severity Scoring:** Flag critical drift (prod env var missing in staging but present in dev = risky).
- **Multiple Source Parsers:** GitHub Actions variables (`.github/workflows/*.yml`), `.env` files, Terraform `.tfvars`, basic JSON configs.

### Out-of-Scope (V1)

- **Actual sync execution:** Read-only in MVP. No writes to CI/CD configs.
- **Secrets handling:** Explicitly excluded. Never diff, report, or sync secrets.
- **CI/CD platforms beyond GitHub Actions:** V2 adds GitLab, Azure DevOps, BitBucket.
- **Web dashboard or SaaS backend:** CLI + GH Action only.
- **RBAC, SSO, audit logs:** Enterprise features for later.
- **Kubernetes ConfigMap/Secret scanning:** Too complex for V1.
- **Real-time alerts:** Scheduled scan only.

# 4. FILE & FOLDER STRUCTURE

```
env-sync/
├── bin/                          # oclif entry points
├── action/                       # GitHub Action wrapper
│   ├── action.yml                # Action definition
│   ├── index.js                  # Thin wrapper calling CLI
│   └── Dockerfile                # (optional, if needed)
├── src/
│   ├── commands/                 # oclif commands
│   │   ├── scan.ts               # `envsync scan` — core drift detection
│   │   ├── report.ts             # `envsync report` — format & output
│   │   └── init.ts               # `envsync init` — generate config
│   ├── parsers/                  # Environment config file parsers
│   │   ├── dotenv.ts             # .env file parser
│   │   ├── yaml.ts               # GitHub Actions workflow YAML parser
│   │   ├── tfvars.ts             # Terraform tfvars parser
│   │   └── json.ts               # JSON config parser
│   ├── engine/                   # Core diff logic
│   │   ├── normalizer.ts         # Normalize keys across formats (FOO_BAR vs foo-bar)
│   │   ├── comparator.ts         # Compare two env sets, produce diff entries
│   │   ├── scorer.ts             # Severity scoring (MISSING in prod = critical)
│   │   └── manifest.ts           # Generate dry-run sync patch
│   ├── types/                    # Shared TypeScript interfaces
│   │   └── index.ts
│   ├── utils/                    # Helpers
│   │   ├── config.ts             # Load .envsync.ts config
│   │   ├── logger.ts             # pino-based structured logging
│   │   └── formatter.ts           # Table rendering (terminal)
│   └── index.ts
├── test/
│   ├── parsers/                  # Parser unit tests
│   ├── engine/                   # Comparator + scorer tests
│   ├── fixtures/                 # Sample .env, .yml, .tfvars files
│   └── integration/              # Full scan pipeline tests
├── .envsync.example.ts           # Sample config
├── package.json
├── tsconfig.json
└── README.md
```

# 5. DATA MODELS

### Configuration (.envsync.ts)

```typescript
interface EnvSyncConfig {
  /** Environments to compare, in order of promotion */
  environments: {
    name: string;                  // e.g., "dev", "staging", "prod"
    sources: SourceConfig[];       // Where to find env config for this env
  }[];

  /** Keys to ignore (e.g., BUILD_TIMESTAMP, COMMIT_SHA) */
  ignorePatterns?: string[];

  /** Severity rules */
  severity?: {
    /** If var present in prod but missing elsewhere, flag as? */
    prodMissingDownstream: 'critical' | 'warning';
    /** If value differs between adjacent environments */
    valueDiffAdjacent: 'warning' | 'info';
  };

  /** Output format */
  output?: 'table' | 'json' | 'markdown';
}

interface SourceConfig {
  type: 'dotenv' | 'yaml' | 'tfvars' | 'json';
  path: string;                     // glob pattern or file path
  /** Prefix to strip from keys (e.g., "MYAPP_" → "FOO") */
  keyPrefix?: string;
  /** Extract env vars from this workflow key path (YAML only) */
  yamlPath?: string;                // e.g., "jobs.deploy.steps[].env"
}
```

### Core Diff Types

```typescript
interface DriftEntry {
  key: string;                    // Normalized key name
  sourceEnvs: {
    env: string;
    value?: string;              // undefined = missing
    source: string;              // Which file this came from
  }[];
  status: 'ALIGNED' | 'MISSING' | 'VALUE_DIFFERS' | 'TYPE_DIFFERS';
  severity: 'critical' | 'warning' | 'info';
  /** Human-readable explanation */
  summary: string;
}

interface DriftReport {
  timestamp: string;
  environments: string[];
  totalKeys: number;
  alignedCount: number;
  driftedCount: number;
  criticalCount: number;
  entries: DriftEntry[];
  /** Estimated hours/week wasted (heuristic from drift volume) */
  estimatedImpact: string;
}

interface SyncManifest {
  /** Source environment to use as truth */
  sourceEnv: string;
  /** Target environment to sync to */
  targetEnv: string;
  changes: {
    key: string;
    action: 'ADD' | 'UPDATE' | 'REMOVE';
    currentValue?: string;
    targetValue: string;
  }[];
  /** Warning: this is a dry-run patch, NOT applied */
  dryRun: true;
}
```

### Estimated Impact Heuristic

```typescript
// Rough heuristic: each drifted key ≈ 15 min/month of manual fix-up
// Critical (prod-missing) = 1 hour/month each
function estimateImpact(critical: number, drifted: number): string {
  const minutes = (critical * 60) + (drifted * 15);
  const hoursPerWeek = (minutes / 60) / 4.3;
  if (hoursPerWeek < 1) return '<1 hr/week';
  return `~${Math.round(hoursPerWeek)} hr/week`;
}
```

# 6. API & INTEGRATION CONTRACTS

### GitHub Action Inputs/Outputs

**Action: `env-sync/scan-action`**

```yaml
- uses: env-sync/scan-action@v1
  with:
    # Path to .envsync.ts config (default: .envsync.ts at repo root)
    config: '.envsync.production.ts'
    # Environments to scan (comma-separated, default: dev,staging,prod)
    environments: 'dev,staging,prod'
    # Output format for comment
    output: 'markdown'
    # Post a PR comment (true) or just log (false)
    comment-on-pr: 'true'
  id: envsync
```

**Outputs:**

```yaml
outputs:
  drift-summary:   # Markdown summary for comment
  drift-count:     # Number of drifted keys
  critical-count:  # Number of critical issues
  report-json:     # Full JSON report (base64 encoded)
```

### CLI Commands

```bash
# Initialize config in current directory
envsync init

# Scan and display drift report in terminal
envsync scan

# Scan with custom config
envsync scan --config .envsync.prod.ts --output json

# Generate dry-run sync manifest (from dev → staging)
envsync report --manifest --source dev --target staging --dry-run

# Run as GitHub Action (called internally)
envsync scan --github-action-output
```

### Parser Contracts

Each parser implements a common interface:

```typescript
interface EnvParser {
  /** Parse a file and return normalized key-value pairs */
  parse(filePath: string, options?: ParseOptions): Promise<Map<string, string>>;
  
  /** File extensions this parser handles */
  extensions: string[];
}
```

# 7. COMPONENT / MODULE BREAKDOWN

### 1. Config Loader (`utils/config.ts`)
Loads `.envsync.ts` from current directory or specified path. Validates schema with Zod. Generates example if none exists.

### 2. Parser Registry (`parsers/`)
Pluggable parser system. Each parser handles one config format and returns a normalized `Map<string, string>`. Registry auto-detects based on file extension.

### 3. Normalizer (`engine/normalizer.ts`)
Strips prefixes, normalizes casing (`FOO_BAR` ↔ `foo-bar` ↔ `FooBar`), removes ignored keys. Produces canonical key set across all environments.

### 4. Comparator (`engine/comparator.ts`)
Takes normalized env sets from each environment. Produces `DriftEntry[]` by comparing presence and values across all environments pairwise.

### 5. Scorer (`engine/scorer.ts`)
Assigns severity to each `DriftEntry` based on configurable rules (prod-missing = critical, staging-prod diff = warning, dev-staging diff = info).

### 6. Report Generator (`commands/report.ts`)
Formats `DriftReport` into terminal table, JSON, or markdown. Calculates estimated impact heuristic.

### 7. Manifest Builder (`engine/manifest.ts`)
Produces a `SyncManifest` showing what changes would be needed to align target environment to source. **Does NOT apply changes.**

### 8. GitHub Action Wrapper (`action/`)
Thin wrapper that runs `envsync scan --github-action-output`, captures JSON report, posts markdown comment on PR using `@actions/core` and `@actions/github`.

# 8. UI/UX SPECIFICATION

### Terminal Output (`envsync scan`)

```
🔍 EnvSync Drift Report
   Environments: dev → staging → prod
   Scanned: 7 files, 142 keys

   ┌─────────────────────────────────────┬─────────────┬──────────┐
   │ VARIABLE                            │ STATUS      │ SEVERITY │
   ├─────────────────────────────────────┼─────────────┼──────────┤
   │ DATABASE_URL                        │ VALUE_DIFF  │ ⚠️ warn  │
   │ API_KEY                             │ MISSING     │ 🔴 crit  │
   │ CACHE_TTL                           │ VALUE_DIFF  │ ℹ️  info  │
   │ LOG_LEVEL                           │ ALIGNED     │ ✅       │
   │ FEATURE_FLAG_NEW_UI                 │ MISSING     │ ℹ️  info  │
   └─────────────────────────────────────┴─────────────┴──────────┘

   Summary: 112 aligned, 18 drifted, 3 critical
   Estimated impact: ~2 hr/week manual fix-up
   
   💡 Run 'envsync report --manifest --source dev --target staging' for sync patch
```

### GitHub PR Comment

```markdown
## 🔍 Environment Drift Scan

| Status | Count |
|--------|-------|
| ✅ Aligned | 112 |
| ⚠️ Drifted | 18 |
| 🔴 Critical | 3 |

### Critical Issues

| Variable | dev | staging | prod |
|----------|-----|---------|------|
| `API_KEY` | ✅ | ✅ | ❌ MISSING |

### Value Differences

| Variable | dev | staging | prod |
|----------|-----|---------|------|
| `DATABASE_URL` | `dev.db.example.com` | `staging.db.example.com` | `prod.db.example.com` ⚠️ |
| `CACHE_TTL` | `300` | `300` | `60` |

---
*Scan completed in 2.3s • Run `envsync scan --help` for options*
```

### Init Output (`envsync init`)

```
✨ Created .envsync.ts with default configuration.

Next steps:
  1. Edit .envsync.ts to define your environments and config sources
  2. Run 'envsync scan' to check for drift
  3. Add to your CI: see https://github.com/<repo>/env-sync#github-action
```

# 9. TESTING STRATEGY

### Unit Tests (Priority: High)

| Test Area | What | How |
|-----------|------|-----|
| Parser tests | Each parser correctly reads its format | Fixtures in `test/fixtures/` — `.env`, `.yml`, `.tfvars`, `.json` |
| Normalizer | Key normalization, prefix stripping, case folding | Direct function tests with edge cases |
| Comparator | Correctly identifies ALIGNED vs MISSING vs VALUE_DIFF | Known input pairs → expected DriftEntry |
| Scorer | Severity assignment matches rules | Configurable rules → expected severity per entry |
| Config loader | Schema validation, defaults, error messages | Zod schema against valid/invalid configs |

### Integration Tests

- **Full pipeline:** Run `envsync scan` against a test repo fixture with known drift → verify report matches expectations.
- **Action simulation:** Run the GitHub Action locally using `nektos/act` → verify comment output format.

### Test Coverage Target

- `engine/` and `parsers/`: ≥85% (core logic must be reliable)
- `commands/`: ≥70% (mostly I/O glue)
- `action/`: ≥60% (thin wrapper)

### Fixture Strategy

```
test/fixtures/
├── repo-dev/
│   ├── .env
│   └── config.json
├── repo-staging/
│   ├── .env
│   └── config.json
├── repo-prod/
│   ├── .env
│   └── config.json
└── workflows/
    └── deploy.yml              # Sample GitHub Actions workflow with env vars
```

# 10. DEPLOYMENT & PUBLISHING PLAN

### Phase 1: NPM Package (Day 1-12)

1. Develop core library + CLI on feature branch.
2. Test with fixture repos.
3. Publish to NPM as `@env-sync/cli`:
   ```bash
   npm publish --access public
   ```
4. Users install: `npm install -g @env-sync/cli`

### Phase 2: GitHub Action (Day 12-14)

1. Publish action to same repo under `action/` directory.
2. Reference in marketplace as `env-sync/scan-action`.
3. Tag release `v1` pointing to action directory.
4. Users consume:
   ```yaml
   - uses: env-sync/env-sync/action@v1
     with:
       environments: 'dev,staging,prod'
   ```

### Phase 3: CI Pipeline for EnvSync Itself

```yaml
# .github/workflows/ci.yml (for the env-sync repo itself)
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm test
  publish:
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - run: npm publish
```

# 11. RISK FLAGS FOR DEVELOPER

1. **False positives from environment-specific values:** DATABASE_URL, API endpoints, and feature flags are *expected* to differ between environments. Mitigation: Implement an `ignorePatterns` config with regex support; default-include common patterns like `*_URL` suffixes with per-environment hostnames.

2. **Large monorepo performance:** Parsing hundreds of env files across many packages can be slow. Mitigation: Add `--concurrency` flag with worker threads; default parallel per-env parsing.

3. **YAML parser edge cases:** GitHub Actions workflows use complex YAML with anchors, matrix builds, and nested `env:` blocks at job/step level. Mitigation: Start with top-level workflow `env:` and job-level `env:` only; document limitations clearly.

4. **Secrets appearing in .env files:** Some teams put API keys in `.env` (bad practice but common). If the scanner reports these, it could leak secrets in PR comments. Mitigation: **Never include values in GitHub Action output.** Report only key names and status (`MISSING`, `VALUE_DIFF`). Use redaction markers like `[DIFFERS]` instead of actual values.

5. **Terraform tfvars complexity:** Interpolation syntax (`var.foo`, `module.bar`) makes direct value comparison meaningless. Mitigation: Mark tfvars entries as `OPAQUE_COMPARISON` — only flag presence/absence, not value differences.

6. **Scope creep temptation:** Developers will ask "can you also sync?" — resist. V1 is read-only audit only. Write capability is a separate product decision.

# 12. DEVELOPMENT CHECKLIST

### Phase 1: Project Setup (Day 1-2)
- [ ] Initialize oclif project (`npx oclif generate`)
- [ ] Set up TypeScript strict config
- [ ] Add Zod for config schema validation
- [ ] Create `.envsync.example.ts` template
- [ ] Set up Vitest with coverage config
- [ ] Create initial `test/fixtures/` directory structure

### Phase 2: Parser Implementation (Day 3-5)
- [ ] Implement `.env` parser (handle comments, quotes, empty values)
- [ ] Implement JSON parser
- [ ] Implement YAML parser (GitHub Actions env block extraction)
- [ ] Implement Terraform `.tfvars` parser (variable declarations only)
- [ ] Write unit tests for each parser with fixture files
- [ ] Build parser registry with auto-detection by extension

### Phase 3: Core Diff Engine (Day 6-9)
- [ ] Implement `Normalizer` (case normalization, prefix stripping, ignore patterns)
- [ ] Implement `Comparator` (multi-environment pairwise comparison)
- [ ] Implement `Scorer` (configurable severity rules)
- [ ] Implement `Manifest` builder (dry-run patch generation)
- [ ] Implement `estimateImpact` heuristic
- [ ] Write comprehensive unit tests (>85% coverage)

### Phase 4: CLI Surface (Day 10-11)
- [ ] Build `scan` command (load config, run pipeline, output report)
- [ ] Build `init` command (generate example config)
- [ ] Build `report` command (format output, manifest mode)
- [ ] Add `--output` flag (table/json/markdown)
- [ ] Add `--dry-run` flag for manifest generation
- [ ] Add colored terminal output with chalk + table layout

### Phase 5: GitHub Action (Day 12-13)
- [ ] Create `action/action.yml` definition
- [ ] Implement thin wrapper calling CLI with `--github-action-output`
- [ ] Set up `@actions/github` for PR comments
- [ ] Test with `nektos/act` locally
- [ ] Add workflow trigger on push to main and PR open

### Phase 6: Polish & Release (Day 14)
- [ ] Write README (quick start, config reference, GitHub Action usage)
- [ ] Run full integration test with fixture repo
- [ ] Fix any remaining type errors and lint issues
- [ ] Publish to NPM as `@env-sync/cli`
- [ ] Create `v1` tag for GitHub Action
- [ ] Add CONTRIBUTING.md for future contributors