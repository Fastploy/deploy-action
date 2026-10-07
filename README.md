# Fastploy Deploy

A GitHub Action that deploys a repository's `fastploy.yaml` to [Fastploy](https://app.fastploy.com) and waits until every service runs. It installs the Fastploy CLI the control plane serves, after checking its SHA-256, and runs `fastploy deploy --yes --json`, `fastploy promote` or `fastploy status --commit --wait`. The job fails when the deployment does, and its exit code says where it stopped.

## Deploy on push

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Fastploy/deploy-action@v1
        with:
          token: ${{ secrets.FASTPLOY_TOKEN }}
```

The repository's `.fastploy/link.json`, written by `fastploy link`, says which Environment it deploys. Link that Environment with `fastploy link --no-auto-deploy`, so a push does not deploy a second time by itself.

## Deploy the images this workflow built

```yaml
      - uses: Fastploy/deploy-action@v1
        with:
          token: ${{ secrets.FASTPLOY_TOKEN }}
          image-tag: ${{ github.sha }}
```

`image-tag` gives every service that runs an image the same tag, so none runs another version. `image` takes one `SERVICE=TAG` per line for some services only.

## Promote Staging to Production

```yaml
      - uses: Fastploy/deploy-action@v1
        with:
          token: ${{ secrets.FASTPLOY_PRODUCTION_TOKEN }}
          command: promote
          from: staging
          environment: production
```

Production gets the commit Staging runs, and every image at the exact digest Staging runs, with Production's own settings and secrets. Keep the Production key in a GitHub Environment with required reviewers so a person approves each promotion.

## Inputs

| Input | Default | What it does |
| --- | --- | --- |
| `token` | required | A REST API key from Settings > Account and security > REST API keys; a Deploy key limited to the Environment is enough |
| `command` | `deploy` | `deploy`, `promote`, or `status` to wait for the deployment a push started |
| `environment` | the linked one | The Environment to act on |
| `from` | | For `promote`, the Environment to promote |
| `image` | | `SERVICE=TAG` per line |
| `image-tag` | | One tag or `@sha256:` digest for every image service |
| `timeout` | `30` | Minutes to wait |
| `idempotency-key` | this run attempt | A re-run step follows the deployment it started instead of starting another |
| `version` | the served one | Fail unless the control plane serves this CLI version |
| `url` | `https://app.fastploy.com` | Fastploy's address |

## Outputs

- `result`: the command's JSON document, with each service's status, addresses and, when it failed, `failedStage` and the end of its log.
- `exit-code`: `0` ran, `1` failed, `2` a person must confirm, `3` not allowed, `4` build or image, `5` pre-deploy step, `6` rollout, `7` timeout.
