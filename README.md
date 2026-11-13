# deploy

Deploys your current git branch to an environment: dispatches a GitHub Actions workflow (e.g. build Docker image → Coolify deploy) from your machine and waits until every job finishes. If a job fails, it prints that step's log and exits non-zero.

Refuses to run if your local branch differs from `origin`, since the workflow builds what's on GitHub.

## Requirements

- `git`
- [`gh`](https://cli.github.com), authenticated (`gh auth login`)
- A workflow with `on: workflow_dispatch` and an `environment` input (see below)

## Install

```bash
composer require --dev a2zwebltd/deploy
```

## Use

```bash
vendor/bin/deploy                        # current branch → production, deploy.yml
vendor/bin/deploy staging                # current branch → staging
vendor/bin/deploy staging release.yml    # custom workflow
```

Or as a composer script (disable the 300s process timeout — deploys take longer):

```json
"scripts": {
    "deploy": [
        "Composer\\Config::disableProcessTimeout",
        "deploy"
    ]
}
```

## GitHub Actions: deploy to Coolify

Two composite actions for the workflow side:

- `setup-coolify` installs coolify-cli and points it at your Coolify instance.
- `coolify-deploy` deploys a resource, streams the deployment log, prints the runtime log, and fails unless the deployment finishes. Coolify waits for the image's `HEALTHCHECK`, so a finished deployment means a healthy container.

Each GitHub environment holds the Coolify app it deploys to (`COOLIFY_RESOURCE`):

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: environment
        required: true
        default: production

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # ... checkout, registry login ...
      - uses: docker/build-push-action@v7
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment: ${{ inputs.environment }}
    concurrency: deploy-${{ inputs.environment }}
    steps:
      - uses: a2zwebltd/deploy/setup-coolify@v1.3.0
        with:
          coolify-url: ${{ vars.COOLIFY_URL }}
          coolify-token: ${{ secrets.COOLIFY_TOKEN }}
          # cli-version: 1.11.0

      - uses: a2zwebltd/deploy/coolify-deploy@v1.3.0
        with:
          resource: ${{ vars.COOLIFY_RESOURCE }}   # Coolify application name
          # runtime-log-lines: 200
```

The deploy runs whatever image tag the Coolify app is configured to pull. The workflow above pushes `:<branch>`, so deploy the branch whose tag the environment's app pulls (e.g. `master` → production).
