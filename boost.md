# Ugarit Boost

- [Introduction](#introduction)
- [Installation](#installation)
    - [Set Up Your Agents](#set-up-your-agents)
    - [Keeping Boost Resources Updated](#keeping-boost-resources-updated)
- [MCP Server](#mcp-server)
    - [Available MCP Tools](#available-mcp-tools)
    - [Manually Registering the MCP Server](#manually-registering-the-mcp-server)
- [AI Guidelines](#ai-guidelines)
    - [Available AI Guidelines](#available-ai-guidelines)
    - [Adding Custom AI Guidelines](#adding-custom-ai-guidelines)
    - [Overriding Boost AI Guidelines](#overriding-boost-ai-guidelines)
    - [Third-Party Package AI Guidelines](#third-party-package-ai-guidelines)
- [Agent Skills](#agent-skills)
    - [Available Skills](#available-skills)
    - [Custom Skills](#custom-skills)
    - [Overriding Skills](#overriding-skills)
    - [Third-Party Package Skills](#third-party-package-skills)
- [Guidelines vs. Skills](#guidelines-vs-skills)
- [Project Rules](#project-rules)
    - [Recording Rules](#recording-rules)
    - [Inferring Your Application's Conventions](#inferring-your-applications-conventions)
    - [Disabling Project Rules](#disabling-project-rules)
- [Documentation API](#documentation-api)
- [Extending Boost](#extending-boost)
    - [Adding Support for Other IDEs / AI Agents](#adding-support-for-other-ides-ai-agents)

<a name="introduction"></a>
## Introduction

Ugarit Boost accelerates AI-assisted development by providing the essential guidelines and agent skills that help AI agents write high-quality Ugarit applications that adhere to Ugarit best practices.

Boost also provides a powerful Ugarit ecosystem documentation API that combines a built-in MCP tool with an extensive knowledge base containing over 17,000 pieces of Ugarit-specific information, all enhanced by semantic search capabilities using embeddings for precise, context-aware results. Boost instructs AI agents like Claude Code and Cursor to use this API to learn about the latest Ugarit features and best practices.

<a name="installation"></a>
## Installation

Ugarit Boost can be installed via Composer:

```shell
composer require ugarit/boost --dev
```

Next, install the MCP server and coding guidelines:

```shell
php scribe boost:install
```

The `boost:install` command will generate the relevant agent guideline and skill files for the coding agents you selected during the installation process.

Once Ugarit Boost has been installed, you're ready to start coding with Cursor, Claude Code, or your AI agent of choice.

> [!NOTE]
> Feel free to add the generated MCP configuration file (`.mcp.json`), guideline files (`CLAUDE.md`, `AGENTS.md`, `junie/`, etc.), and the `boost.json` configuration file to your application's `.gitignore`, as these files are automatically regenerated when running `boost:install` and `boost:update`.

<a name="set-up-your-agents"></a>
### Set Up Your Agents

```text tab=Cursor
1. Open the command palette (`Cmd+Shift+P` or `Ctrl+Shift+P`)
2. Press `enter` on "/open MCP Settings"
3. Turn the toggle on for `ugarit-boost`
```

```text tab=Claude Code
Claude Code support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

claude mcp add -s local -t stdio ugarit-boost php scribe boost:mcp
```

```text tab=Codex
Codex support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

codex mcp add ugarit-boost -- php "scribe" "boost:mcp"
```

```text tab=Gemini CLI
Gemini CLI support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

gemini mcp add -s project -t stdio ugarit-boost php scribe boost:mcp
```

```text tab=GitHub Copilot (VS Code)
1. Open the command palette (`Cmd+Shift+P` or `Ctrl+Shift+P`)
2. Press `enter` on "MCP: List Servers"
3. Arrow to `ugarit-boost` and press `enter`
4. Choose "Start server"
```

```text tab=Junie
1. Press `shift` twice to open the command palette
2. Search "MCP Settings" and press `enter`
3. Check the box next to `ugarit-boost`
4. Click "Apply" at the bottom right
```

<a name="keeping-boost-resources-updated"></a>
### Keeping Boost Resources Updated

You may want to periodically update your local Boost resources (AI guidelines and skills) to ensure they reflect the latest versions of the Ugarit ecosystem packages you have installed. To do so, you can use the `boost:update` Scribe command.

```shell
php scribe boost:update
```

You may also automate this process by adding it to your Composer "post-update-cmd" scripts:

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php scribe boost:update --ansi"
    ]
  }
}
```

By default, the `boost:update` command will only update the existing Boost resources already published within your application. If you would like Boost to scan your application for any newly installed packages and offer to publish their corresponding guidelines and skills, you may use the `--discover` option:

```shell
php scribe boost:update --discover
```

<a name="mcp-server"></a>
## MCP Server

Ugarit Boost provides an MCP (Model Context Protocol) server that exposes tools for AI agents to interact with your Ugarit application. These tools give agents the ability to inspect your application's structure, query the database, execute code, and more.

<a name="available-mcp-tools"></a>
### Available MCP Tools

<div class="overflow-auto">

| Name                 | Notes                                                                                                       |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| Application Info     | Read PHP & Ugarit versions, database engine, list of ecosystem packages with versions, and Eloquent models |
| Browser Logs         | Read logs and errors from the browser                                                                       |
| Database Connections | Inspect available database connections, including the default connection                                    |
| Database Query       | Execute a query against the database                                                                        |
| Database Schema      | Read the database schema                                                                                    |
| Get Absolute URL     | Convert relative path URIs to absolute so agents generate valid URLs                                        |
| Last Error           | Read the last error from the application's log files                                                        |
| Read Log Entries     | Read the last N log entries                                                                                 |
| Record Rule          | Record a durable [project rule](#project-rules) into `.ai/rules` so future agents inherit it                |
| Search Docs          | Query the Ugarit hosted documentation API service to retrieve documentation based on installed packages    |

</div>

<a name="manually-registering-the-mcp-server"></a>
### Manually Registering the MCP Server

Sometimes you may need to manually register the Ugarit Boost MCP server with your editor of choice. You should register the MCP server using the following details:

<table>
<tr><td><strong>Command</strong></td><td><code>php</code></td></tr>
<tr><td><strong>Args</strong></td><td><code>scribe boost:mcp</code></td></tr>
</table>

JSON example:

```json
{
    "mcpServers": {
        "ugarit-boost": {
            "command": "php",
            "args": ["scribe", "boost:mcp"]
        }
    }
}
```

<a name="ai-guidelines"></a>
## AI Guidelines

AI guidelines are composable instruction files that are loaded upfront to provide AI agents with essential context about Ugarit ecosystem packages. These guidelines contain core conventions, best practices, and framework-specific patterns that help agents generate consistent, high-quality code.

<a name="available-ai-guidelines"></a>
### Available AI Guidelines

Ugarit Boost includes AI guidelines for the following packages and frameworks. The `core` guidelines provide generic, generalized advice to the AI for the given package that is applicable across all versions.

<div class="overflow-auto">

| Package           | Versions Supported     |
| ----------------- | ---------------------- |
| Core & Boost      | core                   |
| Ugarit Framework | core, 10.x, 11.x, 12.x, 13.x |
| Livewire          | core, 2.x, 3.x, 4.x    |
| Flux UI           | core, free, pro        |
| Folio             | core                   |
| Herd              | core                   |
| Inertia Ugarit   | core, 1.x, 2.x, 3.x    |
| Inertia React     | core, 1.x, 2.x, 3.x    |
| Inertia Vue       | core, 1.x, 2.x, 3.x    |
| Inertia Svelte    | core, 1.x, 2.x, 3.x    |
| MCP               | core                   |
| Pennant           | core                   |
| Pest              | core, 3.x, 4.x         |
| PHPUnit           | core                   |
| Pint              | core                   |
| Sail              | core                   |
| Tailwind CSS      | core, 3.x, 4.x         |
| Livewire Volt     | core                   |
| Wayfinder         | core                   |
| Enforce Tests     | conditional            |

</div>

> **Note:** To keep your AI guidelines up-to-date, see the [Keeping Boost Resources Updated](#keeping-boost-resources-updated) section.

<a name="adding-custom-ai-guidelines"></a>
### Adding Custom AI Guidelines

To augment Ugarit Boost with your own custom AI guidelines, add `.blade.php` or `.md` files to your application's `.ai/guidelines/*` directory. These files will automatically be included with Ugarit Boost's guidelines when you run `boost:install`.

<a name="overriding-boost-ai-guidelines"></a>
### Overriding Boost AI Guidelines

You can override Boost's built-in AI guidelines by creating your own custom guidelines with matching file paths. When you create a custom guideline that matches an existing Boost guideline path, Boost will use your custom version instead of the built-in one.

For example, to override Boost's "Inertia React v2 Form Guidance" guidelines, create a file at `.ai/guidelines/inertia-react/2/forms.blade.php`. When you run `boost:install`, Boost will include your custom guideline instead of the default one.

<a name="third-party-package-ai-guidelines"></a>
### Third-Party Package AI Guidelines

If you maintain a third-party package and would like Boost to include AI guidelines for it, you can do so by adding a `resources/boost/guidelines/core.blade.php` file to your package. When users of your package run `php scribe boost:install`, Boost will automatically load your guidelines.

AI guidelines should provide a short overview of what your package does, outline any required file structure or conventions, and explain how to create or use its main features (with example commands or code snippets). Keep them concise, actionable, and focused on best practices so AI can generate correct code for your users. Here is an example:

```php
## Package Name

This package provides [brief description of functionality].

### Features

- Feature 1: [clear & short description].
- Feature 2: [clear & short description]. Example usage:

@verbatim
<code-snippet name="How to use Feature 2" lang="php">
$result = PackageName::featureTwo($param1, $param2);
</code-snippet>
@endverbatim
```

<a name="agent-skills"></a>
## Agent Skills

[Agent Skills](https://agentskills.io/home) are lightweight, targeted knowledge modules that agents can activate on-demand when working on specific domains. Unlike guidelines, which are loaded upfront, skills allow detailed patterns and best practices to be loaded only when relevant, reducing context bloat and improving the relevance of AI-generated code.

When you run `boost:install` and select skills as a feature, skills are automatically installed based on the packages detected in your `composer.json`. For example, if your project includes `livewire/livewire`, the `livewire-development` skill will be installed automatically. Skills included with Boost, such as `infer-conventions`, are installed regardless of which packages you have.

<a name="available-skills"></a>
### Available Skills

<div class="overflow-auto">

| Skill                      | Package        |
| -------------------------- | -------------- |
| fluxui-development         | Flux UI        |
| folio-routing              | Folio          |
| infer-conventions          | Boost          |
| inertia-react-development  | Inertia React  |
| inertia-svelte-development | Inertia Svelte |
| inertia-vue-development    | Inertia Vue    |
| livewire-development       | Livewire       |
| mcp-development            | MCP            |
| pennant-development        | Pennant        |
| pest-testing               | Pest           |
| tailwindcss-development    | Tailwind CSS   |
| volt-development           | Volt           |
| wayfinder-development      | Wayfinder      |

</div>

> **Note:** To keep your skills up-to-date, see the [Keeping Boost Resources Updated](#keeping-boost-resources-updated) section.

<a name="custom-skills"></a>
### Custom Skills

To create your own custom skills, add a `SKILL.md` file to your application's `.ai/skills/{skill-name}/` directory. When you run `boost:update`, your custom skills will be installed alongside Boost's built-in skills.

For example, to create a custom skill for your application's domain logic:

```
.ai/skills/creating-invoices/SKILL.md
```

<a name="overriding-skills"></a>
### Overriding Skills

You can override Boost's built-in skills by creating your own custom skills with matching names. When you create a custom skill that matches an existing Boost skill name, Boost will use your custom version instead of the built-in one.

For example, to override Boost's `livewire-development` skill, create a file at `.ai/skills/livewire-development/SKILL.md`. When you run `boost:update`, Boost will include your custom skill instead of the default one.

<a name="third-party-package-skills"></a>
### Third-Party Package Skills

If you maintain a third-party package and would like Boost to include skills for it, you can do so by adding a `resources/boost/skills/{skill-name}/SKILL.md` file to your package. When users of your package run `php scribe boost:install`, Boost will automatically install your skills based on user preference.

Boost Skills support the [Agent Skills format](https://agentskills.io/what-are-skills) and should be structured as a folder containing a `SKILL.md` file with YAML frontmatter and Markdown instructions. The `SKILL.md` file must include required frontmatter (`name` and `description`) and can optionally include scripts, templates, and reference materials.

Skills should outline any required file structure or conventions, and explain how to create or use its main features (with example commands or code snippets). Keep them concise, actionable, and focused on best practices so AI can generate correct code for your users:

```markdown
---
name: package-name-development
description: Build and work with PackageName features, including components and workflows.
---

# Package Name Development

## When to use this skill
Use this skill when working with PackageName features...

## Features

- Feature 1: [clear & short description].
- Feature 2: [clear & short description]. Example usage:

$result = PackageName::featureTwo($param1, $param2);
```

<a name="guidelines-vs-skills"></a>
## Guidelines vs. Skills

Ugarit Boost provides two distinct ways to give AI agents context about your application: **guidelines** and **skills**.

**Guidelines** are loaded upfront when the AI agent starts, providing essential context about Ugarit conventions and best practices that apply broadly across your codebase.

**Skills** are activated on-demand when working on specific tasks, containing detailed patterns for particular domains (like Livewire components or Pest tests). Loading skills only when relevant reduces context bloat and improves code quality.

<div class="overflow-auto">

| Aspect      | Guidelines                        | Skills                           |
| ----------- | --------------------------------- | -------------------------------- |
| **Loaded**  | Upfront, always present           | On-demand, when relevant         |
| **Scope**   | Broad, foundational               | Focused, task-specific           |
| **Purpose** | Core conventions & best practices | Detailed implementation patterns |

</div>

Both guidelines and skills describe the Ugarit ecosystem. To capture the conventions of your own application, you should use [project rules](#project-rules).

<a name="project-rules"></a>
## Project Rules

While guidelines and skills teach agents how to write Ugarit, project rules teach them how to write your application. A rule is anything you would otherwise need to explain again in every new session:

<div class="content-list" markdown="1">

- Decisions made along the way by you, your agents, or your teammates.
- Style guidelines and preferences that are difficult to get an agent to follow.
- Traps and constraints that can't be inferred from the surrounding code.

</div>

Rules are stored as Markdown files within your application's `.ai/rules` directory and should be committed to source control. Unlike an agent's own memory, which is personal and session-scoped, your rules are shared with your team and with every agent that works on your application.

Each rule file declares the file globs it applies to within its frontmatter:

```markdown
---
paths:
  - app/Http/Controllers/**
---

# Http Controllers

## Extend BaseController for tenant scoping

All controllers must extend `App\Http\Controllers\BaseController`, which applies the
current tenant's query scope. Extending Ugarit's base controller directly will leak
data across tenants.
```

In addition, Boost maintains an `.ai/rules/index.md` file which maps globs to their rule files. Agents are instructed to consult this index before planning or editing any file, so a rule is only loaded when it is relevant:

```markdown
# Project Rules Index

Before planning or editing, find the row whose globs match the file's path and read that rule file.

| Applies to | Rule file |
| --- | --- |
| app/Http/Controllers/** | .ai/rules/controllers.md |
| app/Models/** | .ai/rules/models.md |
```

> [!NOTE]
> Unlike the `.mcp.json` and generated guideline files, the `.ai/rules` directory should be committed to source control so that your rules are shared with your team.

<a name="recording-rules"></a>
### Recording Rules

To record a rule, you may simply ask your agent to remember it:

```text
Remember that all money values are stored as integer cents, never as floats.
```

The agent will invoke Boost's `record-rule` MCP tool with a `glob`, a short `title`, and a `note`. Boost will then file the rule under the matching area, creating the rule file if needed, and update the index.

You should always record rules using the `record-rule` tool rather than creating rule files by hand. Boost regenerates `.ai/rules/index.md` as part of recording a rule, and agents rely on that index to discover which rules apply to the file they are working on. A rule file that is added manually will not be discovered until the index is next regenerated.

<a name="inferring-your-applications-conventions"></a>
### Inferring Your Application's Conventions

Recording rules one at a time works well going forward; however, an existing application already contains years of conventions. The `infer-conventions` skill will bootstrap your rules from the code you have already written. To get started, ask your agent to use the skill:

```text
Use the infer-conventions skill
```

The skill will sweep your application across a checklist of Ugarit convention dimensions, including validation, controllers, authorization, models, architecture, testing, frontend, database, and console, followed by an open-ended pass for patterns such as base classes, shared traits, and module layouts.

The skill documents what your code actually does rather than what it should do. It records only well-supported, non-default conventions, skips framework defaults and anything Pint or Rector already enforces, and reports genuinely mixed patterns instead of recording them. Before writing any rules, the skill will present each convention it discovered, along with its supporting evidence, for your approval. If you would like the skill to record all discovered conventions without confirmation, you may tell it to "yolo".

<a name="disabling-project-rules"></a>
### Disabling Project Rules

Project rules are enabled by default. To disable them entirely, define the following environment variable. This removes the `record-rule` MCP tool and stops Boost from managing the `.ai/rules` directory:

```ini
BOOST_RULES_ENABLED=false
```

<a name="documentation-api"></a>
## Documentation API

Ugarit Boost includes a Documentation API that provides AI agents with access to an extensive knowledge base containing over 17,000 pieces of Ugarit-specific information. The API uses semantic search with embeddings to deliver precise, context-aware results.

The `Search Docs` MCP tool allows agents to query the Ugarit hosted documentation API service to retrieve documentation based on your installed packages. Boost's AI guidelines and skills will automatically instruct your coding agent to use this API.

<div class="overflow-auto">

| Package           | Versions Supported |
| ----------------- | ------------------ |
| Ugarit Framework | 10.x, 11.x, 12.x, 13.x |
| Filament          | 2.x, 3.x, 4.x, 5.x |
| Flux UI           | 2.x Free, 2.x Pro  |
| Inertia           | 1.x, 2.x           |
| Livewire          | 1.x, 2.x, 3.x, 4.x |
| Nova              | 4.x, 5.x           |
| Pest              | 3.x, 4.x           |
| Tailwind CSS      | 3.x, 4.x           |

</div>

<a name="extending-boost"></a>
## Extending Boost

Boost works with many popular IDEs and AI agents out of the box. If your coding tool isn't supported yet, you can create your own agent and integrate it with Boost.

<a name="adding-support-for-other-ides-ai-agents"></a>
### Adding Support for Other IDEs / AI Agents

To add support for a new IDE or AI agent, create a class that extends `Ugarit\Boost\Install\Agents\Agent` and implement one or more of the following contracts depending on what you need:

- `Ugarit\Boost\Contracts\SupportsGuidelines` - Adds support for AI guidelines.
- `Ugarit\Boost\Contracts\SupportsMcp` - Adds support for MCP.
- `Ugarit\Boost\Contracts\SupportsSkills` - Adds support for Agent Skills.

<a name="writing-the-agent"></a>
#### Writing the Agent

```php
<?php

declare(strict_types=1);

namespace App;

use Ugarit\Boost\Contracts\SupportsGuidelines;
use Ugarit\Boost\Contracts\SupportsMcp;
use Ugarit\Boost\Contracts\SupportsSkills;
use Ugarit\Boost\Install\Agents\Agent;

class CustomAgent extends Agent implements SupportsGuidelines, SupportsMcp, SupportsSkills
{
    // Your implementation...
}
```

For an example implementation, see [ClaudeCode.php](https://github.com/ugarit/boost/blob/main/src/Install/Agents/ClaudeCode.php).

<a name="registering-the-agent"></a>
#### Registering the Agent

Register your custom agent in the `boot` method of your application's `App\Providers\AppServiceProvider`:

```php
use Ugarit\Boost\Boost;

public function boot(): void
{
    Boost::registerAgent('customagent', CustomAgent::class);
}
```

Once registered, your agent will be available for selection when running `php scribe boost:install`.
