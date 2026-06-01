---
name: handoff
description: Compact the current conversation into a structured handoff document so a fresh agent can resume the work, or pick up an existing handoff to continue where a previous session left off. Use whenever the user wants to hand off, pause, checkpoint, save progress for later, "write a handoff", "pick up where we left off", "resume from the handoff", or run out of context and need continuity across sessions.
argument-hint: "[pickup [file]] | what the next session will focus on"
---

This skill has two modes. Pick the mode from the argument, then follow that section.

- **No argument, or a description of upcoming work** → WRITE mode. Produce a handoff document.
- **Argument starts with `pickup`, `resume`, or `continue`** → PICKUP mode. Load an existing handoff and carry on.

The two modes share one storage convention so a handoff written in one session is findable in the next.

## Storage convention

Handoffs live in the OS temporary directory (not the workspace, so they never get committed). The filename encodes the project and a sortable timestamp so PICKUP mode can find and order them:

```
<os-tmp>/handoff-<project>-<YYYYMMDD-HHMMSS>.md
```

- `<os-tmp>` is the OS temp dir. Resolve it at runtime - do not hardcode `/tmp`. In bash, prefer `${TMPDIR:-/tmp}`; in Node, `os.tmpdir()`.
- `<project>` is the basename of the git repo root (`git rev-parse --show-toplevel`), or the cwd basename if not a repo. Lowercase, spaces to hyphens. This scopes a pickup to the current project so you never resume someone else's work.
- The timestamp is the current local time, sortable so the newest file wins a lexical sort.

Resolve all of this in **one shell call** so nothing is guessed and pickup never turns into a filesystem fishing expedition - the whole point of the convention is that the file's location is computed, not searched for:

```bash
proj=$(basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
dir="${TMPDIR:-/tmp}"
# WRITE - build a fresh, correctly-named path:
path="$dir/handoff-$proj-$(date +%Y%m%d-%H%M%S).md"
# PICKUP - list this project's handoffs, newest first, in a single call.
# find + reverse sort is bash/zsh-safe (a bare glob errors on no-match in zsh);
# the sortable timestamp makes a reverse name sort == newest first.
find "$dir" -maxdepth 1 -name "handoff-$proj-*.md" 2>/dev/null | sort -r
```

## WRITE mode

Goal: a document that lets a fresh agent resume **without re-reading this conversation**. Write for a smart colleague who has the codebase but zero context on what just happened.

1. Compute the path with the one-call snippet above (`$path`). Get the real timestamp from the shell - never guess it.
2. Fill in the template below. Omit a section only if it is genuinely empty - keep the headings stable so every handoff reads the same way and the next agent knows where to look.
3. Save the file, then tell the user the exact path and that they can resume with `/handoff pickup`.

Principles:
- **Reference, don't duplicate.** Plans, PRDs, ADRs, issues, commits, and diffs already exist - link them by path, issue number, or commit SHA. The handoff captures what is *not* written down elsewhere: the live mental state.
- **Next steps must be actionable.** "Continue the auth work" is useless. "Add the `requireCourse` check to `server/routes/course.js:42`, then re-run `npm test`" is a handoff. Order them so the next agent can start at step 1.
- **Capture decisions and their rationale.** The biggest waste in a cold start is relitigating a choice that was already settled. Record what was decided and why, so it stays decided.
- **Redact secrets.** Strip API keys, passwords, tokens, and PII. Reference where a secret lives (e.g. "key in `secrets/gemini-key.json`"), never its value.

### Template

```markdown
# Handoff: <one-line title of the work>

- **Project / branch:** <repo> / <git branch>
- **Written:** <timestamp>
- **Next session focus:** <from the argument, or the obvious next objective>
- **Resume with:** `/handoff pickup`

## Goal
What we are ultimately trying to accomplish, in 1-3 sentences.

## Current state
What is done and working right now. What is in progress and how far along.
Note uncommitted changes, the working branch, and anything half-finished.

## Next steps
Ordered, concrete, actionable. Each step names the file/command/check involved.
1. ...
2. ...

## Key files & locations
Paths the next agent needs, each with a one-line "why it matters".

## Decisions & rationale
What was decided and the reasoning, so it is not relitigated. Include rejected
alternatives when the rejection is non-obvious.

## Blockers & open questions
What is stuck, what needs a decision from the user, what is uncertain.

## How to verify
How to build/run/test this work and confirm it behaves (commands + expected result).

## References
Plans, issues, PRs, commits, docs - by path/number/SHA, not copied content.

## Suggested skills
Skills the next agent should invoke, each with a one-line why.
```

## PICKUP mode

Goal: locate the relevant handoff, load it, orient yourself, and continue the work.

1. **Find candidates.** If the argument names a file (`pickup <file>`), use that path directly. Otherwise run the one-call discovery command from the storage convention (`find ... | sort -r`) - that single line is the entire search. Do not stat, grep, or walk directories looking for handoffs; if that command returns nothing, there is nothing to resume.
2. **Select:**
   - **None found** → tell the user no handoff exists for this project and where you looked. Stop; do not invent one.
   - **Exactly one** → load it without prompting.
   - **Two or more** → list them with timestamp and the `# Handoff:` title line, and ask which to resume. Do not auto-pick the newest when several exist.
3. **Load & orient.** Read the file. Give the user a short orientation: the goal, the current state, and the immediate next step you are about to take. Invoke any skills listed under "Suggested skills".
4. **Resume.** Begin at the first unfinished item in "Next steps". Reconcile lazily: do **not** open an upfront audit of every path the doc mentions - that recreates the slow, multi-call scanning this skill exists to avoid. Trust the doc to point you, and only when you actually go to act on a file that isn't where the handoff says, trust the repo over the document and say what changed (e.g. "handoff says `require-course.js`; the file is `requireCourse.js` - using that").
