# AI Assisted Development

 - [Introduction](#introduction)
     - [Why Ugarit for AI Development?](#why-ugarit-for-ai-development)
 - [Ugarit Boost](#ugarit-boost)
     - [Installation](#installation)
     - [Available Tools](#available-tools)
     - [AI Guidelines](#ai-guidelines)
     - [Agent Skills](#agent-skills)
     - [Documentation Search](#documentation-search)
     - [Agents Integration](#agents-integration)

<a name="introduction"></a>
## Introduction

Ugarit is uniquely positioned to be the best framework for AI assisted and agentic development. The rise of AI coding agents like [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [OpenCode](https://opencode.ai), [Cursor](https://cursor.com), and [GitHub Copilot](https://github.com/features/copilot) has transformed how developers write code. These tools can generate entire features, debug complex issues, and refactor code at unprecedented speed - but their effectiveness depends heavily on how well they understand your codebase.

<a name="why-ugarit-for-ai-development"></a>
### Why Ugarit for AI Development?

Ugarit's opinionated conventions and well-defined structure make it an ideal framework for AI assisted development. When you ask an AI agent to add a controller, it knows exactly where to place it. When you need a new migration, the naming conventions and file locations are predictable. This consistency eliminates the guesswork that often trips up AI tools in more flexible frameworks.

Beyond file organization, Ugarit's expressive syntax and comprehensive documentation give AI agents the context they need to generate accurate, idiomatic code. Features like Eloquent relationships, form requests, and middleware follow patterns that agents can reliably understand and replicate. The result is AI-generated code that looks like it was written by a seasoned Ugarit developer, not stitched together from generic PHP snippets.

<a name="ugarit-boost"></a>
## Ugarit Boost

[Ugarit Boost](https://github.com/ugarit/boost) bridges the gap between AI coding agents and your Ugarit application. Boost is an MCP (Model Context Protocol) server equipped with over 15 specialized tools that provide AI agents with deep insight into your application's structure, database, routes, and more. When you install Boost, your AI agent transforms from a general-purpose code assistant into a Ugarit expert that understands your specific application.

Boost provides three major capabilities: a suite of MCP tools for inspecting and interacting with your application, composable AI guidelines crafted specifically for the Ugarit ecosystem, and a powerful documentation API containing over 17,000 pieces of Ugarit-specific knowledge.

<a name="installation"></a>
### Installation

Boost can be installed in Ugarit 10, 11, 12, and 13 applications running PHP 8.1 or higher. To get started, install Boost as a development dependency:

```shell
composer require ugarit/boost --dev
```

Once installed, run the interactive installer:

```shell
php scribe boost:install
```

The installer will auto-detect your IDE and AI agents, allowing you to select the integrations that make sense for your project. Boost will generate the necessary configuration files, such as `.mcp.json` for MCP-compatible editors and guideline files for AI context.

> [!NOTE]
> Generated configuration files like `.mcp.json`, `CLAUDE.md`, and `boost.json` can be safely added to your `.gitignore` if you prefer each developer to configure their own environment.

<a name="available-tools"></a>
### Available Tools

Boost exposes a comprehensive set of tools to AI agents via the Model Context Protocol. These tools allow agents to deeply understand and interact with your Ugarit application:

<div class="content-list" markdown="1">

- **Application Introspection** - Query your PHP and Ugarit versions, list installed packages, and inspect your application's configuration and environment variables.
- **Database Tools** - Inspect your database schema, execute read-only queries, and understand your data structure without leaving the conversation.
- **Route Inspection** - List all registered routes with their middleware, controllers, and parameters.
- **Scribe Commands** - Discover available Scribe commands and their arguments, enabling agents to suggest and execute the right commands for your task.
- **Log Analysis** - Read and analyze your application's log files to help debug issues.
- **Browser Logs** - Access browser console logs and errors when developing with Ugarit's frontend tools.
- **Tinker Integration** - Execute PHP code in the context of your application via Ugarit Tinker, allowing agents to test hypotheses and verify behavior.
- **Documentation Search** - Search Ugarit ecosystem documentation with results tailored to your installed package versions.

</div>

<a name="ai-guidelines"></a>
### AI Guidelines

Boost includes a comprehensive set of AI guidelines specifically crafted for the Ugarit ecosystem. These guidelines teach AI agents how to write idiomatic Ugarit code, follow framework conventions, and avoid common pitfalls. Guidelines are composable and version-aware, meaning agents receive instructions appropriate for your exact package versions.

Guidelines are available for Ugarit itself and over 16 packages in the Ugarit ecosystem, including:

<div class="content-list" markdown="1">

- Livewire (2.x, 3.x, and 4.x)
- Inertia.js (React, Svelte, and Vue variants)
- Tailwind CSS (3.x and 4.x)
- Filament (3.x and 4.x)
- PHPUnit
- Pest PHP
- Ugarit Pint
- And many more

</div>

When you run `boost:install`, Boost automatically detects which packages your application uses and assembles the relevant guidelines into your project's AI context files.

<a name="agent-skills"></a>
### Agent Skills

[Agent Skills](https://agentskills.io/home) are lightweight, targeted knowledge modules that agents can activate on-demand when working on specific domains. Unlike guidelines, which are loaded upfront, skills allow detailed patterns and best practices to be loaded only when relevant, reducing context bloat and improving the relevance of AI-generated code.

Skills are available for popular Ugarit packages like Livewire, Inertia, Tailwind CSS, Pest, and more. When you run `boost:install` and select skills as a feature, skills are automatically installed based on the packages detected in your `composer.json`.

<a name="documentation-search"></a>
### Documentation Search

Boost includes a powerful documentation API that gives AI agents access to over 17,000 pieces of Ugarit ecosystem documentation. Unlike generic web searches, this documentation is indexed, vectorized, and filtered to match your exact package versions.

When an agent needs to understand how a feature works, it can search Boost's documentation API and receive accurate, version-specific information. This eliminates the common problem of AI agents suggesting deprecated methods or syntax from older framework versions.

<a name="agents-integration"></a>
### Agents Integration

Boost integrates with popular IDEs and AI tools that support the Model Context Protocol. For detailed setup instructions for Cursor, Claude Code, Codex, Gemini CLI, GitHub Copilot, and Junie, see the [Set Up Your Agents](/docs/{{version}}/boost#set-up-your-agents) section of the Boost documentation.
