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

Pass `image-digest` and the action pins the Coolify app to that exact image (`image@sha256:…`) before deploying, so any branch can go to any environment and restarts stay on the same image. To roll back, deploy again with an older digest.

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
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      # ... checkout, registry login ...
      - id: build
        uses: docker/build-push-action@v7
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
      - uses: a2zwebltd/deploy/setup-coolify@v2.0.0
        with:
          coolify-url: ${{ vars.COOLIFY_URL }}
          coolify-token: ${{ secrets.COOLIFY_TOKEN }}
          # cli-version: 1.11.0

      - uses: a2zwebltd/deploy/coolify-deploy@v2.0.0
        with:
          resource: ${{ vars.COOLIFY_RESOURCE }}   # Coolify application name
          image-digest: ${{ needs.build.outputs.digest }}
          # runtime-log-lines: 200
```

The Coolify app must use the same image name the workflow pushes (its tag is overwritten with the digest).
