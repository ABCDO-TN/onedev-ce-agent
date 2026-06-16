# TOD CLI Command Reference

TOD (TheOneDev) is the official CLI for OneDev 15.1+. All commands write raw server response to stdout.

## Table of Contents

1. [Global Options](#global-options)
2. [Issue Commands](#issue-commands)
3. [Pull Request Commands](#pull-request-commands)
4. [Build Commands](#build-commands)
5. [Code Comment Commands](#code-comment-commands)
6. [Config Commands](#config-commands)
7. [Utility Commands](#utility-commands)
8. [Reference Formats](#reference-formats)
9. [Error Handling](#error-handling)

## Global Options

All commands support:
- `--help` / `-h`: Show help for any command
- `--server-url`: Override configured server URL
- `--access-token`: Override configured access token

## Issue Commands

### `tod issue list`
List issues matching query.

```bash
tod issue list --query 'state is "Open" and assignee is me' --count 20 --offset 0
```

Flags: `--query`, `--count`, `--offset`, `--order-by`

### `tod issue get <ref>`
Get issue details (title, description, state, assignee, labels, etc.).

```bash
tod issue get 123
tod issue get myproject#123
tod issue get PROJ-123
```

### `tod issue get-comments <ref>`
Get all comments on an issue.

```bash
tod issue get-comments 123
```

### `tod issue create <project> <title> [description]`
Create a new issue.

```bash
tod issue create myproject "Bug in login" "Users cannot log in with SSO"
```

Flags: `--confidential`, `--assignee`, `--milestone`, `--fields` (custom fields as JSON)

### `tod issue edit <ref>`
Edit issue title, description, or custom fields.

```bash
tod issue edit 123 --title "Updated title" --description "New description"
```

### `tod issue change-state <ref> <state>`
Transition issue to a new state.

```bash
tod issue change-state 123 "In Progress"
```

### `tod issue link <ref> <link-type> <target-ref>`
Link issue to another issue.

```bash
tod issue link 123 depends-on 456
```

### `tod issue add-comment <ref> <comment>`
Add a comment to an issue.

```bash
tod issue add-comment 123 "Investigating the root cause"
```

### `tod issue log-work <ref> <time-spent> [comment]`
Log time spent on an issue.

```bash
tod issue log-work 123 "2h30m" "Fixed the bug"
```

### `tod issue create-branch <ref>`
Create a server-side branch for issue development. Outputs branch name. Idempotent — no-op if branch exists.

```bash
tod issue create-branch 123
# Output: issue-123-fix-login-bug
```

### `tod issue current-reference`
Detect current issue from branch name. Prints `#<n>` when branch matches `[<prefix>/]issue-<n>[-<suffix>]`, empty otherwise.

```bash
tod issue current-reference
```

### `tod issue get-query-description <query>`
Get human-readable description of query.

```bash
tod issue get-query-description 'state is "Open"'
```

### `tod issue get-valid-fields`
List valid fields for issue queries.

### `tod issue get-valid-links`
List valid link types.

## Pull Request Commands

### `tod pr list`
List pull requests.

```bash
tod pr list --query 'status is "Open" and target branch is "main"' --count 10
```

Flags: `--query`, `--count`, `--offset`

### `tod pr get <ref>`
Get PR details.

```bash
tod pr get 456
```

### `tod pr get-comments <ref>`
Get PR review comments.

### `tod pr get-code-comments <ref>`
Get code-level comments on PR. Returns comment IDs for reply/resolve operations.

### `tod pr get-builds <ref>`
Get builds associated with PR.

### `tod pr get-patch <ref>`
Get patch/diff of PR.

### `tod pr create <source-branch> <target-branch> <title> [description]`
Create a pull request.

```bash
tod pr create feature/login main "Add SSO login" "Implements OAuth2 flow"
```

Flags: `--reviewer`, `--milestone`, `--field`

### `tod pr edit <ref>`
Edit PR title/description.

### `tod pr approve <ref>`
Approve a pull request.

### `tod pr request-changes <ref>`
Request changes on a pull request.

### `tod pr merge <ref>`
Merge a pull request.

Flags: `--delete-branch-after-merge`

### `tod pr discard <ref>`
Close/discard a pull request without merging.

### `tod pr add-comment <ref> <comment>`
Add a review comment.

### `tod pr add-code-comment <ref> <file> <line> <comment>`
Add a code-level comment on a specific file and line.

### `tod pr checkout <ref>`
Check out a pull request locally.

```bash
tod pr checkout PROJ-456
```

### `tod pr get-query-description <query>`
Get human-readable description of PR query.

### `tod pr get-title-and-description-requirement <source-branch>`
Get project requirements for PR title/description.

### `tod pr get-commit-message-requirement <branch>`
Get commit message format requirements.

## Build Commands

### `tod build list`
List builds.

```bash
tod build list --query 'job is "ci" and not(successful)' --count 5
```

Flags: `--query`, `--count`, `--offset`

### `tod build get <ref>`
Get build details.

### `tod build get-log <ref>`
Stream build log output.

### `tod build get-code-problems <ref> <report-name> <severity>`
Get code problems from build. Severity: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`.

```bash
tod build get-code-problems 789 sonarqube HIGH
```

### `tod build get-changes-since-success <ref>`
Show changes since last successful build of same job.

### `tod build run [--local|--branch|--tag] <job-name>`
Run a CI/CD job.

```bash
# Run against uncommitted local changes
tod build run --local ci

# Run against branch
tod build run --branch main ci

# Run against tag
tod build run --tag v1.0.0 deploy
```

Flags: `--param` (job parameters), `--retries`

**Security note**: Local changes push to a temporal ref. For job secrets, set authorization to "allow all jobs" — "allow all branches" is insufficient.

### `tod build get-spec-schema`
Get JSON schema for `.onedev-buildspec.yml` validation.

### `tod build check-spec`
Validate and migrate `.onedev-buildspec.yml` to latest version.

### `tod build get-query-description <query>`
Get human-readable description of build query.

## Code Comment Commands

### `tod code-comment add-reply <comment-id> <reply>`
Reply to a code comment.

### `tod code-comment resolve <comment-id>`
Mark a code comment as resolved.

### `tod code-comment unresolve <comment-id>`
Reopen a resolved code comment.

## Config Commands

### `tod config set [property] [value]`
Set config properties interactively or positionally.

```bash
# Interactive
tod config set

# Non-interactive
tod config set server-url https://onedev.example.com
tod config set access-token <token>
tod config set trust-certs-file /path/to/ca.pem
```

### `tod config get [property]`
Get config (token redacted). Get single property: `tod config get server-url`.

### `tod config path`
Print config file path.

## Utility Commands

### `tod get-login-name`
Print current user's login name.

### `tod get-unix-timestamp`
Print current Unix timestamp.

### `tod project`
Print current project path (inferred from git remote).

### `tod remote`
Print git remote name pointing to current OneDev project.

### `tod get-valid-labels`
List valid labels for current project.

### `tod get-commit-message-requirement`
Get commit message format requirements for current project.

### `tod download <url> <output-file>`
Download a file from OneDev (handles auth automatically).

## Reference Formats

Commands accepting `<ref>` support these formats:

| Format | Example | Scope |
|--------|---------|-------|
| Numeric ID | `123` | Current project |
| With hash | `#123` | Current project |
| With project | `myproject#123` | Specified project |
| With key | `PROJ-123` | Resolved via project key |

## Error Handling

- TOD exits non-zero on API errors. Check stderr for error details.
- `tod issue current-reference` returns empty string (not error) when no current issue.
- `tod build run --local` requires clean git state or will error.
- Self-signed certificates: set `trust-certs-file` in config.
