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
vendor/bin/deploy
vendor/bin/deploy main deploy.yml    # custom branch / workflow
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
