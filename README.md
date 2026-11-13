# deploy

Triggers a GitHub Actions deploy workflow (e.g. build Docker image → Coolify deploy) from your machine and waits until every job finishes. If a job fails, it prints that step's log and exits non-zero.

Refuses to run if your local branch differs from `origin`, since the workflow builds what's on GitHub.

## Requirements

- `git`
- [`gh`](https://cli.github.com), authenticated (`gh auth login`)
- A workflow in the project with `on: workflow_dispatch`

## Install

```bash
composer require --dev a2zwebltd/deploy
```

## Use

```bash
vendor/bin/deploy                       # master, deploy.yml
vendor/bin/deploy main release.yml      # custom branch / workflow
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

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  timeout-minutes: 15
  steps:
    - uses: a2zwebltd/deploy/setup-coolify@v1.1.0
      with:
        coolify-url: ${{ vars.COOLIFY_URL }}
        coolify-token: ${{ secrets.COOLIFY_TOKEN }}
        # cli-version: 1.11.0

    - uses: a2zwebltd/deploy/coolify-deploy@v1.1.0
      with:
        resource: ${{ vars.COOLIFY_RESOURCE }}   # Coolify application name
        # runtime-log-lines: 200
```
