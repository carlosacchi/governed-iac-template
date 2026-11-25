# Instructions for GitHub Copilot in this repository

This repository is a governed Infrastructure as Code lab. These instructions are **guidance** for the agent: the controls that matter are the CI gates, the approval in GitHub environments, Azure Policy and resource locks. Do not present them as a security mechanism.

- Never run `terraform apply`, `terraform destroy`, `az ... delete`, `az deployment ... create` or any command that changes Azure. Changes reach Azure only through `.github/workflows/rollout.yml`.
- For workloads, change only the files in `infra/workload/teams/`. Do not change gates (`scripts/*-policy.mjs`), tests, workflows, bootstrap or state to make a change pass.
- When you change a team file, explain the expected impact on the plan. Changing `storage_account_name` forces a replacement (`-/+`) of the existing Storage account: call this out explicitly. For a new classification use `service_label`, which only becomes a tag.
- Each tenant's resource group is decided by the platform (`scripts/plan-policy.mjs`), not by the team files. Do not move resources between tenants.
- After a change, the local checks that need no credentials are `npm test` and `terraform -chdir=infra/workload validate`. Remember that a green `validate` says nothing about the impact on Azure.
