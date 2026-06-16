---
name: onedev-ce-agent
description: Expert-level operation within OneDev Community Edition (CE) instances. Use when the agent needs to interact with OneDev CE for issue tracking, pull request management, CI/CD pipeline authoring (via .onedev-buildspec.yml), build inspection, workspace automation, AI user configuration, project administration, or TOD CLI workflow orchestration. Covers both direct REST API usage and TOD CLI integration, Groovy scripting for automation, and server-side workspace operations. Trigger whenever the user references OneDev, tod CLI, .onedev-buildspec.yml, or requests automation of DevOps workflows inside a OneDev instance.
---

# OneDev CE Agent

Operate OneDev Community Edition with expert-level precision. This skill covers the TOD CLI, RESTful API, build spec authoring, Groovy scripting, AI users, workspaces, and core CE architectural patterns.

## OneDev CE Architecture Overview

OneDev CE is a self-hosted Git server with integrated CI/CD, Kanban issue tracking, package registries, and AI capabilities. Key architectural concepts:

- **Project**: Core organizational unit holding code, issues, builds, packages. Projects form hierarchies for setting inheritance.
- **Build Spec**: CI/CD definition in `.onedev-buildspec.yml` at repository root. Defines jobs, triggers, executors, steps, services.
- **Job Executor**: Server Docker, Remote Docker, Server Shell, Remote Shell, or Kubernetes. Agents run on remote machines.
- **Build**: Result of running a job. Builds can be promoted to trigger downstream jobs.
- **Workspace**: Server-side dev environment (container or shell) for vibe coding with terminal agents.
- **AI User**: Virtual user backed by an LLM (Claude Haiku/Sonnet/Opus) with configurable system prompt and entitlements.

CE differs from EE in: no clustering/lead server, no high availability replication. All other features (CI/CD, packages, AI, workspaces) are fully available in CE.

## TOD CLI (Primary Interface)

Prefer TOD CLI for all mutating operations and most queries. It requires OneDev 15.1+.

### Installation & Configuration

```bash
# macOS/Linux/FreeBSD
curl -fsSL https://code.onedev.io/onedev/tod/~raw/main/install.sh | bash

# Configure
tod config set server-url https://onedev.example.com
tod config set access-token <personal-access-token>
```

Config file at `$XDG_CONFIG_HOME/tod/config` or `~/.config/tod/config` (INI format, mode 0600). Environment overrides: `ONEDEV_SERVER_URL`, `ONEDEV_ACCESS_TOKEN`, `ONEDEV_TRUST_CERTS_FILE`.

### Command Groups

| Resource | Commands |
|----------|----------|
| `tod issue` | `list`, `get`, `get-comments`, `create`, `edit`, `change-state`, `link`, `add-comment`, `log-work`, `create-branch`, `current-reference`, `get-query-description`, `get-valid-fields`, `get-valid-links` |
| `tod pr` | `list`, `get`, `get-comments`, `get-code-comments`, `get-builds`, `get-patch`, `create`, `edit`, `approve`, `request-changes`, `merge`, `discard`, `add-comment`, `add-code-comment`, `checkout`, `get-query-description` |
| `tod build` | `list`, `get`, `get-log`, `get-code-problems`, `get-changes-since-success`, `run` (`--branch`/`--tag`/`--local`), `get-spec-schema`, `check-spec`, `get-query-description` |
| `tod code-comment` | `add-reply`, `resolve`, `unresolve` |
| `tod config` | `set`, `get`, `path` |

### Reference Formats

Commands accepting `<ref>` support: `123`, `#123`, `myproject#123`, `PROJECTKEY-123`. For current issue detection: `tod issue current-reference` returns `#<n>` when branch matches `[<prefix>/]issue-<n>[-<suffix>]`.

### Local CI/CD Runs

```bash
# Run CI job against uncommitted changes
tod build run --local <job-name>

# Run against branch or tag
tod build run --branch main <job-name>
tod build run --tag v1.0.0 <job-name>
```

**Security**: Local changes push to a temporal ref. For job secrets, set authorization to "allow all jobs" — "allow all branches" is insufficient.

### Build Spec Commands

```bash
# Validate and migrate .onedev-buildspec.yml to latest schema
tod build check-spec

# Get JSON schema for IDE support
tod build get-spec-schema
```

## RESTful API (Fallback Interface)

Use when TOD CLI lacks coverage or for server-side automation.

- **Base URL**: `https://<onedev-host>/~api`
- **Auth**: Bearer token in `Authorization: Bearer <token>` header
- **Endpoints**: Projects, issues, pull requests, builds, users, groups, packages, settings

Common patterns:
```bash
# List projects
curl -X GET "https://onedev.example.com/~api/projects" \
  -H "Authorization: Bearer <token>"

# Create project
curl -X POST "https://onedev.example.com/~api/projects" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"demo","description":"Demo project"}'
```

## Build Spec Authoring (.onedev-buildspec.yml)

Define CI/CD pipelines in YAML. Place at repository root.

### Structure

```yaml
version: 1.0.0
jobs:
  - name: ci
    steps:
      - !CheckoutStep
        name: checkout
        cloneDepth: 50
      - !CommandStep
        name: build and test
        image: node:18
        commands: |
          npm ci
          npm test
    triggers:
      - !BranchUpdateTrigger {}
    jobExecutors:
      - !ServerDockerExecutor
        osSelector: ubuntu
```

### Key Elements

- **Steps**: `!CheckoutStep`, `!CommandStep`, `!BuildImageStep`, `!RunContainerStep`, `!PublishArtifactStep`, `!PublishReportStep`, `!PublishJUnitReportStep`, `!PublishCoberturaReportStep`, `!PublishCheckstyleReportStep`, `!PublishDependencyScanResultStep`, `!PublishImageScanResultStep`, `!SetBuildVersionStep`
- **Triggers**: `!BranchUpdateTrigger`, `!TagCreateTrigger`, `!CronTrigger`, `!DependencyIngestTrigger`, `!PullRequestUpdateTrigger`, `!ScheduleTrigger`
- **Executors**: `!ServerDockerExecutor`, `!RemoteDockerExecutor`, `!ServerShellExecutor`, `!RemoteShellExecutor`, `!KubernetesExecutor`
- **Services**: Database, message queue, cache etc. defined at job level
- **Step Templates**: Reusable parameterized step sequences

### Variables

- **Job variables**: `job.token`, `job.workspace`, `job.build`, `job.project`, `job.commit`, `job.branch`, `job.tag`, `job.request`, `job.executor`, `job.retries`
- **Workspace variables**: `workspace.id`, `workspace.token`, `workspace.dir`, `workspace.user`, `workspace.project`
- **Secrets**: Define in project settings, reference via `!Secret` in build spec
- **Conditional execution**: Use `condition` field with Groovy expression

For full schema: `tod build get-spec-schema`. For detailed examples: see `references/build-spec-patterns.md`.

## Groovy Scripting

OneDev exposes a Groovy scripting environment via the `onedev` variable.

### Common Patterns

```groovy
// Access managers for data queries
import io.onedev.server.manager.ProjectManager;
def projectManager = onedev.getInstance(ProjectManager.class);
def projects = projectManager.query();

// Issue custom field scripting (show conditions, choices, defaults)
def module = onedev.editContext.getInputValue("Module");

// Build condition scripting
def branch = job.getBranch();
return branch == "main" || branch.startsWith("release/");
```

### Scripting Locations

- Issue custom field show conditions and choices
- Job step conditions
- Build promotion conditions
- Pull request merge strategies
- Notification templates
- Authorization scripts

## AI Users & Integration

### AI User Setup

AI users are virtual OneDev users backed by LLMs. Configure via Administration > AI Users.

- **Recommended model**: Claude Haiku (cost-effective, fast, good tool use). Use Sonnet/Opus for complex tasks.
- **Entitlements**: Control which projects can use the AI user
- **System prompt**: Customizable per AI user to shape behavior

### AI User Capabilities

- Query with natural language (requires lite model)
- Symbol navigation and code search assistance
- PR title/description generation
- Code snippet explanation
- CI/CD spec editing assistance
- Participate in issue/PR discussions (mention with @)
- Review pull requests (add as reviewer, approve/request changes)

### Context-Aware Assistant

When enabled, AI appears as sidebar assistant that understands current context (issue, PR, code view) and can answer questions or perform actions.

## Workspaces

Server-side development environments for vibe coding.

### Workspace Spec

Define in Project Settings > Workspace Specs:

```yaml
name: dev
image: 1dev/opencode
runInContainer: true
shell: bash
userData:
  - key: node-modules
    path: /workspace/node_modules
  - key: config
    path: /workspace/.config
ports:
  - port: 3000
    name: dev server
setupCommands:
  - command: npm install
shortcutCommands:
  - name: dev
    command: npm run dev
```

### Supported Agents

OpenCode, Claude Code, Codex, or any terminal-based coding agent. Access via browser terminal or port-forwarded services.

### Provisioners

- **Docker Provisioner**: Runs workspace in container
- **Shell Provisioner**: Runs workspace directly on server shell

## Workflow Patterns

### Issue Workflow

1. Create issue branch: `tod issue create-branch <ref>` (outputs branch name)
2. Check out branch locally, implement changes
3. Push commits
4. Create or update PR: `tod pr create` or `tod pr edit`
5. Link issue in PR description for automatic state transition

### Pull Request Review Cycle

1. Check out PR: `tod pr checkout <ref>`
2. Review code, add code comments: `tod pr add-code-comment <ref> <file> <line> <comment>`
3. Add review comment: `tod pr add-comment <ref> <comment>`
4. Approve or request changes: `tod pr approve <ref>` / `tod pr request-changes <ref>`

### Build Failure Investigation

1. Get build details: `tod build get <ref>`
2. Stream logs: `tod build get-log <ref>`
3. Check code problems: `tod build get-code-problems <ref> <report> <severity>`
4. Compare with last successful: `tod build get-changes-since-success <ref>`
5. Fix issues, push changes, monitor rebuild

### Package Registry Operations

OneDev CE includes built-in package registries for Docker, NPM, Maven, NuGet, PyPI, RubyGems. Packages published via CI/CD link automatically to builds.

## Security & Permissions

- **Access tokens**: Create via User Settings > Access Tokens. Scope tokens to specific projects and permissions.
- **Branch protection**: Define rules requiring review or CI verification. Can include AI users as reviewers.
- **Project authorization**: Grant roles (Owner, Admin, Write, Read) to users/groups at project or tree level.
- **Job secrets**: Encrypted values accessible only to authorized jobs. Clear authorization for local CI runs.

## Query Language

OneDev uses a powerful query language across all entities:

```
# Issues
assignee is me and state is "Open" and label in (bug, critical)

# Pull requests
source branch starts with "feature/" and status is "Open" and reviewer is "ai-user"

# Builds
job is "ci" and project is "myproject" and not(successful)

# Packages
project is "myproject" and type is "Container Image" and tag starts with "v1"
```

Save queries for quick access; subscribe to queries for notifications.

## References

- **Build spec patterns and examples**: See `references/build-spec-patterns.md`
- **TOD CLI full command reference**: See `references/tod-cli-reference.md`
- **Groovy scripting API**: See `references/groovy-scripting-api.md`
