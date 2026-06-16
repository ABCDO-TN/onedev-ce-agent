# OneDev CE AI Agent "Genie" - Documentation & Setup Guide

This README details the installation, configuration, and everyday utilization of the AI Agent "Skill". This skill transforms your AI agent into an expert operator, administrator, and power user ("Genie") for a locally installed instance of OneDev Community Edition (CE).

---

## 1. Skill Package Structure

The agent's capabilities and context layer are built out of the following files within the `onedev-ce-agent/` directory:

*   **`onedev-ce-agent/SKILL.md`**: Core skill definition containing the foundational architecture overview, TOD CLI patterns, REST API mappings, build spec authoring guidelines, Groovy scripting details, AI users configuration, workspaces setup, workflow patterns, security constraints, and query language rules.
*   **`onedev-ce-agent/references/build-spec-patterns.md`**: Technical reference containing 10 distinct CI/CD patterns equipped with full YAML examples (covering minimal CI, multi-job pipelines, Docker builds, matrix jobs, services, secrets management, promotions, templates, conditionals, and reuse mechanisms).
*   **`onedev-ce-agent/references/tod-cli-reference.md`**: Comprehensive command reference manual for all TOD resource groups including issues, pull requests, builds, code comments, configurations, and utilities alongside their respective flags and examples.
*   **`onedev-ce-agent/references/groovy-scripting-api.md`**: Automation script patterns leveraging Groovy for configuring custom fields, evaluation job conditions, controlling build promotions, modifying PR merge strategies, creating notification templates, and interacting with core manager classes.

---

## 2. Installation & Directory Setup

To allow your local AI tools or the OneDev server to read these capabilities, place the extracted files at the root of your project directory:

```bash
your-project-root/
└── onedev-ce-agent/
    ├── SKILL.md
    └── references/
        ├── build-spec-patterns.md
        ├── groovy-scripting-api.md
        └── tod-cli-reference.md
