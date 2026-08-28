# Configuration & customization

Everything in this doc is optional. With no configuration at all, the review
applies universal lenses — security, performance, correctness, architecture,
type safety, test coverage — and picks its depth from the size and risk of the
diff.

Configuration lets you do three things: tell the review **what to flag and what
to skip**, set **default flags** so you stop passing them per run, and register
**extra reviewer providers**.

## Guidance sources

The review reads convention and lens guidance from several files, in this order:

| File | Scope | Use it for |
|------|-------|------------|
| `REVIEW.md` (repo root) | Review only | What to flag, what to skip, house style rules |
| `.claude/docs/code-review.md` | Review + agents | Domain checklist with severity lenses |
| `CLAUDE.md` | All Claude Code tasks | Project conventions (also read during review) |
| `~/.claude/adversarial-review.json` | Flag defaults, user-wide | Default `mode` / `comment` / `lanes` for every repo |
| `.claude/adversarial-review.json` | Flag defaults, per repo | Same keys, overrides the user file per key — except executable `lanes` (see below) |
| [`review-protocol.md`](review-protocol.md) | Spec | The normative cross-provider contract: artifacts, schemas, adapters |

## Scoped reviews (`--paths`)

By default a review covers the entire branch diff. `--paths <glob>[,<glob>...]`
restricts it to branch changes in matching files. Reach for it to:

- re-review one subsystem after a large rebase,
- split the review of a big branch into digestible passes, or
- point the pipeline at just the risky directory.

Patterns become git pathspecs: globs get `:(glob)` semantics, and bare
directories match everything beneath them. Change-type classification and review
depth are computed from **in-scope files only**, and every artifact and PR
comment is labeled with the scope so a partial review is never mistaken for a
full one. If no branch changes match, the review stops early and lists the
branch's changed files so you can adjust.

The `/adversarial-review:run` entry point supports it:

```bash
/adversarial-review:run --paths "src/api/**,src/auth/**"
```

The full pathspec-translation and scope-propagation rules are specified in
[`review-protocol.md`](review-protocol.md#review-scope---paths).

## Flag defaults

`adversarial-review.json` recognizes three keys; unknown keys are ignored.

```json
{
  "mode": "no-fix",
  "comment": false,
  "lanes": {
    "gemini": {
      "probe": "gemini --version",
      "exec": "gemini --prompt \"$(cat {prompt_file})\" > {output_file}",
      "guard": true,
      "models": "gemini-3-pro"
    }
  }
}
```

**Precedence** is explicit flag > project config > user config > built-in default:

- `--no-fix` / `--fix` beat the `mode` key.
- `--comment` / `--no-comment` beat the `comment` key.

A malformed config file is noted in the report and skipped — it never blocks
a review.

The review is local-first: `comment` defaults to `false`, so findings only show up in
the report and `summary.md`. Set it to `true` (or pass `--comment`) to also post
findings to the PR/MR as a pending (unpublished) review — the review stays a draft
until the author publishes it.

### Why `lanes` is user-level only

The `lanes` key is the one exception to per-key overriding. Executable adapters
are honored from the **user-level** `~/.claude/adversarial-review.json` only,
because the project-level file ships with the repo under review — and repo
content must never define commands the review executes. A project-level `lanes`
entry may only be `false`, disabling that user-defined lane for the repo.
Provider names must be filename-safe slugs (`^[a-z0-9][a-z0-9_-]{0,31}$`);
anything else is ignored with a note.

## Adding more providers (`lanes`)

Cross-vendor reviewers are defined exclusively through the `lanes` adapter
registry. Any provider with a headless one-shot CLI can join a Claude-led
review as an extra sidecar lane through it — no plugin changes needed.

Because adapters define shell commands the review runs, they must come from your
environment, never from the repo being reviewed (a hostile branch could
otherwise turn review setup into arbitrary execution). Each adapter declares:

| Field | Required | Meaning |
|-------|----------|---------|
| `probe` | yes | Shell command; exit 0 means installed **and** authenticated |
| `exec` | yes | Per-pass command template. Placeholders: `{prompt_file}`, `{output_file}`, `{repo_root}` |
| `guard` | no | Set `true` when the CLI lacks a read-only sandbox, enabling the baseline-aware tracked-file guard |
| `models` | no | Informational model string for provenance tables |

Adapter lanes follow the lifecycle in SKILL.md's "Additional provider lanes":
probe at Step 0, one CLI call per pass, process exit as the only completion
signal, a ~10-minute bound past the Claude wave, and a failed or empty report
never blocks the review. Their reports (`optimizer-<provider>.md` /
`skeptic-<provider>.md`) merge into the same synthesis, and cross-vendor
agreement weighting applies to every lane equally. A project-level `lanes`
entry set to `false` disables that lane for the repo; there is no flag to
disable all cross-vendor lanes at once.

The full schema, orchestration contract, and an add-a-provider checklist live in
[`review-protocol.md`](review-protocol.md#provider-adapter-registry).
