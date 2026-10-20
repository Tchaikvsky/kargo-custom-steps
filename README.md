# Kargo Custom Promotion Steps

## Contents

- [Overview](#overview) · [Why custom steps?](#why-custom-steps) · [Prerequisites](#prerequisites)
- [Architecture](#architecture) · [How it works](#how-it-works)
- [Step catalog](#step-catalog)
- [Examples](#examples): [OPA](#open-policy-agent-test) · [Kyverno](#kyverno-validate) · [Trivy](#trivy-image-scanning) · [Flux](#flux-reconcile) · [Teams](#teams-notifications) · [ORAS](#pushing-manifests-to-oci-with-oras) · [OCI tasks](#oci-promotion-tasks)
- [Putting it all together](#putting-it-all-together)
- [Creating your own custom steps](#creating-your-own-custom-steps) → [full guide](docs/creating-custom-steps.md)

## Overview

Custom Promotion Steps allow Kargo administrators to easily introduce new promotion steps to Kargo that aren't necessarily included right out of the box.

## Why custom steps?

Kargo's built-in steps handle the most common GitOps promotion operations with native integrations for Argo CD, Terraform, OCI, and Git. But every organization has unique requirements such as custom security gates, internal tooling, or compliance checks that don't fit a standard mold. With custom steps, it's possible to customize Kargo to meet these requirements.

Examples of when you would use custom steps include:
* Supply chain security verification: `cosign`, `notation`
* Image vulnerability scanning: `grype`, `trivy`
* Policy validation and linting: `conftest`, `kyverno`, `kubeconform`
* Database schema migrations: `flyway`, `liquibase`
* Infrastructure automation: `ansible`, `chef`, `puppet`
* Bespoke tools, scripts, and internal system integrations

## Prerequisites

Custom promotion steps run as pods in a cluster you own, so they need extra setup on the Kargo side:

1. **Kargo v1.10 or newer.** Check the version on your Kargo instance's overview page in the Akuity Platform.
2. **A self-hosted Kargo agent.** Promotions that use custom steps run as pods inside your own cluster. Because of this, a self-hosted agent is required.
3. **The Promotion Controller enabled on that agent.** Without it, Kargo doesn't recognize custom step kinds and promotions fail with a message that may look similar to this:

   ```
   step kind "<custom-step-name>" is not registered; verify the kind is correct; if you are a Kargo Enterprise user, you may need to enable the promotion controller
   ```

4. **Stages assigned to that agent.** Set `spec.shard` on each Stage to the self-hosted agent's name, unless that agent is your default shard.

> [!IMPORTANT]
> `CustomPromotionStep` is **cluster-scoped**: a step is registered once for the whole Kargo instance, not per project. If two repositories register a step with the same name, they overwrite each other. Check the existing steps before you pick a name.

## How it works

To define a new promotion step, a `CustomPromotionStep` resource is registered which specifies:
* The container `image` to run.
* The `command` to be executed in the container.
* Optional `config` inputs to pass into the command.
* Optional `output` to collect from the result of execution.

```yaml
apiVersion: ee.kargo.akuity.io/v1alpha1
kind: CustomPromotionStep
metadata:
  name: random-number
spec:
  image: busybox:1.37.0
  command:
  - sh
  - -c
  - |
    NUM=$(awk 'BEGIN{srand(); print int(rand() * ${{ config.range }}) + 1}')
    echo "Generated number: $NUM"
    echo "result=$NUM" > $KARGO_OUTPUT
  output:
    source:
      # collects key=value outputs from the $KARGO_OUTPUT pipe
      type: Pipe
```

Once registered, the step can be used like any other Kargo step. Outputs are referenced in subsequent steps via `outputs['<as>'].<key>`:

```yaml
steps:
- uses: random-number
  as: random-number
  config:
    range: 100

- uses: yaml-update
  config:
    path: ./src/values.yaml
    updates:
    - key: num
      value: ${{ outputs['random-number'].result }}
```

## Step catalog

Every step below is registered by a file in [`kargo/custom-steps/`](kargo/custom-steps).

| Step | What it does | Image | Maintained in |
|---|---|---|---|
| `opa-test` | Validates rendered manifests against [`policy/kubernetes.rego`](policy/kubernetes.rego) with `conftest` | `openpolicyagent/conftest` | this repo |
| `kyverno-validate` | Validates Kyverno policies, then checks rendered manifests against them | `ghcr.io/blakepettersson/curated-custom-promotion-tasks/kyverno-validate` | [curated-custom-promotion-tasks](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/kyverno/steps/kyverno-validate) |
| `trivy-image` | Fails on `CRITICAL` vulnerabilities in the promoted image | `ghcr.io/<yourgithubusername>/trivy` | this repo ([`docker/trivy`](docker/trivy)) |
| `grype` | Alternative image scanner to `trivy-image` | `ghcr.io/<yourgithubusername>/grype` | this repo ([`docker/grype`](docker/grype)) |
| `extract-images` | Lists the images referenced by a directory of manifests | `mikefarah/yq` | this repo |
| `random-number` | Minimal example: returns a random number | `busybox` | this repo |
| `flux-reconcile` | Flux counterpart of `argocd-update`: reconciles Flux objects and waits for the promoted revision | `ghcr.io/blakepettersson/curated-custom-promotion-tasks/flux-reconcile` | [curated-custom-promotion-tasks](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/flux/steps/flux-reconcile) |
| `kyverno-policy-check` | Prints the Kyverno CLI version: a smoke test that the CLI image runs in your promotion pods | `ghcr.io/kyverno/kyverno-cli` | [sedemo-platform](https://github.com/akuity/sedemo-platform/tree/main/kargo-shared) |
| `teams-notify` | Posts an Adaptive Card to a Microsoft Teams channel | `curlimages/curl` | [sedemo-platform](https://github.com/akuity/sedemo-platform/tree/main/kargo-shared) |
| `oras-push` | Packages a directory as tar+gzip and pushes it to an OCI registry for Argo CD to sync from | `ghcr.io/oras-project/oras` | [sedemo-platform](https://github.com/akuity/sedemo-platform/tree/main/kargo-shared) |
| `oras-resolve` | Resolves an OCI tag to its digest, for `argocd-update`'s `desiredRevision` | `ghcr.io/oras-project/oras` | [sedemo-platform](https://github.com/akuity/sedemo-platform/tree/main/kargo-shared) |

Steps maintained elsewhere are copied here unchanged, with a header naming the source and commit. To upgrade one, re-copy it (or bump its `image` tag) after reading the upstream changes.

[`catalog/oci/`](catalog/oci) also holds three `ClusterPromotionTask`s for publishing rendered manifests as OCI artifacts. They are **not** applied by default; see [OCI promotion tasks](#oci-promotion-tasks).

## Examples

This repository showcases several real-world examples of custom promotion steps:

* `opa-test` and `kyverno-validate` - validate Kubernetes manifests against centrally managed policies, before proceeding with promotion.
* `trivy-image` - scan the promoted image for vulnerabilities, before proceeding with promotion.
* `flux-reconcile` - deploy with Flux instead of Argo CD.
* `teams-notify` - tell a Teams channel about a promotion.
* `oras-push` and `oras-resolve` - deploy rendered manifests from an OCI registry instead of Git.

### Open Policy Agent Test

Open Policy Agent (OPA) is an open-source, general-purpose policy engine that allows you to
define and enforce fine-grained authorization and compliance rules across your stack using the
Rego policy language. For testing policies against structured configuration files (Kubernetes
manifests, Dockerfiles, Terraform, etc.), the companion tool `conftest` can be used.

This example shows how `conftest` can be used to validate Kubernetes deployment YAML against centrally managed OPA rules.

```yaml
apiVersion: ee.kargo.akuity.io/v1alpha1
kind: CustomPromotionStep
metadata:
  name: opa-test
spec:
  image: openpolicyagent/conftest:v0.67.0
  command:
  - conftest
  - test
  - --no-color
  - --update=https://raw.githubusercontent.com/<yourgithubusername>/kargo-custom-steps/main/policy/kubernetes.rego
  - --policy=/tmp/policy
  - ${{ config.path }}
```

> [!NOTE]
> The policy is downloaded from the URL in `--update` on every run. Point it at your own fork, or edits to `policy/kubernetes.rego` will have no effect.

### Kyverno Validate

[Kyverno](https://kyverno.io) is a Kubernetes-native policy engine whose policies are written as Kubernetes resources rather than in Rego. The `kyverno-validate` step runs the Kyverno CLI against your rendered manifests, so policies you might already enforce at admission time can also gate a promotion.

[`policy/kyverno/`](policy/kyverno) holds the same two rules as `policy/kubernetes.rego` (no privileged containers, no `hostNetwork`), so the pipeline checks every render with both engines. You only need one of them in practice: the pair is here to show both.

```yaml
- uses: kyverno-validate
  as: kyverno
  config:
    policies: ./src/policy/kyverno   # files, directories or globs of policies
    manifests: ./out                 # the helm-template output
```

The step does more than call `kyverno apply`, which can exit `0` for policies it failed to load. It loads each policy on its own, fails on load errors, and returns structured output (`passed`, `violationCount`, `violations`, `message`) that later steps can use. See the [step's README](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/kyverno/steps/kyverno-validate) for every option, including `failOn` to make policies advisory.

`kyverno-policy-check` is a much smaller step: it only runs `kyverno version`. It is useful as a first step when you are wiring up Kyverno, to confirm the CLI image pulls and runs in your promotion pods, but it does not validate anything.

> [!NOTE]
> Kyverno policies are written against `Pod`s. Kyverno automatically generates matching rules for `Deployment`s, `StatefulSet`s and other workload kinds, so rendered Deployments are covered.

### Trivy Image Scanning

Trivy is an open-source, all-in-one security scanner that detects vulnerabilities, misconfigurations, secrets, and license issues across container images, filesystems, Git repositories, and cloud infrastructure.

The following example shows how a `trivy-image` step can scan an image for `CRITICAL` vulnerabilities before proceeding with the promotion.

> **Note:** The custom image `ghcr.io/<yourgithubusername>/trivy` pre-caches the vulnerability databases to avoid downloading them on every promotion run. Build it with the `Release Trivy` workflow in [`.github/workflows`](.github/workflows), then set the package to public. See [`docker/trivy/README.md`](docker/trivy/README.md) for details.

```yaml
apiVersion: ee.kargo.akuity.io/v1alpha1
kind: CustomPromotionStep
metadata:
  name: trivy-image
spec:
  image: ghcr.io/<yourgithubusername>/trivy:0.69.3
  command:
  - trivy
  - image
  - --severity=CRITICAL
  - ${{ config.image }}
```

### Flux Reconcile

If your stages deploy with [Flux](https://fluxcd.io) rather than Argo CD, `flux-reconcile` replaces the final `argocd-update` step. It asks Flux to reconcile now and waits until Flux has applied the revision the promotion produced:

```yaml
- uses: flux-reconcile
  as: reconcile
  config:
    kubeconfig: ${{ secret('flux-cluster').kubeconfig }}
    targets:
      - kind: Kustomization
        name: apps
        desiredRevision: ${{ outputs.push.commit }}
```

It is registered here but not used by the demo pipeline, which deploys with Argo CD. Before using it you need:

* a ServiceAccount in the Flux cluster that can `get` and `patch` Flux objects ([example RBAC](https://github.com/blakepettersson/curated-custom-promotion-tasks/blob/main/flux/steps/flux-reconcile/examples/rbac.yaml)), and
* a kubeconfig for it, stored in the Kargo project as a `generic` credential Secret ([example Secret](https://github.com/blakepettersson/curated-custom-promotion-tasks/blob/main/flux/steps/flux-reconcile/examples/kargo-secret.yaml)).

See the [step's README](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/flux/steps/flux-reconcile) for the full configuration.

### Teams notifications

`teams-notify` posts an Adaptive Card to a Microsoft Teams channel through a Power Automate workflow webhook. Add it after `argocd-update`, so the message is only sent once the promotion has succeeded:

```yaml
- uses: teams-notify
  config:
    webhookURL: ${{ secret('teams-webhook').url }}
    title: Promoted to ${{ ctx.stage }}
    message: ${{ vars.image }}:${{ imageFrom(vars.image).Tag }} is live
    stage: ${{ ctx.stage }}
    freight: ${{ ctx.targetFreight.name }}
```

To set it up:

1. In Teams, open the channel → **Workflows** → **Post to a channel when a webhook request is received**, and copy the webhook URL.
2. Store the URL in the Kargo project as a Secret labelled `kargo.akuity.io/cred-type: generic`, so `secret('teams-webhook')` can read it:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: teams-webhook
     namespace: custom-steps
     labels:
       kargo.akuity.io/cred-type: generic
   stringData:
     url: https://<your-webhook-url>
   ```

The step builds its JSON by pasting `title` and `message` into a string, so a double quote in either breaks the card. Keep them to plain text.

### Pushing manifests to OCI with ORAS

`oras-push` and `oras-resolve` let Argo CD sync rendered manifests from an OCI registry instead of a Git branch. Unlike the tasks in [OCI promotion tasks](#oci-promotion-tasks), they work on released Kargo versions, because they run the [ORAS](https://oras.land) CLI rather than Kargo's built-in `oci-push`.

```yaml
# Render every stage into ./bundle/<stage>/ first (helm-template or kustomize-build), then:
- uses: oras-push
  config:
    registry: ghcr.io
    repository: <yourgithubusername>/custom-steps-manifests
    tag: ${{ imageFrom(vars.image).Tag }}
    path: ./bundle
    orasUsername: ${{ secret('oci-registry').username }}
    orasPassword: ${{ secret('oci-registry').password }}

# oras-push does not output the digest, so look it up.
- uses: oras-resolve
  as: resolve
  config:
    registry: ghcr.io
    repository: <yourgithubusername>/custom-steps-manifests
    tag: ${{ imageFrom(vars.image).Tag }}
    orasUsername: ${{ secret('oci-registry').username }}
    orasPassword: ${{ secret('oci-registry').password }}

- uses: argocd-update
  config:
    apps:
    - name: custom-steps-${{ ctx.stage }}
      sources:
      - repoURL: oci://ghcr.io/<yourgithubusername>/custom-steps-manifests
        desiredRevision: ${{ outputs.resolve.digest }}
```

`argocd-update` needs the **digest**, not the tag: Argo CD records digests in its sync status for OCI sources, so a tag never matches and the step waits until it times out. That is the only reason `oras-resolve` exists.

To use these, the Argo CD Applications need an OCI source (`repoURL: oci://...`, with `path` selecting the stage's subdirectory), and the Kargo project needs a `generic` Secret named `oci-registry` with `username` and `password` (for GitHub, a token with `write:packages`). The demo pipeline in this repository deploys from Git and does not use them.

### OCI promotion tasks

[`catalog/oci/`](catalog/oci) holds three `ClusterPromotionTask`s from [curated-custom-promotion-tasks](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/oci). They are not custom steps: they are built entirely from Kargo's built-in steps, so there is no image to run.

| Task | What it does |
|---|---|
| `publish-manifests-to-oci` | Packages a directory of rendered manifests, publishes it as an OCI artifact, and records the digest on the Freight |
| `deploy-oci-to-argocd` | Points an Argo CD Application's OCI source at that digest and waits for it to sync |
| `await-oci-deploy` | Waits for a cluster Kargo cannot reach to report that it is running a digest |

> [!WARNING]
> These tasks need a Kargo built from `main`: they use `oci-push`'s local-archive mode ([akuity/kargo#6893](https://github.com/akuity/kargo/pull/6893)), which is not in a released version yet. That is why they live outside `kargo/` and are not applied by Argo CD. Once your Kargo instance supports it, apply them with `kargo apply -f catalog/oci/<task>.yaml`.

## Putting it all together

The following `PromotionTask` demonstrates how built-in and custom steps compose into a complete promotion pipeline:

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote
  namespace: custom-steps
spec:
  vars:
  - name: image
    value: ghcr.io/<yourgithubusername>/nginx
  - name: repoURL
    value: https://github.com/<yourgithubusername>/kargo-custom-steps.git
  - name: branch
    value: main

  steps:
  # 1. Clone the repo containing Helm charts and environment values
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - branch: ${{ vars.branch }}
        path: ./src

  # 2. Update the image tag in the environment-specific values file
  - uses: yaml-update
    config:
      path: ./src/env/${{ ctx.stage }}/values.yaml
      updates:
      - key: image.tag
        value: ${{ imageFrom( vars.image ).Tag }}

  # 3. Render Helm templates to produce Kubernetes manifests
  - uses: helm-template
    config:
      path: ./src/charts/app
      releaseName: app
      valuesFiles:
      - ./src/env/${{ ctx.stage }}/values.yaml
      outPath: ./out

  # 4. (Custom) Validate rendered manifests against OPA policy rules
  - uses: opa-test
    config:
      path: ./out

  # 5. (Custom) Validate the same manifests against Kyverno policies
  - uses: kyverno-validate
    as: kyverno
    config:
      policies: ./src/policy/kyverno
      manifests: ./out

  # 6. (Custom) Scan the promoted image for CRITICAL vulnerabilities
  ##   grype or trivy can be used interchangeably here. For this example, we will use trivy
  - uses: trivy-image
    config:
      image: ${{ vars.image }}:${{ imageFrom( vars.image ).Tag }}

  # 7. Commit and push the GitOps change to trigger the deploy
  - uses: git-commit
    config:
      path: ./src
      message: Update ${{ ctx.stage }} to ${{ vars.image }}:${{ imageFrom(vars.image).Tag }}

  - uses: git-push
    config:
      path: ./src

  # 8. Have Argo CD sync these changes
  - uses: argocd-update
    config:
      apps:
      - name: custom-steps-${{ ctx.stage }}
```

The custom steps (`opa-test`, `kyverno-validate` and `trivy-image`) act as policy and security gates — if any of them fails, the promotion is blocked before any changes are committed and synced in Argo CD.

`env/prod-eu/values.yaml` sets `privileged: true` on purpose, so promoting to `prod-eu` shows both policy gates rejecting the same render.

## Creating your own custom steps

**[docs/creating-custom-steps.md](docs/creating-custom-steps.md)** walks through building a step from idea to registered, documented step, in eight stages:

1. [Decide whether you need a custom step](docs/creating-custom-steps.md#1-decide-whether-you-need-a-custom-step), or whether a built-in step or a `PromotionTask` will do
2. [Define the contract](docs/creating-custom-steps.md#2-define-the-contract): inputs, outputs, and what counts as failure
3. [Learn the anatomy of a `CustomPromotionStep`](docs/creating-custom-steps.md#3-learn-the-anatomy-of-a-custompromotionstep)
4. [Prototype with a public image](docs/creating-custom-steps.md#4-prototype-with-a-public-image)
5. [Try it in a throwaway stage](docs/creating-custom-steps.md#5-try-it-in-a-throwaway-stage)
6. [Harden it](docs/creating-custom-steps.md#6-harden-it)
7. [Build, test and publish an image](docs/creating-custom-steps.md#7-build-test-and-publish-an-image)
8. [Register and document it](docs/creating-custom-steps.md#8-register-and-document-it)

## Setup

**[docs/setup.md](docs/setup.md)** takes a fresh fork of this repository to a working pipeline: pointing it at your own account and cluster, bootstrapping Argo CD, preparing Kargo (self-hosted agent, Promotion Controller, shards), applying the Kargo resources, and Git credentials. It ends with a [troubleshooting table](docs/setup.md#troubleshooting) of the errors you are most likely to hit.
