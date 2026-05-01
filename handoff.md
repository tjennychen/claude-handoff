---
description: Save or resume session state. Run before /clear to save, run after /clear to resume.
argument-hint: [optional focus, e.g. "the auth bug"]
---

You are running `/handoff`, a bidirectional session-state command.

The user may pass an optional focus argument: $ARGUMENTS

## Step 1: Determine the project slug

Run these via Bash:

1. `git rev-parse --show-toplevel 2>/dev/null` to get the git root.
2. If a git root is returned, use it as the project dir. Otherwise run `pwd` and use that.
3. Compute slug from the project dir:
   - If the path starts with the user's home dir (`$HOME/`), strip that prefix.
   - Replace `/` with `-`.
   - Replace spaces with `_`.
   - Examples (assuming `$HOME` is `/home/alex`):
     - `/home/alex/code/myapp` becomes `code-myapp`
     - `/home/alex/code/myapp/src/lib` (with git root at myapp) also becomes `code-myapp`
     - `/home/alex/My Project` becomes `My_Project`
   - Fallback: use the basename of the project dir.

The slug is the same whether you launch from the project root or any subfolder, as long as you're inside the same git repo.

## Step 2: Detect mode (SAVE vs LOAD)

Decide based on session context:

- **LOAD mode** if this session is fresh: no files edited yet, no substantive tool calls, conversation is essentially starting. The user is running `/handoff` to resume prior work.
- **SAVE mode** if you have done meaningful work this session: files edited, code changes, multi-turn problem solving, decisions made.
- If `$ARGUMENTS` is non-empty (a focus arg), force SAVE mode.
- If genuinely ambiguous, ask the user once: "Save current state, or load the latest handoff for this folder?"

## Step 3a: LOAD mode

1. Find the latest handoff: `ls -t "$HOME/.claude/handoffs/{slug}_"*.md 2>/dev/null | head -1`
2. If a file is found, Read it. Then output a one-line confirmation: "Resuming from {filename} (saved {timestamp}). Picking up: {one-line task summary}."
3. If no file is found: "No prior handoff for `{project}`. Starting fresh." Stop.
4. **Re-read Key files before acting.** For any file under "Key files" that the Resume instruction will modify or depend on, Read its current state. The handoff is a snapshot at save time; disk may have moved since. Trust disk over the summary, and briefly mention any divergence (one line, only if it changes the plan).
5. **Respect "Don't do".** Anything listed there is a ruled-out approach or dead end. Do not silently retry it. If the Resume instruction conflicts with a Don't do entry, stop and ask the user.
6. Follow the file's "Resume instruction" section as your next action. Do not re-explain the file back to the user; just act on it.

## Step 3b: SAVE mode

1. Ensure the dir exists: `mkdir -p "$HOME/.claude/handoffs/"`
2. Get a timestamp: `date +%Y-%m-%d_%H-%M`
3. Build the absolute path: `$HOME/.claude/handoffs/{slug}_{timestamp}.md`
4. Use the Write tool to create the file. A fresh Claude session must be able to read it cold and pick up the work without asking the user to re-explain.

Use this exact structure:

```markdown
# Handoff: {one-line task summary}

**Created:** {timestamp} from {cwd}

## Task
One or two sentences on what we're working on right now and why.

## Status
- **Done this session:** bullet list of completed items, with file paths where relevant
- **In progress:** what's mid-work right now, including any code that's half-written or commands that were running
- **Next:** the immediate next action a fresh session should take

## Key files
- `/absolute/path/to/file:line`: what changed or what state it's in
- (one bullet per file touched or being read)

## Open threads
- Questions the user asked that aren't resolved yet
- Decisions pending input
- Blockers, error messages (verbatim), surprising discoveries
- Anything they said "let's revisit" or "remind me later"

## Don't do
- Approaches already tried and ruled out, with one-line reason (e.g. "tried X, broke Y because Z")
- Corrections the user made this session ("don't use library X", "stop suggesting Y")
- Dead ends so the next session doesn't retry them. Omit the section entirely if there are none, do not pad.

## Resume instruction
One direct sentence telling the new session what to do first.
```

Rules for content:
- Be specific. Include verbatim error messages, exact file paths with line numbers, decisions made and the reasoning.
- Skip anything already in CLAUDE.md or persistent memory. Capture only the ephemeral session state that would be lost on `/clear`.
- If the session was largely conversational (no code touched), summarize the conversation thread + the user's stated intent.
- Do not pad sections. If "In progress" is empty because everything finished cleanly, write "(none)".
- No secrets. No tokens, API keys, or `.env` content in the handoff file.

5. **Cleanup.** After writing, prune older handoffs for this same slug, keeping only the 3 most recent:
   ```
   ls -t "$HOME/.claude/handoffs/{slug}_"*.md 2>/dev/null | tail -n +4 | xargs rm -f --
   ```
   Run this every save. Bounded growth, no manual maintenance.

6. Output exactly this format (no other prose, no preamble):

```
Handoff saved: {absolute path}

Next: run /clear, then /handoff to resume.
```
