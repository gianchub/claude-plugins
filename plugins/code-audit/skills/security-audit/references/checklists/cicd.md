# CI/CD Pipeline Security

## Scope

GitHub Actions, GitLab CI, Jenkins, CircleCI, Bitbucket Pipelines, Azure Pipelines, and equivalent. The CI/CD pipeline is a high-trust environment (it has access to source code, secrets, and often production credentials) and a frequent target of supply-chain and pull-request-based attacks.

## GitHub Actions

### Pwn-Request (`pull_request_target`)

The most critical class of GitHub Actions vulnerability. The `pull_request_target` event:

- Runs in the context of the *base* repository (the trusted repository).
- Has access to repository secrets (unlike `pull_request` from forks).
- By default checks out the base branch, not the PR's HEAD.

The vulnerability arises when a workflow triggered by `pull_request_target` is changed to check out the PR's HEAD (the *untrusted* code) and then runs that code with secrets in scope:

```yaml
# DANGEROUS
on: pull_request_target
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # untrusted code
      - run: npm install                                  # runs PR scripts with secrets
      - run: npm test                                     # ditto
```

The attacker opens a PR with a malicious `package.json`, `npm` install script, or test that exfiltrates secrets.

### Detection

- Search workflow files for `on: pull_request_target` (or `on:` blocks containing it).
- For each, check whether the workflow checks out PR HEAD.
- If yes: any code execution after that checkout (build, test, install, lint) with secrets in scope is a high finding.

### Defense

- Use `pull_request` (which runs in the fork's context with no secrets) for most CI tasks.
- Use `pull_request_target` only for tasks that legitimately need secrets (commenting on the PR, adding labels) and *do not* check out PR code, or only run on labels added by maintainers.
- Pattern: `pull_request` for build/test, `pull_request_target` for posting status, gated by maintainer-added label.

### Permissions

- **`permissions:` key** — Restrict the `GITHUB_TOKEN` per-job to least privilege. Default is `contents: write` which is broad.
- **`permissions: read-all`** at workflow level + per-job overrides — common pattern.
- **Workflow without `permissions:`** — Runs with default; check repo-level default; restrict.

### Action Versioning

- **`uses: actions/checkout@main`** or **`@master`** — Branch reference; if the branch is moved to malicious code, your workflow runs it. Pin to commit SHA: `actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608`.
- **`uses: org/action@v3`** — Tag reference; can be moved by the maintainer or compromised. Better than branch but pin to digest for high-trust workflows.
- **Third-party actions** — Audit the action's source before use; community actions can change behavior. Pin to verified versions.
- **Verified creator badge** is a weak signal; doesn't prove safety of action content.

### Secrets Handling

- **Secrets in workflow files** — Always use `${{ secrets.X }}`, never inline values.
- **`echo $SECRET`** — Visible in logs even if scrubbed (multi-line / formatted output may not match scrub patterns); flag debug echoes of secrets.
- **`set-output` with secret values** — Outputs may flow to subsequent steps in unexpected ways.
- **Passing secrets to actions** — Audit which actions receive which secrets; minimize.
- **Logs retention** — Default 90 days; consider shorter for sensitive workflows.

### Self-Hosted Runners

- **Self-hosted runners on public repos** — Catastrophic risk; PR can run arbitrary commands on the runner. Restrict to private repos or short-lived ephemeral runners.
- **Persistent runner state** — Workflow run leaves files / processes; subsequent jobs may inherit. Use ephemeral runners (run-once) for sensitive workflows.
- **Runner labels** — Verify the workflow targets the intended runner pool.

### Environments

- **Protected environments** — `environment:` key with required reviewers and wait timer; verify production deploys gated. Without it, anyone with write access can deploy.
- **Environment-scoped secrets** — Production secrets only available in production environment; staging secrets in staging.

### Other GitHub Actions Concerns

- **`workflow_dispatch` inputs** — Untrusted user-supplied inputs flowing into commands; validate / quote.
- **`issue_comment` triggers** — Anyone can comment; if a workflow triggers on a comment containing a magic string, attackers can trigger.
- **Deploy keys vs. PATs** — Personal access tokens with write access leaking via logs / debug output; revoke and rotate.
- **`ARTIFACTS`** — Uploaded build artifacts public on public repos; verify no secrets leak.
- **Reusable workflows** (`workflow_call`) — Verify caller / callee trust relationship; same versioning concerns as actions.
- **Composite actions** — Same versioning concerns.

## GitLab CI

### Variables

- **Protected variables** — Available only on protected branches; verify production secrets are protected.
- **Masked variables** — Logged output redacts the value; verify mask works for the format (multi-line tokens may not).
- **Variable visibility** — Group, project, instance scopes; minimize exposure.

### `image:` and `services:`

- Pin to digests; verify trusted registries.
- `services:` block exposes additional surface; verify.

### Trigger Tokens

- **Pipeline trigger tokens** — Each token can trigger pipelines; verify scope.
- **Job tokens** — Used for cross-pipeline operations; permission scope needs checking.

### Runners

- **Shared vs. group-specific vs. project-specific runners** — Sensitive jobs should not run on shared runners exposed to untrusted projects.
- **Runner registration tokens** — Treat as secrets; rotate when compromised.

### Workflow

- **`merge_request_event` source** — Be careful when using merge request artifacts that include source from the MR; same `pull_request_target` class of issue.
- **Auto DevOps** — Default templates; verify they match security expectations.

## Jenkins

### Plugin Trust

- Jenkins plugin ecosystem has historic supply-chain incidents. Audit installed plugins; remove unused.

### Credential Storage

- **Credentials plugin** — Store secrets here, not in Jenkinsfiles or job configs.
- **Credential scope** — Per-job vs. global; per-job preferred.

### Build Trust

- **Untrusted builds** — Sandboxed Groovy in Pipelines; verify sandbox enabled. CSP for Jenkins UI.
- **`jenkinsfileRunner` outside controlled context** — Risk of arbitrary code execution.
- **Master/Agent separation** — Build agents shouldn't have access to controller secrets.

### CSRF and Authentication

- Jenkins CSRF protection enabled; "Anyone can do anything" matrix is dangerous.
- LDAP/SSO for authentication; local user accounts limited.

## CircleCI

- **Contexts** — Scoped secret containers; restrict to specific projects/groups.
- **Orbs** — Reusable config; pin to specific versions; verify orb trust.
- **`run-on-fork`** disabled or carefully reviewed.

## Azure Pipelines

- **Variable groups** — Scope to specific pipelines; mark secret values.
- **Service connections** — Audit Azure / AWS / GCP service connections; least privilege; rotation.
- **YAML pipelines** — Same versioning concerns as GitHub Actions templates.

## Cross-Platform Patterns

### Branch Protection

- **Required reviews** — Cannot merge without review; verify CI/CD-related files (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`) require maintainer review.
- **Status checks required** — CI must pass before merge.
- **Restrict who can push to default branch** — No direct pushes; only via reviewed PRs.
- **Signed commits required** — Commit signing verifies author; defense against compromised credentials pushing.
- **CODEOWNERS** — Review automatically requested from owners; verify CI/CD files have owners.

### Pinning All The Things

- Container images: digest, not tag.
- Actions / Orbs / templates: SHA, not tag.
- Dependency artifacts: lockfile.
- OS packages installed during build: pinned versions where reproducibility matters.

### Build Provenance

- **SBOM** generated as part of build; published with artifact.
- **Sigstore / Cosign** signing of container images and other artifacts.
- **npm provenance, PyPI digital attestations** — Generate as part of publish.
- **SLSA level** — Document target level; CI configuration matches.

### Test and Deploy Separation

- Test stage runs PR code with no production secrets.
- Deploy stage runs on merged code with production secrets, gated by environment protection.
- Cross-contamination via job sharing or shared runners is a finding.

### Workflow File Audit

- **Secrets in commits** — `.github/workflows/` files are committed; any secret in them is leaked. Use the platform secret store.
- **Debug output** — `ACTIONS_STEP_DEBUG` enabled on workflows / re-runs; secrets may leak; ensure flag isn't set in default config.
- **Run logs as evidence** — Public repos have public run logs; verify no secret in error messages or stack traces in CI logs.

## Common Findings Patterns

- **Workflow on `pull_request_target` checks out PR HEAD and runs scripts** — Critical (pwn-request); high in nearly all cases.
- **Action pinned to branch / tag** — Medium (supply-chain risk if dependency moves).
- **`GITHUB_TOKEN` with default permissions** — Low to Medium depending on what the workflow does.
- **Secrets in workflow files (literal values)** — Critical to High depending on secret value.
- **Self-hosted runner on public repo** — Critical.
- **No environment protection on production deploy job** — Medium.
- **No required reviews / status checks on default branch** — Low to Medium (process / governance finding).
- **Old runner / agent versions** — Low to Medium; CVEs in runner software.
- **Build artifact upload includes test fixtures with secrets** — Medium to High; depends on artifact visibility.

## Recommendation Patterns

- Use `pull_request` for build/test of untrusted PRs; `pull_request_target` only for trusted operations and never check out PR code with secrets in scope.
- Pin actions to commit SHAs; pin Docker images to digests; use lockfiles.
- Restrict `GITHUB_TOKEN` permissions per-job; default to read-only at workflow level.
- Use environment protection for production deploys; require reviewers; protected environments for production secrets.
- Scope secrets narrowly; use OIDC (GitHub Actions → AWS / GCP) instead of long-lived credentials when possible.
- Audit CI/CD workflow files in PRs with extra scrutiny; require maintainer review via CODEOWNERS.
- Generate SBOM and sign artifacts; document the build provenance approach.
- Use ephemeral runners; never persistent self-hosted runners on public repos.
- Enable branch protection: required reviews, required status checks, no force-push to default branch, signed commits if practicable.
