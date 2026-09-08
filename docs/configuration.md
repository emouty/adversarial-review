# Configuration & customization

Everything in this doc is optional. With no configuration at all, the review
applies universal lenses — security, performance, correctness, architecture,
type safety, test coverage — and picks its depth from the size and risk of the
diff.

Configuration lets you do two things: tell the review **what to flag and what
to skip**, and set **default flags** so you stop passing them per run.

## Guidance sources

The review reads convention and lens guidance from several files, in this order:

| File | Scope | Use it for |
|------|-------|------------|
| `REVIEW.md` (repo root) | Review only | What to flag, what to skip, house style rules |
| `.claude/docs/code-review.md` | Review + agents | Domain checklist with severity lenses |
| `CLAUDE.md` | All Claude Code tasks | Project conventions (also read during review) |
| `~/.claude/adversarial-review.json` | Flag defaults, user-wide | Default `mode` / `comment` for every repo |
| `.claude/adversarial-review.json` | Flag defaults, per repo | Same keys, overrides the user file per key |

## Scoped reviews (`--paths`)

By default, a review covers the entire branch diff. `--paths <glob>[,<glob>...]`
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
`claude/skills/run/SKILL.md` (Step 0).

## Flag defaults

`adversarial-review.json` recognizes two keys; unknown keys are ignored.

```json
{
  "mode": "no-fix",
  "comment": false
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
