# CLAUDE.md — dilenglish

dilenglish is a **Next.js 15 (App Router)** web app — an English-learning product — built with React 18, Tailwind CSS, Radix UI, and Genkit/Firebase (Data Connect + `firebase-admin`), deployed to **Firebase App Hosting** (`apphosting.yaml`; project `studio-5054979338-10033`).

## Ship mechanism

This repo has **no GitHub Actions workflow** — there is no CI that deploys or publishes on push. It goes live through **Firebase App Hosting**, which builds and rolls out the Next.js app from the connected Git branch (`main`). Depending on how the App Hosting backend is configured in the Firebase console, a rollout is triggered either automatically on a push to the connected branch or manually (`firebase apphosting:rollouts:create` / a rollout from the console). There is **no npm publish** (`package.json` is `private`) and no GitHub Release step. Local verification uses the `package.json` scripts: `npm run dev` (`next dev` on port 9002), `npm run lint`, `npm run typecheck` (`tsc --noEmit`), and `npm run build` (`next build`).

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
