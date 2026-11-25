# Hardening checklist: keeping AI agents and IaC under control

This page collects the principles behind this template, a list of checks to run on your own environment, what to enable right away and what later, and snippets to copy and adapt. The complete lab, working on Azure, is the rest of this repository.

## Contents

1. [Principles](#1-principles)
2. [Ten checks to run on your environment](#2-ten-checks-to-run-on-your-environment)
3. [Easy wins: enable right away](#3-easy-wins-enable-right-away)
4. [Later: for stronger security](#4-later-for-stronger-security)
5. [Snippets to copy and adapt](#5-snippets-to-copy-and-adapt)
6. [Sources](#6-sources)

---

## 1. Principles

1. **Decide on the plan, not on the syntax.** A green `terraform validate` or `bicep build` says nothing about the effect. The effect shows in `terraform plan` and in What-If, and that is where an automated check that can say no belongs.
2. **Blast radius is architecture, not an instruction.** What an agent (or a person) can break depends on the credentials it can reach, not on what the prompt says.
3. **A safety net inside Azure, outside the pipeline.** Locks and Azure Policy also apply to whoever goes through the portal or the command line. Sooner or later someone bypasses the pipeline.
4. **Knowing is not deciding.** An agent can know that a change is a *replace*: it reads it in the provider documentation or in a plan. That does not give it the right to do it. The decision stays with the platform and a person.
5. **MCP brings context, not judgment.** Azure MCP Server gives the agent the real state of Azure, with the credentials of whoever starts it. Use it read-only, with a pinned version and a dedicated identity.
6. **Going back means going forward.** Rolling back configuration is a new release of the good version, with the same controls. Data is different: it needs soft delete, backups and tested restores. No roll-forward brings back a deleted blob.

## 2. Ten checks to run on your environment

Each check is a question. If the answer is the "needs fixing" one, the remedy is in section 3 or 4. The commands use Azure CLI (`az`) and GitHub CLI (`gh`); replace the values in `< >`.

| # | Question | Healthy answer | How to check |
|---|---|---|---|
| 1 | Is the agent's terminal already signed in to Azure? | No | In the VS Code integrated terminal: `az account show`. If it answers with a subscription, the agent has your permissions |
| 2 | Do commands proposed by the agent run without your confirmation? | No | VS Code → Settings → search for *auto approve*: automatic command approval must be off |
| 3 | Can a PR to `main` be merged without anyone approving it? | No | `gh api repos/<owner>/<repo>/branches/main/protection --jq '.required_pull_request_reviews'`: you need at least one approval, or code owner review on critical files |
| 4 | Does the production deploy wait for approval from someone other than who started it? | Yes | `gh api repos/<owner>/<repo>/environments --jq '.environments[] \| {name, rules: .protection_rules}'`: required reviewers and *prevent self-review*. GitHub separates accounts, not people: making sure the reviewer really is another person is a team rule |
| 5 | Does something read the plan before apply and block deletes and replacements? | Yes | Look in the pipeline for a step after `terraform plan` that fails if the plan contains `delete` (snippet 5.1) |
| 6 | Does the pipeline use passwords or secrets to reach Azure? | No, it uses OIDC | Search the workflows for `client-secret`, `ARM_CLIENT_SECRET` or `AZURE_CREDENTIALS`: if they are there, you are using a secret |
| 7 | Does the same identity run plan and apply, on every environment? | No | `az role assignment list --assignee <client-id> --all -o table`: an identity with Contributor on the whole subscription is a huge blast radius |
| 8 | Who has Owner or Contributor on the whole subscription? | Only those who must | `az role assignment list --scope /subscriptions/<id> --query "[].{who:principalName, role:roleDefinitionName}" -o table`: roles inherited from above cross every boundary between teams |
| 9 | Do the resources that must not disappear have a lock? | Yes | `az lock list -g <resource-group> -o table` |
| 10 | If someone deletes blobs, can they be recovered? | Yes | `az storage account blob-service-properties show -n <storage> -g <rg> --query "{blob: deleteRetentionPolicy, container: containerDeleteRetentionPolicy}"`: soft delete must be `enabled: true` |

## 3. Easy wins: enable right away

They take a few hours, need no architecture change, and close the most common gaps.

| What | Why | Time | Snippet |
|---|---|---|---|
| **Agent terminal without `az login`** and mandatory command confirmation | The agent does not act with your permissions by mistake | 10 minutes | 5.9 |
| **Repository instructions for Copilot** | The agent flags replacements and leaves gates and pipelines alone. It is guidance, not a control | 15 minutes | 5.10 |
| **Approval on `main`** (at least one review, or code owner review on platform files) | A PR, even one written by an agent, does not get in on its own | 15 minutes | 5.5 |
| **Production environment with a reviewer and *prevent self-review*** | Whoever starts the deploy cannot approve it | 15 minutes | 5.4 |
| **`CanNotDelete` resource lock** on critical resource groups | A delete is refused, wherever it comes from | 10 minutes | 5.6 |
| **Soft delete** for blobs and containers | A data mistake becomes recoverable | 10 minutes | 5.12 |
| **Minimal gate on the Terraform plan** | The pipeline stops if the plan deletes or replaces something | 1 hour | 5.1 |
| **Review of Owner and Contributor on the subscription** | Removes inherited permissions that override everything else | 1 hour | check 8 |

## 4. Later: for stronger security

These take more work and some architecture decisions. The order is the recommended one.

1. **Identities without secrets (OIDC)** for GitHub Actions or Azure DevOps, with **separate identities for plan and apply** and one apply identity per environment or team, each limited to its resource group (snippet 5.3).
2. **The approved plan is the applied plan.** Save the plan as an artifact, apply exactly that file, and verify that it comes from the same run and the same commit (snippet 5.3; the full version in this template: [scripts/plan-artifact.mjs](../scripts/plan-artifact.mjs)).
3. **Azure Policy `DenyAction`** on deleting critical resources, with *cascade* on the resource group (snippet 5.7).
4. **A gate on What-If too** if you use Bicep or ARM: in incremental mode a rename does not delete, but it leaves an orphaned resource (snippet 5.2).
5. **Platform files have an owner.** Gates, workflows and modules have a code owner other than whoever proposes changes, or live in a separate repository (snippet 5.5).
6. **Release in waves (rings)** with a health check between one wave and the next: first one team or environment, then the others.
7. **A truly isolated agent:** a dev container or VM without mounted credentials, instead of a terminal with a different environment variable. Azure MCP Server read-only with a dedicated identity holding the Reader role (snippet 5.11).
8. **Data:** backups with tested restores, not just configured ones. Soft delete protects against a mistake, not against everything.
9. **Documented break-glass:** who can remove locks and policies, with which approval, and how it is recorded. Destructive operations must be rare and deliberate, not a button in the pipeline.

## 5. Snippets to copy and adapt

These are simplified versions of what runs in this template; the real code is in [scripts/](../scripts/), [.github/workflows/](../.github/workflows/) and [infra/](../infra/). Each snippet says what to adapt. In production, pin GitHub Actions to a commit SHA instead of a tag, as this repository's workflows do.

### 5.1 Minimal gate on the Terraform plan

Stops the pipeline if the plan deletes or replaces a resource (a replace is `delete` + `create`).

```sh
#!/usr/bin/env bash
# gate.sh: usage ./gate.sh plan.json
set -euo pipefail
plan_json="${1:-plan.json}"

destructive=$(jq -r '
  .resource_changes[]?
  | select(.change.actions | any(. == "delete"))
  | "\(.address): \(.change.actions | join("/"))"' "$plan_json")

if [ -n "$destructive" ]; then
  echo "DENY: the plan deletes or replaces resources:"
  echo "$destructive"
  exit 1
fi
echo "PASS: no delete or replace in the plan."
```

How to produce `plan.json`:

```sh
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
./gate.sh plan.json
```

**Adapt:** if some deletes are legitimate, add an allow list of addresses instead of removing the check. This template also checks resource groups, tags and security properties: [scripts/plan-policy.mjs](../scripts/plan-policy.mjs).

### 5.2 Minimal gate on Bicep / ARM What-If

```sh
az deployment group what-if \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters @main.parameters.json \
  --no-pretty-print > whatif.json

# Fails if What-If contains deletes or changes it cannot evaluate
jq -e '[.changes[] | select(.changeType == "Delete" or .changeType == "Unsupported")] | length == 0' whatif.json \
  && echo "PASS" || { echo "DENY: deletes or unevaluable changes in What-If"; exit 1; }

# Implicit rename signal: a resource is created while one of the same type stays outside the template
jq -r '.changes[] | select(.changeType == "Create" or .changeType == "Ignore") | "\(.changeType) \(.resourceId)"' whatif.json
```

**Adapt:** in incremental mode, resources outside the template show up as `Ignore`, not `Delete`. A `Create` plus an `Ignore` of the same type is often a rename: this template turns it into `DENY` ([scripts/whatif-policy.mjs](../scripts/whatif-policy.mjs)), but tune the rule to your case.

### 5.3 GitHub Actions: plan, gate, approval, apply of the same plan, no secrets

```yaml
name: deploy
on: workflow_dispatch

permissions:
  contents: read
  id-token: write          # required for OIDC

env:
  ARM_USE_OIDC: "true"
  ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}

jobs:
  plan:
    runs-on: ubuntu-latest
    env:
      ARM_CLIENT_ID: ${{ vars.AZURE_PLAN_CLIENT_ID }}     # read-only identity
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -out=tfplan
      - run: terraform show -json tfplan > plan.json
      - run: ./gate.sh plan.json                          # snippet 5.1
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    environment: production        # required reviewer + prevent self-review (snippet 5.4)
    env:
      ARM_CLIENT_ID: ${{ vars.AZURE_APPLY_CLIENT_ID }}    # apply identity, limited to its resource group
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - run: terraform init
      - run: terraform apply tfplan                       # applies exactly the approved plan
```

**Adapt:** the two identities are managed identities (or app registrations) with a *federated credential* for the repository: the plan one on the `main` branch, the apply one on the `production` environment. Configure the state backend with Entra ID authentication. The full version, with verification of run, commit and plan sha256, is in [.github/workflows/tenant-ring.yml](../.github/workflows/tenant-ring.yml).

### 5.4 Environment with mandatory approval

In GitHub: *Settings → Environments → New environment* (`production`):

- **Required reviewers:** a person or a team, with **Prevent self-review** enabled;
- **Allow administrators to bypass:** off;
- **Deployment branches:** `main` only.

Check:

```sh
gh api repos/<owner>/<repo>/environments --jq '.environments[] | {name, admin_bypass: .can_admins_bypass, rules: [.protection_rules[] | {type, prevent_self_review, reviewers: [.reviewers[]?.reviewer.login]}]}'
```

### 5.5 Protecting platform files: CODEOWNERS and branch protection

`.github/CODEOWNERS`:

```text
# Platform files have an owner other than whoever proposes changes
/.github/          @<org>/platform-team
/scripts/          @<org>/platform-team
/infra/modules/    @<org>/platform-team

# Team configurations deliberately have no owner: gates and deploy approval decide
/infra/teams/
```

Enable code owner review on the `main` branch:

```sh
gh api -X PATCH repos/<owner>/<repo>/branches/main/protection/required_pull_request_reviews \
  -F require_code_owner_reviews=true -F dismiss_stale_reviews=true
```

**Adapt:** do not set a catch-all `*` owner if whoever opens PRs is also the only owner: nobody can approve their own PR, so every PR would be blocked. If branch protection does not exist yet, create it first from *Settings → Branches*.

### 5.6 Resource lock

```sh
az lock create \
  --name no-delete \
  --lock-type CanNotDelete \
  --resource-group <rg> \
  --notes "Deletes only through an approved procedure"
```

The lock applies to the group and everything in it, even for an Owner: to delete, you must remove it first, which makes deletion a deliberate act.

### 5.7 Azure Policy DenyAction: nobody deletes critical resources

`rules.json`:

```json
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
      { "field": "tags['critical']", "equals": "true" }
    ]
  },
  "then": {
    "effect": "[parameters('effect')]",
    "details": {
      "actionNames": ["delete"],
      "cascadeBehaviors": { "resourceGroup": "deny" }
    }
  }
}
```

`params.json`:

```json
{
  "effect": {
    "type": "String",
    "allowedValues": ["DenyAction", "Disabled"],
    "defaultValue": "DenyAction"
  }
}
```

```sh
az policy definition create --name deny-delete-critical-storage --mode Indexed \
  --rules rules.json --params params.json
az policy assignment create --name deny-delete-critical-storage \
  --policy deny-delete-critical-storage \
  --scope /subscriptions/<id>/resourceGroups/<rg>
```

**Adapt:** change the resource type and the condition. Beware: if protection depends on a tag, whoever can change that tag can take the resource out of the policy. On dedicated resource groups, protect every resource of that type, with no tag condition. The policy protects the *control plane*: it does not prevent deleting the data inside the resource.

### 5.8 Terraform `prevent_destroy`

```hcl
resource "azurerm_storage_account" "state" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

Terraform rejects any plan that destroys this resource. Ideal for state and for resources that must never disappear. It only applies to Terraform users: locks and policy (5.6, 5.7) also protect against other tools.

### 5.9 Agent terminal without Azure credentials (VS Code)

In your VS Code user settings (`settings.json`), the line for your operating system:

```jsonc
"terminal.integrated.env.osx":     { "AZURE_CONFIG_DIR": "/tmp/agent-no-azure" },
"terminal.integrated.env.linux":   { "AZURE_CONFIG_DIR": "/tmp/agent-no-azure" },
"terminal.integrated.env.windows": { "AZURE_CONFIG_DIR": "C:\\Temp\\agent-no-azure" }
```

In the integrated terminal, `az account show` must ask you to run `az login`. This is a precaution, not a boundary: the terminal still runs as your user. For real isolation use a dev container or a VM without mounted credentials.

### 5.10 Repository instructions for Copilot

`.github/copilot-instructions.md`:

```markdown
# Instructions for GitHub Copilot

- Never run `terraform apply`, `terraform destroy` or `az ... delete` commands:
  changes reach Azure only through the pipeline.
- Change only team configurations. Do not change gates, tests or workflows
  to make a change pass.
- When you change infrastructure, explain the expected impact on the plan and
  explicitly flag every replacement (`-/+`) or deletion.
- A green `terraform validate` says nothing about the impact on Azure.
```

These are guidance for the agent, not a control: gates and approval still decide.

### 5.11 Azure MCP Server read-only (VS Code)

`.vscode/mcp.json`:

```json
{
  "servers": {
    "azure": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@azure/mcp@<version>", "server", "start",
               "--read-only", "--namespace", "storage", "--namespace", "policy"]
    }
  }
}
```

**Adapt:**
- pin the version (`npm view @azure/mcp version` shows the latest) instead of using `@latest`;
- expose only the namespaces you need;
- the server uses the credentials of whoever starts it: if an agent uses it, give it a dedicated identity with the Reader role, as the official Copilot cloud agent setup does (`azd coding-agent config`).

### 5.12 Soft delete for blobs and containers

```sh
az storage account blob-service-properties update \
  --account-name <storage> --resource-group <rg> \
  --enable-delete-retention true --delete-retention-days 14 \
  --enable-container-delete-retention true --container-delete-retention-days 14
```

A blob or container deleted by mistake stays recoverable for 14 days. It does not replace a backup with a tested restore.

## 6. Sources

- Incident post-mortems behind these controls: [docs/case-study.md](case-study.md).
- Microsoft Learn: [Azure resource locks](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources), [Azure Policy DenyAction effect](https://learn.microsoft.com/azure/governance/policy/concepts/effect-deny-action), [ARM What-If](https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-what-if), [Blob soft delete](https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview), [Container soft delete](https://learn.microsoft.com/azure/storage/blobs/soft-delete-container-overview), [OIDC from GitHub Actions to Azure](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect), [Azure MCP Server](https://learn.microsoft.com/azure/developer/azure-mcp-server/).
- Terraform: [`prevent_destroy` lifecycle meta-argument](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle).
- GitHub: [deployment environments](https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment), [CODEOWNERS](https://docs.github.com/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners), [Copilot coding agent risks and mitigations](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/risks-and-mitigations).

---

Questions, reports or improvements: open an issue in this repository.
