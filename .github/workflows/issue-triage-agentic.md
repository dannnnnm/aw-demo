---
emoji: 🧭
description: Triage newly opened issues with labeling, duplicate detection, clarification, and owner assignment.
on:
  issues:
    types: [opened]
permissions:
  contents: read
  issues: read
  pull-requests: read
tools:
  github:
    mode: gh-proxy
    toolsets: [default]
steps:
  - name: Prefetch triage context
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      REPO: ${{ github.repository }}
      ISSUE_NUMBER: ${{ github.event.issue.number }}
    run: |
      mkdir -p /tmp/gh-aw/data
      gh api "repos/$REPO/labels?per_page=100" \
        --jq '[.[] | {name, description}]' \
        > /tmp/gh-aw/data/labels.json
      gh api "repos/$REPO/issues?state=open&per_page=100" \
        --jq '[.[] | select(.pull_request|not) | {number, title, body, labels:[.labels[].name], assignees:[.assignees[].login], created_at, updated_at}]' \
        > /tmp/gh-aw/data/open_issues.json
      gh api "repos/$REPO/collaborators?per_page=100" \
        --jq '[.[] | select((.type // "") != "Bot") | {login, permissions}]' \
        > /tmp/gh-aw/data/collaborators.json
safe-outputs:
  add-labels:
    max: 3
  add-comment:
    max: 1
  assign-to-user:
    max: 1
  close-issue:
    max: 1
    state-reason: [duplicate]

models:
  providers:
    github-copilot:
      models:
        'auto':
          cost:
            # Placeholder values — replace with actual pricing for this model
            input: "0e0"      # $0.00 per million input tokens
            output: "0e0"     # $0.00 per million output tokens
            # cache_read: "0e0"  # $0.00 per million cache-read tokens
            # cache_write: "0e0" # $0.00 per million cache-write tokens
---

# Issue Triage Agentic Workflow

## Task

You are triaging a newly opened issue in this repository.

Use the triggering issue plus:

- `/tmp/gh-aw/data/labels.json`
- `/tmp/gh-aw/data/open_issues.json`
- `/tmp/gh-aw/data/collaborators.json`

Complete triage in this order:

1. **Detect duplicates first**
   - Compare the triggering issue against open issues (excluding itself) for same root problem, not just keyword overlap.
   - If you find a likely duplicate, do all of the following:
     - Add a duplicate label (`duplicate` when available, otherwise closest equivalent).
     - Add one short comment linking to the canonical issue number and explaining why it matches.
     - Close the triggering issue as duplicate using `close-issue` with `duplicate_of`.
     - Stop after duplicate handling.

2. **Assess clarity**
   - If the issue description is missing key reproduction/context details, add a short clarifying comment with specific questions needed to proceed.
   - Add a needs-information label (`needs-info` when available, otherwise closest equivalent).
   - Continue with type/priority labeling and assignment even when questions are asked.

3. **Apply exactly one type label**
   - Choose the best-fit type label from existing labels when possible.
   - Preferred canonical names: `type:bug`, `type:feature`, `type:question`, `type:documentation`, `type:maintenance`.
   - If canonical names are unavailable, use the closest existing type/kind/category label.

4. **Apply exactly one priority label**
   - Preferred canonical names: `priority:p0`, `priority:p1`, `priority:p2`, `priority:p3`.
   - Map urgency/impact:
     - `p0`: production down, security incident, or total blocker
     - `p1`: high user/business impact, urgent
     - `p2`: normal priority
     - `p3`: low impact or enhancement backlog
   - If canonical names are unavailable, use closest existing priority/severity labels.

5. **Assign the issue to the right team member**
   - Choose exactly one assignee from `/tmp/gh-aw/data/collaborators.json`.
   - Prefer a collaborator explicitly mentioned by the reporter.
   - Otherwise choose the collaborator whose role best matches issue type (for example docs/tooling/product areas inferred from issue text).
   - If no clear specialist is evident, assign a maintainer-level collaborator (`permissions.push == true` or `permissions.admin == true`) as default triage owner.

## Safe Outputs

- Use only configured safe outputs for labels, comments, assignment, and duplicate closure.
- Use `noop` with a short reason if no changes are needed.
