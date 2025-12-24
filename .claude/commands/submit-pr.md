# Submit PR

Analyzes current git changes and automatically creates a pull request with an intelligently generated description based on commit subject, message, and changed files.

## Arguments

- `base` (optional): Base branch to compare against (default: `main`)
- `fixes` (optional): Issue number that this PR fixes (e.g., `fixes=123` for `Fixes #123`)
- `related` (optional): Related issue or PR number (e.g., `related=456` for `Related Issue/PR #456`)
- `title` (optional): Custom PR title (defaults to first commit subject)
- `draft` (optional): Create as draft PR (`true`/`false`, default: `false`)

## Steps

### 1. Prerequisites Check

Verify GitHub CLI is installed and authenticated:

```bash
echo "🔍 Checking prerequisites..."

# Check if gh CLI is installed
if command -v gh &> /dev/null; then
    echo "✅ GitHub CLI found"
else
    echo "❌ GitHub CLI (gh) is required but not installed"
    echo "Install it from: https://cli.github.com/"
    exit 1
fi

# Check if authenticated
if gh auth status &> /dev/null; then
    echo "✅ GitHub CLI authenticated"
else
    echo "❌ GitHub CLI not authenticated"
    echo "Run: gh auth login"
    exit 1
fi

echo "✅ GitHub CLI ready"
```

### 2. Parse Arguments and Extract Variables

Parse command arguments and set up variables:

```bash
echo "⚙️  Parsing arguments..."

# Extract variables from arguments
BASE_BRANCH="${base:-main}"
FIXES_ISSUE="$fixes"
RELATED_ISSUE="$related"
PR_TITLE="$title"
IS_DRAFT="${draft:-false}"

# Log what will be used
if [ -n "$FIXES_ISSUE" ]; then
    echo "📌 Will link to issue: #$FIXES_ISSUE"
fi

if [ -n "$RELATED_ISSUE" ]; then
    echo "🔗 Will reference related: #$RELATED_ISSUE"
fi

if [ "$IS_DRAFT" = "true" ]; then
    echo "📝 Will create as draft PR"
fi
```

### 3. Validate Branch and Changes

Check that we're on a feature branch and have commits to create a PR for:

```bash
echo "🔍 Validating branch state..."

# Get current branch
CURRENT_BRANCH=$(git branch --show-current)

# Check if we're on main/base branch
if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
    echo "❌ Cannot create PR from $BASE_BRANCH branch"
    echo "Switch to a feature branch with your changes first"
    exit 1
fi

# Check if there are commits different from base branch
COMMIT_DIFF_COUNT=$(git rev-list --count HEAD ^$BASE_BRANCH 2>/dev/null || echo "0")
if [ "${COMMIT_DIFF_COUNT:-0}" -gt 0 ]; then
    echo "✅ Found $COMMIT_DIFF_COUNT commits different from $BASE_BRANCH"
else
    echo "❌ No commits found different from $BASE_BRANCH branch"
    echo "Make some commits on your feature branch first"
    echo "Debug: Current HEAD: $(git rev-parse HEAD)"
    echo "Debug: $BASE_BRANCH HEAD: $(git rev-parse $BASE_BRANCH 2>/dev/null || echo 'not found')"
    exit 1
fi

# Ensure branch is pushed to remote and up-to-date
echo "🚀 Ensuring branch is pushed to remote..."
if git push -u origin "$CURRENT_BRANCH" 2>&1; then
    echo "✅ Branch pushed successfully"
else
    echo "❌ Failed to push branch to remote"
    exit 1
fi

# Verify remote branch has the commits
echo "🔍 Verifying remote branch state..."
sleep 2  # Give GitHub time to process the push

# Check remote commit count
REMOTE_COMMIT_DIFF=$(git rev-list --count origin/$BASE_BRANCH..origin/$CURRENT_BRANCH 2>/dev/null || echo "0")
if [ "${REMOTE_COMMIT_DIFF:-0}" -gt 0 ]; then
    echo "✅ Remote branch has $REMOTE_COMMIT_DIFF commits ahead of $BASE_BRANCH"
else
    echo "❌ Remote branch validation failed"
    echo "Debug: Checking if remote branch exists..."
    git ls-remote --heads origin "$CURRENT_BRANCH" | head -3
    echo "Debug: Local vs remote status:"
    git status -uno
    exit 1
fi

echo "✅ Branch: $CURRENT_BRANCH (ready for PR against $BASE_BRANCH)"
```

### 4. Analyze Commits and Changes

Analyze commit subject, message, and changed files:

```bash
echo "🔍 Analyzing commits and changes..."

# Get the most recent commit for PR title (if not provided)
LATEST_COMMIT_SUBJECT=$(git log -1 --format='%s' HEAD)
LATEST_COMMIT_BODY=$(git log -1 --format='%b' HEAD | grep -v '^Signed-off-by:' | sed '/^$/d')

# Set PR title
if [ -z "$PR_TITLE" ]; then
    PR_TITLE="$LATEST_COMMIT_SUBJECT"
fi

echo "📝 PR Title: $PR_TITLE"

# Get list of changed files
CHANGED_FILES=$(git diff --name-only $BASE_BRANCH...HEAD)

# Calculate file count properly (handle empty case)
if [ -z "$CHANGED_FILES" ]; then
    FILE_COUNT=0
else
    FILE_COUNT=$(echo "$CHANGED_FILES" | wc -l)
fi

# Get commit messages for context
COMMIT_MESSAGES=$(git log --oneline $BASE_BRANCH..HEAD)
COMMIT_COUNT=$(git rev-list --count $BASE_BRANCH..HEAD)

# Ensure counts are valid numbers (fallback to 0 if empty or invalid)
FILE_COUNT=${FILE_COUNT:-0}
COMMIT_COUNT=${COMMIT_COUNT:-0}

# Analyze file types changed
API_CHANGES=$(echo "$CHANGED_FILES" | grep -E '^api/' || echo "")
CONTROLLER_CHANGES=$(echo "$CHANGED_FILES" | grep -E '^controllers/' || echo "")
TEST_CHANGES=$(echo "$CHANGED_FILES" | grep -E '^tests?' || echo "")
DOC_CHANGES=$(echo "$CHANGED_FILES" | grep -E '\.(md|adoc)$' || echo "")
CHART_CHANGES=$(echo "$CHANGED_FILES" | grep -E '^chart/' || echo "")
VERSION_CHANGES=$(echo "$CHANGED_FILES" | grep -E 'versions\.yaml|go\.(mod|sum)' || echo "")

echo "📊 Analysis: $FILE_COUNT files, $COMMIT_COUNT commits"
echo "🔧 Debug - First 5 changed files:"
echo "$CHANGED_FILES" | head -5 | sed 's/^/  - /' || echo "  (none)"
```

### 5. Determine PR Type

Auto-detect the primary type of change based on commit messages and file patterns:

```bash
echo "🏷️  Determining PR type..."

PR_TYPE=""

# Analyze commit subject and body for keywords (prioritize most recent commit)
FULL_COMMIT_TEXT="$LATEST_COMMIT_SUBJECT $LATEST_COMMIT_BODY $COMMIT_MESSAGES"

if echo "$FULL_COMMIT_TEXT" | grep -qi -E '\b(fix|bug|issue|error|broken)\b'; then
    PR_TYPE="Bug Fix"
    echo "   - Bug fix keywords detected → Bug Fix"
elif echo "$FULL_COMMIT_TEXT" | grep -qi -E '\b(refactor|cleanup|restructure|reorganize)\b'; then
    PR_TYPE="Refactor"
    echo "   - Refactor keywords detected → Refactor"
elif echo "$FULL_COMMIT_TEXT" | grep -qi -E '\b(optim|perf|performance|faster|improve)\b'; then
    PR_TYPE="Optimization"
    echo "   - Performance keywords detected → Optimization"
elif [ -n "$TEST_CHANGES" ] && [ -z "$API_CHANGES" ] && [ -z "$CONTROLLER_CHANGES" ]; then
    PR_TYPE="Test"
    echo "   - Only test files changed → Test"
elif [ -n "$DOC_CHANGES" ] && [ -z "$API_CHANGES" ] && [ -z "$CONTROLLER_CHANGES" ]; then
    PR_TYPE="Documentation Update"
    echo "   - Only documentation changed → Documentation Update"
else
    PR_TYPE="Enhancement / New Feature"
    echo "   - Default categorization → Enhancement/New Feature"
fi
```

### 6. Generate PR Description

Create the PR description based on commit analysis and template format:

```bash
echo "📝 Generating PR description..."

# Debug: Show all key variables
echo "🔧 Debug - Key variables:"
echo "  PR_TYPE: '$PR_TYPE'"
echo "  FILE_COUNT: '$FILE_COUNT'"
echo "  COMMIT_COUNT: '$COMMIT_COUNT'"
echo "  API_CHANGES: '$API_CHANGES'"
echo "  CONTROLLER_CHANGES: '$CONTROLLER_CHANGES'"
echo "  TEST_CHANGES: '$TEST_CHANGES'"
echo "  DOC_CHANGES: '$DOC_CHANGES'"

# Ensure all variables are set (fallback to safe values)
PR_TYPE="${PR_TYPE:-Enhancement / New Feature}"
FILE_COUNT="${FILE_COUNT:-0}"
COMMIT_COUNT="${COMMIT_COUNT:-0}"
LATEST_COMMIT_SUBJECT="${LATEST_COMMIT_SUBJECT:-No subject}"
LATEST_COMMIT_BODY="${LATEST_COMMIT_BODY:-}"

# Build the PR description following the template format
# First, build the checkbox section
PR_CHECKBOXES=""

# Set checkboxes based on PR_TYPE
if [ "$PR_TYPE" = "Enhancement / New Feature" ]; then
    PR_CHECKBOXES="- [x] Enhancement / New Feature"
else
    PR_CHECKBOXES="- [ ] Enhancement / New Feature"
fi

if [ "$PR_TYPE" = "Bug Fix" ]; then
    PR_CHECKBOXES="$PR_CHECKBOXES
- [x] Bug Fix"
else
    PR_CHECKBOXES="$PR_CHECKBOXES
- [ ] Bug Fix"
fi

if [ "$PR_TYPE" = "Refactor" ]; then
    PR_CHECKBOXES="$PR_CHECKBOXES
- [x] Refactor"
else
    PR_CHECKBOXES="$PR_CHECKBOXES
- [ ] Refactor"
fi

if [ "$PR_TYPE" = "Optimization" ]; then
    PR_CHECKBOXES="$PR_CHECKBOXES
- [x] Optimization"
else
    PR_CHECKBOXES="$PR_CHECKBOXES
- [ ] Optimization"
fi

if [ "$PR_TYPE" = "Test" ]; then
    PR_CHECKBOXES="$PR_CHECKBOXES
- [x] Test"
else
    PR_CHECKBOXES="$PR_CHECKBOXES
- [ ] Test"
fi

if [ "$PR_TYPE" = "Documentation Update" ]; then
    PR_CHECKBOXES="$PR_CHECKBOXES
- [x] Documentation Update"
else
    PR_CHECKBOXES="$PR_CHECKBOXES
- [ ] Documentation Update"
fi

# Build the PR description
PR_DESCRIPTION="#### What type of PR is this?

$PR_CHECKBOXES

#### What this PR does / why we need it:"

# Use commit body as primary content, fallback to subject if body is empty
if [ -n "$LATEST_COMMIT_BODY" ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

$LATEST_COMMIT_BODY"
else
    PR_DESCRIPTION="$PR_DESCRIPTION

$LATEST_COMMIT_SUBJECT"
fi

# Add file context for multi-commit PRs
if [ "${COMMIT_COUNT:-0}" -gt 1 ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

**Multiple commits ($COMMIT_COUNT):**
$(echo "$COMMIT_MESSAGES" | sed 's/^/- /')"
fi

# Build changed files summary
CHANGED_FILES_SUMMARY=""
CHANGE_CATEGORIES=""

# Build list of change categories
if [ -n "$API_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- API/CRD changes
"
fi

if [ -n "$CONTROLLER_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- Controller logic updates
"
fi

if [ -n "$TEST_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- Test modifications
"
fi

if [ -n "$DOC_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- Documentation updates
"
fi

if [ -n "$CHART_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- Helm chart changes
"
fi

if [ -n "$VERSION_CHANGES" ]; then
    CHANGE_CATEGORIES="${CHANGE_CATEGORIES}- Version/dependency updates
"
fi

# If no categories detected, show actual file list (up to 10 files)
if [ -z "$CHANGE_CATEGORIES" ] && [ -n "$CHANGED_FILES" ]; then
    CHANGE_CATEGORIES="$(echo "$CHANGED_FILES" | head -10 | sed 's/^/- /')"
    if [ "$(echo "$CHANGED_FILES" | wc -l)" -gt 10 ]; then
        CHANGE_CATEGORIES="${CHANGE_CATEGORIES}
- ... and $(($(echo "$CHANGED_FILES" | wc -l) - 10)) more files"
    fi
fi

# Add changed files section
PR_DESCRIPTION="$PR_DESCRIPTION

**Files changed (${FILE_COUNT}):**
$CHANGE_CATEGORIES"

# Add issue references section
PR_DESCRIPTION="$PR_DESCRIPTION

#### Which issue(s) this PR fixes:"

# Handle issue references
if [ -n "$FIXES_ISSUE" ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

Fixes #$FIXES_ISSUE"
else
    # Look for issue references in commits
    COMMIT_FIXES=$(echo "$FULL_COMMIT_TEXT" | grep -i -o -E '(fixes?|closes?|resolves?) #[0-9]+' | head -1)
    if [ -n "$COMMIT_FIXES" ]; then
        PR_DESCRIPTION="$PR_DESCRIPTION

$COMMIT_FIXES"
    else
        PR_DESCRIPTION="$PR_DESCRIPTION

Fixes #"
    fi
fi

if [ -n "$RELATED_ISSUE" ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

Related Issue/PR #$RELATED_ISSUE"
else
    PR_DESCRIPTION="$PR_DESCRIPTION

Related Issue/PR #"
fi

# Add additional information section
PR_DESCRIPTION="$PR_DESCRIPTION

#### Additional information:"

# Add testing checklist for relevant changes
if [ -n "$TEST_CHANGES" ] || [ -n "$API_CHANGES" ] || [ -n "$CONTROLLER_CHANGES" ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

**Testing:**
- [ ] Unit tests pass: \`make test\`"

    if [ -n "$API_CHANGES" ] || [ -n "$CONTROLLER_CHANGES" ]; then
        PR_DESCRIPTION="$PR_DESCRIPTION
- [ ] Integration tests pass: \`make test.integration\`
- [ ] E2E tests pass: \`make test.e2e.kind\`"
    fi
fi

# Add API change notes
if [ -n "$API_CHANGES" ]; then
    PR_DESCRIPTION="$PR_DESCRIPTION

**API Changes:**
- Modified CRDs or API types
- Generated CRDs with \`make gen\`"
fi
```

### 7. Create the Pull Request

Use GitHub CLI to create the PR with the generated description:

```bash
echo "🚀 Creating pull request..."

# Prepare gh pr create arguments
GH_ARGS=(
    "--title" "$PR_TITLE"
    "--body" "$PR_DESCRIPTION"
    "--base" "$BASE_BRANCH"
    "--head" "$CURRENT_BRANCH"
)

# Add draft flag if requested
if [ "$IS_DRAFT" = "true" ]; then
    GH_ARGS+=("--draft")
fi

# Final validation before creating PR
echo "🔍 Final validation before creating PR..."
echo "Current branch: $CURRENT_BRANCH"
echo "Base branch: $BASE_BRANCH"
echo "Local commits ahead: $(git rev-list --count $BASE_BRANCH..HEAD 2>/dev/null || echo '0')"
echo "Remote commits ahead: $(git rev-list --count origin/$BASE_BRANCH..origin/$CURRENT_BRANCH 2>/dev/null || echo '0')"

# Create the PR
echo "📤 Executing: gh pr create --title \"$PR_TITLE\" --base \"$BASE_BRANCH\" --head \"$CURRENT_BRANCH\""
if PR_URL=$(gh pr create "${GH_ARGS[@]}" 2>&1); then
    echo "✅ Pull request created successfully!"
    echo "🔗 URL: $PR_URL"

    # Extract PR number from URL for reference
    PR_NUMBER=$(echo "$PR_URL" | grep -o '[0-9]\+$' || echo "unknown")
    echo "📝 PR #$PR_NUMBER: $PR_TITLE"

else
    echo "❌ Failed to create pull request"
    echo "GitHub CLI Error: $PR_URL"
    echo ""
    echo "🔧 Debugging Information:"
    echo "Branch info:"
    git branch -vv | grep -E "($CURRENT_BRANCH|$BASE_BRANCH)" || git branch -vv
    echo ""
    echo "Remote tracking:"
    git remote -v
    echo ""
    echo "Recent commits:"
    git log --oneline -3
    echo ""
    echo "🚨 Common fixes:"
    echo "1. Ensure you have commits: git log --oneline $BASE_BRANCH..HEAD"
    echo "2. Check remote exists: git remote -v"
    echo "3. Verify GitHub auth: gh auth status"
    echo "4. Try manual PR: gh pr create --web"
    exit 1
fi
```

### 8. Success Summary

Display final summary of the created PR:

```bash
echo ""
echo "🎉 Pull Request Creation Complete!"
echo "=================================="
echo ""
echo "📝 Title: $PR_TITLE"
echo "🏷️  Type: $PR_TYPE"
echo "🌿 Branch: $CURRENT_BRANCH → $BASE_BRANCH"
echo "📊 Changes: $FILE_COUNT files, $COMMIT_COUNT commits"
echo "🔗 URL: $PR_URL"
echo ""

# Show what was automatically detected and included
echo "🔍 Auto-detected content:"
if [ -n "$FIXES_ISSUE" ]; then
    echo "   ✅ Links to issue #$FIXES_ISSUE"
elif echo "$FULL_COMMIT_TEXT" | grep -q -i -E '(fixes?|closes?|resolves?) #[0-9]+'; then
    echo "   ✅ Found issue reference in commits"
else
    echo "   ⚠️  No issue reference detected"
fi

if [ -n "$API_CHANGES" ]; then
    echo "   ✅ API changes detected - added testing checklist"
fi

if [ -n "$TEST_CHANGES" ]; then
    echo "   ✅ Test changes detected"
fi

echo ""
echo "📚 Next Steps:"
echo "1. Review the PR description at: $PR_URL"
echo "2. Add reviewers if needed"
echo "3. Run tests locally: make test"

if [ -n "$API_CHANGES" ] || [ -n "$CONTROLLER_CHANGES" ]; then
    echo "4. Run integration tests: make test.integration"
    echo "5. Run E2E tests: make test.e2e.kind"
fi

echo ""
echo "✅ Ready for review!"
```


## Error Handling

**If GitHub CLI not installed:**
```bash
❌ GitHub CLI (gh) is required but not installed
Install it from: https://cli.github.com/
```

**If GitHub CLI not authenticated:**
```bash
❌ GitHub CLI not authenticated
Run: gh auth login
```

**If not in a git repository:**
```bash
❌ Not in a git repository
This command must be run from within the sail-operator repository.
```

**If on main/base branch:**
```bash
❌ Cannot create PR from main branch
Switch to a feature branch with your changes first
```

**If no changes to analyze:**
```bash
❌ No commits found different from main branch
Make some commits on your feature branch first
```

**If PR creation fails:**
```bash
❌ Failed to create pull request
GitHub CLI Error: [specific error from gh CLI]

Common GraphQL errors and fixes:
1. "Head sha can't be blank, Base sha can't be blank, No commits between main and [branch]":
   - Branch not properly pushed: git push -u origin [branch]
   - No actual commits: git log --oneline main..HEAD (should show commits)
   - Timing issue: Wait a few seconds and retry

2. "Head ref must be a branch":
   - Invalid branch name or not pushed to remote
   - Check: git ls-remote --heads origin [branch]

3. "No commits between [base] and [head]":
   - Your branch is identical to base branch
   - Create commits: git commit -m "your changes"
   - Or wrong base branch: check --base argument

Debugging steps:
1. Verify commits exist: git log --oneline main..HEAD
2. Check branch is pushed: git branch -vv
3. Test GitHub auth: gh auth status
4. Check existing PRs: gh pr list --head [branch]
5. Try manual creation: gh pr create --web
6. Verify remote tracking: git remote -v
```

## Advanced Features

**Intelligent commit analysis:**
- Uses commit subject as default PR title
- Analyzes commit message body for additional context
- Detects PR type from commit keywords (fix, refactor, optimize, etc.)
- Handles both single and multi-commit PRs

**Smart file pattern detection:**
- Automatically detects API changes, controller updates, test modifications
- Adds appropriate testing checklists based on changed files
- Categorizes changes for better PR description context

**Issue linking automation:**
- Accepts issue numbers via command arguments (`fixes=123`)
- Scans commit messages for issue references as fallback
- Formats "Fixes #123" statements automatically
- Supports related issue/PR references

**GitHub integration:**
- Creates PR directly using GitHub CLI
- Automatically pushes branch if not on remote
- Supports draft PRs with `draft=true`
- Provides immediate PR URL for access

## Usage Examples

```bash
# Create PR with automatic analysis of commits
/submit-pr

# Create PR linking to a specific issue
/submit-pr fixes=123

# Create PR with custom title
/submit-pr title="Add new authentication method"

# Create draft PR
/submit-pr draft=true

# Create PR against different base branch
/submit-pr base=develop

# Create PR with issue links and custom title
/submit-pr fixes=123 related=456 title="Fix authentication bug"

# Create draft PR against develop branch
/submit-pr base=develop draft=true fixes=789

# Complex example with all options
/submit-pr base=release-1.0 fixes=123 related=456 title="Security fix" draft=true
```

## Workflow Integration

This command is designed to streamline the PR creation process:

1. **Make your changes** and commit with descriptive messages
2. **Run the command**: `/submit-pr fixes=123`
3. **Review the created PR** and add any additional context
4. **Request reviewers** and proceed with the review process

The command automatically:
- Analyzes your commit messages to understand the change
- Detects the type of PR based on files and keywords
- Creates appropriate testing checklists
- Links to issues when specified
- Pushes your branch if needed

## Important Implementation Notes

**Script Execution:**
- This skill contains multiple bash code blocks that must be executed as a single script
- Variables set in one section are used in subsequent sections
- Do NOT execute each bash block separately - they must run in the same shell session
- All variables (FILE_COUNT, PR_TYPE, CHANGED_FILES, etc.) must be preserved across sections

**Variable Scoping:**
- Critical variables are set with fallback values to prevent undefined variable errors
- Debug output shows variable values to help diagnose scoping issues
- The script includes variable validation at key points

## Notes

- **Prerequisites**: GitHub CLI (`gh`) must be installed and authenticated
- **Branch**: Run from your feature branch, not main
- **Commits**: Ensure you have committed your changes before running
- **Commit messages**: Write descriptive commit subjects - they become the PR title
- **Issue linking**: Use `fixes=123` to automatically close issues when merged
- **Review**: Always review the generated PR description before requesting reviews
- **Testing**: Run suggested tests locally before marking the PR ready for review
