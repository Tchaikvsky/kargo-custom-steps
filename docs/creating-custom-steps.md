# Creating your own custom steps

[← Back to the README](../README.md)

A custom step is a container image plus a `CustomPromotionStep` manifest that tells Kargo how to run it. Most steps start as a few lines of shell in a public image and only grow into a dedicated image when other people depend on them. This section walks through that whole path.

## The approach at a glance

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

## 1. Decide whether you need a custom step

Custom steps cost something: an image to keep pulling, a pod per promotion, and a cluster-wide name. Check the cheaper options first.

| If what you need is… | Use | Example |
|---|---|---|
| Something Kargo already has a step for (Git, Helm, Kustomize, YAML edits, Argo CD, OCI) | The [built-in step](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/) | `yaml-update`, `helm-template`, `argocd-update` |
| A call to an HTTP API, with pass/fail decided by the response | The built-in `http` step | Many notification webhooks, including what `teams-notify` does with `curl` |
| A fixed sequence of built-in steps you want to reuse | A `PromotionTask` (one project) or `ClusterPromotionTask` (every project) | [`kargo/promote.yaml`](../kargo/promote.yaml), [`catalog/oci/`](../catalog/oci) |
| A tool binary, or logic Kargo can't express | **A custom step** | `conftest`, `trivy`, `kyverno`, `flyway`, `oras` |

If it's a custom step, check the name isn't taken on your Kargo instance. Names are cluster-wide, and a second registration overwrites the first (see [Prerequisites](../README.md#prerequisites)).

## 2. Define the contract

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

## 3. Learn the anatomy of a `CustomPromotionStep`

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

## 4. Prototype with a public image

The quickest way to find out whether a tool is useful is to avoid building anything:

1. Pick a public image that already contains the tool (`openpolicyagent/conftest`, `curlimages/curl`, `ghcr.io/oras-project/oras`).
2. Write the `command` as the tool's own CLI, or `sh -c` with a short script, using `${{ config.<key> }}` for each input.
3. Choose the simplest output that works: `Pipe` with `KeyValue` (`echo "result=$NUM" > $KARGO_OUTPUT`, as in `random-number`) or `Stdout` with a `transform` (as in `oras-resolve`).
4. Add the manifest under `kargo/custom-steps/<name>/` and let Argo CD register it.

Pin the image to a version tag even at this stage. A prototype on `:latest` changes under you between promotions.

## 5. Try it in a throwaway stage

Test in Kargo before you touch a real pipeline:

1. Add the step to a test Stage, or to `dev`, on its own or with only the steps it needs before it (usually `git-clone` and a render step).
2. Promote **input that should pass** and check the result in the Promotion view: the step's output panel shows exactly what Kargo collected.
3. Promote **input that should fail** and check that the promotion stops at your step. For a gate, this is the important test. `prod-eu` is this repository's built-in bad input.
4. Leave a required `config` key out on purpose, and look at what the step does with the literal `<nil>` Kargo passes in its place.

If step 3 passes when it should fail, the tool is exiting `0` on failure. Fix that before going further; see the next section.

## 6. Harden it

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

## 7. Build, test and publish an image

When a step outgrows an inline script:

1. **Write the script** as its own file (POSIX `sh`, `set -eu`, `shellcheck`-clean) and copy it into a minimal image. Copy tool binaries out of upstream images (`COPY --from=...`) rather than installing them, so the image builds for arm64 without emulation. End with `USER 65532:65532`.
2. **Test it the way Kargo runs it**: mount fixtures as the working directory, set the config variable and `KARGO_OUTPUT`, run the image, then assert on the exit code and the JSON. [`kyverno-validate`'s bats tests](https://github.com/blakepettersson/curated-custom-promotion-tasks/tree/main/kyverno/steps/kyverno-validate/test) are a good template.
3. **Publish from CI**, for both platforms, and only after the tests pass. [`.github/workflows/release-trivy.yaml`](../.github/workflows/release-trivy.yaml) is a minimal example; [`kyverno-validate.yaml`](https://github.com/blakepettersson/curated-custom-promotion-tasks/blob/main/.github/workflows/kyverno-validate.yaml) adds lint, tests, signing and SBOMs. Image names on `ghcr.io` must be lowercase.
4. **Make the package public** (or add `imagePullSecrets` to the step), then **pin the released tag** in the manifest.

## 8. Register and document it

1. Add the manifest under [`kargo/custom-steps/<name>/`](../kargo/custom-steps) and commit. The Argo CD Application that syncs `kargo/` (see [Setup](setup.md)) registers it on the Kargo instance. Without Argo CD, use `kargo apply -f kargo/custom-steps/<name>/<name>.yaml`.
2. If you copied it from another repository, keep a header comment with the source URL and commit, so it can be updated later.
3. Add a row to the [Step catalog](../README.md#step-catalog), and an example under [Examples](../README.md#examples) showing a realistic `config`, any Secrets it needs, and where it belongs in a pipeline.
