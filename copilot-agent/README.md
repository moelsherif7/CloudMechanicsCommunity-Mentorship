# Partner Center Incentive Claims Assistant

A Microsoft 365 Copilot **declarative agent** that helps Microsoft partners
check the status of incentive engagement and co-op claims in Microsoft Partner
Center via the Partner Center Incentives REST API, using delegated (interactive)
sign-in.

> ⚠️ **Scope and accuracy note.** The Partner Center Incentives REST surface
> for engagement/co-op claims is partner-distribution: exact endpoint paths,
> query parameters, and field names are not always public. The OpenAPI spec
> in this folder is a working scaffold based on commonly documented patterns
> (`https://api.partner.microsoft.com/v1.0/incentives/...`) and the documented
> claim lifecycle. **Verify the paths and response shapes against the
> documentation Microsoft provides to your partner organization** and update
> [`openapi.yaml`](./openapi.yaml) before publishing.

## Files

| File | Purpose |
|---|---|
| [`manifest.json`](./manifest.json) | Declarative agent manifest (schema v1.5). Defines name, description, system instructions, conversation starters, and references the API plugin. |
| [`plugin.json`](./plugin.json) | API plugin manifest (schema v2.3). Declares the four functions and wires them to the OpenAPI spec under OAuth (`OAuthPluginVault`). |
| [`openapi.yaml`](./openapi.yaml) | OpenAPI 3.0 spec for the Partner Center Incentives endpoints the agent calls. |

The agent exposes four functions:

| Function | Endpoint | Purpose |
|---|---|---|
| `listIncentivePrograms` | `GET /v1.0/incentives/programs` | Programs the partner is enrolled in (MCI, Co-op, etc.). |
| `listEngagements` | `GET /v1.0/incentives/engagements` | Engagements/workshops under a program. |
| `listClaims` | `GET /v1.0/incentives/claims` | Claims, filterable by `status`, `programId`, `engagementId`, date range. |
| `getClaim` | `GET /v1.0/incentives/claims/{claimId}` | Single claim by ID. |

## Claim status lifecycle (reference)

The agent's instructions teach it to recognize these statuses:

- **Draft** – customer consent received; claim editable; expires if not submitted by deadline.
- **Submitted** – in Microsoft review queue.
- **Approved** – all required elements present; payout follows program cadence.
- **ActionRequired** ("Partner action required") – reviewer needs more info; partner must edit and resubmit.
- **Expired** – first-submission or post-submission workshop expired.
- **Rejected** – denied; partner has 30 days to dispute.

Source: [MCI engagement claims best practices (Partner Center)](https://learn.microsoft.com/en-us/partner-center/incentives/engagement-claim-best-practices).

## Prerequisites

1. **Microsoft 365 tenant** with Microsoft 365 Copilot licensing for the users who'll use the agent.
2. **Partner Center account** with the **Incentive admin** or **Incentive user** role for the partner organization whose claims you want to query. Without this role, the API returns 403.
3. **Microsoft 365 Agents Toolkit** (VS Code extension) or **Teams Toolkit** for packaging and deployment.
4. **Entra ID app registration** in your tenant, with delegated permission `https://api.partner.microsoft.com/.default` and an **OAuth client registration** in the Teams Developer Portal.

## Setup — delegated (interactive) auth

These steps wire `OAuthPluginVault` in `plugin.json` to a real Entra app.

### 1. Register an Entra ID app

In the [Microsoft Entra admin center](https://entra.microsoft.com):

1. **App registrations → New registration**
   - Name: `Partner Center Incentive Claims Assistant`
   - Supported account types: *Accounts in this organizational directory only*
   - Redirect URI (Web): `https://token.botframework.com/.auth/web/redirect` (used by the M365 Copilot OAuth broker)
2. **Certificates & secrets → New client secret.** Copy the secret value.
3. **API permissions → Add a permission → APIs my organization uses → "Microsoft Partner Center"** → **Delegated permissions** → select `user_impersonation` (or the Partner Center scope your tenant exposes), then **Grant admin consent**.
4. **Expose an API / Authentication:** ensure the *Access tokens* and *ID tokens* checkboxes are enabled under **Authentication → Implicit grant and hybrid flows** if your toolkit version requires them.

### 2. Register the OAuth client in Teams Developer Portal

1. Open the [Teams Developer Portal](https://dev.teams.microsoft.com/) → **Tools → OAuth client registration → New OAuth client registration**.
2. Fill in:
   - **Name:** Partner Center Incentives
   - **Client ID:** the Entra app's Application (client) ID
   - **Client secret:** the secret you copied
   - **Authorization endpoint:** `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/authorize`
   - **Token endpoint:** `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/token`
   - **Refresh endpoint:** same token endpoint
   - **Scopes:** `https://api.partner.microsoft.com/.default offline_access`
3. Save and copy the **Registration ID** (a GUID).

### 3. Wire the registration ID into `plugin.json`

`plugin.json` references the value via `${{OAUTH2AUTHCODE_CONFIGURATION_ID}}`.
If you use the Microsoft 365 Agents Toolkit, set this in your `env/.env.dev`:

```
OAUTH2AUTHCODE_CONFIGURATION_ID=<the-registration-id-guid>
```

If you're packaging the app manually, replace `${{OAUTH2AUTHCODE_CONFIGURATION_ID}}`
in `plugin.json` with the GUID directly before zipping.

## Package and deploy

The agent ships as a Microsoft 365 app package (`.zip`) containing:

```
appPackage.zip
├── manifest.json          # Microsoft 365 app manifest (NOT this file — the M365 outer manifest)
├── color.png              # 192x192 icon (you provide)
├── outline.png            # 32x32 transparent icon (you provide)
└── declarativeAgent/
    ├── manifest.json      # the file in this folder
    ├── plugin.json
    └── openapi.yaml
```

The simplest path is the **Microsoft 365 Agents Toolkit** in VS Code:

1. Create a new project: *Custom engine agent → Declarative agent with API plugin*.
2. Replace the generated `appPackage/declarativeAgent/manifest.json`,
   `plugin.json`, and `openapi.yaml` with the three files in this folder.
3. Set `OAUTH2AUTHCODE_CONFIGURATION_ID` in `env/.env.dev`.
4. Run **Provision → Package → Publish to your tenant** from the Toolkit.
5. In **Microsoft 365 admin center → Integrated apps**, approve the app for your test users.

## Try it

After installing the agent in Microsoft 365 Copilot:

- "Show my action-required claims."
- "What's the current status of claim ID `<claim-id>`?"
- "List my Draft claims sorted by deadline."
- "Summarize my MCI engagement claims for FY26 by status."

The first call triggers an interactive sign-in prompt. After consent, Copilot
caches the token and reuses it for subsequent calls until it expires.

## Validating and extending the OpenAPI spec

Before publishing to production users:

1. Sign in to Partner Center, open browser dev tools (Network tab), and inspect
   the calls the Incentives UI makes when you load **Incentives → Claims**.
   Compare paths, query parameters, and response shapes to `openapi.yaml`.
2. If your tenant exposes claim endpoints under a different base URL (some
   partner programs use `https://api.partnercenter.microsoft.com` instead of
   `https://api.partner.microsoft.com`), update the `servers` block.
3. Add any missing fields you need surfaced (for example program-specific
   payout fields, customer association IDs for CPOR claims).
4. Re-validate with the Microsoft 365 Agents Toolkit's manifest validator.

## References

- [Declarative agent schema 1.5 / 1.6 for Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/declarative-agent-manifest-1.5)
- [API plugin manifest schema 2.3](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-manifest-2.3)
- [Configure Authentication for plugins in Agents in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/api-plugin-authentication)
- [Partner Center REST API reference](https://learn.microsoft.com/en-us/partner-center/developer/partner-center-rest-api-reference)
- [Get started with Partner Center APIs](https://learn.microsoft.com/en-us/partner-center/developer/get-started)
- [MCI engagement claims best practices](https://learn.microsoft.com/en-us/partner-center/incentives/engagement-claim-best-practices)
- [Manage incentives co-op claims](https://learn.microsoft.com/en-us/partner-center/incentives/create-incentives-claims)
