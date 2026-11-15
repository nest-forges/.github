# Shared GitHub Actions Workflows

Reusable workflows for CI, deployment and notifications across all projects in this organisation.

---

## Workflows

### `ci-reusable.yml` — CI Quality Gate

Runs security audit, static analysis, tests with coverage and SonarQube scan.

**Inputs**

| Input | Required | Description |
|---|---|---|
| `php-version-primary` | yes | PHP version for coverage run and SonarQube (e.g. `"8.4"`) |
| `php-versions-matrix` | no | JSON array of additional PHP versions for compatibility checks (e.g. `'["8.3","8.4"]'`). Omit to skip the matrix. |

**Secrets**

| Secret | Required | Description |
|---|---|---|
| `GH_PERSONAL_TOKEN` | yes | Personal access token to install private Composer packages |
| `SONAR_TOKEN` | no | SonarQube token; omit to skip the scan |

**Composer scripts expected**

| Script | Purpose |
|---|---|
| `composer phpcs` | Code style check |
| `composer phpstan` | Static analysis |

Tests are run with `./vendor/bin/phpunit -c phpunit.xml`.
Coverage output must be written to `./.runtime/coverage.xml`.

---

### `deploy-reusable.yml` — Build and Deploy

Builds the project with Composer, uploads a zip archive via SCP and unpacks it on the server.

**Inputs**

| Input | Required | Description |
|---|---|---|
| `php-version` | yes | PHP version to build with (e.g. `"8.4"`) |
| `source-zip` | yes | Name of the zip produced by `composer release` (e.g. `"release.zip"`) |
| `target` | yes | Absolute target directory on the server (e.g. `"/var/www/myapp/"`) |
| `env-path` | yes | Path inside the unpacked release where `.env` is placed (e.g. `"app/configs/env"`) |

**Secrets**

| Secret | Required | Description |
|---|---|---|
| `GH_PERSONAL_TOKEN` | yes | Personal access token to install private Composer packages |
| `SSH_HOST` | yes | Hostname or IP of the deployment server |
| `SSH_USERNAME` | yes | SSH user on the deployment server |
| `SSH_PASSWORD` | yes | SSH password on the deployment server |

**Composer scripts expected**

| Script | Purpose |
|---|---|
| `composer release` | Builds and packages the project into `release/<source-zip>` |

#### Server prerequisites

The target directory must contain a `shared/` folder with the environment file before the first deploy:

```
<target>/
└── shared/
    └── .env          ← must exist before first deploy
```

The workflow copies `shared/.env` into the unpacked release at `<env-path>/.env` on every deploy, so the file is never committed to the repository.

The deploy user needs write access to `<target>` and the `unzip` command must be available on the server.

---

### `notify-reusable.yml` — Deploy Notification

Checks whether the website is reachable after a deploy and sends an email with the result.

**Inputs**

| Input | Required | Description |
|---|---|---|
| `ref-name` | yes | Git ref that was deployed (tag or branch name) |
| `website` | yes | Full URL to check and include in the email (e.g. `"https://example.com"`) |

**Secrets**

| Secret | Required | Description |
|---|---|---|
| `GMAIL_APP_PASSWORD` | yes | Gmail app password for the sending account |

The workflow performs an HTTP check with a 15-second timeout. A 2xx or 3xx response is considered success. The subject and body of the email differ depending on the result:

- `✅ Deploy <ref> erfolgreich` — site is reachable, HTTP status shown
- `⚠️ Deploy <ref> – Website nicht erreichbar` — deploy completed but site returned an unexpected status

---

## GitHub Secrets overview

Set these in **Settings → Secrets and variables → Actions** of each consuming repository, or at organisation level to share them.

| Secret | Used by |
|---|---|
| `GH_PERSONAL_TOKEN` | CI, Deploy |
| `SONAR_TOKEN` | CI (optional) |
| `SSH_HOST` | Deploy |
| `SSH_USERNAME` | Deploy |
| `SSH_PASSWORD` | Deploy |
| `GMAIL_APP_PASSWORD` | Notify |

---

## Usage example

```yaml
jobs:
  ci:
    uses: nest-forges/.github/.github/workflows/ci-reusable.yml@main
    secrets:
      GH_PERSONAL_TOKEN: ${{ secrets.GH_PERSONAL_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    with:
      php-version-primary: "8.4"

  deploy:
    needs: ci
    uses: nest-forges/.github/.github/workflows/deploy-reusable.yml@main
    secrets:
      GH_PERSONAL_TOKEN: ${{ secrets.GH_PERSONAL_TOKEN }}
      SSH_HOST: ${{ secrets.SSH_HOST }}
      SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
      SSH_PASSWORD: ${{ secrets.SSH_PASSWORD }}
    with:
      php-version: "8.4"
      source-zip: "release.zip"
      target: "/var/www/myapp/"
      env-path: "app/configs/env"

  notify:
    needs: deploy
    uses: nest-forges/.github/.github/workflows/notify-reusable.yml@main
    secrets:
      GMAIL_APP_PASSWORD: ${{ secrets.GMAIL_APP_PASSWORD }}
    with:
      ref-name: ${{ github.ref_name }}
      website: "https://example.com"
```
