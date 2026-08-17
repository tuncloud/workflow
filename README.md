# TunCloud Reusable Workflows

Shared GitHub Actions workflows for building, deploying and restarting TunCloud services.

| Workflow | Purpose |
| --- | --- |
| [`ci-build-push.yml`](.github/workflows/ci-build-push.yml) | Version bump, build and push container images |
| [`rollout-deployment.yml`](.github/workflows/rollout-deployment.yml) | Update a deployment's image through the TunCloud deploy gateway |
| [`restart-deployment.yml`](.github/workflows/restart-deployment.yml) | Restart a deployment through the TunCloud deploy gateway |
| [`cd-deploy.yml`](.github/workflows/cd-deploy.yml) | Deploy or restart with `kubectl` over a Cloudflare tunnel |
| [`notify-telegram.yml`](.github/workflows/notify-telegram.yml) | Standalone Telegram message sender |

## Which deployment workflow?

| | `rollout-deployment` | `restart-deployment` | `cd-deploy` |
| --- | --- | --- | --- |
| Changes the image | yes | no | yes (`deployment-type: rollout`) |
| Auth | GitHub OIDC | GitHub OIDC | `KUBECONFIG` + Cloudflare Access secrets |
| Cluster secrets needed | none | none | 4 secrets |
| Multi-container update | one container per call | n/a | all containers in one call (`targets`) |
| Auto-rollback on failure | no | n/a | yes |
| Telegram notification | not yet | `notify: true` | `notify: true` |

Prefer the gateway workflows (`rollout-deployment` / `restart-deployment`) for new
services — they need no cluster credentials at all. Use `cd-deploy` when you must update
several containers atomically or want the built-in rollback.

---

## Deploy gateway workflows

`rollout-deployment.yml` and `restart-deployment.yml` share one mechanism. Neither ever
touches your cluster credentials: the job asks GitHub for a short-lived OIDC token and
sends it to the TunCloud deploy gateway (`https://gateway.tuando.app`), which performs the
operation on your behalf. There is **no `KUBECONFIG`, no Cloudflare Access secret and no
long-lived token** to rotate.

Both follow the same sequence:

1. `actions/github-script` requests an OIDC token for the audience `https://gateway.tuando.app`
   and masks it in the logs.
2. The operation is submitted to the gateway, which answers `202 Accepted` with an
   `operation_id`. Any other status fails the job immediately.
3. `GET /v1/operations/{id}` is polled every 5 seconds until the operation reaches a terminal
   state or `timeout-seconds` elapses. Non-`200` responses (expired token, gateway outage) fail
   the job right away rather than spinning until the deadline.

### Requirements

1. **The gateway must trust your repository.** The OIDC token is issued for the audience
   `https://gateway.tuando.app`, with the subject identifying the repo, ref and
   environment. The gateway maps that subject to the namespaces and deployments it is
   allowed to act on. A repo that is not registered gets a `403` from the gateway.
2. **The calling job must grant `id-token: write`.** A reusable workflow can only reduce
   the permissions passed down by its caller, never expand them, so declaring the
   permission inside the called workflow is not enough on its own.

### Operation states

| Status | Meaning |
| --- | --- |
| `succeeded` | Operation completed, all pods ready |
| `failed` | Operation failed on the cluster side |
| `timeout` | The gateway gave up on the operation |
| `denied` | The gateway refused the operation for this repository |

If the deadline passes while the operation is still running, the job fails with
`timed out waiting for operation <id>` — the operation itself keeps running on the cluster.

### Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL` | The caller job did not grant `id-token: write` | Add the `permissions` block to the calling job |
| `gateway returned 403` | Repository not registered, or not allowed for that namespace | Register the repo subject with the gateway |
| `gateway returned 404` | Namespace or deployment name does not exist | Check with `kubectl get deploy -n <namespace>` |
| `gateway returned 400` | Rejected payload — usually an unknown `container` name | Check the container names with `kubectl get deploy <name> -n <ns> -o jsonpath='{.spec.template.spec.containers[*].name}'` |
| `gateway auth failed during polling (token expired?)` | The operation outlived the OIDC token lifetime | Lower `timeout-seconds`, or investigate why the rollout is that slow |
| `timed out waiting for operation` | Still in progress at the deadline | Raise `timeout-seconds`; check pod events for `ImagePullBackOff` / failing readiness probes |

---

## `rollout-deployment.yml`

Points a deployment at a new container image and waits until the rollout finishes. This is
the normal "ship a new build" step: pair it with [`ci-build-push.yml`](.github/workflows/ci-build-push.yml)
and feed it the tag that job produced.

### Rollout inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `namespace` | string | yes | — | Kubernetes namespace of the deployment |
| `deployment` | string | yes | — | Deployment name to update |
| `image` | string | yes | — | Full image reference, e.g. `ghcr.io/org/app:sha-abc1234` |
| `container` | string | no | `""` | Container to update. Leave empty for single-container pods — the gateway resolves it. Required when the pod has more than one container. |
| `timeout-seconds` | number | no | `600` | Max seconds to wait for the rollout to complete |

No secrets are required — authentication is entirely OIDC-based.

### Rollout usage

#### Build, then roll out

```yaml
name: Deploy API

on:
  push:
    branches: [main]

jobs:
  build:
    uses: tuncloud/workflow/.github/workflows/ci-build-push.yml@main
    with:
      deployment-name: api
    secrets: inherit

  rollout:
    needs: build
    uses: tuncloud/workflow/.github/workflows/rollout-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: production
      deployment: api
      image: docker.io/tuncloud/api:${{ needs.build.outputs.tag }}
```

`ci-build-push.yml` exposes the version it created as the `tag` output, so the two jobs
stay in sync without hardcoding a version anywhere.

#### Manual rollout button

Drop this in the **service** repository to get a *Run workflow* button that rolls out any
image on demand — handy for hotfixes, for promoting a tag that was built earlier, and for
reverting a bad release:

```yaml
name: Rollout Deployment

on:
  workflow_dispatch:
    inputs:
      namespace:
        description: Namespace
        required: true
        type: choice
        default: production
        options: [production, staging]
      deployment:
        description: Deployment
        required: true
        type: string
      image:
        description: Full image reference, e.g. docker.io/tuncloud/api:v1.4.2
        required: true
        type: string
      container:
        description: Container to update (leave empty for single-container pods)
        required: false
        type: string
        default: ''

jobs:
  rollout:
    uses: tuncloud/workflow/.github/workflows/rollout-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: ${{ inputs.namespace }}
      deployment: ${{ inputs.deployment }}
      image: ${{ inputs.image }}
      container: ${{ inputs.container }}
```

Keep the button in the service repo rather than here: the OIDC token is minted for the
repository the run belongs to, and that repository is the subject the gateway checks
against its allow-list.

An empty `container` is passed straight through and behaves exactly like omitting it — the
workflow drops the key so the gateway resolves the single container itself.

To revert a bad release, run the button again with the previous image reference. This
workflow does **not** roll back on its own — a failed rollout leaves the deployment on the
failed revision.

#### Multi-container deployments

Each call updates a single container, so a pod built from several `ci-build-push` targets
needs one job per container:

```yaml
jobs:
  rollout-api:
    needs: build
    uses: tuncloud/workflow/.github/workflows/rollout-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: production
      deployment: platform
      container: api
      image: docker.io/tuncloud/platform-api:${{ needs.build.outputs.tag }}

  rollout-worker:
    needs: build
    uses: tuncloud/workflow/.github/workflows/rollout-deployment.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      namespace: production
      deployment: platform
      container: worker
      image: docker.io/tuncloud/platform-worker:${{ needs.build.outputs.tag }}
```

Note that these are two separate rollouts — pods restart twice, and the containers are
briefly on mismatched versions. When that matters, use
[`cd-deploy.yml`](.github/workflows/cd-deploy.yml) with `targets: api,worker`, which sets
every image in one `kubectl` call.

---

## `restart-deployment.yml`

Triggers a rolling restart without changing the image, and waits until the rollout
finishes. Use it to recycle pods after a ConfigMap/Secret change, to clear a stuck worker,
or to force a re-pull of a mutable `:latest` tag.

> Restarting only recycles pods. If a build published a **new tag**, use
> [`rollout-deployment.yml`](.github/workflows/rollout-deployment.yml) instead — a restart
> will not pick it up.

### Restart inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `namespace` | string | yes | — | Kubernetes namespace of the deployment |
| `deployment` | string | yes | — | Deployment name to restart |
| `timeout-seconds` | number | no | `600` | Max seconds to wait for the rollout to complete |
| `notify` | boolean | no | `false` | Send a Telegram message on success and on failure |

### Restart secrets

| Name | Required | Description |
| --- | --- | --- |
| `TELEGRAM_CHAT_ID` | only when `notify: true` | Target chat for the notification |
| `TELEGRAM_BOT_TOKEN` | only when `notify: true` | Bot token used to send the message |

### Restart usage

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

The message carries the namespace, deployment, operation id, actor and a link back to the
workflow run. On failure it also reports where the run died, which is not always a gateway
state:

| Reported status | Meaning |
| --- | --- |
| `failed` / `timeout` / `denied` | Terminal state reported by the gateway |
| `timed-out` | `timeout-seconds` elapsed while the operation was still running |
| `rejected` | The gateway did not accept the restart request (non-`202`) |
| `unreachable` | Polling failed — expired token or gateway unavailable |
| `unknown` | The run failed before the gateway replied (e.g. the OIDC step) |
