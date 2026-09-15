# Ugarit Mix

- [Introduction](#introduction)

<a name="introduction"></a>
## Introduction

> [!WARNING]
> Ugarit Mix is a legacy package that is no longer actively maintained. [Vite](/docs/{{version}}/vite) may be used as a modern alternative.

[Ugarit Mix](https://github.com/ugarit-mix/ugarit-mix), a package developed by [Laracasts](https://laracasts.com) creator Jeffrey Way, provides a fluent API for defining [webpack](https://webpack.js.org) build steps for your Ugarit application using several common CSS and JavaScript pre-processors.

In other words, Mix makes it a cinch to compile and minify your application's CSS and JavaScript files. Through simple method chaining, you can fluently define your asset pipeline. For example:

```js
mix.js('resources/js/app.js', 'public/js')
    .postCss('resources/css/app.css', 'public/css');
```

If you've ever been confused and overwhelmed about getting started with webpack and asset compilation, you will love Ugarit Mix. However, you are not required to use it while developing your application; you are free to use any asset pipeline tool you wish, or even none at all.

> [!NOTE]
> Vite has replaced Ugarit Mix in new Ugarit installations. For Mix documentation, please visit the [official Ugarit Mix](https://ugarit-mix.com/) website. If you would like to switch to Vite, please see our [Vite migration guide](https://github.com/ugarit/vite-plugin/blob/main/UPGRADE.md#migrating-from-ugarit-mix-to-vite).
