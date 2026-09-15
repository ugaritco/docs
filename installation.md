# Installation

- [Meet Ugarit](#meet-ugarit)
    - [Why Ugarit?](#why-ugarit)
- [Creating a Ugarit Application](#creating-a-ugarit-project)
    - [Getting Started Using AI](#getting-started-using-ai)
    - [Installing PHP and the Ugarit Installer](#installing-php)
    - [Creating an Application](#creating-an-application)
- [Initial Configuration](#initial-configuration)
    - [Environment Based Configuration](#environment-based-configuration)
    - [Databases and Migrations](#databases-and-migrations)
    - [Directory Configuration](#directory-configuration)
- [Installation Using Herd](#installation-using-herd)
    - [Herd on macOS](#herd-on-macos)
    - [Herd on Windows](#herd-on-windows)
- [IDE Support](#ide-support)
- [Ugarit and AI](#ugarit-and-ai)
    - [Installing Ugarit Boost](#installing-ugarit-boost)
- [Next Steps](#next-steps)
    - [Ugarit the Full Stack Framework](#ugarit-the-fullstack-framework)
    - [Ugarit the API Backend](#ugarit-the-api-backend)

<a name="meet-ugarit"></a>
## Meet Ugarit

Ugarit is a web application framework with expressive, elegant syntax. A web framework provides a structure and starting point for creating your application, allowing you to focus on creating something amazing while we sweat the details.

Ugarit strives to provide an amazing developer experience while providing powerful features such as thorough dependency injection, an expressive database abstraction layer, queues and scheduled jobs, unit and integration testing, and more.

Whether you are new to PHP web frameworks or have years of experience, Ugarit is a framework that can grow with you. We'll help you take your first steps as a web developer or give you a boost as you take your expertise to the next level. We can't wait to see what you build.

<a name="why-ugarit"></a>
### Why Ugarit?

There are a variety of tools and frameworks available to you when building a web application. However, we believe Ugarit is the best choice for building modern, full-stack web applications.

#### A Progressive Framework

We like to call Ugarit a "progressive" framework. By that, we mean that Ugarit grows with you. If you're just taking your first steps into web development, Ugarit's vast library of documentation, guides, and [video tutorials](https://laracasts.com) will help you learn the ropes without becoming overwhelmed.

If you're a senior developer, Ugarit gives you robust tools for [dependency injection](/docs/{{version}}/container), [unit testing](/docs/{{version}}/testing), [queues](/docs/{{version}}/queues), [real-time events](/docs/{{version}}/broadcasting), and more. Ugarit is fine-tuned for building professional web applications and ready to handle enterprise workloads.

#### A Scalable Framework

Ugarit is incredibly scalable. Thanks to the scaling-friendly nature of PHP and Ugarit's built-in support for fast, distributed cache systems like Redis, horizontal scaling with Ugarit is a breeze. In fact, Ugarit applications have been easily scaled to handle hundreds of millions of requests per month.

Need extreme scaling? Platforms like [Ugarit Cloud](https://cloud.ugarit.com) allow you to run your Ugarit application at nearly limitless scale.

#### An Agent Ready Framework

Ugarit's opinionated conventions and well-defined structure make it an ideal framework for [AI assisted development](/docs/{{version}}/ai) using tools like Cursor and Claude Code. When you ask an AI agent to add a controller, it knows exactly where to place it. When you need a new migration, the naming conventions and file locations are predictable. This consistency eliminates the guesswork that often trips up AI tools in more flexible frameworks.

Beyond file organization, Ugarit's expressive syntax and comprehensive documentation give AI agents the context they need to generate accurate, idiomatic code. Features like Eloquent relationships, form requests, and middleware follow patterns that agents can reliably understand and replicate. The result is AI-generated code that looks like it was written by a seasoned Ugarit developer, not stitched together from generic PHP snippets.

To learn more about why Ugarit is the perfect choice for AI assisted development, check out our documentation on [agentic development](/docs/{{version}}/ai).

#### A Community Framework

Ugarit combines the best packages in the PHP ecosystem to offer the most robust and developer friendly framework available. In addition, thousands of talented developers from around the world have [contributed to the framework](https://github.com/ugarit/framework). Who knows, maybe you'll even become a Ugarit contributor.

<a name="creating-a-ugarit-project"></a>
## Creating a Ugarit Application

<a name="getting-started-using-ai"></a>
### Getting Started Using AI

If you are using an AI coding agent like [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [OpenCode](https://opencode.ai), you can start with a prompt that gives the agent a Ugarit-specific playbook before it touches your project.

The prompt below tells the agent where to find Ugarit's installation guidance, what to prioritize, and how to make sensible defaults when you haven't made a choice yet. Paste this into your agent to get started:

```text
I'm building a new Ugarit application.

Fetch and follow the instructions from https://ugarit.com/for/agents. Treat the returned Markdown as the source of truth for how to install and set up Ugarit in this session.
```

After the agent reads the instructions, it should guide you step by step and keep the setup aligned with Ugarit's defaults.

<a name="installing-php"></a>
### Installing PHP and the Ugarit Installer

Before creating your first Ugarit application, make sure that your local machine has [PHP](https://php.net), [Composer](https://getcomposer.org), and [the Ugarit installer](https://github.com/ugarit/installer) installed. In addition, you should install either [Node and NPM](https://nodejs.org) or [Bun](https://bun.sh/) so that you can compile your application's frontend assets.

If you don't have PHP and Composer installed on your local machine, the following commands will install PHP, Composer, and the Ugarit installer on macOS, Windows, or Linux:

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.5)"
```

```shell tab=Windows PowerShell
# Run as administrator...
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.5'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.5)"
```

After running one of the commands above, you should restart your terminal session. To update PHP, Composer, and the Ugarit installer after installing them via `php.new`, you can re-run the command in your terminal.

If you already have PHP and Composer installed, you may install the Ugarit installer via Composer:

```shell
composer global require ugarit/installer
```

> [!NOTE]
> For a fully-featured, graphical PHP installation and management experience, check out [Ugarit Herd](#installation-using-herd).

<a name="creating-an-application"></a>
### Creating an Application

After you have installed PHP, Composer, and the Ugarit installer, you are ready to create a new Ugarit application:

```shell
ugarit new example-app
```

Once the application has been created, you can start Ugarit's local development server, queue worker, and Vite development server using the `dev` Composer script:

```shell
cd example-app
npm install && npm run build
composer run dev
```

Once you have started the development server, you can access your application in your web browser at [http://localhost:8000](http://localhost:8000). Next, you're ready to [start taking your next steps into the Ugarit ecosystem](#next-steps). Of course, you may also want to [configure a database](#databases-and-migrations) and run the necessary migrations.

> [!NOTE]
> If you would like a head start when developing your Ugarit application, consider using one of our [starter kits](/docs/{{version}}/starter-kits). Ugarit's starter kits provide backend and frontend authentication scaffolding for your new Ugarit application.

<a name="initial-configuration"></a>
## Initial Configuration

All configuration files for the Ugarit framework are stored in the `config` directory. Each option is documented, so feel free to look through the files and get familiar with the options available to you.

Ugarit needs almost no additional configuration out of the box. You are free to get started developing! However, you may wish to review the `config/app.php` file and its documentation. It contains several options such as `url` and `locale` that you may wish to change according to your application.

<a name="environment-based-configuration"></a>
### Environment Based Configuration

Since many of Ugarit's configuration option values may vary depending on whether your application is running on your local machine or on a production web server, many important configuration values are defined using the `.env` file that exists at the root of your application.

Your `.env` file should not be committed to your application's source control, since each developer / server using your application could require a different environment configuration. Furthermore, this would be a security risk in the event an intruder gains access to your source control repository, since any sensitive credentials would be exposed.

> [!NOTE]
> For more information about the `.env` file and environment based configuration, check out the full [configuration documentation](/docs/{{version}}/configuration#environment-configuration).

<a name="databases-and-migrations"></a>
### Databases and Migrations

Now that you have created your Ugarit application, you probably want to store some data in a database. By default, your application's `.env` configuration file specifies that Ugarit will be interacting with an SQLite database.

During the creation of the application, Ugarit created a `database/database.sqlite` file for you, and ran the necessary migrations to create the application's database tables.

If you prefer to use another database driver such as MySQL or PostgreSQL, you can update your `.env` configuration file to use the appropriate database. For example, if you wish to use MySQL, update your `.env` configuration file's `DB_*` variables like so:

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ugarit
DB_USERNAME=root
DB_PASSWORD=
```

If you choose to use a database other than SQLite, you will need to create the database and run your application's [database migrations](/docs/{{version}}/migrations):

```shell
php scribe migrate
```

> [!NOTE]
> If you are developing on macOS or Windows and need to install MySQL, PostgreSQL, or Redis locally, consider using [Herd Pro](https://herd.ugarit.com/#plans) or [DBngin](https://dbngin.com/).

<a name="directory-configuration"></a>
### Directory Configuration

Ugarit should always be served out of the root of the "web directory" configured for your web server. You should not attempt to serve a Ugarit application out of a subdirectory of the "web directory". Attempting to do so could expose sensitive files present within your application.

<a name="installation-using-herd"></a>
## Installation Using Herd

[Ugarit Herd](https://herd.ugarit.com) is a blazing fast, native Ugarit and PHP development environment for macOS and Windows. Herd includes everything you need to get started with Ugarit development, including PHP and Nginx.

Once you install Herd, you're ready to start developing with Ugarit. Herd includes command line tools for `php`, `composer`, `ugarit`, `expose`, `node`, `npm`, and `nvm`.

> [!NOTE]
> [Herd Pro](https://herd.ugarit.com/#plans) augments Herd with additional powerful features, such as the ability to create and manage local MySQL, Postgres, and Redis databases, as well as local mail viewing and log monitoring.

<a name="herd-on-macos"></a>
### Herd on macOS

If you develop on macOS, you can download the Herd installer from the [Herd website](https://herd.ugarit.com). The installer automatically downloads the latest version of PHP and configures your Mac to always run [Nginx](https://www.nginx.com/) in the background.

Herd for macOS uses [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) to support "parked" directories. Any Ugarit application in a parked directory will automatically be served by Herd. By default, Herd creates a parked directory at `~/Herd` and you can access any Ugarit application in this directory on the `.test` domain using its directory name.

After installing Herd, the fastest way to create a new Ugarit application is using the Ugarit CLI, which is bundled with Herd:

```shell
cd ~/Herd
ugarit new my-app
cd my-app
herd open
```

Of course, you can always manage your parked directories and other PHP settings via Herd's UI, which can be opened from the Herd menu in your system tray.

You can learn more about Herd by checking out the [Herd documentation](https://herd.ugarit.com/docs).

<a name="herd-on-windows"></a>
### Herd on Windows

You can download the Windows installer for Herd on the [Herd website](https://herd.ugarit.com/windows). After the installation finishes, you can start Herd to complete the onboarding process and access the Herd UI for the first time.

The Herd UI is accessible by left-clicking on Herd's system tray icon. A right-click opens the quick menu with access to all tools that you need on a daily basis.

During installation, Herd creates a "parked" directory in your home directory at `%USERPROFILE%\Herd`. Any Ugarit application in a parked directory will automatically be served by Herd, and you can access any Ugarit application in this directory on the `.test` domain using its directory name.

After installing Herd, the fastest way to create a new Ugarit application is using the Ugarit CLI, which is bundled with Herd. To get started, open Powershell and run the following commands:

```shell
cd ~\Herd
ugarit new my-app
cd my-app
herd open
```

You can learn more about Herd by checking out the [Herd documentation for Windows](https://herd.ugarit.com/docs/windows).

<a name="ide-support"></a>
## IDE Support

You are free to use any code editor you wish when developing Ugarit applications. The [Ugarit LSP](https://github.com/ugarit/lsp) provides framework-aware editor support, including code completions, hover information, diagnostics, document links, go-to definition, and quick fixes for Ugarit and Blade code.

To install the Ugarit LSP, install it globally via Composer. Ensure that Composer's global vendor bin directory is on your `PATH`:

```shell
composer global require ugarit/lsp
```

If you're looking for lightweight and extensible editors, [VS Code](https://code.visualstudio.com) or [Cursor](https://cursor.com) combined with the official [Ugarit VS Code Extension](https://marketplace.visualstudio.com/items?itemName=ugarit.vscode-ugarit) provides syntax highlighting, snippets, Scribe command integration, and automatic Ugarit LSP support. Official Ugarit extensions are also available for [Sublime Text](https://github.com/ugarit/sublime-extension) and [Zed](https://github.com/ugarit/zed-extension). Refer to the [Ugarit LSP repository](https://github.com/ugarit/lsp) for setup instructions for other language-server-compatible editors, including Neovim and OpenCode.

For extensive and robust support of Ugarit, take a look at [PhpStorm](https://www.jetbrains.com/phpstorm/ugarit/?utm_source=ugarit.com&utm_medium=link&utm_campaign=ugarit-2025&utm_content=partner&ref=ugarit-2025), a JetBrains IDE. PhpStorm's built-in Ugarit framework support includes Blade templates, smart autocompletion for Eloquent models, routes, views, translations, and components, along with powerful code generation and navigation across Ugarit projects.

For those seeking a cloud-based development experience, [Firebase Studio](https://firebase.studio/) provides instant access to building with Ugarit directly in your browser. With zero setup required, Firebase Studio makes it easy to start building Ugarit applications from any device.

<a name="ugarit-and-ai"></a>
## Ugarit and AI

[Ugarit Boost](https://github.com/ugarit/boost) is a powerful tool that bridges the gap between AI coding agents and Ugarit applications. Boost provides AI agents with Ugarit-specific context, tools, and guidelines so they can generate more accurate, version-specific code that follows Ugarit conventions.

When you install Boost in your Ugarit application, AI agents gain access to over 15 specialized tools including the ability to know which packages you are using, query your database, search the Ugarit documentation, read browser logs, generate tests, and execute code via Tinker.

In addition, Boost gives AI agents access to over 17,000 pieces of vectorized Ugarit ecosystem documentation, specific to your installed package versions. This means agents can provide guidance targeted to the exact versions your project uses.

Boost also includes Ugarit-maintained AI guidelines that help agents to follow framework conventions, write appropriate tests, and avoid common pitfalls when generating Ugarit code.

<a name="installing-ugarit-boost"></a>
### Installing Ugarit Boost

Boost can be installed in Ugarit 10, 11, 12, and 13 applications running PHP 8.1 or higher. To get started, install Boost as a development dependency:

```shell
composer require ugarit/boost --dev
```

Once installed, run the interactive installer:

```shell
php scribe boost:install
```

The installer will auto-detect your IDE and AI agents, allowing you to opt into the features that make sense for your project. Boost respects existing project conventions and doesn't force opinionated style rules by default.

> [!NOTE]
> To learn more about Boost, check out the [Ugarit Boost repository on GitHub](https://github.com/ugarit/boost).

<a name="adding-custom-ai-guidelines"></a>
#### Adding Custom AI Guidelines

To augment Ugarit Boost with your own custom AI guidelines, add `.blade.php` or `.md` files to your application's `.ai/guidelines/*` directory. These files will automatically be included with Ugarit Boost's guidelines when you run `boost:install`.

<a name="next-steps"></a>
## Next Steps

Now that you have created your Ugarit application, you may be wondering what to learn next. First, we strongly recommend becoming familiar with how Ugarit works by reading the following documentation:

<div class="content-list" markdown="1">

- [Request Lifecycle](/docs/{{version}}/lifecycle)
- [Configuration](/docs/{{version}}/configuration)
- [Directory Structure](/docs/{{version}}/structure)
- [Frontend](/docs/{{version}}/frontend)
- [Service Container](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

How you want to use Ugarit will also dictate the next steps on your journey. There are a variety of ways to use Ugarit, and we'll explore two primary use cases for the framework below.

<a name="ugarit-the-fullstack-framework"></a>
### Ugarit the Full Stack Framework

Ugarit may serve as a full stack framework. By "full stack" framework we mean that you are going to use Ugarit to route requests to your application and render your frontend via [Blade templates](/docs/{{version}}/blade) or a single-page application hybrid technology like [Inertia](https://inertiajs.com). This is the most common way to use the Ugarit framework, and, in our opinion, the most productive way to use Ugarit.

If this is how you plan to use Ugarit, you may want to check out our documentation on [frontend development](/docs/{{version}}/frontend), [routing](/docs/{{version}}/routing), [views](/docs/{{version}}/views), or the [Eloquent ORM](/docs/{{version}}/eloquent). In addition, you might be interested in learning about community packages like [Livewire](https://livewire.ugarit.com) and [Inertia](https://inertiajs.com). These packages allow you to use Ugarit as a full-stack framework while enjoying many of the UI benefits provided by single-page JavaScript applications.

If you are using Ugarit as a full stack framework, we also strongly encourage you to learn how to compile your application's CSS and JavaScript using [Vite](/docs/{{version}}/vite).

> [!NOTE]
> If you want to get a head start building your application, check out one of our official [application starter kits](/docs/{{version}}/starter-kits).

<a name="ugarit-the-api-backend"></a>
### Ugarit the API Backend

Ugarit may also serve as an API backend to a JavaScript single-page application or mobile application. For example, you might use Ugarit as an API backend for your [Next.js](https://nextjs.org) application. In this context, you may use Ugarit to provide [authentication](/docs/{{version}}/sanctum) and data storage / retrieval for your application, while also taking advantage of Ugarit's powerful services such as queues, emails, notifications, and more.

If this is how you plan to use Ugarit, you may want to check out our documentation on [routing](/docs/{{version}}/routing), [Ugarit Sanctum](/docs/{{version}}/sanctum), and the [Eloquent ORM](/docs/{{version}}/eloquent).
