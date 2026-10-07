# Kargo Custom Promotion Steps

## Overview

Custom Promotion Steps allow Kargo administrators to easily introduce new promotion steps to Kargo.

## Why custom steps?

Kargo's built-in steps handle the most common GitOps promotion operations out of the box, with native integrations for Argo CD, Terraform, OCI, and Git. But every organization has unique requirements — custom security gates, internal tooling, or compliance checks that don't fit a standard mold. Custom steps make these possible.

Examples of when you would use custom steps include:
* Supply chain security verification: `cosign`, `notation`
* Image vulnerability scanning: `grype`, `trivy`
* Policy validation and linting: `conftest`, `kyverno`, `kubeconform`
* Database schema migrations: `flyway`, `liquibase`
* Infrastructure automation: `ansible`, `chef`, `puppet`
* Bespoke tools, scripts, and internal system integrations

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

Once registered, the step can be used like any other kargo step. Outputs are referenced in subsequent steps via `outputs['<as>'].<key>`:

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

NOTE: Custom promotion steps require pod-based promotions available only on the Akuity Platform.

## Examples

This repository showcases two real-world examples of custom promotion steps:

* `opa-test` - validate Kubernetes manifests using centrally managed OPA rules, before proceeding with promotion.
* `trivy-image` - scan the promoted image for vulnerabilities, before proceeding with promotion.

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

### Trivy Image Scanning

Trivy is an open-source, all-in-one security scanner that detects vulnerabilities, misconfigurations, secrets, and license issues across container images, filesystems, Git repositories, and cloud infrastructure.

The following example shows how a `trivy-image` step can scan an image for `CRITICAL` vulnerabilities before proceeding with the promotion.

> **Note:** The custom image `ghcr.io/<yourgithubusername>/trivy` pre-caches the vulnerability databases to avoid downloading them on every promotion run. See [`docker/trivy/README.md`](docker/trivy/README.md) for details.

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

  # 5. (Custom) Scan the promoted image for CRITICAL vulnerabilities
  - uses: trivy-image
    config:
      image: ${{ vars.image }}:${{ imageFrom( vars.image ).Tag }}

  # 6. Commit and push the GitOps change to trigger the deploy
  - uses: git-commit
    config:
      path: ./src
      message: Update ${{ ctx.stage }} to ${{ vars.image }}:${{ imageFrom(vars.image).Tag }}

  - uses: git-push
    config:
      path: ./src
```

The custom steps (`opa-test` and `trivy-image`) act as policy and security gates — if either fails, the promotion is blocked before any changes are committed.
