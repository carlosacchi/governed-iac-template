# Case study: the wrong scope, executed correctly

The lab does not invent a scenario. It reproduces, at small scale, a failure mode documented in public post-mortems: **a change that is legitimate in form affects more than the person approving it believes**, and no layer stops it before the irreversible effect.

## Reference cases

### Atlassian, April 2022 (main case)

[Official Post-Incident Review](https://www.atlassian.com/engineering/post-incident-review-april-2022-outage).

- A maintenance script was meant to remove a legacy app (Insight). It received **site IDs** instead of app IDs and deleted them permanently.
- Result: **883 sites (775 customers)** deleted between 07:38 and 08:01 UTC. For some customers, restoration took **up to 14 days**.
- The PIR notes that the script did not check whether the IDs referred to the app or to the whole site. There was a peer review, but it *"focused on which endpoint was being called and how"*; testing in staging could not catch it because those IDs did not exist there. Among the corrective actions, it lists universal soft deletes and, for deletions of this kind, multiple layers of protection, including *staged rollout* and tested rollback.

### Azure DevOps, May 2023 (Azure case)

[Official post-mortem](https://status.dev.azure.com/_event/392143683/post-mortem) ([summary by The Register](https://www.theregister.com/software/2023/06/03/typo-blamed-for-microsoft-azure-devops-outage-in-brazil/508301)).

- A large PR migrated packages from `Microsoft.Azure.Management.*` to `Azure.ResourceManager.*`. Inside it, a typo turned "delete the snapshot database" into "**delete the Azure SQL Server** that hosts it".
- The internal **Ring 0** passed without problems because it contained no snapshot: the job had nothing to delete. In **Ring 1**, in South Brazil, there was a snapshot old enough to trigger the job: the server and **17 production databases** were deleted, with about 10 hours of outage.
- Among the fixes: tests covering the whole job lifecycle and **resource locks** on production SQL servers.

### Agentic cases (same actor, three different perimeters)

| Case | What happened | Lesson |
|---|---|---|
| [PocketOS, April 2026](https://www.theregister.com/software/2026/04/27/cursor-opus-agent-snuffs-out-startups-production-database/5224442) | A coding agent working in staging finds a token in an unrelated file. The token, with broader permissions than expected, lets it delete, with a single API call, a volume holding the production database and the backups stored on the same volume. Railway states it also had disaster recovery backups, from which it later recovered the data. | The credentials an agent can reach are its real perimeter; deletes need a boundary that does not depend on the prompt. |
| Replit / SaaStr, July 2025 | An agent deletes a production database during a declared *code freeze*. | A natural-language instruction is not a control. |
| [Amazon Q Developer, July 2025 (bulletin AWS-2025-015)](https://aws.amazon.com/security/security-bulletins/AWS-2025-015/) | A GitHub token with excessive permissions in the CodeBuild configuration allows malicious code to be inserted into the VS Code extension, released automatically in version 1.84.0. The code did not run only because of a syntax error. | The agent's supply chain is part of the attack surface. |

## From post-mortem to lab

The lab simulates an internal platform with two tenants, Sales and Operations. It is the situation of a large multi-team organization, but also of a single team with separate roles: who writes code, who approves, who runs the platform.

| Observed failure mode | Where it shows in the cases | Control in the lab | Type |
|---|---|---|---|
| The real scope is wider than the perceived one | Atlassian (site instead of app), ADO (server instead of DB) | The gate resolves the RG from the tenant and rejects any resource out of scope. The tenant→RG map is not in the team files, and the platform files have a code owner other than whoever proposes the change. | Preventive, CI |
| The change "looks like" an update but implies a destruction | ADO (typo in a huge PR), rename in Terraform | `terraform plan` + gate: `-/+` and delete → DENY. Bicep What-If + gate: implicit rename → DENY. | Preventive, CI |
| Syntax validation passes | ADO (tests did not cover the condition) | `terraform validate` passes: the decision is based on the **plan**, not on syntax. | Educational |
| The canary does not hit the critical condition | ADO (Ring 0 without snapshots) | Ring 1 starts only if ring 0 passes the health check: private Storage, lock, DenyAction and no drift. The gates run **in every ring**, not only in the canary. | Detective + preventive |
| Credentials that are too broad | PocketOS, Atlassian | Plan MI is Reader with a custom role limited to `whatIf`/`validate`, plus write access to the state containers (a lab simplification). One apply MI per tenant, limited to its RG and its state container. | Least privilege |
| No human between the plan and the effect | All | One GitHub environment per tenant: required reviewer, no self-review, `main` only. The approved artifact is bound to run, commit and sha256. | Approval |
| The delete arrives by another route | PocketOS (direct API) | Azure Policy `DenyAction` and a `CanNotDelete` lock on the RG, independent of the pipeline. | Platform |

The table links failure modes to lab controls. It does not claim that the lab's gate would have stopped those cases: none of them went through a Terraform plan. For that, see the next section.

## In hindsight: which layer would have stopped them

These assessments are the template authors', not the post-mortems': they reason about the fixes the companies chose and about the layers implemented in the lab. They hold by analogy, since only Azure DevOps was already on Azure, and in hindsight: these teams also had controls that seemed sufficient.

| Case | Lab gate alone | With all layers | Thanks to | Caveat |
|---|---|---|---|---|
| Atlassian | No: there was no Terraform plan | Probably avoided | An approval in front of a preview of the effect (type and number of objects); soft delete, their fix, would have shortened restoration | There was a peer review, but it looked at the code |
| Azure DevOps | No: the job decided the delete at run time | Stopped | Resource lock on the SQL server, the fix Microsoft chose | Review and canary were in place |
| PocketOS | No: direct API call, outside any pipeline | Stopped | Token limited to staging, platform-side delete protection, backups on a separate volume | The protection must come from the platform: the endpoint used had no delayed deletion |
| Replit | No: no pipeline | Stopped | Separate development and production, the fix Replit announced | At the time Replit did not separate the two environments |
| Amazon Q | No: not infrastructure | Probably stopped | Least-privilege build token, mandatory review, release approval | Depends on what that token allowed |
| CrowdStrike | No: their gate is exactly what failed | Reduced, not avoided | Staggered rollout with a canary | No person would have spotted the error by reading the content file |

The lab's gate stops one precise class of errors: those visible in the plan. Identities, policy, locks and gradual releases limit the others. Together they reduce risk; they do not eliminate it.

### CrowdStrike, July 2024

- On 19 July 2024 at 04:09 UTC, CrowdStrike released a content update (Rapid Response Content, Channel File 291) for the Windows sensor; it reverted it at 05:27 UTC ([Preliminary Post Incident Review](https://www.crowdstrike.com/en-us/blog/falcon-content-update-preliminary-post-incident-report/)).
- The PIR reports that, due to a bug in the Content Validator, one of the two Template Instances passed validation despite containing problematic content. The instances were released based on the Template Type tests, trust in the Content Validator checks, and previous successful releases.
- The sensor code had a staged rollout (internal use, early adopters, general availability); the fixes for content include a staggered rollout starting from a canary, with more control for customers. The PIR does not explain why content was not already staggered. The [root cause analysis](https://crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf) adds bounds checking in the sensor.
- Microsoft estimates about 8.5 million Windows devices were affected ([Microsoft, 20 July 2024](https://blogs.microsoft.com/blog/2024/07/20/helping-our-customers-through-the-crowdstrike-outage/)); CrowdStrike's technical report gives no count.
- Here it is an analogy, not the same pattern: no agents, IaC or Azure are involved. The lesson is that even a green check can be wrong, and a later layer is needed.

## Copilot: what it already covers and what it does not

The [GitHub Copilot coding agent](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/risks-and-mitigations) already offers important mitigations:

- it works only on its own branches;
- it cannot approve or merge its own PRs;
- workflows on its PRs require approval;
- it has a configurable firewall.

The rule "whoever requested the PR cannot approve it" only takes effect **if branch protection requires at least one approval**. None of these protections knows the Azure scope, the difference between update and replace, or the right tenant: that is the platform's job. The lab's principle is that **the control does not depend on who writes the change**, person or agent.

## Honest limits of the simulation

- One Entra directory and one subscription: isolation is by RG, identity, state and policy. Anyone with roles inherited on the subscription bypasses it.
- The canary is per tenant ring, not a traffic split.
- The DenyAction only covers `delete`. Replacing a workload Storage account is stopped by the gate; `prevent_destroy` only protects the state and the tenant RGs.
