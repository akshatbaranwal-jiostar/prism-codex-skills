---
name: prism-deployment
description: Prepare and raise pull requests for deploying JioHotstar or Prism services to Prism clusters, and optionally create or update the required Vault paths and keys when direct Vault access is available. Use when Codex needs to add or update Prism deployment config for a service in the service repo, `hs-k8s-app-inventory`, `harness-ng-project`, or Prism Vault, especially for requests like "deploy X to prism-npd", "add Prism config for service Y", "prepare PR for Prism", "copy pp Vault config to prism-npd", or "deploy X to prod Prism". Treat `prism-npd` as the default non-prod target. If the user says only "prod", require a specific target cluster such as `prism-mnc`, `prism-icc`, or `prism-cmd` before making changes, modifying Vault, or raising PRs.
---

# Prism Deployment

Prepare repository changes and raise PRs for Prism deployments.

## Work in access-aware mode

Check what is actually available before promising a PR:

- **Git repo access**: if the needed repositories are checked out locally or accessible through a connected tool, edit them directly and create PR-ready branches/commits if the user wants that.
- **GitHub / PR access**: if a connected GitHub tool is available, open PRs on the user's behalf after showing the proposed diff summary. If PR creation is not available, prepare exact branch names, commit messages, and PR titles/bodies for the user.
- **Harness / Vault / k8s dashboard**: treat these as reference systems unless a connected tool, authenticated CLI, or other direct access method is available.

State the mode explicitly for each step: "editing locally", "preparing a diff only", "ready to raise PR but lacking GitHub access", or "Vault read/write available".

Never store Vault tokens or other live credentials in this skill. Accept them only as runtime inputs for the current run.

Before cloning any repo, check whether the repo already exists locally. If it does, reuse that checkout instead of cloning again.

## Step 1: Collect the minimum inputs

Ask for or confirm these inputs before editing:

1. Service name and owning team.
2. Target environment.
3. Service repository GitHub link.
4. `hs-k8s-app-inventory` repository link or local path.
5. `harness-ng-project` repository link or local path.
6. Whether direct Vault access is available, and if so, which auth method or tool should be used.

For runtime Vault tokens, use these aliases:

- `prism-vault`: token for `https://vault.prism-cmd.jiostarprism.com/`
- `hotstar-vault`: token for `https://vault-incubator.hotstar-labs.com/`

Treat these as runtime-only values. Do not write them into skill files, repository files, PR bodies, or logs.

If the user says only "prod", stop and ask which Prism cluster they mean, for example `prism-mnc`, `prism-icc`, or `prism-cmd`.

If the user requests `prism-npd`, proceed as non-prod unless they say otherwise.

If the service is split into multiple deployable parts, repeat the full workflow for each part and report each part separately.

## Step 2: Decide the source config chain

Choose the source environment in this order:

- For a new `prism-npd` setup, clone from `pp` / `pre-prod`.
- For a new prod client cluster such as `prism-mnc` or `prism-icc`, clone from the service's existing `prism-npd` setup.
- Do not clone prod config directly from `pp`.

Treat the change as clone-then-adjust. Copy the nearest valid source config first, then only change confirmed environment-specific values.

## Step 3: Update the required repositories

For each service part, check which repositories need changes:

- **Service repo**: add or update service-level config, Terraform or IaC files, app config, and any code-level Prism branches required for the service.
- **`hs-k8s-app-inventory`**: add or update Prism environment files in the service inventory and infra folder when the service uses IaC Terraform.
- **`harness-ng-project`**: add or update the service's Prism input set or deployment config so Harness points at the correct env and Vault paths.
- **Vault**: create or update the target Prism secret path and keys if direct Vault access is available. Otherwise prepare the exact source path, target path, and key copy plan for the user.

Before finalizing any Prism change, always scan all three repo surfaces for lingering source-env rules and values:

- the service repo
- `hs-k8s-app-inventory`
- `harness-ng-project`

Do not assume the required rename is confined to one repo. Check for old env markers, Vault paths, tfvars filenames, workspace names, override manifest paths, and Prism URL/domain suffixes across all three, then update every confirmed Prism-targeted occurrence.

Also inspect the service repo CI workflow such as `.github/workflows/build-ci.yaml`. If branch-trigger filters are present, add a Prism branch pattern such as `prism-**` so pushes to Prism feature branches still generate the image tag or manifest artifacts needed by Harness.

Use the migration rule from `references/technical-details.md`:

- If `hs-k8s-app-inventory/<team>/<service>/infra/` exists, treat the service as migrated to IaC Terraform and update IaC files there.
- If it does not exist, update the legacy Terraform files in the service repo, for example `prism-npd.tfvars`, `prism-mnc.tfvars`, or the matching target-cluster tfvars file.
- When the source legacy tfvars uses the market-region format, preserve that shape and replace only the environment segment. Example: `in-pp-ap-south-1.tfvars` -> `in-prism-npd-ap-south-1.tfvars`.

Before finalizing, diff against the source environment and call out every intentional line change.

## Step 3A: Manage local repos and branches safely

When local repo access exists:

- Check whether each required repo is already present locally before cloning.
- If a repo already exists locally, use that checkout.
- Do not make changes directly on `main`, `master`, or `develop`.
- If the current branch is `main`, `master`, or `develop`, create and switch to a feature branch before editing.

Preferred branch names:

- For common repos such as `hs-k8s-app-inventory` and `harness-ng-project`, use `prism-feature/<service-short-name>`.
- For the service repo, use an env-specific branch such as `prism-npd-feature/config` or the matching target-specific variant such as `prism-mnc-feature/config`.

Reuse an existing suitable feature branch when it already represents the same in-progress Prism change. Otherwise create a fresh branch before editing.

## Step 4: Apply Prism naming and value rules

- Rename `/pp/` or `/preprod/` path segments to `/prism-npd/` or `/prism-<client>/`.
- Rename `-pp` or `-preprod` suffixes and prefixes to the target Prism environment.
- In cloned tfvars values, also rename trailing env tokens such as `-pp`, `-pp2`, and `/pp2-` to the matching Prism form, for example `user-umfnd-in-pp2` -> `user-umfnd-in-prism-npd`.
- For Vault secret leaf names, preserve the original segment order and replace only the environment token. Example: `pp2-aps1-in` -> `prism-npd-aps1-in`, not `in-prism-npd-ap-south-1`.
- In `hs-k8s-app-inventory` override values, replace env tokens everywhere they appear as delimited tokens, not just at the start or end of the value. This includes forms such as `.pp.`, `.qa.`, `-pp-`, `-qa-`, `-pp`, `-qa`, `pp-`, `qa-`, `pp1`, `pp2`, and `preprod` when they are clearly environment markers inside topics, hosts, namespaces, client ids, file paths, or resource names.
- Do not rename non-environment business values that merely resemble env tokens, for example country or market codes such as `qa` in `config-qa.json`, `/usr/configs/qa/`, `SUPPORTED_COUNTRIES`, or `LAUNCH_DARKLY_ENABLED_COUNTRIES`.
- For Prism NPD URLs and hostnames, normalize service host suffixes to `.prism-npd.jiostarprism.com`. Replace older endings such as `.preprod.hotstar.com`, `.pp.hotstar.com`, and any intermediate `.prism-npd.hotstar.com` form with the Prism domain when the value is an environment-specific service URL or hostname.
- Update explicit environment identifiers such as `ENV`, stage, cluster, Vault path, and DNS hostnames.
- Preserve secrets byte-for-byte unless the value must be regenerated for the new environment.
- If a value is unknown but safely clonable, carry it forward from the source config after applying the required environment rename.
- If a value needs a real decision and cannot be inferred safely, stop and ask.

Read `references/technical-details.md` when editing Terraform, Vault, DNS, Harness, or IAuth-dependent services.

## Step 4A: Handle Vault path creation and copying

When the user wants Vault updated as part of the deployment workflow:

- For `prism-npd`, source values may come from:
  - `https://vault-incubator.hotstar-labs.com/`, for example `secret/non-prod/<team>/<service>/pp-aps1`
  - or an existing Prism path in `https://vault.prism-cmd.jiostarprism.com/`
- For prod client clusters such as `prism-cmd` or `prism-mnc`, create and update the target path only in `https://vault.prism-cmd.jiostarprism.com/`

Apply the same clone-then-adjust rule used for repo config:

- Example source: `secret/non-prod/hs-core/layout-service/pp-aps1`
- Example target for `prism-npd`: `secret/prism-npd/hs-core/layout-service/prism-npd-aps1`
- Example target for `prism-cmd`: `secret/prism-cmd/hs-core/layout-service/prism-cmd-aps1`

Copy all keys from the nearest valid source path first, then update only confirmed environment-specific values. Do not invent missing secret values.

If direct Vault write access is unavailable, hand back:

- source Vault URL and path
- target Vault URL and path
- key list to copy
- keys that require new environment-specific values

If runtime Vault tokens are provided, use them only in-memory for the current task and avoid echoing them back in summaries.

## Step 5: Prepare and raise the PR

When repo access exists, make the changes in the relevant repositories and prepare a PR per repo unless the team's normal workflow requires a combined branch elsewhere.

After the diff is ready and the user wants PRs raised:

- commit the changes in each affected repo with a concise env-specific message
- push the feature branch to the correct writable remote when remote write access exists
- if push succeeds but direct PR creation is unavailable, hand back the exact compare URL, PR title, and PR body
- if push is unavailable, hand back the exact `git push` command and PR metadata instead of stopping at a local diff

For each PR:

1. Use a branch name that includes the service and target env.
2. Write a concise title stating the service and target Prism env.
3. Include a body covering:
   - what was added or cloned,
   - source environment used,
   - any values intentionally left unchanged,
   - any manual follow-up needed in Harness, Vault, DNS, or IAuth.

If direct PR creation is available, raise the PR on the user's behalf after summarizing the diff. If it is not available, hand back the exact PR title, body, branch name, and changed files.

For prod cluster changes, require explicit user confirmation before raising the PR or finalizing the prod-targeted diff.

## Step 6: Hand off deployment follow-through

After the PR is ready, tell the user which Harness project and input set should be used after merge, and list any post-merge checks:

- Harness `prism_service_deployment`
- `SkipTerraform` expectation
- Vault path created or pending
- pod health in the relevant Prism dashboard
- service endpoint reachability if applicable

Report status per repo and per service part: changed, pending input, PR raised, or blocked.
