# Block 2 — Bundle build

Epic: RHOAIENG-93185. Hierarchy: Operator (Block 1) → **Bundle (Block 2)** → FBC (Block 3).

Block 1 is implemented and validated green. This document covers Block 2: what was
built, why it diverges from the upstream ODH implementation where it does, and where
each design-review requirement is discharged.

---

## 1. Shape of the block

Block 2 is inverted the same way Block 1 is. Upstream ODH runs the whole of Block 2
inside the orchestration pipeline — clone, process, prefetch, buildah, apply-tags —
chained through trusted artifacts, and never touches git. That works upstream because
ODH-Build-Config already has a branch the gate can build from.

RHOAI-Build-Config has no stable EG branch (DR-1). So the branch has to be created,
and once it exists the PaC build is the natural thing to trigger from it — which is
also what the deliverable path (`.tekton/` in RHOAI-Build-Config) implies, and what
Block 1 already does.

The prescribed nine tasks therefore split across two files:

| # | Prescribed task | Where it lives |
|---|---|---|
| 1 | `clone-build-config-repo` | `early-gate/tasks/clone-build-config-repo.yaml` |
| 2 | `create-ephemeral-branch` | `early-gate/tasks/create-ephemeral-branch.yaml` |
| 3 | `update-group-snapshot` | `early-gate/tasks/update-group-snapshot.yaml` |
| 4 | `bundle-processor` | `early-gate/tasks/bundle-processor.yaml` |
| 5 | `verify-bundle-images` | `early-gate/tasks/verify-bundle-images.yaml` |
| — | *(push — new, see DR-1d)* | `early-gate/tasks/push-bundle-to-ephemeral-branch.yaml` |
| 6 | `prefetch-dependencies-bundle` | PaC run — `prefetch-input` param |
| 7 | `build-bundle-container` | PaC run — `container-build.yaml` |
| 8 | `apply-tags-bundle` | PaC run — `additional-tags` param |
| — | *(monitor — new, see DR-7e)* | `early-gate/tasks/monitor-pac-build.yaml` |
| 9 | `cleanup-ephemeral-branch` | `early-gate/tasks/cleanup-ephemeral-branch.yaml` (in `finally`) |

Tasks 6–8 are not written by hand: `pipelines/container-build.yaml` already contains
`prefetch-dependencies-oci-ta`, `buildah-oci-ta` and `apply-tags`, and the release
bundle build uses exactly those. Re-implementing them in the orchestration pipeline
would mean the gate exercised a different build than the release does, which defeats
the point of the gate.

Two tasks exist that were not in the prescribed list, both forced by the inversion:
the push that triggers PaC, and the monitor that reads the resulting digest back.

### Data flow

```
Block 1 ──OPERATOR_IMAGE_REF_BY_DIGEST──┐
                                        ▼
clone-build-config ──release-sha──► fork-bundle-branch ──► update-group-snapshot
   │  SOURCE_ARTIFACT                    (eg-bundle-<uid>)      │  bundle-patch.yaml
   └────────────────────────────────────────────────────────────┤  config/snapshot.json
generate-snapshot ──SNAPSHOT_ARTIFACT───────────────────────────┘
                                                                ▼
                                                        bundle-processor
                                                        (-v = eg-<sha>,
                                                         --use-existing-digests)
                                                                ▼
                                                        verify-bundle-images  ← FAILS HERE
                                                                ▼                if release-tagged
                                                   push-bundle-to-ephemeral-branch
                                                                ▼ (PaC fires)
                                                        monitor-pac-build
                                                                ▼
                            BUNDLE_IMAGE_DIGEST / BUNDLE_IMAGE_REF_BY_DIGEST ──► Block 3
```

---

## 2. Design-review traceability

Every DR, where it is addressed, and what actually happens.

### DR-1 — Ephemeral branch strategy (Critical)

| Sub | Requirement | Where | Status |
|---|---|---|---|
| a | Identify latest Y-stream release branch | `clone-build-config-repo.yaml`, `resolve-and-clone` step | ✅ |
| b | Fork ephemeral branch `eg-bundle-<run-id>` | pipeline task `fork-bundle-branch` → `create-ephemeral-branch.yaml` | ✅ |
| c | Run ALL processing on it, never the release branch | `update-group-snapshot`, `bundle-processor`, `verify-bundle-images` all run after the fork; the only git write is the push | ✅ |
| d | Push processed files → PaC triggers the build | `push-bundle-to-ephemeral-branch.yaml` + `odh-operator-bundle-v3-6-eg-push.yaml` CEL | ✅ |
| e | Delete the branch after success | `finally: cleanup-bundle-ephemeral-branch` | ✅ |
| f | Defer cleanup if Block 3 needs it; document the dependency | `defer-cleanup` param; pipeline param `defer-bundle-branch-cleanup`; §4 below | ✅ |

**a.** The same `git ls-remote | sed | grep -E '^[23]\.[0-9]+$' | sort -t. -k1,1n -k2,2n | tail -1`
idiom Block 1's `operator-processor.yaml` uses, deliberately, so the two blocks can
never disagree about which release is current. The regex is strict: it admits
`rhoai-3.6` and rejects `rhoai-3.6-ea.1`, `rhoai-3.5.1` and
`rhoai-3.6-ea.1-dry-run-nightly-bundle`. `build-config-branch-override` pins it for
reproducing a run.

**b.** Forked from the clone's exact SHA rather than from the branch name, so the
ephemeral branch and the clone are guaranteed to be the same tree even if `rhoai-3.6`
moves mid-run. 422 (ref exists) force-resets to the fork point, so a re-run is
idempotent.

**The anti-pattern being avoided.** `RHOAI-Build-Config/.github/workflows/process-operator-bundle.yaml:192`:

```bash
git push ${FORCE_FLAG} origin HEAD:${PUSH_BRANCH}   # PUSH_BRANCH = rhoai-X.Y
```

That workflow processes the bundle and pushes the result straight onto the release
branch. Copying it would put unmerged-PR image digests into `rhoai-3.6`. Three
independent guards make that impossible here:

1. `create-ephemeral-branch` refuses any branch name not matching `eg-*`.
2. `push-bundle-to-ephemeral-branch` repeats the check (it is the task holding a write
   token) and pushes only to the branch it cloned — there is no branch name in the
   push that was not in the fetch. There is no `--force`.
3. `cleanup-ephemeral-branch` repeats it again before calling `DELETE` on a ref.

### DR-2 — Group snapshot update (Missing task — must add)

| Sub | Requirement | Where | Status |
|---|---|---|---|
| a | Explicit task BEFORE bundle-processor | pipeline task `update-group-snapshot`, `runAfter: fork-bundle-branch`, and `bundle-processor` `runAfter: update-group-snapshot` | ✅ |
| b | Decide SNAPSHOT_ARTIFACT vs. separate task | **Both** — see below | ✅ |
| c | Operator image in the group snapshot before processing begins | `inject-operator-digest` step | ✅ |

**b — the decision, and a correction to the premise.** The review framed this as
"either reuse `SNAPSHOT_ARTIFACT` or add a task". It is both, because the two files
are not interchangeable:

| File | Read by `bundle-patch`? | Role |
|---|---|---|
| `bundle/bundle-patch.yaml` | **Yes** | Load-bearing. `fetch_operator_metadata` filters `relatedImages` for the `ODH_OPERATOR` entry. **This is the only channel by which the Block 1 digest reaches the CSV.** |
| `config/snapshot.json` | **No** | Parity, audit trail, and Block 3 input. |

`bundle-processor.py` does have `-sn/--snapshot-json-path`, but it feeds a separate
`snapshot_processor` class; the `bundle-patch` constructor (line 888) never receives
it. Upstream's own task has the same gap — it declares `SNAPSHOT_JSON_PATH` at
`bundle-processor.yaml:168` and then never passes it to the processor.

So `update-group-snapshot` writes both: `bundle-patch.yaml` because it is what works,
and `snapshot.json` because Block 3 and the audit trail want the group pinned
somewhere readable. `SNAPSHOT_ARTIFACT` from `generate-snapshot` seeds the latter.

**The task is separate rather than inline** (upstream does this inline inside the
processing step) because DR-2a asks for it and because it is the better shape: the
injection is now its own green/red step, and `verify-bundle-images` has an
`operator-image` result to compare the CSV against.

It refuses an operator image that is not `@sha256:`-pinned. `monitor-pac-build` warns
rather than fails when it cannot resolve a digest, so an empty or tag-shaped value
can reach here, and a bundle that floats on a mutable tag is not a gate result.

**Why pinning the operator is sufficient for the operands.** `bundle-processor.py:373`
takes the operand images from `build/operands-map.yaml`, fetched **from the operator
repo at the operator image's `git.commit`** — not from a tag lookup. Block 1's
`operator-processor` has already pinned the gated components' EG digests in that file
on the operator's ephemeral branch. Pinning the operator to the EG image therefore
propagates the EG operand digests into the CSV automatically.

### DR-3 — QUAY_TAG must be configurable (code change required)

| Sub | Requirement | Where | Status |
|---|---|---|---|
| a | Pipeline-level param | `bundle-processor.yaml` param `QUAY_TAG`, required, no default | ✅ |
| b | Wire from Block 1's output tag | pipeline: `value: eg-$(tasks.monitor-operator-processor.results.processor-commit)` | ✅ |
| c | Use THIS tag for Quay lookups | wired to `-v/--rhoai-version` | ✅ (and superseded — see below) |
| d | Must not default to the release branch name | no default + active rejection of release-shaped values | ✅ |

**The code change turned out not to be needed, for a reason worth recording.**
Upstream's ODH processor has `-q/--quay-tag`. RHOAI's does not — the argparse block
(`bundle-processor.py:850–870`) has no `-q` and no `-r`. The tag RHOAI's processor
looks images up by is `-v/--rhoai-version`, at line 266:

```python
version_tag = f'{self.rhoai_version}-nightly' if self.build_type.lower() == 'nightly' else self.rhoai_version
```

and the release workflow passes `-v rhoai-${RHOAI_VERSION}` — i.e. **`-v` is literally
the release tag**. That is the knob DR-3 is about. `QUAY_TAG` is wired to it.

Passing a non-version-shaped value is safe: `self.rhoai_version` is read in exactly
one place other than a log line (line 266). It does not reach the CSV version, the
bundle name, or any annotation.

**The stronger guarantee.** Wiring the tag still leaves a Quay-lookup code path that a
mis-set param could point at the release. `--use-existing-digests` removes the path
entirely: at line 257 the processor takes the operator digest straight out of
`bundle-patch.yaml` — which `update-group-snapshot` has already set to Block 1's image
— and never calls `fetch_latest_images_and_git_metadata`. Both are applied. The
`QUAY_TAG` guard is the belt; `--use-existing-digests` is the braces.

**d** is enforced, not just documented:

```bash
case "${QUAY_TAG}" in
  rhoai-*|odh-stable|latest|*-nightly)
    echo "QUAY_TAG='${QUAY_TAG}' looks like a release tag." >&2 ; exit 1 ;;
esac
```

### DR-4 — BRANCH must be configurable (code change required)

| Sub | Requirement | Where | Status |
|---|---|---|---|
| a | Pipeline param wired to the ephemeral branch | every path flag in `bundle-processor.yaml` points into the ephemeral-branch clone | ✅ |
| b | Check `bundle-processor.py` for hardcoded branch validation | **Checked — there is none.** See below | ✅ |
| c | Reusable for the FBC processor | `create-ephemeral-branch.yaml` is generic | ✅ |

**b — the finding.** Upstream's ODH processor has `-r/--branch`. RHOAI's has no `-r`
and no branch concept at all: `process-operator-bundle.yaml` checks the branch out
into a directory named after it and encodes the branch purely by where the path flags
point. There is nothing in the Python to make configurable.

The `rhoai-X.Y` pattern matching DR-4b asks about does exist — but in the **GitHub
workflow**, not the Python:

- `process-operator-bundle.yaml:24–28` — `on.push.branches` filter
- `process-operator-bundle.yaml:53` — `if ! [[ "$VERSION_SOURCE" =~ ^([0-9]+\.[0-9]+(-ea\.[0-9]+)?)$ ]]`

Driving the processor from Tekton bypasses both. **Arbitrary `eg-*` branch names work
with no change to `bundle-processor.py`.**

### DR-5 — Bundle processor execution details

Upstream's six steps, and what each became:

| Upstream step | Here | Note |
|---|---|---|
| `use-trusted-artifact` | same | unchanged |
| `clone-utils` | same | `UTILS_REPO_BRANCH` defaults to `main`, not upstream's `odh` — the `odh` branch has the processor under `utils/bundle-processor/` with a different CLI; RHOAI's is `utils/processors/` |
| install yq + `pip install -r requirements.txt` | split | yq moved into `update-group-snapshot` (that is where the `yq -i` now lives); `pip install -r ${UTILS}/utils/processors/requirements.txt` stays |
| seed snapshot.json + jq `.["odh-operator-ci"].image` | → `update-group-snapshot` | DR-2a |
| `yq -i '(.patch.relatedImages[] \| select(.name == "RELATED_IMAGE_ODH_OPERATOR_IMAGE")).value = env(OPERATOR_IMAGE)'` | → `update-group-snapshot`, **verbatim** | DR-2a |
| run the processor | `process-bundle` step | flags below |
| `cp -r ${RAW_INPUTS_DIR}/* ${SOURCE}/bundle/` | + `find -mindepth 1 -maxdepth 1 -type d -exec rm -rf` first | follows RHOAI's workflow, not upstream's, so a manifest the processor dropped does not survive as a stale file |
| `print-logs` | same, `onError: continue` | |
| `create-trusted-artifact` | same | |

**The invocation**, and why each flag differs from upstream:

```bash
python3 ${UTILS}/utils/processors/bundle-processor.py \
  -op bundle-patch \
  -b "${BUILD_CONFIG_PATH}" \
  -c "${BUNDLE_CSV_PATH}" \
  -p "${PATCH_YAML_PATH}" \
  -o "${OUTPUT_FILE_PATH}" \
  -v "${QUAY_TAG}" \
  -a "${ANNOTATION_YAML_PATH}" \
  "${METADATA_CONFIG_FLAG[@]}" \
  "${DIGEST_FLAG[@]}"
```

| Upstream (odh) | Here (main) | Why |
|---|---|---|
| `utils/bundle-processor/` | `utils/processors/` | the forks diverged |
| `-q ${OPERATOR_TAG}` | `-v ${QUAY_TAG}` | RHOAI has no `-q`; `-v` is the lookup tag (DR-3) |
| `-r ${BRANCH}` | *(omitted)* | RHOAI has no `-r` (DR-4) |
| — | `-mc` (conditional) | SBOM metadata config, matching the release workflow |
| — | `--use-existing-digests` | DR-3, removes the Quay-lookup path |
| `--push-pipeline-yaml-path`, `--push-pipeline-operation enable` | *(omitted)* | RHOAI's equivalents are `-y`/`-x` |
| — | *(no helm flags)* | Block 2 builds the operator bundle; the charts are not part of the gate |

**Why `-y`/`-x` are omitted.** Enabling the release push pipeline is release
management, and an EG run is not a release — the same call was dropped from Block 1's
operator-processor for the same reason. It would also be inert: the file it edits
(`.tekton/odh-operator-bundle-v3-6-push.yaml`) gates on `target_branch == "rhoai-3.6"`,
which an `eg-bundle-*` branch never satisfies. And `process_push_pipeline` works by
toggling a `"non-existent-file.non-existent-ext".pathChanged() &&` prefix on the CEL,
which is not a mutation worth making on a throwaway branch.

**One genuine defect found, mitigated here, needs a proper fix upstream.** See §5.

### DR-6 — Image verification (test requirement)

| Sub | Requirement | Where | Status |
|---|---|---|---|
| a | Verification task AFTER bundle-processor | pipeline task `verify-bundle-images`, `runAfter: bundle-processor` | ✅ |
| b | Inspect CSV, `bundle_build_args.map`, relatedImages | all three scanned | ✅ |
| c | Assert every image is EG-tagged or EG-digest | checks 1, 3, 4 — see the refinement below | ⚠️ refined |
| d | Assert NO image references the release tag | checks 1 + 2 | ✅ |
| e | Fail the pipeline listing offenders | `exit 1` with every offender printed to stderr | ✅ |

It runs **before** the push, so a bundle that would embed a release image never
reaches git and never triggers a build.

The four checks:

1. **Every image reference is pinned by digest.** A floating tag is the signature of a
   tag lookup having leaked in, so it fails regardless of what the tag says.
2. **No release tag** — `rhoai-X.Y`, `odh-stable`, `*-nightly`, `latest`, `stable`,
   plus the specific branch this run forked from.
3. **The operator relatedImage is exactly Block 1's image** — same digest, not merely
   the same repository. Matched on the bare digest as well as the full reference,
   because `build-config.yaml`'s registry/repo replacement rewrites
   `quay.io/rhoai/...` to `registry.redhat.io/rhoai/...`.
4. **Every gated component appears at the digest the snapshot pinned.** This is the
   check that catches the interesting failure: a component whose PR was gated but
   whose *release* digest ended up in the CSV anyway. Joined on repository basename,
   which survives the registry/repo replacement.

The CSV and `bundle_build_args.map` are enforced. `bundle-patch.yaml` is scanned but
advisory — it is an input template, and anything wrong in it necessarily reappears in
the CSV, where it does fail.

**c — the refinement, and why it is not the literal requirement.** "ALL images are
EG-built" would fail every realistic run. Only the components listed in the leader PR
are gated; the rest of the bundle is *supposed* to be current release content, pinned
by release digests. A check demanding otherwise would be red on every run and
therefore ignored within a week.

What is enforced instead is the precise property the smoke test actually wants: no
image is release-**tagged** or unpinned (1, 2), and every image that *should* have
been replaced *was* (3, 4). Ungated components on release digests are counted and
reported, not failed — `release-image-count` and `eg-image-count` are task results, so
the split is visible on the run.

`STRICT_MODE` (pipeline param `bundle-verify-strict`) additionally fails when a gated
component is pinned in the snapshot but absent from the bundle. Off by default because
not every gated component appears in the operator bundle; **this is the switch for
Deepak's smoke test** against a group known to be bundle-visible.

> **Coordination still required.** DR-6 notes this is "critical for the smoke test —
> Deepak requires coordination with Noam and RT". Nothing in this implementation
> discharges that; it provides the mechanism. The smoke test needs a leader PR whose
> gated components are bundle-visible, run with `bundle-verify-strict: "true"`.

### DR-7 — FBC processor awareness (design only)

| Sub | Requirement | Status |
|---|---|---|
| a | Block 3 also needs an ephemeral branch | ✅ `create-ephemeral-branch.yaml` is generic — `repo`, `source-sha`, `branch-name` params, nothing in it knows what a bundle is. Block 3 reuses it with `eg-fbc-<uid>`. `clone-build-config-repo.yaml`, `push-bundle-to-ephemeral-branch.yaml` (via its `paths` param), `monitor-pac-build.yaml` and `cleanup-ephemeral-branch.yaml` are equally reusable. |
| b | Block 3 needs a configurable bundle tag | ✅ analysed — `utils/fbc-processor/fbc-processor.py` has the same CLI shape as the bundle processor: `-v/--rhoai-version`, `--use-existing-digests`, `-sn`, and additionally `-s/--single-bundle-path` and `-cba/--catalog-build-args-file-path`. The DR-3 answer transfers directly: wire the bundle tag to `-v`, pass `--use-existing-digests`, and inject the bundle digest into `catalog/catalog-patch.yaml` the way `update-group-snapshot` injects the operator digest into `bundle-patch.yaml`. |
| c | Block 2 must not conflict with Block 3's requirements | ✅ nothing in Block 2 writes to `catalog/`; the push task's `paths` defaults to `bundle config`. |
| d | Cleanup deferral | ✅ `defer-bundle-branch-cleanup` — see §4. |
| e | `BUNDLE_IMAGE_DIGEST` and `BUNDLE_IMAGE_REF_BY_DIGEST` as **explicit pipeline results**, not internal state | ✅ both declared in `spec.results`, alongside `BUNDLE_IMAGE_URL`, `BUNDLE_COMMIT`, `EG_BUNDLE_BRANCH`, `BUNDLE_RELEASE_BRANCH`, `BUNDLE_BUILD_STATUS`, `BUNDLE_VERIFIED`. |

---

## 3. Reused vs. adapted vs. new

**Reused unchanged from the release path** — these are the whole point; the gate must
exercise what the release exercises:

- `pipelines/container-build.yaml` and therefore `prefetch-dependencies-oci-ta`,
  `buildah-oci-ta`, `apply-tags` (prescribed tasks 6, 7, 8)
- `utils/processors/bundle-processor.py` — invoked, not modified
- `bundle/Dockerfile`, `bundle/bundle-patch.yaml`, `to-be-processed/bundle/`

**Adapted from upstream ODH:**

- `bundle-processor.yaml` — upstream's structure, RHOAI's CLI (see DR-5)
- the `yq -i` relatedImages patch — verbatim, relocated to `update-group-snapshot`
- the snapshot seeding/`jq` logic — relocated likewise
- `early-gate-component-pipeline.yaml` Block 2 section — upstream's ordering, inverted
  at the build boundary

**New, with no upstream counterpart:**

- `clone-build-config-repo.yaml` — upstream uses stock `git-clone-oci-ta` because it
  is handed a fixed revision; DR-1a requires run-time discovery, and
  `git-clone-oci-ta` has nowhere to put that result
- `create-ephemeral-branch.yaml`, `push-bundle-to-ephemeral-branch.yaml`,
  `cleanup-ephemeral-branch.yaml` — upstream never creates a branch or pushes
- `verify-bundle-images.yaml` — DR-6 has no upstream counterpart at all
- `monitor-pac-build.yaml` — a consequence of the inversion
- `pipelineruns/RHOAI-Build-Config/.tekton/odh-operator-bundle-v3-6-eg-push.yaml`

---

## 4. Assumptions, deviations and gaps

### Deployment prerequisites — Block 2 cannot run until these exist

1. **Konflux Component `odh-operator-bundle-gap`** in application `gap-poc`,
   namespace `rhoai-tenant`, with service account
   `build-pipeline-odh-operator-bundle-gap` and push access to
   `quay.io/redhat-user-workloads/rhoai-tenant/odh-operator-bundle-gap`. Mirrors
   Block 1's `rhods-operator-gap`. **Not yet created.**

2. **`odh-operator-bundle-v3-6-eg-push.yaml` must be on `rhoai-3.6` in
   RHOAI-Build-Config.** PaC reads `.tekton/` from the branch the push landed on, and
   `eg-bundle-*` is forked from `rhoai-3.6` — so the file has to be there to be
   inherited. It is authored here under `pipelineruns/RHOAI-Build-Config/.tekton/`,
   which is the sync source; landing it means cherry-picking to konflux-central's
   `rhoai-3.6` branch, since the sync is branch-to-branch.

   It is inert on `rhoai-3.6` itself: its CEL requires
   `target_branch.startsWith("eg-bundle-")`.

   *Alternative if adding a file to `rhoai-3.6` is not acceptable for the smoke test:*
   have the push task write `.tekton/` onto the ephemeral branch too (add `.tekton` to
   its `paths` param and fetch the file from konflux-central first). Not implemented —
   it makes the gate write a file it did not process, which is worse than a reviewed
   one-line addition to a release branch.

3. **Secret `early-gate-secrets`** with `RHOAI_QUAY_API_TOKEN` and
   `REDHAT_USER_WORKLOADS_QUAY_API_TOKEN`. Mounted `optional: true`, so the task fails
   with the processor's own diagnostics rather than at pod start. With
   `--use-existing-digests` the operator lookup is skipped, so these may not be needed
   at all — to be confirmed on the first run.

4. **`eg-github-token` needs write access to RHOAI-Build-Config.** Block 1 only needed
   write on `rhods-operator-gap`. Block 2 creates, pushes to and deletes branches on a
   shared release repository. If branch protection on RHOAI-Build-Config restricts ref
   creation, `eg-bundle-*` must be exempted.

### Deviations from the brief

- **Filename.** The brief asks for
  `pipelineruns/RHOAI-Build-Config/.tekton/odh-operator-bundle-v3-6-push.yaml`. That is
  the *existing release* PipelineRun's name — using it would overwrite the release
  build on the next sync. Written as `odh-operator-bundle-v3-6-**eg**-push.yaml`.

- **`build-source-image: true`** is kept, matching the release file, even though a
  throwaway gate build does not need a source container. The gate is more useful if it
  exercises the same path the release does.

- **`prefetch-input: ""`.** The brief asks for "hermetic build (cachi2 prefetch)".
  `bundle/Dockerfile` is `FROM scratch` and does nothing but `COPY` manifests and
  consume build args — there is no package manager to feed. The prefetch task runs and
  resolves nothing, which is what the release bundle build does too. `hermetic: true`
  is what carries the actual meaning: it proves the bundle needs no network.

- **The release file's
  `files.all.exists(p, !p.matches('^bundle/bundle-patch\.yaml$'))` CEL clause is
  dropped.** On an empty commit `files.all` is empty and `exists()` over an empty list
  is false, so keeping it would veto exactly the no-op case the `event_title` clause
  exists to catch.

### Known gaps and follow-ups

1. **`bundle_build_args.map` key derivation is broken for EG operator images
   (mitigated, needs a real fix).**

   `parse_image_value` (`utils/util.py:89`) treats everything after the org as the
   repo, and `generate_bundle_build_args` upper-cases it. For the release image
   `quay.io/rhoai/odh-rhel9-operator` that yields `ODH_RHEL9_OPERATOR_GIT_URL` — the
   ARG `bundle/Dockerfile` declares. The EG operator image lives at
   `quay.io/redhat-user-workloads/rhoai-tenant/<component>`, a **three-segment** path,
   so the derived repo is `rhoai-tenant/<component>` and the key comes out as
   `RHOAI_TENANT/<COMPONENT>_GIT_URL` — not a valid buildah build-arg name, and
   matching no ARG in the Dockerfile.

   *Mitigation here:* `update-group-snapshot` captures the component name from the
   **pre-patch** (release) image path and `bundle-processor` rewrites the key with
   `sed` after the processor runs.

   *Proper fix:* the processor should derive the component name from the
   `relatedImages` entry name — `RELATED_IMAGE_ODH_OPERATOR_IMAGE` already encodes it —
   or accept an override flag. **Needs a PR to RHOAI-Konflux-Automation.**

2. **`monitor-pac-build` duplicates Block 1's inline `monitor-operator-build`.** Block 1
   was left untouched deliberately — it is validated green and Block 2 is not. Folding
   Block 1 onto the shared task is a follow-up.

3. **No `apply-tags` beyond `additional-tags`.** Prescribed task 8 is satisfied by
   `container-build.yaml`'s `apply-tags` acting on `additional-tags`. If the gate later
   needs semantic tags (`eg-latest`, say), that is a change to the PaC file.

4. **`verify-bundle-images` joins snapshot to CSV on repository basename.** Correct for
   every mapping currently in `build-config.yaml` (they are identity on the repo name),
   but a mapping that renamed a repository across registries would silently fail to
   match and be counted as ungated. A rename would need the join key revisited.

5. **Block 2 is unconditional.** No `enable-bundle-block` flag: the removal of
   `enable-group-snapshot` from this pipeline is on record as the right call, because a
   flag guarding a block that *cannot complete* hides that fact behind a green run. If
   the Component in §4.1 is not onboarded, Block 2 fails and the gate is red — which is
   accurate, because the gate genuinely cannot produce a bundle.

---

## 5. What Block 3 inherits

Reusable as-is: `create-ephemeral-branch`, `clone-build-config-repo`,
`push-bundle-to-ephemeral-branch` (set `paths: "catalog"`), `monitor-pac-build`,
`cleanup-ephemeral-branch`.

Consumed from Block 2: `BUNDLE_IMAGE_REF_BY_DIGEST` (preferred),
`BUNDLE_IMAGE_DIGEST`, `BUNDLE_IMAGE_URL`, `BUNDLE_RELEASE_BRANCH`,
`EG_BUNDLE_BRANCH`.

**The one decision Block 3 has to make**, which determines whether
`defer-bundle-branch-cleanup` needs setting:

- **Fork from the release branch** and take the bundle digest from
  `BUNDLE_IMAGE_REF_BY_DIGEST`. Block 2's branch dies as soon as its build is green;
  cleanup runs normally. **This is the shape Block 2 is designed for** — each block's
  branch lifetime stays inside that block.
- **Fork from Block 2's branch**, to inherit the processed `bundle/`. Then deleting
  Block 2's branch before Block 3 has forked breaks Block 3, and the orchestrating
  pipeline must set `defer-bundle-branch-cleanup: "true"` and own both deletions.

Block 3 needs nothing from `bundle/` that is not in the bundle image, so the first
shape should win — but the switch exists either way.
