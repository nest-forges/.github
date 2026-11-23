# Shared GitHub Actions Workflows

Reusable workflows for CI, deployment and notifications across all projects in this organisation.

---

## Composite Actions

Internal building blocks used by the workflows above. Reference them with the full path and `@main`.

### `.github/actions/infisical` — Fetch secrets from Infisical

Fetches secrets from [Infisical](https://app.infisical.com) and injects them as environment variables.
Uses OIDC authentication — no static credentials required in the runner.

| Input | Required | Default | Description |
|---|---|---|---|
| `secret-path` | yes | — | Infisical path to fetch (e.g. `"/global"`) |
| `identity-id` | no | org default | Infisical machine identity ID |
| `project-slug` | no | org default | Infisical project slug |

### `.github/actions/setup-php-composer` — Set up PHP + Composer

Checkout, fetch Infisical secrets, set up PHP, restore Composer cache and install dependencies.

| Input | Required | Default | Description |
|---|---|---|---|
| `php-version` | yes | — | PHP version (e.g. `"8.4"`) |
| `coverage` | no | `none` | Coverage driver: `none`, `xdebug` or `pcov` |
| `ini-values` | no | `''` | Additional `php.ini` values |
| `secret-path` | no | `/global` | Infisical path override |

### `.github/actions/report` — Coverage Report + SonarQube

Checkout with full history, fetch Infisical secrets, download the coverage artifact,
post a coverage comment on the PR and run SonarQube.

| Input | Required | Default | Description |
|---|---|---|---|
| `php-version-primary` | yes | — | PHP version shown in the coverage comment |
| `github-token` | yes | — | GitHub token for posting PR comments (pass `${{ secrets.GITHUB_TOKEN }}`) |
| `secret-path` | no | `/global` | Infisical path override |

---

## Workflows

### `ci-reusable.yml` — CI Quality Gate

Runs security audit, static analysis, tests with coverage and SonarQube scan.

**No secrets need to be configured in the calling repository.** All tokens are
fetched automatically from [Infisical](https://app.infisical.com) at runtime:

| Secret | Infisical path | Purpose |
|---|---|---|
| `GH_PERSONAL_TOKEN` | `/global` | Composer authentication for private packages |
| `SONAR_TOKEN` | `/global` | SonarQube Cloud scan and Quality Gate |

The `GITHUB_TOKEN` used to post coverage comments on pull requests is provided
automatically by GitHub — no manual configuration required.

SonarQube runs only on `main` and pull requests; feature branches skip the scan
because the SonarCloud Free plan does not support branch analysis.

Setup of PHP, Composer cache and dependency installation is handled by the
composite action at `.github/actions/setup-php-composer/`, which calls
`.github/actions/infisical/` to inject the secrets before each job.

**Inputs**

| Input | Required | Description |
|---|---|---|
| `php-version-primary` | yes | PHP version for coverage run and SonarQube (e.g. `"8.4"`) |
| `php-versions-matrix` | no | JSON array of PHP versions for matrix compatibility checks (e.g. `'["8.4","8.5"]'`). Defaults to `["8.4","8.5","8.6"]`. |

**Secrets**

None — all secrets are managed by Infisical (see above).

**Composer scripts expected**

| Script | Purpose |
|---|---|
| `composer validate --strict` | Validate `composer.json` structure |
| `composer phpcs` | Code style check |
| `composer phpstan` | Static analysis |

Tests are run with `./vendor/bin/phpunit -c phpunit.xml`.
Coverage output must be written to `./.runtime/coverage.xml`.

---

### `deploy-reusable.yml` — Build and Deploy

Builds the project with Composer, uploads a zip archive via SCP and unpacks it on the server.

**No secrets need to be configured in the calling repository.** All tokens and SSH credentials are
fetched automatically from [Infisical](https://app.infisical.com) at runtime:

| Secret | Infisical path | Purpose |
|---|---|---|
| `GH_PERSONAL_TOKEN` | `/global` | Composer authentication for private packages |
| `SSH_HOST` | project path | Hostname or IP of the deployment server |
| `SSH_USERNAME` | project path | SSH user on the deployment server |
| `SSH_PASSWORD` | project path | SSH password on the deployment server |
| `ENV_PROD` | project path | Full contents of the `.env` file written into the release |

**Inputs**

| Input | Required | Description |
|---|---|---|
| `php-version` | yes | PHP version to build with (e.g. `"8.4"`) |
| `source-zip` | yes | Name of the zip produced by `composer release` (e.g. `"release.zip"`) |
| `target` | yes | Absolute target directory on the server (e.g. `"/var/www/myapp/"`) |
| `env-path` | yes | Path inside the unpacked release where `.env` is placed (e.g. `"app/configs/env"`) |
| `infisical-secret-path` | yes | Infisical secret path for app-specific secrets (e.g. `"/myapp"`) |

**Secrets**

None — all secrets are managed by Infisical (see above).

**Composer scripts expected**

| Script | Purpose |
|---|---|
| `composer release` | Builds and packages the project into `release/<source-zip>` |

#### Server prerequisites

The deploy user needs write access to `<target>` and the `unzip` command must be available on the server. No `shared/` folder or pre-existing `.env` file is required — the `.env` is created from the `ENV_PROD` secret fetched from Infisical on every deploy.

---

### `notify-reusable.yml` — Deploy Notification

Checks whether the website is reachable after a deploy and sends an email with the result.

**No secrets need to be configured in the calling repository.** All credentials are
fetched automatically from [Infisical](https://app.infisical.com) at runtime:

| Secret | Infisical path | Purpose |
|---|---|---|
| `GMAIL_APP_PASSWORD` | `/global` | Gmail app password for the sending account |

**Inputs**

| Input | Required | Description |
|---|---|---|
| `ref-name` | yes | Git ref that was deployed (tag or branch name) |
| `website` | yes | Full URL to check and include in the email (e.g. `"https://example.com"`) |

**Secrets**

None — all secrets are managed by Infisical (see above).

The workflow performs an HTTP check with a 15-second timeout. A 2xx or 3xx response is considered success; the email subject and body reflect the result.

---

### `e2e-reusable.yml` — E2E Tests

Runs Playwright end-to-end tests against a deployed website.

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `base-url` | yes | — | URL of the deployed website to run Playwright tests against |
| `node-version` | no | `"24"` | Node.js version to use |
| `e2e-command` | no | `"e2e"` | npm script to run (`npm run <e2e-command>`) |

**Secrets**

None.

**Prerequisites**

The calling repository must have a `package.json` with the configured npm script (default `e2e`) that invokes Playwright. The `BASE_URL` environment variable is set automatically from the `base-url` input.

---

## GitHub Secrets overview

Set these in **Settings → Secrets and variables → Actions** of each consuming repository, or at organisation level to share them.

| Secret | Used by | Notes |
|---|---|---|

> **CI, Deploy and Notify require no secrets in the repository.** All credentials
> are injected at runtime from Infisical (`/global` and the app-specific secret path).

---

## Usage example

```yaml
jobs:
  ci:
    uses: nest-forges/.github/.github/workflows/ci-reusable.yml@main
    with:
      php-version-primary: "8.4"
    # No secrets needed — GH_PERSONAL_TOKEN and SONAR_TOKEN are fetched from Infisical

  deploy:
    needs: ci
    uses: nest-forges/.github/.github/workflows/deploy-reusable.yml@main
    with:
      php-version: "8.4"
      source-zip: "release.zip"
      target: "/var/www/myapp/"
      env-path: "app/configs/env"
      infisical-secret-path: "/myproject"
    # No secrets needed — all fetched from Infisical (/global and app-specific secret path)

  notify:
    needs: deploy
    uses: nest-forges/.github/.github/workflows/notify-reusable.yml@main
    with:
      ref-name: ${{ github.ref_name }}
      website: "https://example.com"
    # No secrets needed — GMAIL_APP_PASSWORD fetched from Infisical (/global)

  e2e:
    needs: deploy
    uses: nest-forges/.github/.github/workflows/e2e-reusable.yml@main
    with:
      base-url: "https://example.com"
      # node-version: "24"   # optional, default "24"
      # e2e-command: "e2e"   # optional, default "e2e"
```
