# Kargo Custom Promotion Steps

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

A custom step is a container image plus a `CustomPromotionStep` manifest that tells Kargo how to run it. Most steps start as a few lines of shell in a public image and only grow into a dedicated image when other people depend on them. This section walks through that whole path.

### The approach at a glance

| # | Stage | The question to answer | Done when |
|---|---|---|---|
| 1 | [Decide](#1-decide-whether-you-need-a-custom-step) | Does Kargo already do this? | You know a built-in step or task can't |
| 2 | [Define the contract](#2-define-the-contract) | What goes in, what comes out, what counts as failure? | You can fill in the worksheet |
| 3 | [Learn the anatomy](#3-learn-the-anatomy-of-a-custompromotionstep) | Which manifest fields do I need? | You have a draft manifest |
| 4 | [Prototype](#4-prototype-with-a-public-image) | Does the tool work inside a promotion at all? | One promotion runs it end to end |
| 5 | [Try it in a stage](#5-try-it-in-a-throwaway-stage) | Does it pass good input and fail bad input? | You've seen both, in Kargo |
| 6 | [Harden](#6-harden-it) | Will it behave when inputs are missing, wrong, or retried? | The checklist passes |
| 7 | [Package](#7-build-test-and-publish-an-image) | Do I need my own image? | A tested, pinned image is published |
| 8 | [Register and document](#8-register-and-document-it) | Can other people find and use it? | It's in the catalog table |

Not every step needs every stage. The steps in this repository sit at different points along it:

| Step | How far it has come | What that means in practice |
|---|---|---|
| `random-number`, `kyverno-policy-check` | 4: prototype | Examples and smoke tests. Not meant to gate anything. |
| `opa-test`, `trivy-image`, `grype`, `extract-images` | 4–5: prototype, used as gates | A public (or lightly customised) image with the tool's CLI and `${{ config.<key> }}` in the command. Good enough for a demo; see 6 before relying on them. |
| `teams-notify`, `oras-push`, `oras-resolve` | 5: prototype with credentials | Work in real pipelines, but read secrets through `config` and build their payloads with shell string interpolation. |
| `kyverno-validate`, `flux-reconcile` | 8: hardened and packaged | Own image, JSON config and output, distinct exit codes, behaviour tests, CI that only publishes tested images. The reference for "done". |

### 1. Decide whether you need a custom step

Custom steps cost something: an image to keep pulling, a pod per promotion, and a cluster-wide name. Check the cheaper options first.

| If what you need is… | Use | Example |
|---|---|---|
| Something Kargo already has a step for (Git, Helm, Kustomize, YAML edits, Argo CD, OCI) | The [built-in step](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/) | `yaml-update`, `helm-template`, `argocd-update` |
| A call to an HTTP API, with pass/fail decided by the response | The built-in `http` step | Many notification webhooks, including what `teams-notify` does with `curl` |
| A fixed sequence of built-in steps you want to reuse | A `PromotionTask` (one project) or `ClusterPromotionTask` (every project) | [`kargo/promote.yaml`](kargo/promote.yaml), [`catalog/oci/`](catalog/oci) |
| A tool binary, or logic Kargo can't express | **A custom step** | `conftest`, `trivy`, `kyverno`, `flyway`, `oras` |

If it's a custom step, check the name isn't taken on your Kargo instance. Names are cluster-wide, and a second registration overwrites the first (see [Prerequisites](#prerequisites)).

### 2. Define the contract

Before writing any YAML, write down what the step promises. These answers become the `config` keys, the `output` section and the exit codes. Here is the worksheet filled in for `teams-notify`:

| Question | `teams-notify` |
|---|---|
| **Inputs**: what must the caller provide? Which are optional, and what are their defaults? | `webhookURL`, `title`, `message`, `stage`, `freight`; all required |
| **Outputs**: what will later steps need? | Nothing |
| **Failure**: what makes the promotion stop? | Teams rejects the request (`curl -f` exits non-zero) |
| **Gate or side effect?** | Side effect: it should run *after* the change is live |
| **Credentials**: what secrets does it need, and from where? | The webhook URL, from a `generic` Secret in the project |
| **Safe to retry?** | Mostly: a retry may post the card twice |
| **Where does it run in the pipeline?** | Last, after `argocd-update` |

Two answers shape the design more than any others:

- **Is it a gate?** A gate (`opa-test`, `kyverno-validate`, `trivy-image`) must *fail* on bad input, which is harder than it sounds (see [Verify the tool, don't trust it](#6-harden-it)). Gates run before `git-commit`, so nothing is pushed when they fail.
- **Does it need outputs?** If a later step depends on a value (`oras-resolve`'s `digest`, `random-number`'s `result`), decide its name and type now. It becomes an API other people's promotions rely on.

### 3. Learn the anatomy of a `CustomPromotionStep`

```yaml
apiVersion: ee.kargo.akuity.io/v1alpha1
kind: CustomPromotionStep
metadata:
  name: my-step                    # what promotions reference with `uses:`; must be unique per Kargo instance
spec:
  image: ghcr.io/example/my-step:v1.0.0   # pin a tag or digest, never :latest
  command: ["/usr/local/bin/my-step"]     # run by Kargo's executor, not as the image ENTRYPOINT
  env:
    - name: MY_STEP_CONFIG
      value: ${{ asJSON(config) }}  # the promotion's `config:` block, as one JSON object
  output:
    source:
      type: Pipe                    # read what the step writes to $KARGO_OUTPUT
      format: JSON                  # or KeyValue; Stdout and File sources also exist
  defaultTimeout: 10m               # hard limit on the step
  defaultErrorThreshold: 1          # attempts before the promotion fails
  resources:                        # all steps of a promotion share one pod
    requests: {cpu: 100m, memory: 64Mi}
    limits: {memory: 256Mi}
```

| Field | What to decide |
|---|---|
| `image` | Which tools the step needs. Use a public image while prototyping; build your own once you need a script, a pre-cached database (see `docker/trivy`), or pinned tool versions. |
| `command` | An absolute path, or `sh -c` with an inline script while prototyping. |
| `env` | How `config` reaches the step. Expressions here can use `config` and `ctx` only. |
| `output` | Where results come from (`Pipe` = `$KARGO_OUTPUT`, `Stdout`, or `File`) and their format. An optional `transform` reshapes them. |
| `defaultTimeout` / `defaultErrorThreshold` | How long the step may run, and how many attempts it gets. Callers can override both per use with `retry:`. |
| `resources` | Requests and limits for the step's container. |

### 4. Prototype with a public image

The quickest way to find out whether a tool is useful is to avoid building anything:

1. Pick a public image that already contains the tool (`openpolicyagent/conftest`, `curlimages/curl`, `ghcr.io/oras-project/oras`).
2. Write the `command` as the tool's own CLI, or `sh -c` with a short script, using `${{ config.<key> }}` for each input.
3. Choose the simplest output that works: `Pipe` with `KeyValue` (`echo "result=$NUM" > $KARGO_OUTPUT`, as in `random-number`) or `Stdout` with a `transform` (as in `oras-resolve`).
4. Add the manifest under `kargo/custom-steps/<name>/` and let Argo CD register it.

Pin the image to a version tag even at this stage. A prototype on `:latest` changes under you between promotions.

### 5. Try it in a throwaway stage

Test in Kargo before you touch a real pipeline:

1. Add the step to a test Stage, or to `dev`, on its own or with only the steps it needs before it (usually `git-clone` and a render step).
2. Promote **input that should pass** and check the result in the Promotion view: the step's output panel shows exactly what Kargo collected.
3. Promote **input that should fail** and check that the promotion stops at your step. For a gate, this is the important test. `prod-eu` is this repository's built-in bad input.
4. Leave a required `config` key out on purpose, and look at what the step does with the literal `<nil>` Kargo passes in its place.

If step 3 passes when it should fail, the tool is exiting `0` on failure. Fix that before going further; see the next section.

### 6. Harden it

A prototype works for the happy path. Before other teams depend on a step, work through this checklist. It is condensed from the conventions in [curated-custom-promotion-tasks](https://github.com/blakepettersson/curated-custom-promotion-tasks/blob/main/CLAUDE.md), where `kyverno-validate` is the reference implementation.

**Inputs**
- **Pass the whole config as one JSON variable** (`${{ asJSON(config) }}`) instead of one `${{ config.<key> }}` per input. A key the caller leaves out renders as the literal string `<nil>`, not as empty, so per-key templating turns a missing input into a wrong one. `asJSON` of an absent config renders as `null`; treat that as `{}` in the script.
- **Validate config once, at the top of the script**, and fail with a clear message if a required key is missing.

**Outputs**
- **Write results as JSON to `$KARGO_OUTPUT`** and declare `output.source: {type: Pipe, format: JSON}`. Later steps can then branch on fields such as `outputs.kyverno.violationCount`.
- **Write the output before exiting non-zero.** Kargo collects it even when the step fails, so the operator sees *why*.
- **Keep it under 256 KiB.** Cap any list you emit (for example, the first 20 violations).
- **Print progress to stdout.** It appears in the step's result message, which is what someone reads when a promotion fails.
- **Never claim more than you checked.** "0 violations" and "nothing was checked" must produce different messages.

**Exit codes**
- `0` passed, `1` the gate found a problem, `2` the step was misconfigured. A distinct code for misconfiguration lets tests tell "bad input" from "bad setup".

**Runtime**
- The container runs as **uid 65532**, non-root, with all capabilities dropped. Write only to the working directory (the promotion workspace shared by every step) and `/tmp`.
- Only `linux/amd64` and `linux/arm64` are supported; build for both.
- **Be idempotent.** A step may be retried. For a deterministic check, set `defaultErrorThreshold: 1`: retrying cannot turn a rejected input into an accepted one.

**Verify the tool, don't trust it**

A gate is only useful if it fails on bad input, so prove that with the real binary. `kyverno apply` is the cautionary example: given a policy it cannot parse, it logs the problem only at `-v 3`, applies whatever else it has, and exits `0`. A step built on that exit code passes every broken policy. Assume any tool can exit `0` on partial failure, and write a test for each failure you rely on catching.

**Credentials**
- Pass secrets with `${{ secret('<name>').<key> }}` from a `generic` Secret in the project, never as literal values in a Stage.
- Never print them. Avoid `set -x` in scripts that handle credentials, and send login output to `/dev/null` (as `oras-push` does).

### 7. Build, test and publish an image

When a step outgrows an inline script:

1. **Write the script** as its own file (POSIX `sh`, `set -eu`, `shellcheck`-clean) and copy it into a minimal image. Copy tool binaries out of upstream images (`COPY --from=...`) rather than installing them, so the image builds for arm64 without emulation. End with `USER 65532:65532`.
2. **Test it the way Kargo runs it**: mount fixtures as the working directory, set the config variable and `KARGO_OUTPUT`, run the image, then assert on the exit code and the JSON. [`kyverno-validate`'s bats tests](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/kyverno/steps/kyverno-validate/test) are a good template.
3. **Publish from CI**, for both platforms, and only after the tests pass. [`.github/workflows/release-trivy.yaml`](.github/workflows/release-trivy.yaml) is a minimal example; [`kyverno-validate.yaml`](https://github.com/blakepettersson/curated-custom-promotion-tasks/blob/main/.github/workflows/kyverno-validate.yaml) adds lint, tests, signing and SBOMs. Image names on `ghcr.io` must be lowercase.
4. **Make the package public** (or add `imagePullSecrets` to the step), then **pin the released tag** in the manifest.

### 8. Register and document it

1. Add the manifest under [`kargo/custom-steps/<name>/`](kargo/custom-steps) and commit. The Argo CD Application that syncs `kargo/` (see [Setup](#setup)) registers it on the Kargo instance. Without Argo CD, use `kargo apply -f kargo/custom-steps/<name>/<name>.yaml`.
2. If you copied it from another repository, keep a header comment with the source URL and commit, so it can be updated later.
3. Add a row to the [Step catalog](#step-catalog), and an example under [Examples](#examples) showing a realistic `config`, any Secrets it needs, and where it belongs in a pipeline.

## Setup

These steps take a fresh fork to a working pipeline. They assume an Argo CD instance and a Kargo instance on the Akuity Platform, and one cluster for the demo workloads.

### 1. Make the repository yours

Replace every reference to the original author's account:

* `repoURL` in [`argocd/appset.yaml`](argocd/appset.yaml), [`argocd/appproj.yaml`](argocd/appproj.yaml) and [`kargo/promote.yaml`](kargo/promote.yaml)
* the policy URL in [`kargo/custom-steps/opa-test/opa-test.yaml`](kargo/custom-steps/opa-test/opa-test.yaml)
* the `image` in [`kargo/custom-steps/trivy-image/trivy-image.yaml`](kargo/custom-steps/trivy-image/trivy-image.yaml) and [`kargo/custom-steps/grype/grype.yaml`](kargo/custom-steps/grype/grype.yaml) (lowercase)

Then run the **Release Trivy** (and optionally **Release Grype**) workflow under the repository's **Actions** tab, and set each package to **public** in GitHub Packages.

### 2. Point the apps at your cluster

Set `destination.name` in [`argocd/appset.yaml`](argocd/appset.yaml) and the destination in [`argocd/appproj.yaml`](argocd/appproj.yaml) to your cluster's name, exactly as it appears in the Akuity Platform.

### 3. Bootstrap Argo CD

1. In the Akuity Platform, open the Argo CD instance → **Settings** → **General**, and enable **Control Plane Cluster (in-cluster)**.
2. In Argo CD, create an Application with project `default`, path `argocd`, and destination **`in-cluster`** (`https://kubernetes.default.svc`), namespace `argocd`. It creates the AppProject and the ApplicationSet, which generates one `custom-steps-<stage>` app per folder in `env/`.

### 4. Prepare Kargo

1. In the Akuity Platform, open the Kargo instance → **Agents** → **Register an agent**. Turn on **Self-hosted Kargo Agent**, connect it to your Argo CD instance, and install it on your cluster.
2. Open the agent's settings and enable **Promotion Controller**.
3. Set `shard:` in each Stage in [`kargo/stages.yaml`](kargo/stages.yaml) to the agent's name.

### 5. Apply the Kargo resources

In Argo CD, create an Application with project `default`, path `kargo`, destination your Kargo instance's cluster (for example `https://cluster-kargo:6445`), and **Directory Recurse enabled**:

```yaml
spec:
  source:
    path: kargo
    directory:
      recurse: true
```

Without recurse, Argo CD only applies the files directly in `kargo/` and skips `kargo/custom-steps/`. The app still shows green, and promotions fail later with `CustomPromotionStep ... not found`.

### 6. Give Kargo push access

Create a GitHub fine-grained token for this repository with **Contents: Read and write**. In the Kargo UI, open the `custom-steps` project → **Settings** → **Credentials**, and add a Git credential for the repository URL with your username and the token. Cloning a public repository works without it; pushing does not.

### 7. Promote

Promote the Warehouse's Freight to `dev`, then onwards. `prod-eu` is expected to fail the policy gates (see [Putting it all together](#putting-it-all-together)).

### Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `step kind "opa-test" is not registered ... enable the promotion controller` | The Stage runs on an agent without the Promotion Controller | Enable it on a self-hosted agent and set `shard:` on the Stage (Setup step 4) |
| `CustomPromotionStep.ee.kargo.akuity.io "opa-test" not found` | The step manifests were never applied | Enable Directory Recurse on the Kargo app (Setup step 5) |
| `SharedResourceWarning: CustomPromotionStep/<name> is part of applications ...` | Another repository registers a step with the same name | Rename one of them, or exclude the file from one app (`directory.exclude`) |
| `fatal: could not read Username for 'https://github.com'` on `git-clone` | The repository URL is wrong (GitHub asks for credentials when a repository doesn't exist) | Check the spelling of `repoURL` |
| The same error on `git-push` | Kargo has no write credentials | Add a Git credential (Setup step 6) |
| `application destination ... do not match any of the allowed destinations in project 'custom-steps'` | An app outside the demo uses the locked-down `custom-steps` AppProject | Use project `default` for the bootstrap and Kargo apps |
| `ImagePullBackOff` for `nginx:latest` | The chart's default tag is `latest`, which the demo image doesn't publish | Set `image.tag` in the stage's `env/<stage>/values.yaml` |
