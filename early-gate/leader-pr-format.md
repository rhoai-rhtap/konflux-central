# Leader PR body format

`check-leader-pr`, the first task in [`eg-operator-build.yaml`](eg-operator-build.yaml),
reads the leader PR in `red-hat-data-services/gated-artifacts-promoter` and derives two
results from its **body text**:

| result | values | meaning |
|---|---|---|
| `source-branch` | `main` / `stable` | branch the ephemeral EG branch is forked from |
| `skip-eg-build` | `true` / `false` | `true` skips the whole EG build; the Task 1 PR build image is reused |

## There is no schema

The body is not parsed as a structured document. The task scans the entire body for
occurrences of this pattern and deduplicates them case-insensitively:

```
rhoai-rhtap/[A-Za-z0-9._-]+
```

Everything follows from that set of matches:

- **`source-branch`** is `main` if the set contains the operator repo
  (`operator-repo`, default `rhoai-rhtap/rhods-operator-gap`), otherwise `stable`.
- **`skip-eg-build`** is `true` only if the operator repo is present **and** the set has
  exactly one entry.

Because it is a plain scan, any of these forms match equally — the real GAP leader
template uses the first:

```markdown
- [kserve](https://github.com/rhoai-rhtap/kserve/pull/18) (`new`)
- rhoai-rhtap/kserve#18
- `rhoai-rhtap/kserve`
```

## Gotchas

**The whole body is scanned, not just the child-PR list.** A passing mention of
`rhoai-rhtap/konflux-central` in a description or footer is counted as a component PR.
This matters most for `skip-eg-build`, which requires a count of exactly one: a single
stray reference silently turns a skip into a full build. Refer to other repos without
the `rhoai-rhtap/` prefix in leader PR bodies.

**The org prefix is hardcoded.** Only `rhoai-rhtap/` matches. A leader PR listing
`red-hat-data-services/...` child PRs yields zero matches, which reads as
"no operator PR" — `source-branch=stable`, `skip-eg-build=false`. Today's GAP leader
template links to `rhoai-rhtap`, so this holds; it would need revisiting if the promoter
starts targeting production repos.

**`.` is inside the character class.** `rhoai-rhtap/kserve.` captures the trailing dot
as part of the name, and `rhoai-rhtap/rhods-operator-gap.git` does not equal the operator
repo, so neither matches what you meant. Avoid trailing punctuation directly against a
repo reference.

**An unreadable leader PR is not an error.** Any non-200 from the API — wrong number,
no access, placeholder value — is logged, and the task falls back to the pipeline's
`source-branch` param with `skip-eg-build=false`. The build proceeds as it did before
`check-leader-pr` existed rather than failing.

## Test fixtures

Three open PRs in `red-hat-data-services/gated-artifacts-promoter`, one per outcome.
They carry empty commits — the fixture *is* the body — and are safe to close and reopen.

| PR | branch | repos in body | `source-branch` | `skip-eg-build` |
|---|---|---|---|---|
| [#4](https://github.com/red-hat-data-services/gated-artifacts-promoter/pull/4) | `test-leader-with-operator` | dashboard, kserve, **operator** | `main` | `false` |
| [#5](https://github.com/red-hat-data-services/gated-artifacts-promoter/pull/5) | `test-leader-without-operator` | dashboard, kserve, modelmesh | `stable` | `false` |
| [#6](https://github.com/red-hat-data-services/gated-artifacts-promoter/pull/6) | `test-leader-operator-only` | **operator** only | `main` | `true` |

Point a pipeline at one by setting `leader-pr-number` (and `leader-pr-repo`, whose
default is already the promoter repo) in
`.tekton/rhods-operator-gap-eg-build.yaml`.

## Checking a body without running the pipeline

This is the task's logic verbatim; it answers what the two results will be for any PR:

```bash
PR_BODY=$(gh pr view <number> --repo red-hat-data-services/gated-artifacts-promoter --json body -q .body)
OP_REPO=rhoai-rhtap/rhods-operator-gap

LISTED=$(printf '%s' "${PR_BODY}" \
  | grep -oiE 'rhoai-rhtap/[A-Za-z0-9._-]+' | tr 'A-Z' 'a-z' | sort -u)
PR_COUNT=$(printf '%s' "${LISTED}" | grep -c .)

if printf '%s' "${LISTED}" | grep -qixF "${OP_REPO}"; then
  [ "${PR_COUNT}" -le 1 ] && echo "main / skip=true" || echo "main / skip=false"
else
  echo "stable / skip=false"
fi
```
