# Groovy Scripting API

OneDev exposes a Groovy scripting environment. The `onedev` variable is always available. Scripts run in a sandbox with access to OneDev's internal APIs.

## Table of Contents

1. [The `onedev` Variable](#the-onedev-variable)
2. [Issue Custom Field Scripting](#issue-custom-field-scripting)
3. [Job Condition Scripting](#job-condition-scripting)
4. [Build Promotion Conditions](#build-promotion-conditions)
5. [Pull Request Merge Strategies](#pull-request-merge-strategies)
6. [Notification Templates](#notification-templates)
7. [Common Manager Classes](#common-manager-classes)
8. [Security Considerations](#security-considerations)

## The `onedev` Variable

The `onedev` variable provides access to OneDev's dependency injection container and context objects.

### Core Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `onedev.getInstance(Class<T>)` | `T` | Get a manager/service by class |
| `onedev.editContext` | `EditContext` | Access form input values during editing |
| `onedev.build` | `Build` | Current build (in build contexts) |
| `onedev.job` | `Job` | Current job (in job contexts) |
| `onedev.project` | `Project` | Current project |
| `onedev.user` | `User` | Current user |

## Issue Custom Field Scripting

### Show Conditions

Return boolean to determine if field should be visible:

```groovy
// Show "Regression Version" only for bug issues
def issueType = onedev.editContext.getInputValue("Type");
return issueType == "Bug";
```

### Choice Options

Return Map (display -> value) or List of choices:

```groovy
import io.onedev.server.manager.GroupManager;

// Populate module choices from groups
def choices = [:];
for (group in onedev.getInstance(GroupManager.class).query()) {
    choices.put(group.name, group.name);
}
return choices;
```

### Cascading Choices

```groovy
import io.onedev.server.manager.GroupManager;

// Assignee choices depend on selected module
def module = onedev.editContext.getInputValue("Module");
def choices = [];
if (module != null) {
    def group = onedev.getInstance(GroupManager.class).find(module);
    if (group != null) {
        for (user in group.members) {
            choices.add(user.facade);
        }
    }
}
return choices;
```

### Default Values

```groovy
import io.onedev.server.manager.GroupManager;

// Default assignee based on module selection
def module = onedev.editContext.getInputValue("Module");
if (module == "Front End")
    return "tracy";
else if (module == "Back End")
    return "robin";
else
    return null;
```

### User Choice Fields

Return user login names or `UserFacade` objects:

```groovy
import io.onedev.server.manager.UserManager;

// All active users as choices
def choices = [];
for (user in onedev.getInstance(UserManager.class).query()) {
    if (user.disabled == false) {
        choices.add(user.facade);
    }
}
return choices;
```

## Job Condition Scripting

Used in build spec `condition` fields:

```groovy
// Run only on main or release branches
def branch = job.getBranch();
return branch == "main" || branch.startsWith("release/");
```

```groovy
// Skip for draft PRs
def request = job.getRequest();
return request == null || !request.isDraft();
```

```groovy
// Run only if specific files changed
def changes = build.getChanges();
for (change in changes) {
    if (change.path.endsWith(".java")) {
        return true;
    }
}
return false;
```

## Build Promotion Conditions

Used to gate build promotion:

```groovy
// Promote only if all tests passed and coverage > 80%
def build = onedev.build;
def coverageReport = build.getReport("coverage");
if (coverageReport != null) {
    def lineCoverage = coverageReport.getLineCoverage();
    return lineCoverage > 0.80;
}
return false;
```

## Pull Request Merge Strategies

Custom merge commit messages:

```groovy
// Generate merge commit with issue references
def pr = onedev.pullRequest;
def title = pr.getTitle();
def issues = pr.getLinkedIssues();
def message = title;
if (!issues.isEmpty()) {
    message += "\n\nFixes: ";
    for (issue in issues) {
        message += "#" + issue.getNumber() + " ";
    }
}
return message;
```

## Notification Templates

Used in webhook/email notifications:

```groovy
// Build notification payload
def build = onedev.build;
def project = build.getProject();
def json = [:];
json.project = project.getName();
json.build = build.getNumber();
json.status = build.getStatus().toString();
json.url = build.getUrl();
return new groovy.json.JsonBuilder(json).toString();
```

## Common Manager Classes

| Manager Class | Purpose |
|---------------|---------|
| `io.onedev.server.manager.ProjectManager` | CRUD operations on projects |
| `io.onedev.server.manager.IssueManager` | Issue queries and updates |
| `io.onedev.server.manager.PullRequestManager` | PR queries and updates |
| `io.onedev.server.manager.BuildManager` | Build queries |
| `io.onedev.server.manager.UserManager` | User management |
| `io.onedev.server.manager.GroupManager` | Group management |
| `io.onedev.server.manager.RoleManager` | Role/permission management |
| `io.onedev.server.manager.SettingManager` | System settings |
| `io.onedev.server.manager.AgentManager` | Agent management |

### Example: Querying Issues

```groovy
import io.onedev.server.manager.IssueManager;
import io.onedev.server.model.Issue;
import io.onedev.server.model.Project;

def issueManager = onedev.getInstance(IssueManager.class);
def project = onedev.project;

// Query open bugs in current project
def query = "project is \"" + project.getName() + "\" and state is \"Open\" and label is Bug";
def issues = issueManager.query(project, query, true, 0, 100);

for (issue in issues) {
    println issue.getTitle() + " (#" + issue.getNumber() + ")";
}
```

### Example: Creating an Issue

```groovy
import io.onedev.server.manager.IssueManager;
import io.onedev.server.model.Issue;
import io.onedev.server.model.Project;

def issueManager = onedev.getInstance(IssueManager.class);
def project = onedev.project;

def issue = new Issue();
issue.setProject(project);
issue.setTitle("Automated issue from script");
issue.setDescription("Created via Groovy scripting");
issueManager.open(issue);
return issue.getNumber();
```

## Security Considerations

- Groovy scripts run with the permissions of the invoking user
- Scripts can access all OneDev managers — validate inputs carefully
- Avoid executing user-provided strings as Groovy code
- Use `onedev.editContext.getInputValue()` for sanitized form input
- Manager queries respect the current user's permission scope
