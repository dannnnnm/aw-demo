---
on:
  schedule: weekly

permissions:
  contents: read
  pull-requests: read

safe-outputs:
  create-pull-request:
    allowed-branches: [docs/automation]
    title-prefix: "[docs] "
    draft: true

engine: claude
---

# Documentation Updater

Review code and documentation changes from the last seven days.

Identify outdated setup steps, missing functionality descriptions, and examples that no longer match current behavior. Update or create the relevant documentation files and open a draft pull request describing the changes and any areas that still require human review.