# Early Gate — pipeline overview

One run, three blocks: **Operator → Bundle → FBC fragment**. Each block builds an image
and hands its **digest** to the next, so what the gate finally produces is a catalog
assembled entirely from unmerged PRs.

Nothing is ever written to a release branch. Every write goes to an ephemeral `eg-*`
branch, and `finally` deletes all three.

Pipeline: [`early-gate/early-gate-component-pipeline.yaml`](../early-gate-component-pipeline.yaml).
Block 2's design rationale: [`block2-bundle-design.md`](block2-bundle-design.md).

---

## 1. The image chain

```
Block 1  operator image ──┐
                          ├─► Block 2  bundle image ──┐
     (pinned into the     │        (pinned into       ├─► Block 3  FBC fragment
      bundle's CSV)     ──┘         the catalog)    ──┘
```

Every hand-off is by digest — `repo@sha256:…`, never a tag. A tag is mutable: rebuild
the same commit and it moves, so a gate that chained on tags could pass on one image
and ship another. Two consequences fall out of that:

- `update-group-snapshot` **refuses** a Block 1 reference that is not digest-pinned,
  rather than building a bundle that floats on a tag.
- each block's verify task independently re-checks that the previous block's digest
  actually arrived in the artifact — see §3.

| Result | Produced by | Consumed by |
|---|---|---|
| `OPERATOR_IMAGE_REF_BY_DIGEST` | `monitor-operator-build` | `update-group-snapshot` → the bundle CSV |
| `BUNDLE_IMAGE_REF_BY_DIGEST` | `monitor-bundle-build` | `fbc-processor` → the catalog's `olm.bundle` |
| `FBC_IMAGE_REF_BY_DIGEST` | `monitor-fbc-build` | the gate's final answer |

---

## 2. Tasks

### Block 0 — decide what is being tested

| Task | What it does |
|---|---|
| `resolve-group-configuration` | Reads the leader PR, collects its child component PRs, decides which release branch to fork from |
| `generate-snapshot` | Resolves every component's current image digest from Quay into one Snapshot |
| `audit-snapshot` | Sanity-checks it and records which components were omitted |

### Block 1 — Operator

| Task | What it does |
|---|---|
| `fork-operator-branch` | Creates the ephemeral `eg-<uid>` branch off main/stable |
| `trigger-operator-processor-on-ephemeral-branch` | Dispatches the operator-processor at that branch |
| `monitor-operator-processor` | Waits while it pins the component digests into the operator's config |
| `monitor-operator-build` | Watches the build the pinned commit triggers |

**Out:** `OPERATOR_IMAGE_DIGEST`.

### Block 2 — Bundle

| Task | What it does |
|---|---|
| `clone-build-config` | Clones RHOAI-Build-Config into a trusted artifact |
| `fork-bundle-branch` | Creates `eg-bundle-<uid>` |
| `update-group-snapshot` | Injects Block 1's operator digest; refuses a tag-only reference |
| `bundle-processor` | Regenerates the bundle CSV with every image pinned by digest |
| `verify-bundle-images` | Independent check — all digest-pinned, no release tags, operator digest present |
| `push-bundle-to-ephemeral-branch` | Commits `bundle/` to the ephemeral branch |
| `monitor-bundle-build` | Watches the bundle build |

**Out:** `BUNDLE_IMAGE_DIGEST`.

### Block 3 — FBC fragment

| Task | What it does |
|---|---|
| `fork-fbc-branch` | Creates `eg-fbc-<uid>` |
| `fbc-processor` | Points the semver template at Block 2's bundle, runs `opm alpha render-template semver`, regenerates `catalog/v4.19` |
| `verify-fbc-images` | Checks the new `olm.bundle` exists, points at Block 2's exact digest, carries Block 1's digest, is reachable from a channel, and has no release-tagged references |
| `push-fbc-to-ephemeral-branch` | Commits `catalog/` |
| `monitor-fbc-build` | Watches the FBC fragment build |

**Out:** `FBC_IMAGE_DIGEST`, `FBC_IMAGE_REF_BY_DIGEST`.

### `finally`

`cleanup-ephemeral-branch`, `cleanup-bundle-ephemeral-branch`, `cleanup-fbc-ephemeral-branch`
— so the branches go away even when the run fails.

---

## 3. Why each block verifies the one before it

The verify tasks are deliberately **independent oracles**: they re-derive the answer from
the committed artifact rather than trusting the processor that wrote it. The case that
motivates this is `fbc_processor.patch_olm_bundles`, which only inserts when the bundle
name is not already present and **writes nothing at all otherwise** — a silent no-op that
leaves a perfectly valid, entirely unmodified release catalog behind. Without check 1 in
`verify-fbc-images` the gate would build that catalog and pass.

---

## 4. How the builds are triggered (PaC)

The gate does not run `buildah` itself. It pushes to an ephemeral branch, and
Pipelines-as-Code starts the real build from the `.tekton/` file in that repo. The gate
then follows the build over the GitHub check-runs API, so it needs no read access to
Tekton in whatever cluster the build landed in.

| Block | PaC file | Fires on |
|---|---|---|
| 1 | `rhods-operator/.tekton/odh-operator-eg-push.yaml` | push to `eg-*` |
| 2 | `RHOAI-Build-Config/.tekton/odh-operator-bundle-eg-push.yaml` | push to `eg-bundle-*` |
| 3 | `RHOAI-Build-Config/.tekton/rhoai-fbc-fragment-eg-push.yaml` | push to `eg-fbc-*` |

> **TODO(prod):** none of these three files, nor the Konflux Components behind them,
> exist in `red-hat-data-services` yet. The names above are what the pipeline params
> currently expect; confirm them when the Components are created. Each must be a
> *dedicated early-gate* Component pushing to `quay.io/rhoai/pull-request-pipelines` —
> never the release Component, which pushes to a release repository.

Each EG arm matches on **branch prefix plus an `early-gate:` commit subject** — Block 1
also accepts a changed `build/operands-map.yaml`, since that is what its processor writes:

```
event == "push"
&& target_branch.startsWith("eg-fbc-")
&& event_title.startsWith("early-gate:")
```

The subject is load-bearing and `pathChanged()` cannot replace it on its own:

- creating the branch is itself a push event, and a fresh fork inherits the base branch's
  tip — an ordinary merge commit — so the subject is what separates "the gate pushed a
  processed artifact" from "the gate created a branch";
- the processed push may legitimately change nothing, and the push tasks commit with
  `--allow-empty` for exactly that reason, so there is not always a diff to match on.

The push tasks refuse any message not starting with `early-gate:`, so the guarantee holds
from both ends.

The prefixes are also kept disjoint on purpose. Block 3 could have reused Block 2's
branch, but the bundle's CEL fires on `eg-bundle-*` with an `early-gate:` subject — which
is precisely the shape of Block 3's catalog push. Sharing a branch would mean every FBC
push also rebuilt the bundle.

**Two things to know when reading a red run:**

- A CEL **parse** error is not scoped to the clause that caused it. The whole annotation
  becomes unevaluable, every arm dies with it, and PaC skips the PipelineRun **without
  registering a check-run** — which from the gate looks exactly like a build that never
  started. Regex escapes inside CEL string literals need `\\.`, not `\.`.
- The git resolver resolves each `taskRef` when that task **starts**, not when the
  pipeline starts, so a merge to `konflux-central` main can land mid-run.

---

## 5. Credentials

Consumed only as Kubernetes secret workspaces or `secretKeyRef` — never echoed, never on
a command line, never in a remote URL.

| Secret | Used for |
|---|---|
| `eg-github-token` | Branch operations and the workflow dispatch (needs `actions:write`, which the PaC token does not carry) |
| `rhoai-quay-gap-ro` | Reading digests from the private `quay.io/rhoai` — TODO(prod): POC secret name; confirm or rename when the production secret is created |

Registry auth is written to a file by a stdlib-only Python helper under `umask 077`,
rather than `skopeo login -p` / `cosign login -p`, so no secret ever reaches the process
list or a `set -x` trace. The helper prints registry names only.

---

## 6. Temporary configuration — revert before real use

The current config is tuned for iteration speed, not for gating.

| Setting | Current | Cost |
|---|---|---|
| `skip-checks: "true"` on the FBC build | drops `validate-fbc` | **highest.** `validate-fbc` is what catches a structurally invalid catalog. `verify-fbc-images` checks image *references*, not `opm` graph validity — nothing else covers this |
| `skip-checks` on operator/bundle | drops sast-snyk and the other scanners | no security scanning |
| operator build | x86_64 only | three architectures unproven |
| `build-source-image: false` | all three components | no source images |
| `defer-fbc-branch-cleanup: "true"` | keeps `eg-fbc-*` after the run | branches must be deleted by hand |

Also open: the fixed-name artifact tags (`eg-<leader-pr>.snapshot` / `.bundle` / `.fbc`)
collide if two gate runs overlap, and `rhoai-version` / `build-type` cannot be passed
because this fork's `pipelines/fbc-fragment-build.yaml` is stale against release and does
not declare them.
