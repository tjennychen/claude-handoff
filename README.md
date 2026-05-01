# /handoff

A single Claude Code slash command for surviving `/clear`. One command, two modes: SAVE before you clear, LOAD after.

Designed for the moment your context window gets heavy and you want to pick up tomorrow without re-explaining anything.

---

## Why

`/clear` keeps Claude Code sessions fast and cheap, but it wipes everything. The fix is a thin markdown snapshot on disk: small enough to read cold, complete enough to resume from.

This skill is opinionated about three things:

1. **One command, two modes.** No separate save/load. The command auto-detects whether you want to capture state or restore it, based on whether the session has done meaningful work yet.
2. **Per-project history.** Handoffs are timestamped and grouped by project, so you can rewind to where you were this morning, not just last save. Auto-pruned to the 3 most recent per project.
3. **Out of the repo.** Files live in `~/.claude/handoffs/`, never in the repo root. No `.gitignore` upkeep, no risk of pushing session state into a PR.

---

## Install

Drop the command file into your global Claude Code commands directory:

```bash
mkdir -p ~/.claude/commands
curl -fsSL https://raw.githubusercontent.com/tjennychen/claude-handoff/main/handoff.md \
  -o ~/.claude/commands/handoff.md
```

Verify by running `/handoff` in any Claude Code session.

If your global settings restrict the working directory scope (or you start sessions outside `~/.claude/`), add the handoffs folder to `permissions.additionalDirectories` in `~/.claude/settings.json` so the command can read and write there from any cwd:

```json
{
  "permissions": {
    "additionalDirectories": ["~/.claude/handoffs"]
  }
}
```

---

## Usage

```
You: [a long coding session, multiple files edited, problem half-solved]
You: /handoff

Claude: Handoff saved: /Users/you/.claude/handoffs/myapp_2026-05-01_14-22.md

        Next: run /clear, then /handoff to resume.

You: /clear
You: /handoff

Claude: Resuming from myapp_2026-05-01_14-22.md (saved 14:22). Picking up:
        wiring auth middleware into the request pipeline.

        [proceeds to execute the next step]
```

Optional focus argument forces SAVE mode and signals what the handoff is centered on:

```
/handoff the migration bug
```

---

## Storage

Handoffs go to `~/.claude/handoffs/<slug>_<timestamp>.md` where:

- `<slug>` is computed from the git root (or cwd, outside a repo). Subfolder launches resolve to the same slug as the repo root, so handoffs from `myapp/` and `myapp/src/lib/` go to the same project file group.
- `<timestamp>` is `YYYY-MM-DD_HH-MM`.
- Old handoffs auto-prune on every save: only the 3 most recent per project survive.

Effects:

- `ls ~/.claude/handoffs/` is a global view of in-flight work across every project.
- The repo stays clean. No `HANDOFF.md` to gitignore, no risk of pushing it.
- Rewinding to "where I was this morning" is just reading the second-most-recent file for that project.

---

## File format

```markdown
# Handoff: <one-line task summary>

**Created:** <timestamp> from <cwd>

## Task
What this session was working on and why.

## Status
- Done this session
- In progress
- Next

## Key files
- `/absolute/path:line` — what state it's in

## Open threads
Unresolved questions, blockers, verbatim error messages.

## Don't do
Approaches already ruled out, with one-line reason. Omitted if there are none.

## Resume instruction
One direct sentence for the next session.
```

The "Don't do" section is the one most often skipped, and it's also the most valuable. Without it, a fresh session can re-explore the dead ends you already ruled out. Be specific: `tried X, broke Y because Z`.

---

## Behavior on resume

When `/handoff` runs in LOAD mode, it:

1. Loads the latest handoff for the current project.
2. **Re-reads files listed under "Key files"** before acting. The handoff is a snapshot; disk may have moved since save. Trust disk over the summary.
3. **Respects "Don't do".** If the Resume instruction conflicts with a ruled-out entry, it stops and asks.
4. Executes the Resume instruction without re-explaining the handoff back to you.

---

## What it skips on save

- Anything already in `CLAUDE.md` or persistent memory. Handoffs are for ephemeral session state, not stable project facts.
- Padding. Empty sections are written as `(none)` or omitted.
- Secrets. No tokens, API keys, `.env` content.

---

## License

MIT
