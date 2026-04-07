# Fix #4: ConnectedApp Cannot Deploy to Sandbox

**Priority:** P4 — documentation only
**Agent:** None (manual)

---

## Root Cause

Some Salesforce sandboxes restrict Connected App creation via metadata deployment. This is an org-level setting, not a code issue. The Connected App deploys fine via 2GP managed package installs.

## Action: Add to Setup Guide

Add the following note to the installation/setup documentation.

---

### Setup Guide Addition

**File:** `MessageForge.Documentation/reference/setup-guide.md` (create if doesn't exist, or add to existing install docs)

```markdown
## Connected App — Sandbox Note

The `MessageForge_Backend` Connected App is included in the managed package and deploys automatically to subscriber orgs via AppExchange install.

**Sandbox limitation:** Some sandbox environments restrict Connected App creation via metadata deployment. If you see this error during `sf project deploy`:

> "You can't create a connected app. To enable connected app creation, contact Salesforce Customer Support."

**Workaround:** Create the Connected App manually in your sandbox:

1. Go to **Setup → App Manager → New Connected App**
2. Configure:
   - **Name:** MessageForge Backend
   - **Contact Email:** your admin email
   - **Enable OAuth Settings:** checked
   - **Callback URL:** `https://login.salesforce.com/services/oauth2/callback`
   - **OAuth Scopes:** Full access (api), Perform requests at any time (refresh_token, offline_access)
   - **Use digital signatures:** Upload the certificate from `force-app/main/default/certs/` or generate a new self-signed cert
3. Save and note the Consumer Key for Go backend configuration

This is NOT required for production AppExchange installs.
```

---

## Optional: Add to .forceignore

If sandbox deploys consistently fail on the Connected App, add it to `.forceignore`:

```
# Skip ConnectedApp for sandbox deploys (created manually)
# Remove this line for 2GP package builds
# force-app/main/default/connectedApps/MessageForge_Backend.connectedApp-meta.xml
```

**Decision:** Leave commented out for now. Only uncomment if it blocks regular deploys.

---

## Execution Prompt

```
This is a documentation-only task. Add a "Connected App — Sandbox Note" section to the setup documentation explaining that sandbox environments may require manual Connected App creation. Include the step-by-step manual creation instructions. No code changes needed.
```
