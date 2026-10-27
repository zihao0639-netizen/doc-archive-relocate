---
name: doc-archive-relocate
description: 'Archive or relocate scattered Markdown files without breaking a single reference. Trigger on "my docs are a mess", "archive the old ones", "where should this go", "keep cleaning", or whenever the docs root has piled up hundreds of md files. Core three steps: scan references and classify consumers into three groups, sort every file into archive / relocate / keep, then move and rewrite only the full-path references in the active zone, and re-verify zero broken links. Ships eleven hard-won pitfalls, the most important being: when your own verification tool misbehaves, it makes you reach the opposite conclusion.'
license: MIT
compatibility: 'Needs macOS or Linux with git, grep and python3. Works in Claude Code, Claude.ai, Codex, Cursor and any SKILL.md-compatible agent.'
metadata:
  author: zihao0639-netizen
  version: '1.0.0'
  tags: 'markdown,docs,archive,relocate,zero-broken-links'
---

# Markdown Archive and Relocate (Zero Broken Links)

> **Scope**: the docs root mixes "living authority" (contracts / specs / entrypoints)
> with "one-shot process artifacts" (reports / evidence / postmortems / build logs).
> Goal = **leave only living files at the root**, while **breaking not a single reference**.
> ⚠️ Binary assets (images, videos) need the symlink-preserving route — here you can
> only **rewrite text**.

---

## 🔴 Step 0: Establish criteria before touching anything (never judge by "the filename looks old")

Criteria in one sentence: **"Will I open this file again next time?"**

| Verdict | Condition | Destination |
|---|---|---|
| **Keep** | Living authority: contracts / specs / entrypoints (`_*.md`) / templates / checklists / decision bases (`rulings/`) | Stay put |
| **Relocate** | A home directory already exists (`review/` `rulings/` `plans/` `taskbooks/` `research/`…) ⇒ group with peers | That directory |
| **Archive** | Zero active references + one-shot process artifact (report / evidence / postmortem / cleanup / triage / analysis / build log / finding) | `docs/archive/<batch>/` |

★ **Relocate ≠ Archive** — relocate is "put it in the right place", archive is "exit the active view".

---

## The three steps (order is mandatory)

### ① Scan references, **split consumers into three groups**

```python
# Only read directories that 【may reference docs】 — no project-wide os.walk (see pitfall 2)
ROOTS = ["docs", ".workbuddy/memory", "tools"] + root-level *.md
SKIP  = {"archive","delivery","pending","to-sign","_backup","__pycache__","_archive_oneshot","target"}
```

For each candidate, count **active references** and split referrers into three groups:
- **Active documents** ← 🔴 **if it has these, the file basically cannot move**
- **To-sign** (taskbooks not yet run) ← means "this work is not finished"
- **Daily memory** (`.workbuddy/memory/`) ← historical record, **not a consumer**

### ② Sort (three questions that must be asked)

**Question 1: Is the place that references it still alive?**
Evidence: a taskbook ledger file looks like it has only **1** reference (safest),
but that 1 reference is an **F4 "resume the driver" item in a refactor master table, and F4 is not done yet**
⇒ archiving it = **hiding the instructions for pending work**.
> **"Few references" ≠ "safe to move"** — ask instead: **"Is the place referencing it still alive?"**

**Question 2: Is the reference a 「full path」or a 「bare filename」?**

| Type | What happens if you move the file | Action |
|---|---|---|
| **Full path** `` `docs/review_v2_A.md` `` | 🔴 **breaks** | **must rewrite** |
| **Bare filename** `review_v2_A.md` | does not break (just text) | leave it |

**Question 3: Is any code using it?**
Evidence: an analysis document is referenced by **a criteria script (code!)** ⇒ **cannot move**.
⇒ when scanning, **count `.py/.sh/.toml` as consumers too**.

### ③ mv + sync references + re-verify

```bash
# Move (list files explicitly, no wildcards; git mv keeps history, git recognizes R100 = rename)
git mv docs/<old-name> docs/archive/<batch>/<old-name>
```

When syncing references, **only touch the active zone**:

```python
SKIP_PARTS = {".git","assets","archive","delivery","_backup","_archive_oneshot","__pycache__"}
# 🔴 .workbuddy/memory must also be excluded — daily memory is history; editing it = tampering
if ".workbuddy/memory" in path: continue
```

Re-verify (**the active zone must end with zero leftovers**):

```bash
# ✅ use ripgrep, or grep -E
grep -rE "old-filename" docs --include="*.md"
# 🔴 the only allowed leftover = deliberately preserved historical notes
```

---

## 🔴 Eleven pitfalls (all of them stepped on)

> ⚠️ Pitfalls 1 and 9 are the same family: **your verification tool itself is wrong ⇒ the conclusion flips direction**.
> ⇒ before concluding, ask: **"Is my checker itself correct?"** (a positive control costs seconds).

### 🚨 Pitfall 1 (the worst): **BSD grep does not support `\|`**

```bash
grep -v "archive\|delivery"   # 🔴 excludes nothing! Looks "normal", actually filters zero
grep -n "A\|B\|C" f           # 🔴 no output ⇒ you misread it as "not in the file" (it is)
```

**How bad it gets**:
- References inside `archive/` and `delivery/` all count as "active references" ⇒ movable files get misjudged as unmovable;
- When a search finds nothing you misread it as "last run did not take" ⇒ you redo finished work.

**✅ Mandatory move (stop relying on memory)**:
```bash
grep -E "archive|delivery"          # use -E
grep -vE "archive|delivery"         # exclude with -vE
grep -nE "A|B|C" f                  # alternation with -nE
```
**For batch jobs always use ripgrep** — it supports `|` and is faster.

> ★ **Generalization**: **a broken verification/filter tool is more dangerous than no verification** — it makes you reach the **opposite conclusion**.
> So after every "exclude with grep", **verify once that it really excluded** (test with a reference known to be inside `archive/`).

### 🚨 Pitfall 2: project-wide `os.walk` killed by SIGTERM (exit 137)

**It lies**: the same script **runs in the morning, dies an hour later** —
because **another session on the machine started heavy work** (large-file processing etc.; a 16 GB machine cannot take it).

**✅ Mandatory move**: scan **only the necessary directories**, and skip files >2MB:

```python
ROOTS = ["docs", "tools"] + root-level *.md     # 🔴 never os.walk(".") or scan big asset dirs
SKIP  = {"archive","delivery","_backup","__pycache__","_captures","_log_archive","_archive_oneshot","target"}
# and do not forget .workbuddy/memory (daily memory is history, untouched)
```

★ **Corollary**: in such projects **"can the script finish" depends on whether another session is running heavy work** —
　so **long scans must be narrowed or delegated**, do not bet on an idle machine.

### Pitfall 3: **the path changed, the semantics did not follow**

Example: a health-checker's `LEGACY_START` table means "**these old must-read files are still being read**";
after the file is archived the code **still reads it** ⇒ **a false alarm, forever**.
⇒ the fix is **removing it from the table**, not rewriting the path ⇒ after editing, ask "**is the semantics of the code/table that reads it still correct?**"

### Pitfall 4: **the script's existence check misses the path prefix**

```python
if not os.path.exists(old)          # 🔴 old is a bare filename, but you cd'ed to project root ⇒ always "missing"
if not os.path.exists("docs/"+old)   # ✅
```
★ **"Scan references first" catches this** — so that step is not just link protection, **it doubles as a pre-flight health check**.

### Pitfall 5: directories like `_archive_oneshot` slip past SKIP

SKIP with **exact directory-name matching** misses them. ⇒ to decide "is this a history zone", judge by **semantics** (does it contain the word "archive"?), not exact equality.

### Pitfall 6: entry notes of archive/relocate zones not updated

**Relocating a group of files must update that directory's `_index.md` / `README.md`**, otherwise the relocation did not really happen.
(a new directory **must** be created together with its entry.)

### 🚨 Pitfall 7: **brace-shorthand references get missed** (silent partial rewrite)

```markdown
`docs/{START.md,status.md,eval_goal_mode.md,rules.md}`   ← shorthand form
```

The sync script looks for `docs/<full filename>` — **it cannot match this form** ⇒ after the move it **silently points at the wrong path**.

**✅ Mandatory move**: scan for this form before moving:
```bash
grep -nE 'docs/\{[^}]*\}' <active-zone>        # find brace shorthand
```
On a hit ⇒ **hand-edit each** (or rewrite the block). ★ if "files mentioned inside the braces" and "the referrer" end up in the **same directory** after the move, bare names still work.

### 🚨 Pitfall 8: **half-width double quotes in a commit message truncate `-m`** (one wasted round)

```bash
git commit -m "……per the 「trace what happened」 principle……"     # 🔴 half-width quote closes early
# ⇒ if ** appears later, zsh treats it as 【recursive glob】⇒ no matches found ⇒ commit fails
```

**✅ Mandatory move**: **always wrap commit messages in single quotes** (`git commit -m '...'`), use full-width 「」 for inner quotes.
★ the upside: staging is not lost, just rerun commit once; but a round is wasted, and **the failure looks like "the command is broken" when it is really the quoting**.

### 🚨 Pitfall 9: **to check "named / covered / included", use literal containment, not regex** (it makes you report fake problems)

When checking "does this index/table name every file", **regex shreds filenames**:

```python
# 🔴 wrong: incomplete char class ⇒ long names / names with full-width parens get shredded
re.findall(r'([A-Za-z0-9_\u4e00-\u9fff\uff00-\uffef\-\.]+\.md', txt)
#   observed fragments: `_lead_review_20260927.md` (real name is longer) · `one_checked）.md` · `final.md`
#   ⇒ computed "85 of 88 named" ⇒ almost reported "delivery overstated coverage" = a fake problem

# ✅ right: the filesystem gives real names, so check each literally
for f in sorted(os.listdir(d)):
    if f.endswith(".md") and f != "_index.md" and f not in txt:   # literal containment
        miss.append(f)
```

**Rule: checking "named / covered / included / referenced" always uses literal containment (`f in text`), never regex parsing of text.**
★ twice the same day (the other was BSD `grep -v "A\|B"` doing nothing) — **common root: the verification tool itself was wrong ⇒ the conclusion flipped direction**,
more dangerous than not verifying. **after every check, ask "is my checker itself correct?"** (positive/negative controls are the cheapest method).

### 🚨 Pitfall 10: **stage files one by one, never by directory** (you will commit another session's work)

```bash
git add -- docs/delivery/          # 🔴 whole directory ⇒ sweeps in "just appeared, not mine" files
git add -- docs/delivery/a.md docs/delivery/b.md   # ✅ explicit listing
```

**Why this gets its own entry**: the workspace often has **another session working in parallel** (one session running batch downloads, another organizing docs).
`git add <directory>` commits the other session's **just-written, uncommitted files** ⇒ **someone else's in-flight work lands in my commit**
(and the commit message never mentions it ⇒ later "where did this file come from").

**✅ Mandatory move**: after staging, **always look at** `git diff --cached --name-status`, and ask of each line "**is this mine?**".
On a mistaken sweep ⇒ `git rm --cached <file>` (**disk untouched**) ⇒ back to "untracked", left to the session that created it.

---

### 🚨 Pitfall 11: **before moving code files, scan three kinds of implicit dependencies** (miss one and you get a silent broken link)

> Pitfalls 1~10 are all about **documents**. This one is about **code/scripts** — completely different:
> document references are "paths written in text" (visible); code dependencies hide **in many more places**.

**① External hardcoded paths** — another file pins its **exact location**:

```bash
# you must grep each of these 「hardcoded manifest」 files:
#   git hooks (.git/hooks/ or core.hooksPath target)
#   launchd plist (~/Library/LaunchAgents/*.plist)
#   protected set / allowlist / path tables (object paths hardcoded in scripts)
grep -rn "<moved-file-name>" .git/hooks/ tools/hooks/ ~/Library/LaunchAgents/ tools/
```
★ Example: a hook contains `gate="$root/tools/gate_script.py"` — move it ⇒ **hook fail-closed ⇒ every commit is rejected from then on**.
★ Example: a driver `protected_list()` hardcodes 21 absolute paths ⇒ move any one ⇒ **baseline MISSING ⇒ the scheduler refuses to run**.

**② Same-directory relative dependencies** (**the most hidden**) — loading neighbor files via `HERE` / `__file__`:

```bash
python3 - <<'PY'
import os
for f in sorted(os.listdir("tools")):                     # ← swap in the target dir
    p=os.path.join("tools",f)
    if not (os.path.isfile(p) and f.endswith(".py")): continue
    for i,l in enumerate(open(p,encoding="utf-8").read().splitlines(),1):
        s=l.strip()
        if s.startswith("#"): continue
        if ".py" in s and ("HERE" in s or "__file__" in s or 'os.path.join(ROOT, "' in s):
            print(f"{f}:{i}  {s[:110]}")
PY
```
★ Example: a gate script = `os.path.join(HERE, "health_check.py")` ⇒ **moving only one breaks it** (the other stays behind).
★ Example: a QC script hangs 3 neighbor files at once (`HERE / "xxx.py"`) ⇒ **a chain of 4 files**.
★ Example: `os.path.join(ROOT, "tools/batch_job.py")` ⇒ **ROOT-relative**, after moving ROOT changed ⇒ silently computes the wrong path.

**③ Is the content hash protected by a 「baseline / fingerprint table」** — some projects record sha256 of protected files:

```bash
grep -n "<filename>" ~/.config/*/baseline.sha 2>/dev/null   # or any fingerprint table
```
★ Example: a permission script's hash is in `baseline.sha` ⇒ **changing its content also moves the trust anchor** ⇒ requires approval + **rewrite the baseline** afterwards.
★ **Corollary rule**: **"path must not move" and "content must not change" are two independent things**, verify separately:

| | path | content |
|---|---|---|
| hardcoded by a hook | 🔴 no | ✅ yes (unless hash in baseline)|
| hash in baseline | 🔴 no | 🔴 no |

**④ After all three scans, still ask: "which dimension do the protection clauses actually cover?"**

★ **Common root cause**: a taskbook says "directory X and Y must not be touched" —
**it protects the "directory" dimension but misses the "root-level loose files" dimension** ⇒ three protected files are left naked.
⇒ **when writing protection clauses, list by 「object」 (trace every file), not by 「directory」.**

**⑤ Self-proving delivery** (write it into the taskbook, or the executor will not know what to prove):
- `ls -la` of the protected files (proving they are still in place)
- `shasum -a 256` **before AND after values** (proving content untouched)
- the **actual output** of the ② scan command (not hand-copied)

---

## Report format (give this after every run)

1. **"old path → new path" mapping** (or grouped: N archived / M relocated)
2. **References synced**: how many files, how many spots
3. **Re-verification result**: leftovers in the active zone (**must be 0**, exceptions stated)
4. **List of files judged 【keep】 + reasons** (write into that batch's README, **so the next round does not retry**)
5. Before/after health numbers (root file count / files over 32KB / injection layer)
