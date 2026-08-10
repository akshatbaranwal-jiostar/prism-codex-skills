# Using the `prism-deployment` Skill

This document explains how to use the Codex `prism-deployment` skill to prepare Prism deployment changes for a service. It uses the `um-token-service` Prism NPD work as the example.

## What This Skill Does

The `prism-deployment` skill helps prepare Prism deployment changes across the repositories that normally participate in a service rollout:

- Service repository: app config, build workflow, Terraform/tfvars, and any service-owned deployment files.
- `hs-k8s-app-inventory`: Kubernetes manifest overrides and IaC inventory files.
- `harness-ng-projects`: Harness service, input set, and environment override files for the `prism_service_deployment` pipeline.
- Vault: copy-plan or direct secret path creation when authenticated Vault access is available.

The skill follows a clone-then-adjust model. For `prism-npd`, it normally clones the nearest `pp` or `preprod` config and changes only confirmed environment-specific values.

## What To Provide In The Prompt

Give Codex enough context to find the service and all related repo surfaces without guessing.

Required details:

- Skill name: `$prism-deployment`
- Target environment: for example `prism-npd`
- Service name and owning team
- Service repository path or GitHub link
- `hs-k8s-app-inventory` service path or repo path
- `harness-ng-projects` service file path or repo path
- Whether Vault should be updated directly or only documented as a follow-up
- Branch names to use, if you need specific branch naming

Optional but useful details:

- Source environment to clone from, if not obvious
- Whether the service is split into multiple deployable parts
- Known Vault source and target paths
- Known DNS or endpoint requirements
- Whether PRs should be created, or only local diffs prepared

## Prompt Template

```text
$prism-deployment

Prepare <target-env> deployment changes for <service-name>.

Service:
- name: <service-name>
- team: <team-name>
- service repo: <local-path-or-github-url>

Related repos:
- hs-k8s-app-inventory path: <local-path>
- harness-ng-projects service path: <local-path>

Branching:
- service repo branch: <branch-name>
- hs-k8s-app-inventory branch: <branch-name>
- harness-ng-projects branch: <branch-name>

Vault:
- <direct Vault update required | prepare copy plan only>
- source path: <optional-source-path>
- target path: <optional-target-path>

Please clone from <source-env>, update all required repo surfaces, scan for lingering source-env values, and summarize the intentional diffs and follow-ups.
```

## Example Prompt Used For `um-token-service`

```text
$prism-deployment prism-npd changes in um-token-service and its corresponding changes in hs-k8s-app-inventory /Users/<UserName>/projects/hs-k8s-app-inventory/umsp/token-service and harness-ng-projects /Users/<UserName>/projects/harness-ng-projects/Org/prism/Projects/Identity/Services/um-token-service.yaml and other related files for prism-npd deployment
```

Branch follow-up prompt:

```text
create separate branch prism-feature/um-token-service in k8s and harness to do the changes
and branch prism-feature/config in token-service
```

## What Codex Checks

When using the skill, Codex should:

1. Confirm local repo access and current branches.
2. Create or switch to feature branches before editing.
3. Identify the source config, usually `pp` or `preprod` for `prism-npd`.
4. Check whether the service uses inventory IaC:
   - if `hs-k8s-app-inventory/<team>/<service>/infra/` exists, update IaC there;
   - otherwise update legacy Terraform in the service repo.
5. Update service config and CI workflow as needed.
6. Update `hs-k8s-app-inventory` Prism override files.
7. Update Harness Prism override/input set references.
8. Scan all three repos for lingering source environment values such as `pp`, `preprod`, old Vault paths, old tfvars names, old workspace names, and old domains.
9. Validate syntax and run relevant tests where practical.
10. Summarize changed files, intentional differences from source, and manual follow-ups.

## `um-token-service` Changes Made By This Workflow

For the `um-token-service` Prism NPD setup, the workflow updated:

- Service repo branch: `prism-feature/config`
- `hs-k8s-app-inventory` branch: `prism-feature/um-token-service`
- `harness-ng-projects` branch: `prism-feature/um-token-service`

Service repo changes:

- Added `prism-**` push trigger in `.github/workflows/build-ci.yaml`.
- Added `library/config/config_files/prism-npd.json` cloned from `pp.json` with Prism environment tokens.
- Added `terraform/tfvars/in-prism-npd-ap-south-1.tfvars` cloned from `in-pp-ap-south-1.tfvars` with Prism account, role, environment, stage, and VPC context.

Inventory changes:

- Updated `umsp/token-service/overrides/prism-npd.yaml` to mount `prism-npd.json`.
- Set config environment variables:
  - `CONFIG_FILE_MOUNT_PATH=/etc/config/env`
  - `DEFAULT_CONFIG_FILE_MOUNT_PATH=/etc/config/default`
  - `UM_TOKEN_SERVICE_CONFIG_FILE_ENV=prism-npd`

Harness changes:

- Updated Prism override values in `Org/prism/Projects/Identity/Overrides/account.prism_npd/um-token-service.yaml`.
- Pointed `TerraformVarFilePath` to `terraform/tfvars/in-prism-npd-ap-south-1.tfvars`.
- Pointed `VaultPath` to `secret/prism-npd/umsp/um-token-service/prism-npd-aps1-in`.
- Updated workspace to `tokenservice-in-prism-npd-ap-south-1`.
- Updated `APP_IAM_ROLE` to the Prism NPD assumed role.

## Vault Follow-Up

If direct Vault access is not available, ask Codex to provide a copy plan instead of writing secrets.

For this service, the target path was:

```text
secret/prism-npd/umsp/um-token-service/prism-npd-aps1-in
```

The expected process is:

1. Read the nearest valid `pp` or existing Prism source path.
2. Copy keys to the Prism target path.
3. Rename only confirmed environment-specific values.
4. Do not invent secrets or commit credentials.

## Post-Merge Checks

After PRs are merged:

- Use Harness pipeline `prism_service_deployment`.
- Confirm input set starts from `prism-npd`.
- Confirm `SkipTerraform` expectation. For first bring-up it is usually `False`.
- Confirm the target Vault path exists.
- Deploy and check pod health in Prism NPD.
- Test the health endpoint, for example:

```bash
curl -i "https://origin-hs-um-token-service.internal.prism-npd.jiostarprism.com/health"
```

## Notes And Guardrails

- Do not edit directly on `master`, `main`, or `develop`.
- Do not commit secrets, Vault tokens, API keys, or credentials.
- For `prod`, always provide the exact Prism target cluster such as `prism-mnc`, `prism-icc`, or `prism-cmd`.
- Do not copy prod config directly from `pp`; use `prism-npd` as the source for prod client clusters.
- Always scan all three repos before calling the change complete.
