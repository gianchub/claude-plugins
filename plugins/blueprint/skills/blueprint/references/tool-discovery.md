# Tool Discovery

Detect the project's tooling before generating any plan. The discovered tool chain populates the Phase 3 verification commands in every step.

## Config File Detection

Scan the project root (and common subdirectories) for configuration files. Each file implies specific tools.

### Python
- **pyproject.toml**: Parse `[tool.*]` sections.
  - `[tool.ruff]` or `[tool.ruff.lint]` → linter: `ruff check .`
  - `[tool.mypy]` → type checker: `mypy .`
  - `[tool.pytest]` or `[tool.pytest.ini_options]` → test runner: `pytest`
  - `[tool.black]` → formatter: `black --check .`
  - `[tool.isort]` → import sorter: `isort --check .`
  - `[tool.coverage]` → coverage: `pytest --cov`
- **setup.cfg / tox.ini**: Check `[mypy]`, `[flake8]`, `[isort]` sections.

### JavaScript / TypeScript
- **package.json**: Parse `"scripts"` and `"devDependencies"`.
  - `eslint` in devDeps → linter: `npx eslint .` (or the script name)
  - `prettier` in devDeps → formatter: `npx prettier --check .`
  - `jest` in devDeps → test runner: `npx jest`
  - `vitest` in devDeps → test runner: `npx vitest run`
  - `typescript` in devDeps → type checker: `npx tsc --noEmit`
  - `biome` in devDeps → linter+formatter: `npx biome check .`
- **tsconfig.json**: Confirms TypeScript usage; check `"strict"` setting.

### Rust
- **Cargo.toml**: Rust project confirmed.
  - Clippy (ships with rustup) → linter: `cargo clippy -- -D warnings`
  - Rustfmt (ships with rustup) → formatter: `cargo fmt --check`
  - Test runner: `cargo test`
- **.clippy.toml** / **clippy.toml**: Custom clippy configuration.

### Go
- **go.mod**: Go project confirmed.
  - Built-in → vet: `go vet ./...`
  - Check for `golangci-lint` binary or `.golangci.yml` → linter: `golangci-lint run`
  - Test runner: `go test ./...`
  - Formatter: `gofmt -l .`

### Ruby
- **Gemfile**: Ruby project confirmed.
  - `.rubocop.yml` or `rubocop` in Gemfile → linter: `bundle exec rubocop`
  - `rspec` in Gemfile → test runner: `bundle exec rspec`
  - `minitest` → test runner: `bundle exec rake test`

### Java / Kotlin
- **pom.xml**: Maven project.
  - Test runner: `mvn test`
  - Linter: check for checkstyle/spotbugs plugins.
  - Formatter: check for spotless plugin.
- **build.gradle** / **build.gradle.kts**: Gradle project.
  - Test runner: `./gradlew test`
  - Linter: check for checkstyle/detekt/ktlint plugins.

### Build Orchestration
- **Makefile**: Parse target names — look for `test`, `lint`, `check`, `fmt`, `build`.
- **justfile**: Same as Makefile — parse recipe names.
- **scripts/**: Scan for executable scripts with suggestive names (`test.sh`, `lint.sh`, `ci.sh`).

## Lock File Signals

Lock files confirm the language ecosystem and package manager.

| Lock file | Ecosystem | Package manager |
|---|---|---|
| `uv.lock` | Python | uv |
| `poetry.lock` | Python | poetry |
| `Pipfile.lock` | Python | pipenv |
| `requirements.txt` (alone) | Python | pip |
| `package-lock.json` | Node.js | npm |
| `yarn.lock` | Node.js | yarn |
| `pnpm-lock.yaml` | Node.js | pnpm |
| `Cargo.lock` | Rust | cargo |
| `go.sum` | Go | go modules |
| `Gemfile.lock` | Ruby | bundler |
| `composer.lock` | PHP | composer |

## CI Configuration

CI pipelines are a reliable source of truth for the project's actual tool chain. Parse them for commands.

**GitHub Actions**: Read all files matching `.github/workflows/*.yml`. Look for `run:` steps containing test, lint, format, type-check, and build commands. Pay attention to matrix strategies — they may reveal multiple tool versions or configurations.

**GitLab CI**: Read `.gitlab-ci.yml`. Look for `script:` arrays in stages named `test`, `lint`, `check`, etc.

**CircleCI**: Read `.circleci/config.yml`. Look for `run:` commands in job steps.

**General parsing strategy**: Extract the raw shell commands. Ignore environment setup commands (installing dependencies, setting env vars). Focus on the commands that validate code quality.

## Script Conventions

### npm Scripts
Parse `"scripts"` in `package.json`. Common keys: `test`, `lint`, `format`, `typecheck`, `build`, `check`. Prefer the project's script names over raw tool commands — they may include project-specific flags.

### Makefile / justfile Targets
Parse target names and their commands. Prefer the target name (e.g., `make lint`) over the raw command — the target may chain multiple tools.

### scripts/ Directory
Scan for executables. Match common names: `test`, `lint`, `check`, `ci`, `fmt`, `format`, `typecheck`, `build`.

## Presentation Format

After discovery, present the findings to the user for confirmation. Order by verification sequence (the order they run in Phase 3).

```
Discovered tools for this project:

1. Package manager: uv (from uv.lock)
2. Test runner: pytest (from pyproject.toml [tool.pytest])
3. Test coverage: pytest --cov (from pyproject.toml [tool.coverage])
4. Linter: ruff check . (from pyproject.toml [tool.ruff])
5. Formatter: ruff format --check . (from pyproject.toml [tool.ruff.format])
6. Type checker: mypy . (from pyproject.toml [tool.mypy])
7. CI pipeline: GitHub Actions (from .github/workflows/ci.yml)

Add, remove, or reorder? (or confirm to proceed)
```

**Rules**:
- Always show the source of each discovery in parentheses.
- Order by verification sequence: tests first, then linting, formatting, type checking.
- If CI commands conflict with config file commands, prefer CI commands (they reflect what actually runs).
- If no tools are discovered for a category, omit it — do not guess.
- Wait for user confirmation before proceeding. The user may add tools not detected (e.g., manual review steps, deployment checks) or remove tools that are not relevant to the plan.
