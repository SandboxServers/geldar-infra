# Release identity: geldar-bot and the Scriptorium release workflow

How the `geldar-scriptorium` release workflow gets permission to publish the template, without any long-lived secret in GitHub. Campaign decision D-SC-02 in `geldar-scriptorium` (`docs/analysis/scriptorium/README.md`).

## Pieces

| Piece | Where | What it is |
|---|---|---|
| `geldar-bot` | GitHub App owned by the SandboxServers org | Installed on **`geldar-player-template` only**, with repository contents read/write and metadata read. No webhook. Its installation token is what pushes a kit release to the template repo. |
| `rg-geldar-infra` | Azure, North Central US | Resource group for geldar infrastructure (`project=geldar`). |
| `kv-geldar-infra` | Key Vault in `rg-geldar-infra` | Access through Azure roles, purge protection on, 90-day soft delete. Holds `geldar-bot-private-key` (PEM) and `geldar-bot-app-id`. |
| `geldar-github-release` | Entra ID app registration and its service principal | The identity GitHub Actions signs in as. It has **Key Vault Secrets User** on those two secrets only, not on the vault. |

## The trust

`geldar-github-release` has one flexible federated credential, `geldar-scriptorium-kit-release-tags`. It trusts GitHub's OIDC issuer (`https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`) only when all three of these hold:

- `sub` matches `repo:SandboxServers/geldar-scriptorium:ref:refs/tags/kit-v*`, meaning a workflow running for a `kit-v*` tag;
- `repository_id` equals the numeric ID of `geldar-scriptorium`;
- `repository_owner_id` equals the numeric ID of the SandboxServers org.

The two numeric IDs mean a renamed or recreated repo with the same name can't inherit the trust. Branch runs and pull requests can't sign in at all.

## How the release job uses it

`geldar-scriptorium` has repository **variables**, not secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` and `KEY_VAULT_NAME`. The release workflow, on a `kit-v*` tag:

1. requests `permissions: id-token: write`, then signs in with `azure/login` using those variables (no client secret);
2. reads `geldar-bot-app-id` and `geldar-bot-private-key` from the vault, and masks them in the logs;
3. exchanges them for a short-lived installation token, for example with `actions/create-github-app-token`, pinned by SHA;
4. pushes the release commit to `geldar-player-template` with that token.

The private key never sits in GitHub, and the installation token expires within an hour.

## Operating it

- **Rotate the key:** in the GitHub App's settings, generate a new private key, then `az keyvault secret set --vault-name kv-geldar-infra -n geldar-bot-private-key --file <pem>`. Delete the old key in GitHub and the local PEM file.
- **Revoke publishing at once:** suspend the `geldar-bot` installation in the org's GitHub App settings, or remove the federated credential from `geldar-github-release`.
- **Widen what the bot may do** (for example births, lineage or archival under spec open decision 2f): change the app's permissions and install it on the extra repositories deliberately, then record that as a decision.
- **Created:** 2026-11-08, by the planning session in Claude Code. The app was made through GitHub's app-manifest flow, so the private key went straight from GitHub into the vault.
