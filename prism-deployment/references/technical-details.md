# Prism Deployment Technical Details

Read this when preparing concrete file edits or PRs.

## Repository mapping

Expect changes across up to three repositories:

- **Service repo**: service-specific config, legacy Terraform, app config, and any service code changes needed for Prism.
- **`hs-k8s-app-inventory`**: Prism environment inventory and IaC Terraform files for migrated services.
- **`harness-ng-project`**: service deployment config and input sets used by the `prism_service_deployment` pipeline.

Ask the user for the GitHub links or local paths for all three. Do not assume repo locations from memory.

Vault may be a fourth system in scope. Treat it separately from git repos.

Before cloning any repository, check whether it already exists locally and reuse that checkout when possible.

Cross-repo verification rule:

- always scan the service repo, `hs-k8s-app-inventory`, and `harness-ng-project` for lingering source-env values before calling the Prism change complete
- verify old env markers, tfvars names, workspace names, Vault paths, override manifest paths, and service URL/domain suffixes in all three repos
- leave non-target source files such as `preprod` reference files unchanged unless the user explicitly asks to migrate them too

## Local branch policy

When working in a local checkout:

- never edit directly on `main`, `master`, or `develop`
- create a feature branch before making Prism changes
- for common repos like `hs-k8s-app-inventory` and `harness-ng-project`, prefer `prism-feature/<service-short-name>`
- for the service repo, prefer env-specific names such as `prism-npd-feature/config` or the matching target-specific branch

If a suitable feature branch already exists for the same Prism change, reuse it instead of creating another branch.

## Environment targeting

- Default non-prod target: `prism-npd`
- Prod targets must be explicit: `prism-mnc`, `prism-icc`, `prism-cmd`, or another named Prism client cluster confirmed by the user
- Source chain:
  - `pp` or `pre-prod` -> `prism-npd`
  - `prism-npd` -> `prism-<client>`

If the user says only "prod", stop and ask which cluster.

## Terraform and migration

Migration check:

- If `hs-k8s-app-inventory/<team>/<service>/infra/` exists, the service is on IaC Terraform. Update Prism files there.
- If it does not exist, update the legacy Terraform files in the service repo, such as `deploy/terraform/variables/prism-npd.tfvars`, `prism-mnc.tfvars`, or the matching `prism-<client>.tfvars`.
- If the source tfvars uses a market-region filename, keep that filename pattern and replace only the env segment. Example: `in-pp-ap-south-1.tfvars` -> `in-prism-npd-ap-south-1.tfvars`.

Legacy Terraform main-file checks:

- inspect `main.tf` for a Vault Kubernetes auth path and change it to `auth/kubernetes-prism-prd-eks-mum-cmd-cluster1/login` when preparing Prism-targeted legacy Terraform
- inspect the Terraform S3 backend for a hardcoded `role_arn`; when it is Prism-specific backend access, use `arn:aws:iam::245594728378:role/harness-delegate-prism-assumer-role`

CI workflow check:

- inspect `.github/workflows/build-ci.yaml` or the equivalent service build workflow
- if push branch filters are present, make sure Prism branches are included, for example `prism-**`
- this is needed so Prism feature branch pushes still generate the image tag or manifest outputs consumed by Harness

Role ARN rules:

- Hardcoded Terraform `role_arn` values commonly use `arn:aws:iam::245594728378:role/harness-delegate-prism-assumer-role`
- Backend service `.tfvars` `aws_provider_iam_role` for `prism-npd` commonly uses `arn:aws:iam::279473425884:role/harness-delegate-assumed-role`
- S3 backend role usage commonly follows the Prism CMD assumer role

For first-time environment setup, prefer cloning an existing nearest-environment file instead of authoring Terraform from scratch.

## App config and Vault

Common file patterns:

- Service repo config: `config/prism-npd.yaml`, `config/prism-npd.json`, or target-cluster equivalents
- Vault path: `secret/prism-npd/<team>/<service>/prism-npd` or `secret/prism-<client>/<team>/<service>/prism-<client>`

Vault hosts:

- Non-prod source paths may live in `https://vault-incubator.hotstar-labs.com/`
- Prism target paths for `prism-npd`, `prism-cmd`, `prism-mnc`, and other Prism client clusters should live in `https://vault.prism-cmd.jiostarprism.com/`

Runtime token aliases:

- `hotstar-vault` -> `https://vault-incubator.hotstar-labs.com/`
- `prism-vault` -> `https://vault.prism-cmd.jiostarprism.com/`

These aliases are conventions for runtime credential injection only. Never persist their token values in the skill, repos, or generated PR content.

Path examples:

- Source in incubator: `secret/non-prod/hs-core/layout-service/pp-aps1`
- Existing Prism source: `secret/prism-npd/hs-core/layout-service/pp-aps1`
- Target for `prism-npd`: `secret/prism-npd/hs-core/layout-service/prism-npd-aps1`
- Target for `prism-cmd`: `secret/prism-cmd/hs-core/layout-service/prism-cmd-aps1`
- Target for `prism-mnc`: `secret/prism-mnc/hs-core/layout-service/prism-mnc-aps1`

Rename rules:

- `/pp/` or `/preprod/` -> `/prism-npd/` or `/prism-<client>/`
- `-pp` or `-preprod` -> `-prism-npd` or `-prism-<client>`
- Trailing env tokens inside resource names should follow the same rename rule, including forms like `-pp2` or `/pp2-` when they are environment markers.
- Vault path leaf names should keep the source ordering and swap only the env token. Example: `secret/non-prod/umsp/um/pp2-aps1-in` -> `secret/prism-npd/umsp/um/prism-npd-aps1-in`, not `secret/prism-npd/umsp/um/in-prism-npd-ap-south-1`.
- For `hs-k8s-app-inventory` overrides, treat delimited env markers as global replacements across the full value, including middle segments in hosts, topics, namespaces, client ids, and source paths. Examples: `api-qa.pp...` -> `api-prism-npd.prism-npd...`, `in.qa.um...` -> `in.prism-npd.um...`, `um-profile-recollection-pp` -> `um-profile-recollection-prism-npd`.
- Exclude non-env identifiers from that rewrite, especially country and market codes such as `qa` in `config-qa.json`, `/usr/configs/qa/`, or country lists.
- Prism NPD service URLs should end in `.prism-npd.jiostarprism.com` after renaming. That rule also applies when an earlier rewrite produced `.prism-npd.hotstar.com`; convert that intermediate form to the final Prism domain.

Value rules:

- Update explicit env identifiers, Vault paths, DNS names, and known hostnames
- Preserve secret material unless the destination env requires fresh credentials
- If a value is unknown and safely clonable, keep the source value after required renaming

Vault copy procedure:

1. Read the nearest valid source path.
2. Create the target Prism path in the correct Vault host.
3. Copy all keys.
4. Update only known env-specific values.
5. Record any keys that still need owner-provided secrets or regenerated credentials.

If the source path is already in `vault.prism-cmd.jiostarprism.com`, stay in that Vault for the copy. Prod cluster targets should never be created in `vault-incubator.hotstar-labs.com/`.

## Vault access requirements

Direct Vault modification requires an authenticated access path. The skill can only modify Vault if one of these is available in the environment:

- a connected Vault MCP or API tool with read and write permissions
- an authenticated Vault CLI session the agent can use
- a Vault token or comparable credential already wired into the environment

Preferred runtime inputs:

- `prism-vault` token for Prism Vault operations
- `hotstar-vault` token for incubator Vault reads when `prism-npd` is cloned from non-prod

If the runtime exposes environment variables instead of aliases, map them without persisting the raw values.

Minimum permissions needed:

- read access to the source path
- create, read, and update access to the target Prism path
- access to both Vault hosts if the source is in incubator and the target is in Prism

If direct access is not available, the skill should still prepare the exact copy plan, target paths, and key-level follow-up list.

## Harness

- Pipeline: `prism_service_deployment`
- Org/account context: Prism / Hotstar depending on the team's setup
- Project is team-specific
- Input sets are service-specific and sometimes part-specific for split services

Check `harness-ng-project` for:

- Prism environment input set existence
- Vault path references
- `SkipTerraform` behavior

Default `SkipTerraform` to `False` for new env bring-up unless the repo state shows Terraform has already provisioned the required outputs.

## DNS and endpoints

Default endpoint pattern:

- `origin-<service-name>.<env>.jiostarprism.com`

Split services get one endpoint per deployable part.

Custom DNS prefixes are exceptions. Use them only when the user confirms the service needs one, then note the extra `hs-k8s-dns-manager` PR and DNS pipeline follow-up in the PR body or handoff.

## IAuth-dependent services

If the service depends on IAuth:

- confirm the correct Prism-compatible SDK version
- register the service for the target environment to obtain env-specific credentials
- update config or Vault entries such as `IAUTH_AUTH_ENDPOINT`, `IAUTH_ENV`, `IAUTH_MANAGE_ENDPOINT`, and `IAUTH_PUBLIC_KEY`

Do not carry service ids, secrets, or S2S tokens across environments unless the owning team explicitly confirms that behavior.

## PR guidance

When raising a PR on the user's behalf:

- include the source environment used for cloning
- list the repositories changed
- state whether the change is `prism-npd` or a specific prod cluster
- call out manual follow-ups in Harness, Vault, DNS, or IAuth
- note any values intentionally left unchanged because they were unknown or copied from source config
