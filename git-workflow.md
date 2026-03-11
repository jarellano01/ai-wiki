# Git Workflow & GitHub PR Guide

A simple, effective branching strategy built around `main` as the single source of truth.

## Prerequisites

- **Git** installed (`git --version` to check, `brew install git` to install)
- **SSH key configured with GitHub** — required for private repos, recommended for all repos. Follow the [SSH Setup for GitHub](ssh-github-setup.md) guide first if you haven't done this yet.

## Repo Setup

### New project (no code yet)

**Using GitHub CLI:**

```bash
mkdir my-project && cd my-project   # Create a folder and move into it
git init                             # Turn this folder into a git repository
gh repo create my-project --public --source . --push
# ↑ Creates a public repo on GitHub, links it to this folder, and pushes your code up
```

**Manual approach:**

```bash
mkdir my-project && cd my-project   # Create a folder and move into it
git init                             # Turn this folder into a git repository
```

Then create the repo on GitHub:

1. Go to [github.com/new](https://github.com/new)
2. Name it `my-project`, set to Public, **don't** add a README (you already have local files)
3. Click **Create repository**

GitHub will show you the commands to connect your local repo — run them:

```bash
git remote add origin https://github.com/YOUR_USERNAME/my-project.git
# ↑ Tells your local repo where the GitHub repo lives (this link is called "origin")

git branch -M main
# ↑ Renames your default branch to "main" (some git versions default to "master")

git push -u origin main
# ↑ Uploads your commits to GitHub. The -u flag means future "git push" commands
#   will automatically push to this same place without you having to specify it again.
```

### Existing code you want to put on GitHub

You already have a folder with code in it and want to get it on GitHub.

**Using GitHub CLI:**

```bash
cd my-project                        # Move into your project folder
git init                             # Turn this folder into a git repository
git add .                            # Stage all files — tells git "I want to include these"
git commit -m "Initial commit"       # Save a snapshot of all staged files with a message
gh repo create my-project --public --source . --push
# ↑ Creates the GitHub repo, links it, and pushes your initial commit in one step
```

**Manual approach:**

```bash
cd my-project                        # Move into your project folder
git init                             # Turn this folder into a git repository
git add .                            # Stage all files — marks everything to be included in the next commit
git commit -m "Initial commit"       # Save a snapshot of your code. This is your first checkpoint.
```

Then create the repo on [github.com/new](https://github.com/new) (same steps as above — don't add a README), and connect it:

```bash
git remote add origin https://github.com/YOUR_USERNAME/my-project.git
# ↑ Links your local repo to the GitHub repo you just created

git branch -M main
# ↑ Ensures your branch is named "main"

git push -u origin main
# ↑ Pushes your code to GitHub. After this, your code is live on the repo page.
```

> **What just happened?** Your local code now has a full history (the commit) and a link to GitHub (the remote). From here on, `git push` sends your new commits to GitHub, and `git pull` downloads any changes from it.

### Private repo

Private repos require [SSH to be set up](ssh-github-setup.md) so git can authenticate without prompting for credentials.

**Using GitHub CLI:**

```bash
gh repo create my-project --private --source . --push
# ↑ Same as before, but only you (and people you invite) can see the repo.
#   gh handles authentication automatically.
```

**Manual approach:**

Select **Private** instead of Public when creating on [github.com/new](https://github.com/new). When connecting the remote, **use the SSH URL** (starts with `git@`) instead of HTTPS:

```bash
git remote add origin git@github.com:YOUR_USERNAME/my-project.git
# ↑ The SSH URL uses your SSH key to authenticate.
#   HTTPS URLs would require a personal access token for private repos.

git branch -M main
git push -u origin main
```

> **Already using HTTPS on a private repo?** Switch to SSH:
> ```bash
> git remote set-url origin git@github.com:YOUR_USERNAME/my-project.git
> ```

### Cloning an existing repo

Someone else already created the repo and you want a local copy to work on.

**Using GitHub CLI:**

```bash
gh repo clone owner/repo   # Downloads the repo and sets up the remote automatically
cd repo                     # Move into the downloaded folder
```

**Manual approach:**

```bash
git clone https://github.com/owner/repo.git
# ↑ Downloads the full repo (all files + full history) and sets "origin" to point back to GitHub

cd repo                     # Move into the downloaded folder
```

> After cloning, you're ready to work. The remote is already configured — `git push` and `git pull` work out of the box.

### After setup: add a .gitignore

Every repo should have one. A `.gitignore` file tells git which files to **not** track — things like installed dependencies, build output, and secret files. Without it, you'd accidentally commit thousands of files from `node_modules/` or expose your `.env` secrets.

GitHub maintains templates by language:

```bash
curl -sL https://raw.githubusercontent.com/github/gitignore/main/Node.gitignore > .gitignore
# ↑ Downloads a pre-made .gitignore for Node.js projects from GitHub's official templates

# Common ones: Node.gitignore, Python.gitignore, Go.gitignore, Rust.gitignore
```

Or create a minimal one:

```gitignore
node_modules/   # Installed packages (reinstalled with npm install)
.env            # Secret keys, API tokens — never commit these
.env.*          # .env.local, .env.production, etc.
dist/           # Build output (regenerated with npm run build)
.DS_Store       # macOS system file, not relevant to the project
```

> **Tip:** You can also add a `.gitignore` when creating a repo on GitHub — there's a dropdown to pick a template. If you do this on a repo that already has local commits, you'll need to `git pull origin main --rebase` before pushing.

## GitHub Repo Settings

Once your repo is on GitHub, configure it so that `main` is protected, PRs are required, and merges are clean. Do this **once** per repo.

### 1. Only allow squash merges

Squash merging combines all commits from a branch into a single commit on `main`. This keeps the main branch history clean and readable — one commit per feature/fix instead of dozens of "wip" commits.

**Using GitHub CLI:**

```bash
gh repo edit --enable-squash-merge --disable-merge-commit --disable-rebase-merge
# ↑ Turns on squash merge and disables the other two merge strategies.
#   This means the only way to merge a PR is via squash — no one can accidentally
#   do a regular merge that dumps all branch commits onto main.

gh repo edit --default-branch main
# ↑ Makes sure "main" is set as the default branch (PRs target this by default).
```

**Manual approach:**

1. Go to your repo on GitHub
2. **Settings → General** (scroll down to **Pull Requests**)
3. **Uncheck** "Allow merge commits"
4. **Uncheck** "Allow rebase merging"
5. **Check** "Allow squash merging"
6. Under the squash merge option, set the **default commit message** to:
   - **Default to pull request title and description**
   - This means when someone clicks "Squash and merge", the commit message will automatically use the PR title as the commit subject and the PR body as the commit description — no manual editing needed.
7. **Check** "Always suggest updating pull request branches"
   - This shows an "Update branch" button on PRs when they're behind `main`
8. **Check** "Automatically delete head branches"
   - Cleans up branches after merging so they don't pile up

### 2. Protect the main branch

Branch protection rules prevent anyone from pushing directly to `main` and ensure every change goes through a PR with checks.

**Using GitHub CLI:**

```bash
gh api repos/{owner}/{repo}/rulesets -X POST -f 'name=main-protection' \
  -f 'target=branch' -f 'enforcement=active' \
  --json 'conditions={"ref_name":{"include":["refs/heads/main"],"exclude":[]}}' \
  --json 'rules=[
    {"type":"pull_request","parameters":{"required_approving_review_count":1,"dismiss_stale_reviews_on_push":true,"require_last_push_approval":false}},
    {"type":"required_status_checks","parameters":{"strict_status_checks_policy":true,"required_status_checks":[{"context":"validate"}]}}
  ]'
# ↑ Creates a ruleset that requires PRs with at least 1 approval and passing CI checks.
#   "validate" should match the job name in your CI workflow file.
```

> The CLI approach for rulesets can be tricky. The manual approach below is more straightforward and gives you a visual overview of all settings.

**Manual approach:**

1. Go to your repo on GitHub
2. **Settings → Branches → Add branch protection rule** (or **Settings → Rules → Rulesets** on newer repos)
3. Branch name pattern: `main`
4. Enable each of these:

| Setting | What it does |
|---------|-------------|
| **Require a pull request before merging** | No one can `git push` directly to `main`. All changes must go through a PR. |
| **Require approvals → 1** | At least one teammate must review and approve the PR before the merge button is enabled. |
| **Dismiss stale pull request approvals when new commits are pushed** | If someone approves but then the author pushes more changes, the approval is reset — the reviewer needs to re-approve the updated code. |
| **Require status checks to pass before merging** | The merge button stays disabled until your CI workflow (lint, test, build) passes. Select your CI job name (e.g., `validate`) from the search box. |
| **Require branches to be up to date before merging** | The PR branch must include the latest `main` commits. If `main` moved forward since the PR was opened, the author has to update their branch first. This prevents "it works on my branch but breaks on main" scenarios. |

5. Click **Create** / **Save changes**

### 3. Verify your setup

After configuring, test that everything works:

```bash
# Try pushing directly to main — this should be rejected
git push origin main
# ↑ Should fail with "protected branch hook declined" or similar

# The correct flow: create a branch, open a PR
git checkout -b test/verify-setup
echo "test" > test.txt && git add test.txt && git commit -m "Test branch protection"
git push -u origin test/verify-setup
gh pr create --title "Test: verify branch protection" --body "Testing that PR checks work"
# ↑ The PR should show pending/running status checks.
#   The merge button should say "Review required" until someone approves.
```

Clean up the test PR when done:

```bash
gh pr close --delete-branch
```

## Branch Strategy

```
main (production)
 └── feature/my-feature
 └── fix/bug-description
 └── chore/cleanup-task
```

- **`main`** is always deployable
- Every change starts as a branch off `main`
- No long-lived branches — merge often, delete after merge

## Branch Naming

Use a prefix that describes the type of work:

| Prefix | Use for |
|--------|---------|
| `feature/` | New functionality |
| `fix/` | Bug fixes |
| `chore/` | Maintenance, deps, config |
| `refactor/` | Code restructuring (no behavior change) |
| `docs/` | Documentation only |

Examples: `feature/user-auth`, `fix/login-redirect`, `chore/upgrade-node`

## Daily Workflow

### 1. Start a branch

```bash
git checkout main
# ↑ Switch to the main branch so your new branch starts from the latest production code

git pull origin main
# ↑ Download the latest changes from GitHub — someone else may have merged since you last pulled

git checkout -b feature/my-feature
# ↑ Create a new branch AND switch to it. The -b flag means "create new".
#   You're now on your own isolated copy — changes here won't affect main until you merge.
```

### 2. Work in small commits

```bash
git add src/auth.ts src/auth.test.ts
# ↑ Stage specific files. Staging = telling git "include these in my next commit".
#   Prefer listing files explicitly over "git add ." to avoid committing something by accident.

git commit -m "Add JWT token validation"
# ↑ Save a snapshot of the staged files. This commit only exists on YOUR machine for now.

# Keep working...
git add src/middleware.ts
git commit -m "Add auth middleware to protected routes"
# ↑ Each commit is a checkpoint you can go back to if something breaks.
```

Write commits that explain **why**, not what. The diff shows what changed.

```bash
# Bad
git commit -m "Update auth.ts"

# Good
git commit -m "Validate JWT expiry to prevent stale session access"
```

### 3. Stay up to date with main

While you're working, other people may merge changes to `main`. Staying up to date avoids painful merge conflicts later.

```bash
git fetch origin
# ↑ Downloads new commits from GitHub but doesn't change your files yet.
#   Think of it as "check for updates" without installing them.

git rebase origin/main
# ↑ Replays YOUR commits on top of the latest main. This keeps your branch history
#   linear and clean, as if you just started your branch from the current main.
```

If you hit conflicts during rebase, git will pause and show you which files conflict:

```bash
# Open the conflicting files — look for <<<<<<< and >>>>>>> markers, pick the right code
git add <resolved-files>    # Mark the conflict as resolved
git rebase --continue       # Continue replaying the rest of your commits
```

### 4. Push your branch

```bash
git push -u origin feature/my-feature
# ↑ First push: uploads your branch to GitHub. The -u flag links your local branch
#   to the remote one, so future pushes just need "git push".

git push
# ↑ Subsequent pushes: sends any new commits since your last push.

git push --force-with-lease
# ↑ Use this ONLY after a rebase. Rebase rewrites your commit history, so git will
#   reject a normal push. --force-with-lease overwrites the remote branch BUT refuses
#   if someone else pushed to it since your last fetch (safer than --force).
```

## Pull Requests

A pull request (PR) is how you propose merging your branch into `main`. It gives your team a place to review the code, discuss changes, and run automated checks before anything hits production.

### Creating a PR

**Using GitHub CLI:**

```bash
gh pr create --title "Add user authentication" --body "$(cat <<'EOF'
## Summary
- Add JWT-based authentication
- Protect API routes with auth middleware
- Add login/logout endpoints

## Test plan
- [ ] Unit tests pass
- [ ] Manual test: login flow
- [ ] Manual test: protected route returns 401 without token
EOF
)"
# ↑ Creates a PR from your current branch targeting main.
#   The title shows up in the PR list, the body is the detailed description.
```

Or just `gh pr create` and fill in the interactive prompts.

**Manual approach:**

1. Push your branch (see step 4 above)
2. Go to your repo on GitHub — you'll see a banner saying "Compare & pull request" for your recently pushed branch
3. Click it, fill in a title and description, then click **Create pull request**

### PR Best Practices

- **Keep PRs small** — aim for under 400 lines changed. Smaller PRs get better reviews and merge faster.
- **One concern per PR** — don't mix a feature with an unrelated refactor.
- **Write a description** — reviewers shouldn't have to read every line to understand the intent.
- **Link issues** — use `Closes #123` in the PR body to auto-close issues on merge.
- **Add screenshots** for UI changes.

### Reviewing PRs

**Using GitHub CLI:**

```bash
gh pr list
# ↑ Shows all open PRs so you can find the one to review

gh pr checkout 42
# ↑ Downloads PR #42's branch to your machine so you can run and test the code locally

gh pr review 42 --approve
# ↑ Approve the PR — signals the code looks good to merge

gh pr review 42 --request-changes --body "Need to handle the null case in auth.ts:35"
# ↑ Request changes — blocks merging until the author addresses your feedback
```

**Manual approach:**

1. Go to the **Pull requests** tab on the repo page
2. Click into a PR to see the description, commits, and a diff of all changed files
3. Click **Review changes** (top right of the diff), leave comments, and select Approve or Request changes

### Merging

Use **squash merge** for feature branches to keep `main` history clean:

**Using GitHub CLI:**

```bash
gh pr merge 42 --squash --delete-branch
# ↑ --squash: collapses all your branch commits into ONE clean commit on main
#   --delete-branch: removes the branch since it's done (keeps the repo tidy)
```

**Manual approach:**

1. On the PR page, click the dropdown arrow next to the green **Merge** button
2. Select **Squash and merge**
3. Edit the commit message if needed, click **Confirm**
4. Click **Delete branch** when prompted

> **Why squash?** Your branch might have 15 messy commits ("fix typo", "wip", "actually fix it"). Squash merging combines them into one meaningful commit on `main`, keeping the history readable.

## CI/CD with GitHub Actions

GitHub Actions lets you run automated checks and deployments every time code is pushed or a PR is opened. The config lives in `.github/workflows/` as YAML files — GitHub picks them up automatically.

### How It Fits the Workflow

```
push to branch  →  CI runs validation (lint, test, typecheck, build)
create PR       →  Same checks run as PR status checks
                    Optional: deploy to staging
merge to main   →  Deploy to production
```

### PR Validation Workflow

Create `.github/workflows/ci.yml` — this runs on every push to `main` and on every PR targeting `main`:

```yaml
name: CI
# ↑ The name that shows up in the GitHub Actions tab

on:
  push:
    branches: [main]          # Run when code is pushed/merged to main
  pull_request:
    branches: [main]          # Run when a PR is opened or updated targeting main

jobs:
  validate:
    runs-on: ubuntu-latest    # Use a fresh Linux VM provided by GitHub (free for public repos)
    steps:
      - uses: actions/checkout@v4       # Download your code onto the VM

      - uses: actions/setup-node@v4     # Install Node.js
        with:
          node-version: 20
          cache: npm                    # Cache node_modules between runs for speed

      - run: npm ci             # Install dependencies (clean install, uses lockfile exactly)
      - run: npm run lint       # Check code style and catch common mistakes
      - run: npm run typecheck  # Verify TypeScript types are correct
      - run: npm run test       # Run your test suite
      - run: npm run build      # Make sure the project compiles without errors
```

Adapt the steps to your stack — the pattern is the same regardless of language:

| Stack | Typical checks |
|-------|---------------|
| Node/TS | `lint`, `typecheck`, `test`, `build` |
| Python | `ruff check`, `mypy`, `pytest` |
| Go | `go vet`, `go test ./...`, `go build` |
| Rust | `cargo clippy`, `cargo test`, `cargo build` |

> **Note:** Make sure you've completed the [GitHub Repo Settings](#github-repo-settings) section above — that's where branch protection, required reviews, and squash merge are configured.

### Deploy to Staging on PR

A staging deploy lets you preview the change in a real environment before merging. This workflow only runs on PRs (not on pushes to `main`):

```yaml
name: Deploy Staging

on:
  pull_request:
    branches: [main]          # Only PRs targeting main trigger this

jobs:
  validate:
    # ... same CI steps as above ...

  deploy-staging:
    needs: validate            # Only deploy if validation passes
    runs-on: ubuntu-latest
    environment: staging       # Links to a GitHub "environment" (see below)
    steps:
      - uses: actions/checkout@v4
      # Your deploy steps here — varies by platform:
      # Vercel, Railway, Fly.io, AWS, etc.
```

### Deploy to Production on Merge

When a PR is squash-merged, the resulting commit lands on `main`, which triggers this workflow:

```yaml
name: Deploy Production

on:
  push:
    branches: [main]          # Runs whenever main gets a new commit (i.e., after a merge)

jobs:
  validate:
    # ... same CI steps ...

  deploy:
    needs: validate            # Only deploy if validation passes
    runs-on: ubuntu-latest
    environment: production    # Links to the "production" GitHub environment
    steps:
      - uses: actions/checkout@v4
      # Your deploy steps here
```

The `environment` field ties into GitHub's [deployment environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment), which give you:
- **Environment-specific secrets** — different API keys for staging vs production
- **Required reviewers** — optional approval gate before prod deploy (someone has to click "approve" in the GitHub UI)
- **Deployment history** — see what's deployed and when on the repo page

## The Full Picture

```
1. git checkout -b feature/x        ← Create a branch off main
2. Write code, commit often          ← Small focused commits (checkpoints you can revert to)
3. git push -u origin feature/x      ← Upload your branch to GitHub
4. gh pr create (or use GitHub UI)   ← Open a PR — proposes merging your branch into main
5. CI runs automatically             ← GitHub Actions checks your code (lint, test, build)
6. Optional: deploys to staging      ← Preview the change in a real environment
7. Teammate reviews, approves        ← Code review catches issues before they hit production
8. Squash merge the PR               ← All your commits become one clean commit on main
9. CI runs on main                   ← Validates one more time after merge
10. Auto-deploy to production        ← Your change is live
```
