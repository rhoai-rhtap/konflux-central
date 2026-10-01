# PR-specific digests in the early gate

**Status: option A (by commit) is implemented.** `generate-snapshot-for-group-testing`
now probes, in order:

1. `quay.io/<quay_path>:rhoai-pr-<N>` — the component's own repository, ODH-style
2. `quay.io/rhoai/pull-request-pipelines:<component>-<commit>` — what RHOAI actually publishes
3. `quay.io/<quay_path>:rhoai-3.5` — the release fallback

Probe 2 is the one that hits on RHOAI. It needs the head SHA of each child PR,
which `resolve-group-configuration` reads from the GitHub API while it is
already walking the child PR list and carries in each component's `sha` field.

The rest of this note is the measurement that led there, kept because it also
explains why the other options were rejected.

Before probe 2 existed, the gate had only probes 1 and 3, so every component in
every group fell through to the release tag — meaning the gate validated the
operator against *released* component images rather than the proposed ones.

## What the gate looks for

For a component resolved from `component_repo_map.json` — say
`odh-kserve-controller-v3-5` → `rhoai/odh-kserve-controller-rhel9` — carrying
PR number 18, the task inspects:

```
quay.io/rhoai/odh-kserve-controller-rhel9:rhoai-pr-18      # preferred
quay.io/rhoai/odh-kserve-controller-rhel9:rhoai-3.5        # fallback
```

That is, the PR image is expected in the component's **own** repository, under
a tag naming the PR.

## What RHOAI actually publishes

Every PR pipeline in `red-hat-data-services/konflux-central` publishes to one
**shared** repository instead:

```yaml
- name: output-image
  value: quay.io/rhoai/pull-request-pipelines:odh-kserve-controller-{{revision}}
- name: additional-tags
  value:
  - 'pr-{{pull_request_number}}-into-{{target_branch}}'
```

Measured over `konflux-central@main`:

| | count |
|---|---|
| `*-pull-request.yaml` files | 129 |
| publishing to `quay.io/rhoai/pull-request-pipelines` | **129 (all)** |
| publishing to the component's own repo | 0 |
| also tagging `pr-<N>-into-<branch>` | 71 |
| producing a `rhoai-pr-<N>` tag | **0** |

On `rhoai-3.5` there are only 8 PR pipelines, and none of them carries the
`pr-<N>-into-<branch>` tag.

So there are three independent mismatches, not one:

1. **Wrong repository.** The image is in `rhoai/pull-request-pipelines`, not
   `rhoai/<component>-rhel9`.
2. **Wrong tag.** The primary tag is `<component>-<commit-sha>`; the secondary,
   where present, is `pr-<N>-into-<branch>`. Neither is `rhoai-pr-<N>`.
3. **Not on 3.5.** Even the secondary tag is a 3.6-era addition.

This is why every component in every early-gate run so far has fallen back to
`rhoai-3.5` — including the runs that were otherwise fully green.

Note the contrast with ODH, whose `odh-pr-<N>` convention the upstream task was
written against: ODH components push PR images to their own repositories, so
the probe works there unmodified.

## Options

### A. Teach the gate the shared-repo layout

No change outside this repository. `generate-snapshot` would probe

```
quay.io/rhoai/pull-request-pipelines:pr-<N>-into-<source-branch>
```

before falling back, using the branch the gate forked from.

Cheapest, but it inherits a real ambiguity: the tag names a PR number and a
target branch, and nothing else. Two different repositories whose PR numbers
collide — PR #42 in `kserve` and PR #42 in `odh-dashboard`, both into
`rhoai-3.5` — write the same tag in the same repository, and the second wins.
The gate has no way to tell which component it just resolved. It would also
only work for the 71 of 129 components that emit the tag at all, and for none
on 3.5.

A safer variant is to probe by commit instead —
`pull-request-pipelines:<component>-<sha>` — which is unambiguous and emitted
by all 129. That needs the head SHA of each child PR, which
`resolve-group-configuration` can read from the GitHub API while it is already
walking the child PR list. This is the option worth pursuing.

### B. Have component PR pipelines tag their own repositories

Add to each `*-pull-request.yaml`:

```yaml
- name: additional-tags
  value:
  - 'rhoai-pr-{{pull_request_number}}'
```

and point `output-image` at the component's own repository.

Matches what the gate already expects and removes the ambiguity, because the
repository disambiguates the component. But it is a change to 129 files in a
repository this POC does not own, it puts unreviewed PR builds into the same
repositories that hold release images, and it needs a tag-expiry policy so PR
images do not accumulate. Not a POC-scale change.

### C. Leave the fallback as the only path

What happens today. The gate verifies that the operator builds and that the
snapshot is internally consistent, against the last released component images
rather than the proposed ones. That is a weaker guarantee than group testing is
meant to give — a component PR that would break the operator is not caught,
because its image is never the one tested — but it is honest, and it is what is
currently verified end to end.

## What was done

**A, in the by-commit form.** Contained to `generate-snapshot` and
`resolve-group-configuration`, unambiguous, and it works for every component
that has a PR pipeline. C remains the behaviour whenever probe 2 misses, which
is still the common case — most component PRs never trigger a build, because
the PR pipelines are gated behind `on-label` / `on-comment`.

B is the right long-term shape but is a konflux-central-wide change with
release-registry and retention implications that need an owner outside this
POC. It is not proposed here.

### Evidence for the tag shape

Read live from `quay.io/rhoai/pull-request-pipelines` (anonymously readable):

| | count |
|---|---|
| tags sampled | 3000 |
| matching `<name>-<40-hex>` | 474 |
| distinct `<name>` prefixes among those | 36 |
| prefixes declared by a `*-pull-request.yaml` in `konflux-central@main` | 32 |
| not declared | 4 (`main`, `main-rocm`, `odh-mod-arch-data-registry`, `odh-operator-pr-51`) |

And the derivation the probe relies on — component name in
`component_repo_map.json` minus its `-vX-Y` suffix equals the PR-pipeline tag
prefix — holds for 93 of the 110 PR pipelines against `rhoai-3.6` and 74 of 110
against `rhoai-3.5`. The shortfall is components that have a PR pipeline but no
push pipeline on that release branch, not a break in the rule.

### What still gates it in practice

A component PR only produces an image if its PR pipeline actually runs, and
most are behind `on-label` (`kfbuild-all`, `kfbuild-<component>`) or
`on-comment` (`/build-konflux <component>`). A gated group whose child PRs were
never labelled still resolves entirely by fallback. That is a property of how
RHOAI triggers PR builds, not of the gate.

One further trap, found on `red-hat-data-services/kserve@test-gap`: PR
pipelines resolve their definition from `konflux-central` at
`{{ target_branch }}`, which only exists for `rhoai-X.Y`. On any other base
branch the resolver 404s and no PipelineRun is created at all — no build and no
check-run, which reads exactly like "the pipeline never triggered".
`red-hat-data-services/kserve#4636` pins it for that branch.
