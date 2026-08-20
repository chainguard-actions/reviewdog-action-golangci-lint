<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-golangci-lint/v2.10.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-golangci-lint/v2.10.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a) violation: `${{ matrix.golangci-lint-version }}` is directly interpolated inside a `run:` shell command string. Before the shell ever sees the command, GitHub Actions performs YAML template substitution, allowing a matrix value to inject arbitrary shell metacharacters. Offending line: `run: golangci-lint config verify --config=.github/.golangci.${{ matrix.golangci-lint-version }}.yml`

Locations:

- `.github/workflows/check-golangci-lint-config.yml:30`

### unsafe-shell (severity: high)

Remote content is piped directly to a shell interpreter without being downloaded to a file first. The install script is fetched from a mutable URL (using a tag ref) and immediately executed: `curl -sSfL "https://raw.githubusercontent.com/golangci/golangci-lint/refs/tags/$TAG_NAME/install.sh" | sh -s -- -b "$(go env GOPATH)/bin" "$TAG_NAME"`. If the remote content is tampered with or the tag is moved, arbitrary code will execute on the runner.

Locations:

- `.github/workflows/check-golangci-lint-config.yml:27`

### missing-permissions (severity: medium)

These workflow files have no top-level `permissions:` key and no job-level `permissions:` key on any of their jobs. Without explicit permissions, the GITHUB_TOKEN is granted its default (often broad) permissions, violating the principle of least privilege. Each file should declare minimal required permissions at the top level or per job.

Locations:

- `.github/workflows/check-dist.yml:1`
- `.github/workflows/check-golangci-lint-config.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/reviewdog.yml:1`
- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, missing-permissions

**Notes:**

Fixed 3 findings across 5 workflow files:

1. **unsafe-shell** (check-golangci-lint-config.yml): Replaced `curl ... | sh -s -- -b ...` with a two-step approach: download to a temp file with `curl ... -o "$INSTALL_SCRIPT"`, then execute `sh "$INSTALL_SCRIPT" -b "$(go env GOPATH)/bin" "$TAG_NAME"`. Dropped the `--` as it was the shell's option terminator (not the script's).

2. **script-injection** (check-golangci-lint-config.yml): Moved `${{ matrix.golangci-lint-version }}` out of the `run:` shell string into the step's `env:` block as `GOLANGCI_LINT_VERSION`, then referenced it as `${GOLANGCI_LINT_VERSION}` in the shell command.

3. **missing-permissions**: Added top-level `permissions:` blocks to all 5 workflow files with minimal required permissions:
   - check-golangci-lint-config.yml: `contents: read`
   - check-dist.yml: `contents: read`
   - release.yml: `contents: write` + `pull-requests: write`
   - reviewdog.yml: `contents: read` + `pull-requests: write` + `checks: write`
   - test.yml: `contents: read`

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'Set TAG_NAME' step in .github/workflows/check-golangci-lint-config.yml. The `gh release list` output is now captured into a `raw_tag` variable, sanitized with `printf '%s' "$raw_tag" | tr -d '\n\r'` to strip embedded newlines/carriage returns, and then the sanitized value is written to $GITHUB_ENV. This prevents a GitHub release name containing embedded newlines from injecting additional key=value pairs into the runner's environment.

