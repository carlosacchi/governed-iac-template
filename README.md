# Governed IaC template

A template for letting people **and AI coding agents** change Azure infrastructure as code without letting them decide on their own what reaches Azure.

It addresses one failure mode documented in public post-mortems: a change that is correct in form but wrong in scope. A rename turns into a replacement, or a cleanup reaches the wrong resource group. Several independent layers each assume the previous one can fail. Background and references are in [docs/case-study.md](docs/case-study.md). To apply the same ideas to your own environment, start from the [hardening checklist](docs/hardening-checklist.md): ten checks, quick wins and snippets to copy.

> The example values use the fictional `contoso` prefix and two tenants, `sales` and `operations`, in `italynorth`. Replace them before use: see [Adopting the template](#adopting-the-template).

## What it implements

| Capability | Where |
|---|---|
| Platform engineering | Self-service per team ([team files](infra/workload/teams/)). Gates, approval, identities and policy belong to the platform, not to whoever writes the change |
| Multi-tenant (simulated) | One resource group per tenant, with a separate state container and a separate apply identity per tenant ([bootstrap](infra/bootstrap/main.tf)) |
| Terraform | [Workload](infra/workload/main.tf) built on one [Storage module](infra/modules/workload-storage/main.tf), with [versioned team files](infra/workload/teams/) |
| Bicep | An [equivalent template](infra/compare/storage.bicep) used **only** for ARM What-If, never applied |
| Plan / what-if | A saved `terraform plan` and `az deployment group what-if --validation-level ProviderNoRbac` on the same tenant scope |
| Policy gates | [Terraform gate](scripts/plan-policy.mjs) and [What-If gate](scripts/whatif-policy.mjs): both must allow the change |
| Policy as code in Azure | Azure Policy `Deny` (public Storage) and `DenyAction` (delete of the lab Storage accounts) on every tenant resource group |
| Canary deploy | [rollout.yml](.github/workflows/rollout.yml): ring 0 (`sales`), then ring 1 (`operations`) only if the canary is healthy |
| Human approval | The apply job is bound to the tenant's GitHub environment (required reviewer, no self-review) |

Each ring runs [tenant-ring.yml](.github/workflows/tenant-ring.yml):

```text
plan (plan MI)                       apply (tenant MI)                   verify (plan MI)
├─ RG from the gate, not team files  ├─ environment = tenant → human     ├─ Storage private and tagged
├─ saved terraform plan              ├─ same run, sha256, commit         ├─ CanNotDelete lock present
├─ Bicep What-If (ProviderNoRbac)    ├─ gates re-checked                 ├─ DenyAction assigned and enforced
├─ Terraform gate + What-If gate     └─ apply of THAT plan               └─ plan -detailed-exitcode = 0
├─ summary for the reviewer
└─ artifact bound to the run
```

## Layers of defense

Each layer assumes the previous one can fail, including when an agent or a person edits the code.

1. **Plan gates** reject deletes and replacements (`-/+`), resource groups outside the tenant, non-compliant properties, and implicit renames in incremental Bicep (which create a new Storage account and leave the original orphaned). The tenant→resource group map lives in the gate, not in the team files. Platform files (gates, workflows, tests, bootstrap, modules) have an owner in [CODEOWNERS](.github/CODEOWNERS): with *Require review from Code Owners* enabled on `main`, admins included, a PR that touches them waits for the platform reviewer. Team files deliberately have no owner.
2. **Human approval** in the tenant's environment, by a reviewer other than whoever started the rollout. Before approving, the reviewer reads the run summary: gate results, add/change/destroy counts, affected resources, the full plan and the SHA256 of the saved plan.
3. **Least-privilege identities**: the `sales` apply identity has *Storage Account Contributor* only on the `sales` resource group and writes only to `tfstate-sales`. The plan identity is Reader on the resource groups, has a custom role limited to `whatIf`/`validate`, and writes to the tenant state containers because plan takes the state lock (a lab simplification: in production, split it per tenant).
4. **Azure Policy `DenyAction`** on deleting the lab Storage accounts, including resource group cascade. It holds even if someone bypasses the pipeline.
5. **`CanNotDelete` resource locks** on the tenant resource groups and on the state Storage account, plus `prevent_destroy` in Terraform.

Resource group names are constrained to `contoso-*` (and never the shared `contoso-iac` group) in the bootstrap, the workload and the gate, so that the lab cannot touch other workloads in the same subscription.

## Running a rollout

```sh
# 1. Canary: plan + what-if + gates on sales; apply waits for approval in the "sales" environment
gh workflow run rollout.yml --ref main -f scenario=baseline -f rings=canary-only

# 2. Widen: sales again (no changes expected), then operations after the health check
gh workflow run rollout.yml --ref main -f scenario=baseline -f rings=all

# 3. Deterministic rename scenario: both gates must answer DENY, with no apply
gh workflow run rollout.yml --ref main -f scenario=rename-demo -f rings=canary-only
```

A typical walkthrough:

1. Ask an agent to rename the `sales` Storage account in its [team file](infra/workload/teams/sales.tfvars.json). The PR passes `validate`: `terraform validate` does not see the problem.
2. After merge, run the baseline rollout. The plan shows `-/+` and the What-If shows a new Storage account with the original ignored. **Both gates answer DENY**, the job fails before the artifact is uploaded, and the environment is never reached.
3. Ask the agent to fix it: keep the original physical name and put the new classification in `service_label` (a tag only). The new rollout is an in-place tag update: approval in the environment, apply, health check, then ring 1.
4. Safety net: even a manual `az storage account delete` is refused, first by the lock and, once the lock is removed, by the DenyAction.

`rename-demo` reads the same file and appends `r` through an explicit override. It is a repeatable fallback, not a diff written by an agent.

## Reset and rollback

The starting state is versioned in [demo/baseline/](demo/baseline/): both Storage accounts exist with the baseline configuration. The walkthrough only changes team files and tags, so the reset **deletes nothing**: it rolls forward to the baseline through the same pipeline.

```sh
npm run demo:status   # drift between team files and baseline, open team PRs, unfinished rollouts
npm run demo:reset    # PR that restores the files → CI validate → merge → baseline rollout rings=all
```

- It is idempotent: if files and Azure are already aligned it opens no PR, and the rollout skips apply (only the health and drift check runs).
- If Azure differs, for example because the `serviceLabel` tag was changed, every ring asks for approval in its environment like any other change. Rollback is governed too.
- On an empty lab the first run creates the baseline Storage accounts, so the same command also seeds the lab.
- Rollouts are serialized: if a run is waiting for approval, the reset waits in the queue. If branch protection requires an approval, the script stops after CI and tells you how to continue (`node scripts/demo-reset.mjs rollout` after the merge).

Deleting the Storage accounts (teardown) is a break-glass platform operation: first set the DenyAction to `effect=Disabled` and remove the lock, always with explicit approval.

## Local checks

Prerequisites: Node.js 22+, Terraform 1.14.9 (as in CI) and Azure CLI 2.76+ with Bicep. No Azure credentials are needed.

```sh
npm test
terraform fmt -check -recursive infra
terraform -chdir=infra/bootstrap init -backend=false -lockfile=readonly
terraform -chdir=infra/bootstrap validate
terraform -chdir=infra/bootstrap test
terraform -chdir=infra/workload init -backend=false -lockfile=readonly
terraform -chdir=infra/workload validate
terraform -chdir=infra/workload test -verbose -json > terraform-test-events.jsonl
node scripts/check-mock-plans.mjs terraform-test-events.jsonl
az bicep build --file infra/compare/storage.bicep --stdout > /dev/null
```

## Adopting the template

Repository settings are not part of the code: you configure them once per repository.

1. **Example values.** Replace the `contoso` prefix, the project tag, the region and the tenant names in the bootstrap, the workload, the Bicep template, the gates, the workflows, the tests and the team files. Give each tenant a globally unique Storage account name.
2. **CODEOWNERS.** Replace `@YOUR-ORG/platform-team` in [.github/CODEOWNERS](.github/CODEOWNERS) with a team or account other than the people who author changes.
3. **Bootstrap** (platform operator, run locally, never by CI):
   1. Copy [terraform.tfvars.example](infra/bootstrap/terraform.tfvars.example) to `terraform.tfvars` (ignored by git). Get the OIDC subject prefix from the [oidc-subject.yml](.github/workflows/oidc-subject.yml) workflow.
   2. If the local state was lost, regenerate the imports of existing resources with `node scripts/bootstrap-imports.mjs <state-storage-account>`. The generated `imports.generated.tf` is ignored by git.
   3. `terraform -chdir=infra/bootstrap plan -out=bootstrap.tfplan`. The expected plan contains no `destroy`. Apply only after review.
4. **Repository variables**: `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, `AZURE_PLAN_CLIENT_ID`, `TF_STATE_ACCOUNT`. There are no client secrets: every identity is federated through OIDC.
5. **One GitHub environment per tenant** (`sales`, `operations`) with:
   - a required reviewer and *prevent self-review*;
   - deployments from `main` only;
   - the variable `AZURE_APPLY_CLIENT_ID` set to the bootstrap output `apply_client_ids[<tenant>]`. The apply job refuses a missing ID or one equal to the plan ID.
6. **Branch protection on `main`**: require pull requests, at least one approval, *Require review from Code Owners*, the `validate` check, and apply the rules to administrators too.

## Known limits

- **One Entra directory and one subscription.** Tenants are resource groups with separate identities, state and policy: the isolation of an internal multi-team platform, not of a multi-tenant SaaS. Anyone with roles inherited on the subscription, such as Contributor, bypasses it.
- **The canary is per tenant (ring)**, not a traffic split: Storage has no traffic to route.
- **What-If is a preview.** With `ProviderNoRbac` it does not check write permissions. Confirm the custom role (`whatIf`/`validate` plus Reader) on the first real run.
- **DenyAction only covers `delete`.** Replacing a workload Storage account is stopped by the gate, not by policy: `prevent_destroy` only protects the state and the tenant resource groups. Locks and DenyAction make cleanup deliberate too: remove them first, then delete, always with approval.
- The state backend uses a public endpoint authenticated with Entra ID, without Shared Key, and a named exception to the network policy. Workloads have no Private Endpoint: the lab demonstrates control-plane governance, not data access.
- In a public repository, code, logs and artifacts are public. Use only synthetic resources and data.
- `CODEOWNERS` and branch protection do not replace the environment approval, and repository admins can reconfigure them.
- [.github/copilot-instructions.md](.github/copilot-instructions.md) are guidance for the agent, not a control: the gates decide regardless.

## License

[MIT](LICENSE)
