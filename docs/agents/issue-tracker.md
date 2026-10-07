# Issue tracker: GitHub

GitHub Issues is the single source of truth for task and progress tracking. The M1
scope lives in issue #1.

Use **`gh-axi`** for issue and PR operations; plain `gh api` is the escape hatch for a
raw field. The repo is inferred from `git remote -v`.

- **Create**: `gh-axi issue create --title "..." --body-file <path>`
- **Read**: `gh-axi issue view <number> --comments --full` (the default truncates, and a
  truncated body reads exactly like a short one)
- **List**: `gh-axi issue list --state open` (`--label`, `--state`, `--limit`)
- **Comment**: `gh-axi issue comment <number> --body-file <path>`
- **Labels**: `gh-axi issue edit <number> --add-label ...` / `--remove-label ...`
- **Close**: `gh-axi issue close <number> --reason completed --comment "..."`

⚠️ **Pass bodies as `--body-file`, never `--body "..."` or an unquoted heredoc.** An
unquoted heredoc containing backticks executes them; that replaced #12's 53k-character
body with a mangled 101k one. `scripts/check-heredocs.sh` refuses the unquoted form in
committed scripts; one typed at a prompt still has to be quoted by hand.

## Rewriting an existing issue body goes through the script

```bash
scripts/issue-body.py get  12 /tmp/i12.md   # fetch, verified against the API's own count
#   ...edit /tmp/i12.md...
scripts/issue-body.py put  12 /tmp/i12.md   # write, then RE-FETCH to prove it stored
scripts/issue-body.py diff 12 /tmp/i12.md   # is the remote still what this file says?
```

`put` verifies by reading back, because GitHub accepting the request is not evidence:
the corrupting write was accepted cleanly. Adding a comment needs none of this; prefer a
comment whenever the content is additive.

When a skill says "publish to the issue tracker", create a GitHub issue. When it says
"fetch the relevant ticket", `gh-axi issue view <number> --comments --full`.
