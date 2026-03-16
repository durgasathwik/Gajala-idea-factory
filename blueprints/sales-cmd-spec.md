PROJECT: SalesCmd
SPEC_ID: spec_SalesCmd_09ba
ARCHITECT_VERSION: 1.0

# 1. EXECUTIVE SUMMARY
SalesCmd is a high-velocity, keyboard-first terminal interface (CLI) designed for power users of enterprise CRMs. The modern CRM UI (HubSpot/Salesforce) is often bloated and slow for frequent tasks. SalesCmd provides a "Sync Bridge" that allows users to manage pipelines and update deal statuses directly from their terminal, significantly reducing friction and increasing sales velocity. The MVP focuses exclusively on HubSpot integration.

# 2. TECH STACK
We are using "boring," reliable technology to ensure stability and developer velocity.
- **Language:** TypeScript (TS)
- **Runtime:** Node.js (v18+)
- **CLI Framework:** [oclif](https://oclif.io/) (Open CLI Framework)
- **SDK:** [@hubspot/api-client](https://www.npmjs.com/package/@hubspot/api-client)
- **State Management:** Local JSON storage (via `conf` or similar)
- **HTTP Client:** Axios (included in HubSpot SDK)

# 3. FEATURE SCOPE (MVP Focus)
- **Auth (OAuth 2.0):** Securely connect to HubSpot using an OAuth flow or Private App Access Token.
- **Pipeline List:** Fetch and display all active sales pipelines and stages.
- **Deal Status Update:** Quickly search for a deal by name or ID and update its stage/status.
- **Context Switching:** Support for multiple HubSpot portals/accounts.

# 4. OUT OF SCOPE (V1)
- Salesforce integration (Reserved for V2).
- Bulk deal creation/import.
- Advanced reporting or analytics.
- Contact/Task management (Focus remains strictly on Deals/Pipelines for MVP).
- Real-time webhooks (Sync is pull-based or trigger-based).

# 5. FILE & FOLDER STRUCTURE
```text
salescmd/
├── bin/                # Executable entry points
├── src/
│   ├── commands/       # oclif command files
│   │   ├── auth/       # login, logout, status
│   │   ├── deals/      # list, update
│   │   └── pipelines/  # list
│   ├── core/           # Business logic & HubSpot Bridge
│   │   ├── api.ts      # HubSpot SDK wrapper
│   │   └── state.ts    # Config & Local storage management
│   ├── types/          # TS Interfaces
│   └── utils/          # Formatters, UI helpers (Chalk, Ora)
├── test/               # Mocha/Chant tests
├── package.json
└── tsconfig.json
```

# 6. DATA MODELS
### Config (Local Store)
```typescript
interface Config {
  activePortalId: string;
  portals: {
    [portalId: string]: {
      accessToken: string;
      refreshToken?: string;
      name: string;
    }
  };
}
```
### Sync State (Cache)
```typescript
interface SyncState {
  lastSync: string;
  pipelines: Pipeline[];
}
```

# 7. API CONTRACTS
### HubSpot OAuth
- **Scope:** `crm.objects.deals.read`, `crm.objects.deals.write`, `crm.schemas.deals.read`.
- **Flow:** Local server listener (port 3000) to capture redirect code.

### HubSpot CRM API (v3)
- `GET /crm/v3/objects/deals`: List deals.
- `PATCH /crm/v3/objects/deals/{dealId}`: Update deal stage.
- `GET /crm/v3/pipelines/deals`: Fetch pipeline/stage mapping.

# 8. COMPONENT BREAKDOWN
- **Command Dispatcher:** Oclif-based routing for CLI arguments.
- **HubSpot Bridge:** Abstracted client to handle rate limiting and token refreshing.
- **Table Renderer:** Formats deal lists into a clean, terminal-width table (using `cli-ux` or `table`).
- **Input Prompts:** Interactive fuzzy-search for deal selection (using `inquirer` or `enquirer`).

# 9. UI/UX SPEC
### Command Structure
- `scmd login`: Initiates OAuth flow.
- `scmd pipelines`: Lists available pipelines.
- `scmd deals list --pipeline="Sales"`: Lists deals in a specific pipeline.
- `scmd deals update <dealId> --stage="Closed Won"`: Quick update.
- `scmd deals status`: Interactive prompt to select a deal and move it.

### Style
- Success messages in green.
- Errors in bold red.
- Use spinners (`ora`) for API calls.

# 10. TESTING STRATEGY
- **Unit Tests:** Mock HubSpot SDK responses to test command logic.
- **Integration Tests:** Use a HubSpot Sandbox account for end-to-end command validation.
- **Coverage Target:** 70% (Focus on core logic in `src/core`).

# 11. DEPLOYMENT PLAN
- **Distribution:** Published as a scoped package on NPM (`@salescmd/cli`).
- **Binary:** Use `oclif-dev pack` to generate standalone binaries for macOS/Linux/Windows.
- **CI/CD:** GitHub Actions for automated testing and publishing on tag creation.

# 12. RISK FLAGS & MITIGATION
- **API Rate Limits:** Implement exponential backoff in the HubSpot Bridge.
- **Token Security:** Store tokens in the system keychain (using `keytar`) instead of plain text JSON where possible.
- **Complexity:** Stick to `oclif` defaults to avoid configuration rabbit holes.

---
**Architect's Note:** Developer Agent, focus on the `deals update` command first, as it's the highest-value "wedge" for the user. Keep the UI extremely snappy. Report status to @Gajala.
