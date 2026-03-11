# Automating GitHub Issues with Claude Code

Use Claude Code as an AI agent that triages issues and auto-implements them as PRs — triggered by comments on GitHub issues.

## What You Get

- **`@claude triage`** on any issue — Claude analyzes the codebase, posts a diagnostic summary, and labels the issue as ready for dev
- **`@claude implement`** on any issue — Claude reads the issue, explores the codebase, writes the code, runs checks, and opens a PR

Both are triggered by GitHub Actions workflows that run the [Claude Code Action](https://github.com/anthropics/claude-code-action).

## How It Works

```
Issue opened
  │
  ├─ Someone comments "@claude triage"
  │    → GitHub Actions triggers a workflow
  │    → Claude reads the issue + explores the codebase
  │    → Posts a triage comment (affected files, complexity, approach)
  │    → Applies "Ready For Dev" label if issue is clear enough
  │
  └─ Someone comments "@claude implement"
       → GitHub Actions triggers a workflow
       → Claude reads the issue, explores the codebase, writes the code
       → Runs your project's checks (lint, test, build, etc.)
       → Creates a branch, commits, pushes, and opens a PR
       → Links the PR back to the issue
```

## Prerequisites

- A GitHub repo with [GitHub Actions](https://docs.github.com/en/actions) enabled
- A Claude Code OAuth token (see [Authentication](#1-authentication) below)
- A `CLAUDE.md` file in your repo root (gives Claude context about your project)

## Setup

### 1. Authentication

The Claude Code Action needs a token to authenticate. You have two options:

**Option A: OAuth Token (Recommended)**

Generate a token from your Claude Code CLI:

```bash
claude oauth generate
# ↑ Outputs a token. Copy it.
```

Then add it as a GitHub secret:

1. Go to your repo on GitHub
2. **Settings → Secrets and variables → Actions → New repository secret**
3. Name: `CLAUDE_CODE_OAUTH_TOKEN`
4. Value: paste the token
5. Click **Add secret**

**Option B: API Key**

If you have an Anthropic API key, add it as a secret named `ANTHROPIC_API_KEY` instead. In the workflow files below, replace `claude_code_oauth_token` with `anthropic_api_key`.

### 2. Add CLAUDE.md

Create a `CLAUDE.md` file in your repo root. This is how you give Claude context about your project — it reads this file before doing any work. Think of it as onboarding docs for an AI teammate.

```markdown
# Project Name

## Architecture
- Brief description of your stack (e.g., "Next.js frontend, Express API, PostgreSQL")
- Package/directory structure

## Development Commands
- How to install dependencies (e.g., `npm install`)
- How to run checks (e.g., `npm run lint && npm test && npm run build`)
- How to start the dev server

## Code Patterns
- Naming conventions
- File organization patterns
- Any code generation steps (e.g., GraphQL codegen, ORM migrations)

## Important Notes
- Anything Claude should know before making changes
```

The more context you provide, the better Claude's triage and implementation will be. Include things like:
- Which directories contain what (e.g., "API routes are in `src/routes/`")
- Naming conventions (e.g., "Test files use `*.test.ts` suffix")
- Required steps after certain changes (e.g., "Run `npm run codegen` after modifying GraphQL schema")

### 3. Create the Triage Workflow

Create `.github/workflows/issue-triage.yml`:

```yaml
name: Issue Triage

on:
  issue_comment:
    types: [created]          # Fires when someone comments on an issue
  issues:
    types: [opened]           # Fires when a new issue is created

jobs:
  triage:
    # Only run on issues (not PRs), when a human (not a bot) mentions @claude triage
    if: |
      !github.event.issue.pull_request &&
      (
        (github.event_name == 'issue_comment' && github.event.comment.user.type != 'Bot' && contains(github.event.comment.body, '@claude triage')) ||
        (github.event_name == 'issues' && contains(github.event.issue.body, '@claude triage'))
      )
    runs-on: ubuntu-latest
    permissions:
      contents: read            # Read the codebase
      issues: write             # Post comments and add labels
      id-token: write           # Required for Claude Code OAuth
    steps:
      # React with 👀 so the user knows the workflow was triggered
      - name: Add eyes reaction
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            if (context.eventName === 'issue_comment') {
              await github.rest.reactions.createForIssueComment({
                owner, repo,
                comment_id: context.payload.comment.id,
                content: 'eyes'
              });
            } else {
              await github.rest.reactions.createForIssue({
                owner, repo,
                issue_number: context.payload.issue.number,
                content: 'eyes'
              });
            }

      # Post a comment linking to the workflow run so the user can watch progress
      - name: Post status comment
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            const runUrl = `${context.serverUrl}/${owner}/${repo}/actions/runs/${context.runId}`;
            await github.rest.issues.createComment({
              owner, repo,
              issue_number: context.payload.issue.number,
              body: `## Triage Started\n\nClaude is analyzing this issue.\n\n**[View progress →](${runUrl})**`
            });

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1        # Shallow clone is fine for triage (read-only)

      # This is where Claude does the actual work
      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # Restrict tools to read-only operations + issue management
          claude_args: |
            --allowedTools "Bash(gh issue view:*),Bash(gh issue comment:*),Bash(gh issue edit:*),Bash(git log:*),Bash(git diff:*),Bash(git show:*)"
          prompt: |
            Run /triage ${{ github.event.issue.number }}

      # React with 🚀 on success so the user knows it's done
      - name: Add success reaction
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            if (context.eventName === 'issue_comment') {
              await github.rest.reactions.createForIssueComment({
                owner, repo,
                comment_id: context.payload.comment.id,
                content: 'rocket'
              });
            } else {
              await github.rest.reactions.createForIssue({
                owner, repo,
                issue_number: context.payload.issue.number,
                content: 'rocket'
              });
            }
```

### 4. Create the Implement Workflow

Create `.github/workflows/issue-implement.yml`:

```yaml
name: Issue Auto-Implement

on:
  issue_comment:
    types: [created]

jobs:
  implement:
    # Only run on issues (not PRs), when a human mentions @claude implement
    if: |
      !github.event.issue.pull_request &&
      github.event.comment.user.type != 'Bot' &&
      contains(github.event.comment.body, '@claude implement')
    runs-on: ubuntu-latest
    permissions:
      contents: write           # Push branches and commits
      issues: write             # Comment on issues
      pull-requests: write      # Create PRs
      id-token: write           # Required for Claude Code OAuth
    steps:
      - name: Add eyes reaction
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            await github.rest.reactions.createForIssueComment({
              owner, repo,
              comment_id: context.payload.comment.id,
              content: 'eyes'
            });

      - name: Post status comment
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            const runUrl = `${context.serverUrl}/${owner}/${repo}/actions/runs/${context.runId}`;
            await github.rest.issues.createComment({
              owner, repo,
              issue_number: context.payload.issue.number,
              body: `## Auto-Implementation Started\n\nClaude is working on implementing this issue.\n\n**[View progress →](${runUrl})**`
            });

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0          # Full history needed for branching and git operations
          token: ${{ secrets.GITHUB_TOKEN }}

      # --- Customize this section for your project's runtime ---
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci
      # --- End customization section ---

      # Configure git so Claude can commit and push
      - name: Configure Git
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          claude_args: |
            --model claude-opus-4-6
            --allowedTools "Edit,Write,Read,Glob,Grep,Bash(gh issue:*),Bash(gh pr:*),Bash(git:*),Bash(npm:*)"
          prompt: |
            Run /implement ${{ github.event.issue.number }}

      # On success: react and label the PR
      - name: Add success reaction and label PR
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;

            await github.rest.reactions.createForIssueComment({
              owner, repo,
              comment_id: context.payload.comment.id,
              content: 'rocket'
            });

            // Find the PR that was just created (most recent, linking to this issue)
            const pulls = await github.rest.pulls.list({
              owner, repo, state: 'open', sort: 'created', direction: 'desc', per_page: 5
            });
            const issuePR = pulls.data.find(pr =>
              pr.body && pr.body.includes(`#${context.payload.issue.number}`)
            );
            if (issuePR) {
              await github.rest.issues.addLabels({
                owner, repo,
                issue_number: issuePR.number,
                labels: ['claude-implemented']
              });
            }

      # On failure: react and comment so the user knows something went wrong
      - name: Handle failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            await github.rest.reactions.createForIssueComment({
              owner, repo,
              comment_id: context.payload.comment.id,
              content: 'confused'
            });
            const runUrl = `${context.serverUrl}/${owner}/${repo}/actions/runs/${context.runId}`;
            await github.rest.issues.createComment({
              owner, repo,
              issue_number: context.payload.issue.number,
              body: `## Auto-Implementation Failed\n\nClaude encountered an error. Check the [workflow run](${runUrl}) for details.`
            });
```

### 5. Create the Triage Skill

Skills tell Claude *how* to perform a task. They're markdown files with a YAML frontmatter header.

Create `.claude/skills/issue-triage/SKILL.md`:

```markdown
---
name: triage
description: Triage a GitHub issue with code-level diagnostics.
---

# Issue Triage

Analyze a GitHub issue and provide code-level diagnostics.
**This skill is for information gathering ONLY — never implement or write code changes.**

## Workflow

### 1. Check for Existing Triage

Look for previous triage comments on the issue:

\`\`\`bash
gh issue view <issue-number> --json comments --jq '.comments[] | select(.body | contains("## Triage:")) | .body'
\`\`\`

If already triaged with no new information, apply "Ready For Dev" label and stop.
If new information exists since last triage, post an updated triage.

### 2. Understand the Issue

Identify:
- **Problem**: What is broken or needs to be added?
- **Expected vs Actual**: What should happen vs what is happening?
- **Affected Area**: Which part of the app?
- **Reproduction**: Steps or conditions mentioned?
- **Acceptance Criteria**: What defines "done"?

### 3. Search the Codebase

Use the Explore agent to find relevant code:
- Search for keywords from the issue (component names, error messages)
- Find related files, functions, or patterns
- Check recent git history for changes to affected areas

### 4. Post Triage Summary

Post a comment with findings:

\`\`\`bash
gh issue comment <issue-number> --body "$(cat <<'TRIAGEEOF'
## Triage: #<number> - <issue title>

### Problem Summary
<1-2 sentence summary>

### Affected Files
- \`path/to/file.ts:line\` - <what this file does>

### Analysis
<Current code behavior and what needs to change>

### Implementation Approach
<Brief description — NOT actual code>

### Complexity
<Low/Medium/High> - <reasoning>

### Ready for Implementation?
<Yes/No>

To auto-implement, comment: \`@claude implement\`
TRIAGEEOF
)"
\`\`\`

### 5. Apply Label

If ready for implementation:

\`\`\`bash
gh issue edit <issue-number> --add-label "Ready For Dev"
\`\`\`

## Rules

- **NEVER implement** — information gathering only
- **Be specific** — include file paths and line numbers
- **Ask questions** if information is missing rather than guessing
```

Customize the "Search the Codebase" and "Affected Files" sections to match your project's structure. For example, if you have a Rails app, you'd reference `app/controllers/`, `app/models/`, etc.

### 6. Create the Implement Skill

Create `.claude/skills/implement-issue/SKILL.md`:

```markdown
---
name: implement
description: Auto-implement a GitHub issue end-to-end.
argument-hint: <issue-number>
---

# Auto-Implement GitHub Issue

Implement a GitHub issue and create a PR — fully autonomous, no confirmation needed.

## Arguments

- `$ARGUMENTS[0]` - Issue number or GitHub issue URL

## Workflow

### 1. Ensure Clean State

\`\`\`bash
git checkout main
git pull origin main
\`\`\`

### 2. Fetch Issue Details

\`\`\`bash
gh issue view <issue-number> --json number,title,body,labels,state
\`\`\`

### 3. Assess Readiness

Check that the issue has:
- [ ] Clear problem statement or feature description
- [ ] Expected behavior or outcome
- [ ] Specific area of the codebase affected

If NOT enough information, post a comment asking for clarification and **stop**.

### 4. Explore and Plan

Use the Explore agent to find relevant code. Create a mental plan:
- Which files need changes
- What the changes are
- Existing patterns to follow

### 5. Implement

Make the code changes, then run your project's checks:

\`\`\`bash
npm run lint
npm test
npm run build
\`\`\`

Fix any failures before proceeding.

### 6. Create Branch and Commit

\`\`\`bash
git checkout -b feat/issue-<number>-<kebab-case-summary>
git add .
git commit -m "feat: <description>

Closes #<issue-number>"
\`\`\`

### 7. Push and Create PR

\`\`\`bash
git push -u origin <branch-name>

gh pr create --title "feat: <description>" --body "$(cat <<'EOFPR'
## Summary
<bullet points describing what this PR does>

Closes #<issue-number>

## Changes
<files changed and what was modified>

## Test Plan
- [ ] <specific test step>
EOFPR
)"
\`\`\`

### 8. Label and Comment

\`\`\`bash
gh pr edit <pr-number> --add-label "claude-implemented" --add-label "ready-for-review"

gh issue comment <issue-number> --body "Implementation complete — see PR #<pr-number>."
\`\`\`

## Rules

- **Be autonomous** — don't ask for confirmation
- **Quality first** — never skip checks; fix issues before creating PR
- **Link everything** — reference the issue in commits, PR, and comments
```

Customize the check commands (`npm run lint`, `npm test`, etc.) to match your project.

## Customization Guide

### Changing the Trigger Keywords

In the workflow YAML, change what comment text triggers the workflow:

```yaml
# Default
contains(github.event.comment.body, '@claude triage')

# Custom examples
contains(github.event.comment.body, '/triage')         # Slash command style
contains(github.event.comment.body, 'claude please triage')
```

### Restricting Who Can Trigger

Add a check for team membership or specific users:

```yaml
# Only allow org members
if: |
  !github.event.issue.pull_request &&
  github.event.comment.author_association == 'MEMBER' &&
  contains(github.event.comment.body, '@claude implement')

# Only specific users
if: |
  !github.event.issue.pull_request &&
  (github.event.comment.user.login == 'your-username' || github.event.comment.user.login == 'teammate') &&
  contains(github.event.comment.body, '@claude implement')
```

`author_association` values: `OWNER`, `MEMBER`, `COLLABORATOR`, `CONTRIBUTOR`, `NONE`

### Changing the Model

In the implement workflow, the model is set in `claude_args`:

```yaml
claude_args: |
  --model claude-opus-4-6           # Best quality, slower, more expensive
  --model claude-sonnet-4-6         # Good balance of quality and speed
```

Triage doesn't specify a model (uses the default), which is fine since it's read-only analysis.

### Adjusting Allowed Tools

The `--allowedTools` flag controls what Claude can do. Be restrictive for safety:

```yaml
# Triage: read-only (can't edit files or push code)
--allowedTools "Bash(gh issue view:*),Bash(gh issue comment:*),Bash(gh issue edit:*),Bash(git log:*),Bash(git diff:*),Bash(git show:*)"

# Implement: full access to edit, write, and run project commands
--allowedTools "Edit,Write,Read,Glob,Grep,Bash(gh issue:*),Bash(gh pr:*),Bash(git:*),Bash(npm:*)"
```

The pattern `Bash(command:*)` means Claude can run that command with any arguments. You can be more specific:

```yaml
# Only allow npm run, not npm install or npm publish
--allowedTools "Bash(npm run:*)"
```

### Adding Your Project's Build Steps

The implement workflow needs your project's runtime. Customize the setup section:

**Node.js (npm):**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: npm
- run: npm ci
```

**Node.js (pnpm):**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '22'
- uses: pnpm/action-setup@v4
- run: pnpm install --frozen-lockfile
```

**Python:**
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
- run: pip install -r requirements.txt
```

**Go:**
```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
```

Update the allowed tools to match your package manager:

```yaml
# npm
--allowedTools "...,Bash(npm:*)"

# pnpm
--allowedTools "...,Bash(pnpm:*)"

# Python
--allowedTools "...,Bash(pip:*),Bash(python:*),Bash(pytest:*)"

# Go
--allowedTools "...,Bash(go:*)"
```

### Adding Auto-Triage on Issue Creation

To automatically triage every new issue without needing `@claude triage`:

```yaml
on:
  issues:
    types: [opened]

jobs:
  triage:
    # Remove the comment-body check — just run on every new issue
    if: "!github.event.issue.pull_request"
    # ... rest of the workflow
```

### Adding More Skills

Skills are just markdown files in `.claude/skills/`. The directory structure:

```
.claude/
  skills/
    issue-triage/
      SKILL.md          # Triage skill
    implement-issue/
      SKILL.md          # Implement skill
    your-new-skill/
      SKILL.md          # Add more skills here
```

Each skill needs a YAML frontmatter with at least `name` and `description`:

```markdown
---
name: my-skill
description: What this skill does. Use when user says "trigger phrase".
---

# Skill Title

Instructions for Claude...
```

Skills are invoked with `/skill-name` in the prompt. The workflow triggers them with:

```yaml
prompt: |
  Run /skill-name ${{ github.event.issue.number }}
```

## File Structure Overview

After setup, your repo will have these new files:

```
.github/
  workflows/
    issue-triage.yml          # Workflow: triggers on @claude triage
    issue-implement.yml       # Workflow: triggers on @claude implement
.claude/
  skills/
    issue-triage/
      SKILL.md                # Skill: how to triage an issue
    implement-issue/
      SKILL.md                # Skill: how to implement an issue
CLAUDE.md                     # Project context for Claude
```

## Usage

Once everything is committed and pushed:

1. **Create an issue** describing a bug or feature
2. **Comment `@claude triage`** — Claude analyzes the issue and posts findings
3. **Comment `@claude implement`** — Claude implements the fix and opens a PR
4. **Review the PR** — Claude's code goes through your normal review process
5. **Merge** — squash merge as usual

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Workflow doesn't trigger | Check that the comment contains exactly `@claude triage` or `@claude implement`. Bot comments are ignored by design. |
| "Resource not accessible by integration" | Check the `permissions` block in the workflow YAML. Implement needs `contents: write` and `pull-requests: write`. |
| Claude doesn't know about your project | Improve your `CLAUDE.md` — add directory structure, naming conventions, and common patterns. |
| Implementation fails on checks | The skill tells Claude to run checks and fix failures. If it still fails, the issue may be too complex for auto-implementation. |
| PR doesn't link to issue | Make sure the skill includes `Closes #<number>` in the commit message and PR body. |
| OAuth token expired | Generate a new one with `claude oauth generate` and update the GitHub secret. |
