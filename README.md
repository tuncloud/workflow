# TunCloud Reusable Workflows

Shared GitHub Actions workflows for building, deploying and restarting TunCloud services.

| Workflow | Purpose |
|---|---|
| [`ci-build-push.yml`](.github/workflows/ci-build-push.yml) | Version bump, build and push container images |
| [`cd-deploy.yml`](.github/workflows/cd-deploy.yml) | Deploy to Kubernetes with `kubectl` over a Cloudflare tunnel |
| [`restart-deployment.yml`](.github/workflows/restart-deployment.yml) | Restart a deployment through the TunCloud deploy gateway (no kubeconfig needed) |
| [`notify-telegram.yml`](.github/workflows/notify-telegram.yml) | Standalone Telegram message sender |

---

## `restart-deployment.yml`

Triggers a rolling restart of a Kubernetes deployment and waits until the rollout
finishes.

Unlike [`cd-deploy.yml`](.github/workflows/cd-deploy.yml), this workflow never touches
your cluster credentials. It asks GitHub for a short-lived OIDC token and sends it to
the TunCloud deploy gateway (`https://gateway.tuando.app`), which performs the restart
on your behalf. There is **no `KUBECONFIG`, no Cloudflare Access secret and no long-lived
token** to rotate.

Use it when you want to recycle pods without changing the image — picking up a new
ConfigMap/Secret, clearing a stuck worker, or forcing a re-pull of a mutable `:latest` tag.

### Requirements

1. **The gateway must trust your repository.** The OIDC token is issued for the audience
   `https://gateway.tuando.app`, with the subject identifying the repo, ref and
   environment. The gateway maps that subject to the namespaces and deployments it is
   allowed to restart. A repo that is not registered gets a `403` from the gateway.
2. **The calling job must grant `id-token: write`.** A reusable workflow can only reduce
   the permissions passed down by its caller, never expand them, so declaring the
   permission inside `restart-deployment.yml` is not enough on its own.

### Inputs

| Name | Type | Required | Default | Description |
|---|---|---|---|---|
| `namespace` | string | yes | — | Kubernetes namespace of the deployment |
| `deployment` | string | yes | — | Deployment name to restart |
| `timeout-seconds` | number | no | `600` | Max seconds to wait for the rollout to complete |
| `notify` | boolean | no | `false` | Send a Telegram message on success and on failure |

### Secrets

| Name | Required | Description |
|---|---|---|
| `TELEGRAM_CHAT_ID` | only when `notify: true` | Target chat for the notification |
| `TELEGRAM_BOT_TOKEN` | only when `notify: true` | Bot token used to send the message |

No cluster secrets are required — authentication is entirely OIDC-based.

### Usage

#### Manual restart button

```yaml
name: Restart Service

on:
  workflow_dispatch:
    inputs:
      namespace:
        description: Namespace
        required: true
        default: production
      deployment:
        description: Deployment
        required: true

jobs:
  restart:
    uses: tuncloud/workflow/.github/workflows/restart-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: ${{ inputs.namespace }}
      deployment: ${{ inputs.deployment }}
```

#### With Telegram notifications

```yaml
jobs:
  restart:
    uses: tuncloud/workflow/.github/workflows/restart-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: production
      deployment: api
      timeout-seconds: 900
      notify: true
    secrets:
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
```

`secrets: inherit` also works if the caller repository already exposes both secrets and
you do not need to remap them.

#### Chained after a build

```yaml
jobs:
  build:
    uses: tuncloud/workflow/.github/workflows/ci-build-push.yml@main
    with:
      deployment-name: api

  restart:
    needs: build
    uses: tuncloud/workflow/.github/workflows/restart-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: production
      deployment: api
      notify: true
    secrets: inherit
```

> Restarting only recycles pods. If the new image is published under a **new tag**, use
> [`cd-deploy.yml`](.github/workflows/cd-deploy.yml) with `deployment-type: rollout`
> instead so the tag is actually applied.

### How it works

1. `actions/github-script` requests an OIDC token for the audience `https://gateway.tuando.app`
   and masks it in the logs.
2. `POST /v1/deployments/restart` is called with the namespace and deployment. The gateway
   answers `202 Accepted` with an `operation_id`; any other status fails the job immediately.
3. `GET /v1/operations/{id}` is polled every 5 seconds until the operation reaches a terminal
   state or `timeout-seconds` elapses. Non-`200` responses (expired token, gateway outage) fail
   the job right away rather than spinning until the deadline.
4. If `notify: true`, a Telegram message is sent with the namespace, deployment, operation id,
   actor and a link back to the workflow run.

### Operation states

The message reports the state the run ended in:

| Reported status | Meaning |
|---|---|
| `succeeded` | Rollout completed, all pods ready |
| `failed` | Rollout failed on the cluster side |
| `timeout` | The gateway gave up on the rollout |
| `denied` | The gateway refused the operation for this repository |
| `timed-out` | `timeout-seconds` elapsed while the operation was still running |
| `rejected` | The gateway did not accept the restart request (non-`202`) |
| `unreachable` | Polling failed — expired token or gateway unavailable |
| `unknown` | The run failed before the gateway replied (e.g. the OIDC step) |

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL` | The caller job did not grant `id-token: write` | Add the `permissions` block to the calling job |
| `gateway returned 403` | Repository not registered, or not allowed for that namespace | Register the repo subject with the gateway |
| `gateway returned 404` | Namespace or deployment name does not exist | Check the values with `kubectl get deploy -n <namespace>` |
| `gateway auth failed during polling (token expired?)` | The rollout outlived the OIDC token lifetime | Lower `timeout-seconds`, or investigate why the rollout is that slow |
| `timed out waiting for operation` | Rollout still in progress at the deadline | Raise `timeout-seconds`; check pod events for `ImagePullBackOff` / failing readiness probes |
| Job passes but no Telegram message | `notify` left at `false`, or the secrets were not passed through | Set `notify: true` and pass both `TELEGRAM_*` secrets |
