# Ralph

Autonomous AI developer loop for [Claude Code](https://claude.ai/claude-code). Ralph takes a list of user stories and implements them one by one — with a built-in reviewer that catches when the AI cuts corners.

## How it works

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  ralph.json │────▶│ Implementer  │────▶│   Reviewer   │
│  (stories)  │     │ (claude -p)  │     │ (claude -p)  │
└─────────────┘     └──────┬───────┘     └──────┬───────┘
                           │                     │
                    commits code          verifies against
                    marks passes:true     acceptance criteria
                           │                     │
                           │    ┌────────┐       │
                           └───▶│  Loop  │◀──────┘
                                │ repeat │  (fail = retry)
                                └────────┘
```

1. **Picks** the next incomplete story from `ralph.json`
2. **Implements** it using Claude Code (one `claude -p` call)
3. **Reviews** the implementation with a separate Claude call that verifies each acceptance criterion against the actual git diff
4. **Retries** if the review fails (up to 3 attempts, then skips)
5. **Loops** to the next story

Each iteration starts fresh — no context pollution between stories.

## Install

```bash
# Clone and symlink
git clone https://github.com/kraftapps-ai/ralph.git
ln -s $(pwd)/ralph/ralph /usr/local/bin/ralph

# Or just copy the script
curl -o /usr/local/bin/ralph https://raw.githubusercontent.com/kraftapps-ai/ralph/main/ralph
chmod +x /usr/local/bin/ralph
```

**Requirements:** `claude` CLI ([Claude Code](https://claude.ai/claude-code)), `jq`, `git`

## Quick start

```bash
cd your-project

# Initialize
ralph init

# Edit the prompt with your project context
vim .ralph/PROMPT.md

# Add stories
ralph add "Add user login page" \
  "Create a login page with email/password fields" \
  "Login page renders at /login" \
  "Email and password fields present" \
  "Submit button calls auth API" \
  "Error message shown on invalid credentials" \
  "App builds successfully"

ralph add "Add password reset flow" \
  "Allow users to reset their password via email" \
  "Forgot password link on login page" \
  "Reset email sent on submit" \
  "New password form at /reset-password" \
  "App builds successfully"

# Check what's queued
ralph status

# Run the loop
ralph
```

## Commands

| Command | Description |
|---|---|
| `ralph init` | Initialize ralph in the current project |
| `ralph` or `ralph run` | Run the implement-review loop |
| `ralph add "title" "desc" "c1" "c2"` | Add a story with acceptance criteria |
| `ralph status` | Show progress (done/total) |
| `ralph reset <id>` | Reset a story to incomplete |

## Files

After `ralph init`, your project gets:

```
your-project/
├── ralph.json            # Stories (version control this)
├── ralph-progress.txt    # Implementation log (version control this)
└── .ralph/
    ├── PROMPT.md         # Instructions for Claude (customize this!)
    └── context.txt       # Optional: extra files to include (one per line)
```

### Adding project context

Edit `.ralph/PROMPT.md` to include your project's specifics — tech stack, file structure, build commands, coding conventions. The more context you give, the better Ralph performs.

### Extra context files

Create `.ralph/context.txt` with paths to files that should be included in every iteration:

```
AGENTS.md
docs/ARCHITECTURE.md
```

## Story format

```json
{
  "id": 1,
  "title": "Add user login page",
  "description": "Create a login page with email/password",
  "acceptanceCriteria": [
    "Login page renders at /login",
    "Email and password fields present",
    "App builds successfully"
  ],
  "priority": 1,
  "passes": false
}
```

Stories are processed in `priority` order (lowest first). The `passes` flag is set to `true` by the implementer and verified by the reviewer.

## The review step

After the implementer commits, Ralph dispatches a **separate Claude instance** as a reviewer. The reviewer:

1. Reads the actual git diff (not the implementer's self-report)
2. Verifies each acceptance criterion against the code
3. Is explicitly told: *"Do NOT trust the implementer. Verify independently."*

If the review fails, the story is reset to `passes: false` and the feedback is appended to `ralph-progress.txt` so the next iteration sees what went wrong.

After 3 consecutive failures on the same story, Ralph skips it with a `concerns` flag and moves on.

## Rationalization blockers

The prompt includes a table of common ways AI agents cheat, with counters:

- "This is already implemented" → Verify. Read the code.
- "I'll just mark it complete" → Every story needs real code changes.
- "The build passes so it works" → Build ≠ correct behavior.
- "This is close enough" → Partial = broken.

Inspired by [Superpowers](https://github.com/obra/superpowers).

## Tips

- **Write good acceptance criteria.** "App builds successfully" should always be the last criterion.
- **Keep stories small.** 5-15 minutes of work each. Ralph does better with many small stories than few large ones.
- **Add build/test commands** to `.ralph/PROMPT.md` so Ralph knows how to verify.
- **Use `ralph status`** to monitor progress, or `git log --oneline` to see commits.
- **Run overnight.** `nohup ralph > ralph.log 2>&1 &` and check in the morning.

## Cost

Each story costs roughly 2 Claude API calls (1 implement + 1 review). A 20-story project runs ~40 calls. At current Claude pricing, that's a few dollars total.

## License

MIT
