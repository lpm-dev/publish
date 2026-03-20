# LPM.dev Publish

GitHub Action to publish packages to [LPM](https://lpm.dev). Sets up Node.js, installs dependencies, and publishes — all in one step.

Supports OIDC Trusted Publishers (no secrets needed) and token-based authentication.

## Quick Start (OIDC — recommended)

No secrets to manage. Configure a Trusted Publisher at [lpm.dev/dashboard/packages/YOUR_PACKAGE/workflow](https://lpm.dev).

```yaml
name: Publish
on:
  push:
    tags: ["v*"]

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/publish@v1
```

That's it. Triggers on version tags, handles everything automatically.

## With Token

For CI without OIDC support:

```yaml
name: Publish
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/publish@v1
        with:
          oidc: "false"
          token: ${{ secrets.LPM_TOKEN }}
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `oidc` | Use OIDC for secret-free auth | `"true"` |
| `token` | LPM auth token (when OIDC is off) | `""` |
| `node-version` | Node.js version | `"22"` |
| `cli-version` | LPM CLI version | `"latest"` |
| `install-command` | Install command before publish | `"npm ci"` |
| `dry-run` | Validate without uploading | `"false"` |

## Examples

### Dry run on pull requests

Validate the package builds and passes quality checks on every PR, publish only on tags:

```yaml
name: CI + Publish
on:
  push:
    tags: ["v*"]
  pull_request:

permissions:
  id-token: write
  contents: read

jobs:
  validate:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/publish@v1
        with:
          dry-run: "true"

  publish:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/publish@v1
```

### Build + test before publish

Use `lpm-dev/install` for the build job and `lpm-dev/publish` for the publish job:

```yaml
name: CI + Publish
on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/install@v1
      - run: npm run build
      - run: npm test

  publish:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lpm-dev/publish@v1
```

## Setting Up Trusted Publishers

1. Go to your package in the LPM dashboard
2. Open the **Trusted Publishers** tab
3. Click **Add Trusted Publisher**
4. Enter your repository owner, name, workflow file, and optional environment
5. Save

The repository, workflow file, and environment must match exactly what the CI pipeline sends.

## How It Works

1. GitHub generates a signed identity token (JWT)
2. LPM verifies the JWT and matches it against your Trusted Publisher config
3. LPM issues a 15-minute publish token
4. The CLI publishes with full provenance (commit SHA, workflow, actor)

Each published version records exactly which commit, workflow, and user published it.

## License

MIT
