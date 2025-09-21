# STA E2E Shared Actions

Centralized GitHub Actions for STA E2E repositories. This repository contains reusable composite actions that can be used across multiple repositories in the organization.

### E2E Repos

Find STA E2E repos in AEMDEMOS: https://github.com/search?q=org%3Aaemdemos+sta+e2e&type=repositories

_**NOTE**_ - other STA E2E tests have been created in other orgs as well 🤷.

## Available Actions

### 🧹 E2E Issue Branch Cleanup

**Path:** `.github/actions/e2e-issue-branch-cleanup`

Automatically performs two cleanup operations:
1. **Issue Branches**: Deletes branches matching the `issue-<number>` pattern (e.g., `issue-123`, `issue-4567`) 
2. **Old Backup Branches**: Deletes backup branches older than 1 month matching the `backup-YYYY-MM-DD-HH-MM` pattern

#### Usage

Add this workflow to your other E2E repository at `.github/workflows/nightly-cleanup.yaml`:

```yaml
name: Nightly E2E and Backup Branch Cleanup
on:
  schedule:
    - cron: '0 23 * * *'  # Every night at 11 PM UTC
  workflow_dispatch:      # Allow manual trigger

permissions:
  contents: write   # 👈 this is required to delete branches

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout this repository
        uses: actions/checkout@v4
      
      - name: Checkout shared actions
        uses: actions/checkout@v4
        with:
          repository: aemdemos/sta-e2e-shared
          path: .shared-actions
          
      - name: Run E2E branch cleanup
        uses: ./.shared-actions/.github/actions/e2e-issue-branch-cleanup
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `github-token` | GitHub token with repo access | Yes | `${{ github.token }}` |

#### What Gets Cleaned Up

**Issue Branches (fixed pattern):**
- Pattern: `issue-123`, `issue-4567`, etc. (exactly `issue-<number>`)

**Backup Branches (automatic):**
- Pattern: `backup-YYYY-MM-DD-HH-MM` (e.g., `backup-2025-01-15-14-30`)
- Only deletes branches older than 1 month

## How to Use Shared Actions in General

### General Pattern

1. **Checkout your repository** (establishes context)
2. **Checkout this shared actions repository** 
3. **Use the specific action** from the checked-out path

```yaml
steps:
  - name: Checkout this repository
    uses: actions/checkout@v4
  
  - name: Checkout shared actions
    uses: actions/checkout@v4
    with:
      repository: aemdemos/sta-e2e-shared
      path: .shared-actions
      
  - name: Use shared action
    uses: ./.shared-actions/.github/actions/[ACTION_NAME]
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      # Example of additional parameters that future actions might need:
      param1: 'value1'
      param2: 'value2'
      param3: true
      param4: 123
```

### Benefits

✅ **Centralized Logic**: Maintain action code in one place  
✅ **No Cross-Repo Tokens**: Each repo uses its own `GITHUB_TOKEN`  
✅ **Easy Updates**: Update logic once, all repos benefit  
✅ **Version Control**: Pin to specific commits/tags if needed  
✅ **Consistent Behavior**: Same logic across all repositories  

### Advanced Usage

**Pin to a specific version:**

```yaml
- name: Checkout shared actions
  uses: actions/checkout@v4
  with:
    repository: aemdemos/sta-e2e-shared
    ref: v1.0.0  # Pin to specific tag
    path: .shared-actions
```

**Use in matrix builds:**

```yaml
strategy:
  matrix:
    environment: [dev, staging, prod]
steps:
  - name: Checkout shared actions
    uses: actions/checkout@v4
    with:
      repository: aemdemos/sta-e2e-shared
      path: .shared-actions
      
  - name: Environment-specific cleanup
    uses: ./.shared-actions/.github/actions/e2e-issue-branch-cleanup
    with:
      branch-pattern: '^${{ matrix.environment }}-issue-\\d+$'
```

## Contributing

When adding new shared actions:

1. Create a new directory under `.github/actions/[action-name]`
2. Add `action.yaml` with proper metadata and composite steps
3. Update this README with usage examples
4. Test in a consuming repository before merging

## Repository Structure

```
sta-e2e-shared/
├── .github/
│   └── actions/
│       ├── e2e-issue-branch-cleanup/
│       │   └── action.yaml
│       └── [future-action]/
│           └── action.yaml
└── README.md
```

## Troubleshooting

### Common Issues

**Permission Errors:**
- Ensure the repository's `GITHUB_TOKEN` has write access to branches
- Check that protected branch rules don't prevent deletion

**No Branches Found:**
- The action only processes the first 100 branches (GitHub API limit) - no paging implemented
- Verify branch naming follows the exact patterns: `issue-123` or `backup-2025-01-15-14-30`

**Workflow Not Triggering:**
- Check cron syntax in your scheduled workflow
- Ensure workflow file is in `.github/workflows/` directory
- Verify the calling repository can access `aemdemos/sta-e2e-shared`

### Best Practices

- **Test First:** Use `workflow_dispatch` to manually run before relying on schedule
- **Monitor Logs:** Check the Actions tab to see what branches were processed
- **Branch Limits:** If you have >100 branches, consider running cleanup more frequently
- **Backup Safety:** Only backup branches older than 1 month are deleted
