# Sandbox Setup Notes

## Connected App -- Manual Creation Required

The `MessageForge_Backend` Connected App is included in the managed package and deploys automatically to subscriber orgs via AppExchange install.

Some sandbox environments restrict Connected App creation via metadata deployment. If you see this error during `sf project deploy`:

> "You can't create a connected app. To enable connected app creation, contact Salesforce Customer Support."

### Workaround

Create the Connected App manually in your sandbox:

1. Go to **Setup > App Manager > New Connected App**
2. Configure:
   - **Name:** MessageForge Backend
   - **Contact Email:** your admin email
   - **Enable OAuth Settings:** checked
   - **Callback URL:** `https://login.salesforce.com/services/oauth2/callback`
   - **OAuth Scopes:** Full access (api), Perform requests at any time (refresh_token, offline_access)
   - **Use digital signatures:** Upload the certificate from `force-app/main/default/certs/` or generate a new self-signed cert
3. Save and note the Consumer Key for Go backend configuration

This is NOT required for production AppExchange installs.

## Record Pages -- Create via App Builder

Record Pages (`Messenger_Chat_Record_Page`, `Messenger_Channel_Record_Page`) may fail to deploy due to template name mismatches across org editions.

### Workaround

Create record pages via Lightning App Builder UI, then retrieve:

```bash
sf project retrieve start --metadata FlexiPage --target-org sandbox
```

This captures the exact template name the org uses.
