# doc-archive-relocate

Archive or relocate scattered Markdown files **without breaking a single reference**.

A skill for Claude Code, WorkBuddy, Codex and any SKILL.md-compatible agent.

## Install

```bash
npx skills add zihao0639-netizen/doc-archive-relocate
```

or via SkillHQ:

```bash
skillhq install doc-archive-relocate
```

## What it does

Your docs root has quietly piled up hundreds of md files — contracts, specs, one-shot
reports, postmortems, build logs — all mixed together. Moving things sounds easy until
you discover, weeks later, that three references silently broke.

This skill enforces a three-step discipline:

1. **Scan references first** — split referrers into three consumer groups (active docs,
   unsigned taskbooks, daily memory). A file with few references is not necessarily
   movable; what matters is whether the place referencing it is still alive.
2. **Sort into three buckets** — keep (living authority) / relocate (has a home directory)
   / archive (zero active references + one-shot artifact). Code files count as consumers too.
3. **Move + sync + re-verify** — rewrite only full-path references in the active zone,
   never touch daily memory, and finish with zero leftovers.

## Why it exists

It ships **eleven hard-won pitfalls**, including:

- BSD grep silently ignoring `\|` alternation — a broken verification tool makes you reach
  the **opposite** conclusion, which is more dangerous than not verifying at all
- full-path vs bare-filename references (only one kind breaks)
- brace-shorthand references that sync scripts cannot match
- three kinds of implicit dependencies when moving **code** files (hardcoded hook paths,
  `HERE`-relative sibling loads, baseline hash protection)

The recurring lesson: after any verification, ask "is my checker itself correct?" —
a positive control costs seconds.

## Contents

```
skills/doc-archive-relocate/
└── SKILL.md    # the full methodology (single self-contained file)
```

## License

MIT
