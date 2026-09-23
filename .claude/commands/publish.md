---
description: Security-scan the project, update the README and GitHub About section, set up GitHub Pages via Actions, and push to a GitHub repo
argument-hint: <owner/repo | https://github.com/owner/repo | git@github.com:owner/repo.git> [branch]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(grep:*), Bash(ls:*), Bash(cat:*), Bash(which:*), Bash(command -v:*), Bash(gitleaks:*), Read, Write, Edit, Glob, Grep
---

# Publish this project to GitHub

Target repo and optional branch: `$ARGUMENTS`

Work through the steps below in order. Stop and report to the user if any step fails. Never skip step 2.

## 1. Preflight

1. Parse the repo from `$ARGUMENTS`. Accept `owner/repo`, an HTTPS URL or an SSH URL, and normalise it to `owner/repo`. The branch is the second argument, defaulting to `main`. If no repo was given, ask the user for it (suggest the current `origin` from `git remote -v` if there is one) and stop until they answer.
2. Check that `gh` is installed (`command -v gh`) and signed in (`gh auth status`). If not, tell the user to run `brew install gh` and `gh auth login`, then stop.
3. Run `git status` and `git remote -v`. If `origin` points at a different repo than the one given, ask the user before changing it.
4. Check whether the repo exists with `gh repo view owner/repo`. If it doesn't, ask the user whether to create it (and whether public or private) with `gh repo create`. GitHub Pages on a free plan needs a public repo, so say so.

## 2. Security scan (must pass before anything leaves this machine)

Scan every file that will be pushed: tracked files plus anything new that will be committed (`git ls-files` and `git status --porcelain`). Check for:

- **Secrets.** If `gitleaks` is installed, run `gitleaks detect --source . --no-banner` (this also covers git history). Otherwise grep for API keys, tokens, private keys, passwords and connection strings. Useful patterns: `AKIA[0-9A-Z]{16}`, `ghp_`, `github_pat_`, `sk-`, `xox[baprs]-`, `-----BEGIN .*PRIVATE KEY-----`, `password\s*[:=]`, `secret\s*[:=]`, `token\s*[:=]`, `api[_-]?key`.
- **Personal data.** Email addresses, phone numbers and internal hostnames or IPs. In this project, `FORMSUBMIT_ENDPOINT` in `index.html` contains the recipient email address. Point out that anyone who views the page source will see it, and ask the user whether that is acceptable or whether to swap it for a FormSubmit random-string alias.
- **Files that shouldn't be published.** `.env*`, `*.pem`, `*.key`, `id_rsa*`, `*.sqlite`, `*.log`, `.DS_Store`, `node_modules/`, `.claude/settings.local.json`. Offer to add them to `.gitignore`.
- **Client-side code risks.** User values written into `innerHTML` without `escapeHtml()`, `eval`/`new Function`, `document.write`, and third-party `<script src>` without `integrity`. Also check that external calls use `https://` only.

Report the findings as a short table (severity, file:line, issue, suggested fix). If you find a **critical or high** issue, such as a real secret, stop: do not commit or push until the user has fixed it or explicitly accepted the risk. If a secret was ever committed, tell the user to rotate it, because removing it from the latest commit does not remove it from history.

## 3. README

Read `index.html` and `CLAUDE.md` so the README describes the project as it actually is. Create `README.md`, or update it if it exists and keep any sections the user wrote. Include:

- Title and a one-paragraph description
- Live demo link: `https://<owner>.github.io/<repo>/` (lower-case owner)
- Features (drag and drop plus keyboard moves, filters, summary, validation, email notification, accessibility)
- How to run it locally (open `index.html`, or `python3 -m http.server`)
- Configuration: where `FORMSUBMIT_ENDPOINT` is set and the one-time FormSubmit activation step
- Deployment: GitHub Pages via the Actions workflow
- Project structure (single `index.html`, no build step)

Don't invent features, badges or licences that aren't there.

## 4. GitHub Pages workflow

Create or update `.github/workflows/pages.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [<branch>]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - name: Prepare site
        run: |
          mkdir -p _site
          cp index.html _site/
          touch _site/.nojekyll
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site
      - id: deployment
        uses: actions/deploy-pages@v4
```

Only publish the site files, not `CLAUDE.md`, `.claude/` or other repo files. If more static assets are added later, copy them into `_site` as well.

## 5. Commit and push

1. Show the user `git status` and `git diff --stat`, then stage the files by name. Don't use `git add -A`.
2. Commit with a clear message.
3. Set or confirm `origin` (step 1.3) and run `git push -u origin <branch>`. Never force-push without explicit permission.

## 6. Enable Pages and set the About section

1. Switch Pages to build from Actions:
   `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow`
   If Pages already exists, use `-X PUT` with the same field instead.
2. Update the About section. Write a one-line description from the README, set the homepage to the Pages URL, and add relevant topics:
   `gh repo edit <owner>/<repo> --description "..." --homepage "https://<owner>.github.io/<repo>/" --add-topic kanban --add-topic html --add-topic javascript --add-topic github-pages`
3. Check the workflow run with `gh run list --workflow=pages.yml --limit 1`, and use `gh run watch` if it's still running.

## 7. Report

Finish with a short summary covering:
- The security scan result and anything the user accepted
- The files created or changed
- The commit SHA and branch pushed
- The About description, homepage and topics set
- The Pages URL and the workflow run status
