---
description: Deploy this project to GitHub Pages (checks prerequisites, creates the repo, deploys, prints the URL)
argument-hint: [repo-name]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(npm:*), Bash(node:*), Bash(curl:*), Read, Write, Edit, Glob, Grep
---

Deploy this local Vite + React project to GitHub Pages using GitHub Actions. Optional repo name: `$ARGUMENTS`

Work through the steps below in order. Every step must be safe to re-run: check the current state first and skip anything that's already done, so a second run just redeploys. Tell the user briefly what you're checking at each step. When something needs them to act, say exactly what to do, then wait.

Project facts:
- `npm run build` outputs to `build/` (not `dist/`).
- `vite.config.ts` sets no `base`. Do **not** edit it. The base path is passed on the build command line instead, so `npm run dev` keeps working at `/`.

## 1. Check the tools
Run `git --version`, `gh --version` and `node --version`. If any is missing, tell the user how to install it, then stop:
- git: https://git-scm.com/downloads or `winget install Git.Git`
- GitHub CLI: https://cli.github.com or `winget install GitHub.cli` / `brew install gh`
- Node.js 20+: https://nodejs.org

## 2. Check the GitHub login
- Run `gh auth status`.
  - If not logged in: tell the user to run `! gh auth login` in the prompt. Suggest GitHub.com, HTTPS and login with a web browser. Wait until they're done, then run `gh auth status` again.
  - If the token's scopes don't include `workflow` (needed to push `.github/workflows/`): tell the user to run `! gh auth refresh -h github.com -s workflow`, and wait.
- Run `gh auth setup-git` so `git push` uses the gh credentials.
- Get the username with `gh api user --jq .login`. Call it `<user>` below.

## 3. Pick the repo name
- Use `$ARGUMENTS` if it's not empty.
- Otherwise, if `git remote get-url origin` already points at a GitHub repo, use that repo's name.
- Otherwise suggest the current folder name and confirm it with the user.

Call it `<repo>` below. Work out the base path:
- `/` if `<repo>` is `<user>.github.io`
- `/<repo>/` otherwise.

## 4. Set up local git
- If `git rev-parse --is-inside-work-tree` fails, run `git init` and then `git branch -M main`.
- Make sure `.gitignore` contains `node_modules` and `build`. Add any that are missing.
- Check `git config user.name` and `git config user.email`. If either is empty, ask the user for the values and set them, locally to this repo.

## 5. Create the GitHub repo
- Run `gh repo view <user>/<repo>`.
- If the repo doesn't exist:
  1. Tell the user it will be created as a **public** repo. GitHub Pages on a free account needs a public repo.
  2. Get their confirmation.
  3. Run `gh repo create <repo> --public --source=. --remote=origin`.
- If the repo exists but there's no `origin` remote, run `git remote add origin https://github.com/<user>/<repo>.git`.
- If `origin` points at a different repo, stop and ask the user what to do.

## 6. Turn on GitHub Pages
- Run `gh api repos/<user>/<repo>/pages`. If it succeeds and `build_type` is `workflow`, skip ahead to step 7.
  - If Pages exists with a different `build_type`, run `gh api -X PUT repos/<user>/<repo>/pages -f build_type=workflow`.
  - If Pages doesn't exist (404), run `gh api -X POST repos/<user>/<repo>/pages -f build_type=workflow`.
- If the API refuses, for example because the repo has no commits yet, carry on. Try this step again after the push in step 9.
- If it still fails after the push, tell the user to open `https://github.com/<user>/<repo>/settings/pages` and set **Source** to **GitHub Actions**.

## 7. Write the deploy workflow
Create or update `.github/workflows/deploy.yml` with the content below. Replace `<BASE>` with the base path from step 3.

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build -- --base=<BASE>
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: build
      - id: deployment
        uses: actions/deploy-pages@v4
```

## 8. Build locally first
- If `node_modules` is missing, run `npm install`.
- Run `npm run build -- --base=<BASE>`.
- If the build fails, show the error, stop, and don't push.

## 9. Commit and push
- Run `git add -A`, then `git status --short`, and show the user what will be committed.
- Confirm with the user before pushing.
- Commit with a message like `Deploy to GitHub Pages`. If there's nothing new to commit, skip the commit.
- Run `git push -u origin main`.
- If step 6 didn't succeed earlier, do it now.

## 10. Watch the deploy
- Get the latest run ID with `gh run list --workflow=deploy.yml --limit 1 --json databaseId,status,conclusion`.
  - If nothing new was pushed, or the run started before Pages was enabled and failed, start a fresh run with `gh workflow run deploy.yml`. Wait a few seconds, then list the runs again.
- Follow the run with `gh run watch <id> --exit-status`.
- If the run fails:
  1. Run `gh run view <id> --log-failed`.
  2. Explain the cause, fix it if the problem is in this repo, then push again and re-run.
  3. Don't loop more than twice. After that, hand the problem to the user.

## 11. Check the site and report
- Check that the site responds with `curl -sI https://<user>.github.io<BASE>`.
  - Pages can take a minute or so to come up. Retry a few times with short pauses before deciding it's broken.
- End with a short summary that includes:
  - **Repository:** `https://github.com/<user>/<repo>`
  - **Live game:** `https://<user>.github.io<BASE>`
  - **Actions runs:** `https://github.com/<user>/<repo>/actions`
- Remind the user that next time they can run `/deploy_github_page` again, or just `git push` to `main`.
