# Dependencies and Supply Chain

## Scope

Third-party packages, lock files, install-time hooks, transitive dependencies, version pinning, and supply-chain attack surface. Without external CVE data feeds (the skill is configured for pure-Claude analysis), CVE detection relies on training-cutoff knowledge — incomplete and stale, but still useful for known-bad packages and obviously old versions. The skill should still be exhaustive on patterns regardless of CVE knowledge: outdated, unmaintained, install-script-running, typosquat-vulnerable packages are findings even without specific CVE attribution.

## Manifest Files to Inspect

| Ecosystem | Files |
|-----------|-------|
| Node.js | `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `npm-shrinkwrap.json` |
| Python | `requirements.txt`, `requirements-*.txt`, `pyproject.toml`, `Pipfile`, `Pipfile.lock`, `poetry.lock`, `setup.py`, `setup.cfg`, `uv.lock` |
| Java | `pom.xml`, `build.gradle`, `build.gradle.kts`, `gradle.lockfile`, `dependencies/*.lockfile`, `settings.gradle`, `bom.xml` |
| .NET | `*.csproj`, `packages.config`, `packages.lock.json`, `Directory.Packages.props`, `NuGet.Config` |
| Ruby | `Gemfile`, `Gemfile.lock`, `*.gemspec` |
| PHP | `composer.json`, `composer.lock` |
| Go | `go.mod`, `go.sum`, `vendor/modules.txt` |
| Rust | `Cargo.toml`, `Cargo.lock` |
| Swift | `Package.swift`, `Package.resolved` |
| Dart/Flutter | `pubspec.yaml`, `pubspec.lock` |
| Container base | `Dockerfile` (FROM lines), `docker-compose.yml` |
| OS packages | `apt`, `apk`, `yum` install lines in Dockerfiles or scripts |
| Generic | `LICENSE`, `THIRD_PARTY`, `vendor/` directory inspection |

## Version Pinning

### Patterns

- **Unpinned ranges** — `"react": "^18.0.0"`, `"requests": ">=2.0"`. Resolves to latest within range; introduces drift, surprising upgrades, supply-chain risk if a new minor version is compromised.
- **Pinned to exact version** — `"react": "18.2.0"`. Predictable; requires explicit upgrade.
- **Pinned to git ref / branch** — `"react": "git+https://...#main"`. Branch can change without notice; use SHA pin if necessary.
- **Lock file present** — Records the exact resolved version of every dependency including transitives. Always commit; `package.json` alone is insufficient for reproducibility.
- **Lock file out of sync** — `package.json` requires `"^18.0.0"` but lock file shows 17.x; CI may resolve differently. Verify lock matches manifest.
- **Floating tag for container base** — `FROM node:latest` — image content changes; pin to digest (`FROM node:18.17.0@sha256:...`) for reproducibility.

### Trade-offs

- Tight pinning + auto-upgrade tooling (Renovate, Dependabot) is the typical balance.
- Pinning without an upgrade plan stagnates security; document the upgrade cadence.

## Outdated and Unmaintained Packages

### Heuristics for "Outdated"

- Last release > 1-2 years ago for an active package category.
- Major version many behind current.
- Maintainer count = 1 with no recent activity.
- Repository archived / read-only.
- Package marked "deprecated" in the registry (npm has explicit deprecation).
- Dependency on an EOL runtime version (e.g., Node 12, Python 3.7).

### What to Report

- Each instance with package name, current version, signal of staleness, and reachable functions (is it a deeply-buried transitive or a primary dependency?).
- Recommendation: upgrade or replace.

## Known-Vulnerable Versions (Limited by Training Data)

Without external feeds, focus on:

- **Notorious packages and ranges** — log4j 2.x < 2.17.0 (CVE-2021-44228 et al.), Spring Framework < 5.3.20 (CVE-2022-22965 Spring4Shell), Apache Commons Text < 1.10.0 (CVE-2022-42889), older Jackson default-typing, older Newtonsoft.Json with TypeNameHandling, older snakeyaml < 2.0, lodash < 4.17.21 (prototype pollution fixes), event-stream@3.3.6 (compromised), node-ipc@10.1.1 (compromised), `colors`@1.4.1 (compromised), etc.
- **Cryptographic libraries** — Older OpenSSL, libgcrypt, mbedTLS bundled in dependencies.
- **Document the limitation** — In the report, note: "CVE assessment based on training-cutoff knowledge of public advisories. For comprehensive CVE coverage, recommend running an SCA tool (Snyk, GitHub Dependabot, Trivy, Grype, OSV-Scanner) on this lock file."

## Install-Time Hooks

Some package managers execute scripts during installation. These are full-trust execution on the build machine and any install target.

### Risk

- npm: `preinstall`, `install`, `postinstall`, `prepublish`, `prepare`. Malicious package executes during `npm install`.
- pip: `setup.py` executes during `pip install` (egg/wheel install). Modern wheels limit some surface but still execute on install.
- Gradle / Maven: build scripts execute during build.
- Cargo: `build.rs` executes; restricted scope but still surface.

### Detection

- Audit `package.json` of each direct dependency for install scripts. (Manual; or `npm install --ignore-scripts` and inspect what fails.)
- For Python: `pip install --no-build-isolation` or wheel-only installs limit surface.
- Dependabot / Renovate: review every dependency PR's release notes when unfamiliar packages have install scripts.

## Typosquatting and Dependency Confusion

### Typosquatting

Attacker publishes a package named similarly to a popular one (`requests` vs. `reqeusts`, `colors` vs. `clors`). Dev installs typo, gets malicious code.

- Verify each dependency name carefully.
- Periodically run `npm audit` / equivalent for typosquat-pattern detection.
- Internal package naming conventions reduce collision (`@org/foo` instead of `foo`).

### Dependency Confusion

Attacker publishes a public package with the same name as an internal/private one. If the package manager prefers the public registry, the attacker's code is installed instead of the internal package.

- Lock-file with explicit registry per scope.
- npm: `.npmrc` with `@org:registry=https://internal.example.com/`.
- pip: `--index-url` and `--extra-index-url` ordering matters; use `--index-url` for the trusted registry only.
- Maven / Gradle: configure repositories with explicit ordering.

## Transitive Dependency Risks

A direct dependency on `safe-package` may transitively pull in vulnerable or compromised packages. Lock file shows the full tree.

- For each direct dependency, audit the transitive set in the lock file.
- Use `npm ls`, `pip show`, `mvn dependency:tree`, etc., to enumerate.
- "Bundling" frameworks (webpack, esbuild) include the transitive set into the final artifact.

## License Compliance (Adjacent)

- Compatibility with project's license; verify each direct and transitive dependency.
- Particularly: GPL/AGPL in commercial proprietary; LGPL static linking; Creative Commons NC.
- Not strictly a security concern but often surfaced in the same audit; mention in scope if relevant to the user.

## Container Base Images

- **`FROM` line** — Pin to a specific tag and ideally a digest. `FROM ubuntu:latest` is reproducibility risk and cannot be CVE-tracked.
- **Distro EOL** — Old distro releases stop receiving security updates; flag.
- **Minimal images** — Distroless, alpine, slim variants reduce attack surface.
- **Multi-stage builds** — Verify final stage doesn't include build tools (compilers, dev shells) accessible at runtime.

## OS Packages in Containers

- `apt-get install ...` lines in Dockerfiles install distro packages; verify with `--no-install-recommends`, version pinning if reproducibility matters, and `apt-get clean && rm -rf /var/lib/apt/lists/*` to reduce image size.
- Same for `apk add ...`, `yum install ...`, `dnf install ...`.
- Identify obviously old or known-vulnerable distro package versions.

## Build Artifacts and Provenance

- **Reproducible builds** — Same input → same output. Documents the build environment.
- **SBOM** (Software Bill of Materials) — `cyclonedx`, `syft`, `spdx` formats. Useful for downstream security; recommend if not present.
- **SLSA** (Supply chain Levels for Software Artifacts) — Framework for build provenance. Mention if compliance level relevant to the user.
- **Signed artifacts** — Sigstore / Cosign for container images, npm provenance, PyPI digital attestations. Adoption is growing; flag absence as a recommendation for security-critical deployments.

## Vendoring and Mirroring

- **Vendoring** — Dependencies copied into the repo. Pros: fully self-contained build, registry compromise survives. Cons: harder to update, may pin to ancient versions.
- **Internal mirror / proxy** — Artifactory, Sonatype Nexus, JFrog. Reduces direct exposure to public registry; still requires update discipline.
- **Audit vendored code** — When vendored, the repo includes the vendored code; audit it as part of the codebase if in scope (or explicitly note exclusion).

## Update Tooling

- **Dependabot / Renovate** — Automated PRs for dependency updates; verify configured for the target repo. Verify auto-merge policies don't bypass review.
- **GitHub Advisory / GitLab Security Center** — Surface known CVEs via the platform.
- **OSS-Fuzz / OSV** — Public fuzz / vulnerability databases; not directly run but the data feeds into Dependabot.

## Common Findings Patterns

- **Dependency listed but never imported** — Dead dependency; reduce surface by removing.
- **Dev dependency in production** — `devDependencies` ending up in shipped artifact (bundling misconfig); inflates surface and may include test secrets.
- **Vulnerable package in critical path** — A package with a known CVE used in the critical path (auth, crypto). Severity rises with proximity to security-sensitive code.
- **Pinned to compromised version** — Specific version bricked or compromised (e.g., `colors@1.4.1` self-DoS); verify pinned versions don't fall in known compromised ranges.
- **Multiple versions of the same package** — npm tree often resolves to multiple versions of one package; check whether security-relevant package has multiple resolved versions, some vulnerable.
- **Transitive vulnerability with no upgrade path** — Direct dep pins a vulnerable transitive; resolution is `--force` resolution or upgrading the direct dep. Document.

## Recommendation Patterns

- Lock file committed; CI ensures lockfile-manifest consistency.
- Automated update tooling (Dependabot / Renovate) configured; review SLA documented.
- SCA tool integrated into CI for CVE detection beyond training-cutoff (Trivy, Snyk, Grype, OSV-Scanner).
- Disable install-time scripts where feasible (`npm install --ignore-scripts` for CI, with periodic full installs for trusted dependencies).
- Pin container base images to digest; rebuild on cadence.
- Pin internal package scope to internal registry; use lockfile registry-per-scope.
- Generate and publish SBOM; sign artifacts where applicable (cosign / npm provenance / PyPI attestations).
- Document an upgrade cadence in `SECURITY.md`.
