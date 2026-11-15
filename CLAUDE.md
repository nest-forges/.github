# Agent Instructions for nest-forges/.github

## README maintenance

`README.md` documents every reusable workflow in `.github/workflows/`.
Keep it in sync with the workflow files at all times.

### When a workflow file changes

After editing any `.github/workflows/*.yml` file, check `README.md` for each of
the following and update it before finishing:

- **Inputs** — every `inputs:` key must appear in the README table with the
  correct `required` flag and description.
- **Secrets** — every `secrets:` key must appear in the README table and in the
  global secrets overview table.
- **Composer scripts** — if a new `composer <script>` call is added or removed,
  reflect it in the "Composer scripts expected" section.
- **Server prerequisites** — if the deploy workflow gains new assumptions about
  the server (new directories, tools, file paths), document them.
- **Notification logic** — if the notify workflow changes its success/failure
  conditions or email format, update the description in the README.
- **Usage example** — if any input or secret is renamed or added, update the
  example block at the bottom of the README.

### When a new workflow file is added

1. Add a new `###` section to `README.md` following the same structure as the
   existing sections:
   - One-line description of what the workflow does.
   - **Inputs** table: `Input | Required | Description`.
   - **Secrets** table: `Secret | Required | Description`.
   - Any tools, scripts or external services the calling repository must provide.
2. Add every new secret to the global **GitHub Secrets overview** table.
3. Add the workflow to the **Usage example** block.

### When a workflow file is deleted

Remove its section from `README.md`, remove its secrets from the overview table
and remove it from the usage example.

### Consistency checks to run before finishing

Verify the following before completing any task that touches workflows or the README:

1. Every `inputs:` key in each workflow file has a matching row in the README.
2. Every `secrets:` key in each workflow file has a matching row in the README
   and in the secrets overview table.
3. No README section references a workflow file that no longer exists.
4. The usage example uses the correct input and secret names (no stale names).

If any check fails, fix the README and note the discrepancy in your response.

---

## Naming conventions

- **Workflow file names** — `<purpose>-reusable.yml` for reusable workflows
  called via `workflow_call`, e.g. `ci-reusable.yml`, `deploy-reusable.yml`.
- **Job names** — emoji prefix followed by a short description in sentence case,
  e.g. `🚀 Build and deploy release to server`.
- **Step names** — emoji prefix followed by a short description, e.g.
  `📦 Install dependencies`. Use consistent emojis per action type:
  - 🛒 Checkout
  - 🐘 Set up PHP
  - ♻️ Cache
  - 🛡️ Auth / security
  - 📦 Install dependencies
  - 🧪 Tests
  - 📤 Upload / 📥 Download artifact
  - 🚀 Deploy
  - 📧 Notify
- **Input and secret keys** — kebab-case, e.g. `php-version`, `source-zip`,
  `SSH_HOST` (secrets in SCREAMING_SNAKE_CASE as required by GitHub).
- **Workflow `name:` field** — emoji prefix followed by a human-readable title,
  e.g. `🛡️ CI Quality Gate (Reusable)`.

---

## Action versions

Pin every third-party action to its current major version tag. Do not mix
versions of the same action across workflow files. The canonical versions for
this repository are:

| Action | Version |
|---|---|
| `actions/checkout` | `@v7` |
| `actions/cache` | `@v6` |
| `actions/upload-artifact` | `@v7` |
| `actions/download-artifact` | `@v8` |
| `actions/github-script` | `@v9` |
| `shivammathur/setup-php` | `@v2` |
| `appleboy/scp-action` | `@v1` |
| `appleboy/ssh-action` | `@v1` |
| `dawidd6/action-send-mail` | `@v24` |
| `SonarSource/sonarqube-scan-action` | `@v8` |

When adding a new action, use its latest major version tag and add it to this
table. When upgrading an existing action to a new major version, update every
occurrence across all workflow files and update this table.

---

## Security

- Never hardcode values that should be secrets (passwords, tokens, hostnames).
  Always pass them via `secrets:` and reference with `${{ secrets.NAME }}`.
- Every secret used in a workflow must be declared in its `on.workflow_call.secrets:`
  block with a `description`.
- Every input must have a `description` field.

---

## New workflow template

When creating a new reusable workflow, start from this skeleton:

```yaml
name: <emoji> <Title> reusable

on:
  workflow_call:
    inputs:
      example-input:
        required: true
        type: string
        description: <what this input controls>
    secrets:
      EXAMPLE_SECRET:
        required: true
        description: <what this secret is used for>

jobs:
  <job-id>:
    name: <emoji> <Job description>
    runs-on: ubuntu-latest
    steps:
      - name: <emoji> <Step description>
        ...
```

Then immediately add a section for it in `README.md` (see README maintenance
section above).

---

## Validation before finishing

Before completing any task that adds or modifies workflow files, confirm:

1. `yamllint` passes on every changed file (or manually verify indentation and
   quoting are consistent with the existing files).
2. All `uses:` versions match the canonical versions table above.
3. All inputs and secrets have a `description`.
4. README is up to date (see consistency checks above).
