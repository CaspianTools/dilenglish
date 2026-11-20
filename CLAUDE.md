# CLAUDE.md — dilenglish

dilenglish is a **Next.js 15 (App Router)** web app — an English-learning product — built with React 18, Tailwind CSS, Radix UI, and Genkit/Firebase (Data Connect + `firebase-admin`), deployed to **Firebase App Hosting** (`apphosting.yaml`; project `studio-5054979338-10033`).

## Ship mechanism

This repo has **no GitHub Actions workflow** — there is no CI that deploys or publishes on push. It goes live through **Firebase App Hosting**, which builds and rolls out the Next.js app from the connected Git branch (`main`). Depending on how the App Hosting backend is configured in the Firebase console, a rollout is triggered either automatically on a push to the connected branch or manually (`firebase apphosting:rollouts:create` / a rollout from the console). There is **no npm publish** (`package.json` is `private`) and no GitHub Release step. Local verification uses the `package.json` scripts: `npm run dev` (`next dev` on port 9002), `npm run lint`, `npm run typecheck` (`tsc --noEmit`), and `npm run build` (`next build`).

## Owner actions live in GitHub issues, linked to the session that asked

**Rule (owner, 20 Nov 2026).** Anything the owner has to do, decide or check
is a GitHub issue **in this repository**, opened in the same turn it comes
up — never only in the chat. The chat disappears when the session ends; the
issue does not. Link the issue in the reply. Search the open issues first and
add to an existing one rather than opening a duplicate.

### Mark who it is for, twice

| For | Label | Title starts with |
| --- | --- | --- |
| The owner — something to do or check | `for: owner` | `[YOU]` |
| The owner — a choice to make | `for: owner` + `decision` | `[YOU · DECIDE]` |
| Claude — development work to pick up | `for: claude` | `[CLAUDE]` |

Create the labels if the repository does not have them yet. When an issue
changes hands, change both markers.

### The shape of a `[YOU]` issue: steps first, reasons last

```
## Do this

1. One action, naming the site, repository or console it happens in.
   A command, or a Claude for Chrome script for anything done in a
   browser, goes in its own block under the step.

2. Next action.

## Why

Two sentences at most: what is wrong and what doing the steps changes.

**Urgency:** one word, with a half-sentence if it needs one.

## Sessions

**Changed by:** https://claude.ai/code/session_… (pull request owner/repo#N)

**Asked by:** https://claude.ai/code/session_… (go back here with the result)

<details>
<summary>Background</summary>

- Everything else: how it was found, what Claude could not do and why,
  the exact names of projects, services and settings involved.

</details>
```

- **Steps are the issue**: one imperative action each, a blank line between
  every block, under about 120 words above `<details>`. Explanation goes
  inside `<details>`, collapsed. One issue, one thing to do.
- **The owner is not a developer.** Write in plain words what happens to
  the business, the site or the customer; literal text only in a command.
- `[YOU · DECIDE]` uses the same frame with `## Decide` (the options as a
  numbered list, the recommendation marked) in place of `## Do this`.
- **Anything in a browser is a Claude for Chrome script** inside the step,
  never a list of clicks: numbered steps, one action each, the address in
  every step that navigates, the exact on-screen words to expect, stop at
  the first step that misbehaves, nothing it must not touch, and a check of
  the deploy first when the step depends on one. It ends: "Give me step,
  pass or fail, and the exact wording of anything that failed. If every
  step passed, comment the results on <issue link> and close it; otherwise
  comment the failing step and leave it open. End the comment with: Report
  this back to <Asked by session link>."
- **A changed script is a new comment**, never an edit: headed
  `Updated script (<date>) — use this one`, holding the whole script. A
  "run it again" is a new comment too.
- **Commands for the owner** run on Windows in PowerShell and Command
  Prompt alike: full paths, no shell variables, no `&&`, no pipes.

### Link the session that asked

An issue alone does not say which conversation wants the answer, so every
one carries the session's link, and the owner can go back to that session
and report what was done.

- **Where the link comes from:** the `Claude-Session:` line of the commit
  attribution (`https://claude.ai/code/session_…`); if it is missing,
  `get_session` with no id gives the session id.
- **Every `[YOU]` and `[YOU · DECIDE]` issue has the `## Sessions` section**:
  **Changed by** (the session that made the change being checked, with its
  pull request) and **Asked by** (the session waiting for the result). When
  they are the same, one line: `**Session:** <link> (made the change and is
  waiting for the result)`.
- **Every comment that asks the owner to run a script** starts with
  `**Asked by:** <link>`; the newest comment's session is the one to go back
  to. A `[CLAUDE]` issue names the session that opened it the same way.
- A session that finds a result on an issue acts on it there, even when
  another session asked: the link is for the owner, not a lock.

### Close what is finished

Close an issue once the evidence is in (the owner's report, a script's
comment, output they pasted), with a one-line comment saying what proves
it; `not planned` when nothing was done. Never close on an assumption.
End every issue and comment with the Claude Code attribution footer.

## Worktrees & the ship rule

Claude Code can run parallel sessions in isolated **git worktrees** (`claude --worktree <name>`, or ask it to "work in a worktree" → the `EnterWorktree` tool). A worktree lives under `.claude/worktrees/<name>/` on branch `worktree-<name>`, branched **`fresh` from `origin/main`** by default (set `worktree.baseRef: "head"` in `.claude/settings.json` to carry local HEAD instead). `.claude/worktrees/` is gitignored and **`.worktreeinclude`** copies any local secrets (`.env*`, `*.pem`) into new worktrees — see those two files. `node_modules` is *not* copied: run `npm install` in each new worktree.

**The catch:** landing to `main` is what makes a change go live — Firebase App Hosting builds and rolls out the Next.js app from the connected `main` branch. A worktree sits on `worktree-<name>`, so a plain "commit + push" *inside* a worktree pushes a feature branch and **ships nothing** (App Hosting only watches `main`). So a standing "commit + push at end of task" flow **adapts inside a worktree** — do NOT blindly push `main` from one:

1. **Commit, pause before landing.** Auto-commit finished work on the `worktree-<name>` branch, then **stop and report**. Never merge to `main` / push `main` (the step that triggers the App Hosting rollout) without the owner's explicit go-ahead. *(On `main` — the normal solo flow — the rule is unchanged: commit, then push when the owner is ready.)*
2. **Serialize landings — one at a time.** Never land two worktrees to `main` in parallel. If another worktree/session is still in flight, wait for it to land first. There's no live cross-session signal, so "wait" means: at land time `git fetch` and rebase onto whatever `origin/main` now is; if the owner says another is mid-flight, hold until told it's done.
3. **Resolve conflicts in the worktree, never on `main`.** At land time: `git fetch origin` → **rebase `worktree-<name>` onto the latest `origin/main`** → resolve every conflict *there*, so `main` only ever receives an already-merged, clean tree.
4. **Finalize the version/release bump last.** If a change touches the version in `package.json` (or any release metadata), don't fix the number until after the rebase — take *current-main + 1* so two shippable worktrees don't collide on it, then regenerate anything derived from it.
5. **Re-verify + rebuild after resolving.** Re-run this repo's checks — `npm run lint`, `npm run typecheck`, and `npm run build` (`next build`) — before landing. A conflict resolution that isn't re-verified is a bug waiting to ship.
6. **Only then ship.** Fast-forward `main` to the clean, verified branch → `git push origin main`, which lets Firebase App Hosting build and roll out the new revision. Confirm the rollout succeeded in the Firebase console (App Hosting → your backend → Rollouts). **Never push a conflicted or failing tree to `main`.**

For solo, single-stream work that ships immediately, **skip worktrees and work on `main` directly** — the rule needs no adaptation. Reserve worktrees for genuine parallelism (two tasks at once) or experiments you may not ship.
