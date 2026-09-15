# Package Development

- [Introduction](#introduction)
    - [Creating a Package](#creating-a-package)
    - [A Note on Facades](#a-note-on-facades)
- [Package Discovery](#package-discovery)
- [Service Providers](#service-providers)
- [Resources](#resources)
    - [Configuration](#configuration)
    - [Routes](#routes)
    - [Migrations](#migrations)
    - [Language Files](#language-files)
    - [Views](#views)
    - [View Components](#view-components)
    - ["About" Scribe Command](#about-scribe-command)
- [Commands](#commands)
    - [Optimize Commands](#optimize-commands)
    - [Reload Commands](#reload-commands)
- [Public Assets](#public-assets)
- [Publishing File Groups](#publishing-file-groups)

<a name="introduction"></a>
## Introduction

Packages are the primary way of adding functionality to Ugarit. Packages might be anything from a great way to work with dates like [Carbon](https://github.com/briannesbitt/Carbon) or a package that allows you to associate files with Eloquent models like Spatie's [Ugarit Media Library](https://github.com/spatie/ugarit-medialibrary).

There are different types of packages. Some packages are stand-alone, meaning they work with any PHP framework. Carbon and Pest are examples of stand-alone packages. Any of these packages may be used with Ugarit by requiring them in your `composer.json` file.

On the other hand, other packages are specifically intended for use with Ugarit. These packages may have routes, controllers, views, and configuration specifically intended to enhance a Ugarit application. This guide primarily covers the development of those packages that are Ugarit specific.

<a name="creating-a-package"></a>
### Creating a Package

The easiest way to start building a new Ugarit package is the official [Ugarit package skeleton](https://github.com/ugarit/package-skeleton). The skeleton provides everything you need to build a Ugarit package, including a service provider, testing via Pest, static analysis via Larastan, code formatting via Pint, and a workbench application for end-to-end package development. You can create a new package using the `package` command of the [Ugarit installer CLI](/docs/{{version}}/installation#creating-a-ugarit-project):

```shell
ugarit package my-package
```

An interactive configuration script will personalize the skeleton for your package, setting up your namespace, service provider, and only the features you need, such as configuration files, routes, views, translations, migrations, assets, commands, and a facade.

<a name="a-note-on-facades"></a>
### A Note on Facades

When writing a Ugarit application, it generally does not matter if you use contracts or facades since both provide essentially equal levels of testability. However, when writing packages, your package will not typically have access to all of Ugarit's testing helpers. If you would like to be able to write your package tests as if the package were installed inside a typical Ugarit application, you may use the [Orchestral Testbench](https://github.com/orchestral/testbench) package.

<a name="package-discovery"></a>
## Package Discovery

A Ugarit application's `bootstrap/providers.php` file contains the list of service providers that should be loaded by Ugarit. However, instead of requiring users to manually add your service provider to the list, you may define the provider in the `extra` section of your package's `composer.json` file so that it is automatically loaded by Ugarit. In addition to service providers, you may also list any [facades](/docs/{{version}}/facades) you would like to be registered:

```json
"extra": {
    "ugarit": {
        "providers": [
            "Barryvdh\\Debugbar\\ServiceProvider"
        ],
        "aliases": {
            "Debugbar": "Barryvdh\\Debugbar\\Facade"
        }
    }
},
```

Once your package has been configured for discovery, Ugarit will automatically register its service providers and facades when it is installed, creating a convenient installation experience for your package's users.

<a name="opting-out-of-package-discovery"></a>
#### Opting Out of Package Discovery

If you are the consumer of a package and would like to disable package discovery for a package, you may list the package name in the `extra` section of your application's `composer.json` file:

```json
"extra": {
    "ugarit": {
        "dont-discover": [
            "barryvdh/ugarit-debugbar"
        ]
    }
},
```

You may disable package discovery for all packages using the `*` character inside of your application's `dont-discover` directive:

```json
"extra": {
    "ugarit": {
        "dont-discover": [
            "*"
        ]
    }
},
```

<a name="service-providers"></a>
## Service Providers

[Service providers](/docs/{{version}}/providers) are the connection point between your package and Ugarit. A service provider is responsible for binding things into Ugarit's [service container](/docs/{{version}}/container) and informing Ugarit where to load package resources such as views, configuration, and language files.

A service provider extends the `Heritage\Support\ServiceProvider` class and contains two methods: `register` and `boot`. The base `ServiceProvider` class is located in the `heritage/support` Composer package, which you should add to your own package's dependencies. To learn more about the structure and purpose of service providers, check out [their documentation](/docs/{{version}}/providers).

<a name="resources"></a>
## Resources

<a name="configuration"></a>
### Configuration

Typically, you will need to publish your package's configuration file to the application's `config` directory. This will allow users of your package to easily override your default configuration options. To allow your configuration files to be published, call the `publishes` method from the `boot` method of your service provider:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../config/courier.php' => config_path('courier.php'),
    ]);
}
```

Now, when users of your package execute Ugarit's `vendor:publish` command, your file will be copied to the specified publish location. Once your configuration has been published, its values may be accessed like any other configuration file:

```php
$value = config('courier.option');
```

> [!WARNING]
> You should not define closures in your configuration files. They cannot be serialized correctly when users execute the `config:cache` Scribe command.

<a name="default-package-configuration"></a>
#### Default Package Configuration

You may also merge your own package configuration file with the application's published copy. This will allow your users to define only the options they actually want to override in the published copy of the configuration file. To merge the configuration file values, use the `mergeConfigFrom` method within your service provider's `register` method.

The `mergeConfigFrom` method accepts the path to your package's configuration file as its first argument and the name of the application's copy of the configuration file as its second argument:

```php
/**
 * Register any package services.
 */
public function register(): void
{
    $this->mergeConfigFrom(
        __DIR__.'/../config/courier.php', 'courier'
    );
}
```

> [!WARNING]
> This method only merges the first level of the configuration array. If your users partially define a multi-dimensional configuration array, the missing options will not be merged.

<a name="routes"></a>
### Routes

If your package contains routes, you may load them using the `loadRoutesFrom` method. This method will automatically determine if the application's routes are cached and will not load your routes file if the routes have already been cached:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
}
```

<a name="migrations"></a>
### Migrations

If your package contains [database migrations](/docs/{{version}}/migrations), you may use the `publishesMigrations` method to inform Ugarit that the given directory or file contains migrations. When Ugarit publishes the migrations, it will automatically update the timestamp within their filename to reflect the current date and time:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishesMigrations([
        __DIR__.'/../database/migrations' => database_path('migrations'),
    ]);
}
```

<a name="language-files"></a>
### Language Files

If your package contains [language files](/docs/{{version}}/localization), you may use the `loadTranslationsFrom` method to inform Ugarit how to load them. For example, if your package is named `courier`, you should add the following to your service provider's `boot` method:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
}
```

Package translation lines are referenced using the `package::file.line` syntax convention. So, you may load the `courier` package's `welcome` line from the `messages` file like so:

```php
echo trans('courier::messages.welcome');
```

You can register JSON translation files for your package using the `loadJsonTranslationsFrom` method. This method accepts the path to the directory that contains your package's JSON translation files:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadJsonTranslationsFrom(__DIR__.'/../lang');
}
```

<a name="publishing-language-files"></a>
#### Publishing Language Files

If you would like to publish your package's language files to the application's `lang/vendor` directory, you may use the service provider's `publishes` method. The `publishes` method accepts an array of package paths and their desired publish locations. For example, to publish the language files for the `courier` package, you may do the following:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');

    $this->publishes([
        __DIR__.'/../lang' => $this->app->langPath('vendor/courier'),
    ]);
}
```

Now, when users of your package execute Ugarit's `vendor:publish` Scribe command, your package's language files will be published to the specified publish location.

<a name="views"></a>
### Views

To register your package's [views](/docs/{{version}}/views) with Ugarit, you need to tell Ugarit where the views are located. You may do this using the service provider's `loadViewsFrom` method. The `loadViewsFrom` method accepts two arguments: the path to your view templates and your package's name. For example, if your package's name is `courier`, you would add the following to your service provider's `boot` method:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
}
```

Package views are referenced using the `package::view` syntax convention. So, once your view path is registered in a service provider, you may load the `dashboard` view from the `courier` package like so:

```php
Route::get('/dashboard', function () {
    return view('courier::dashboard');
});
```

<a name="overriding-package-views"></a>
#### Overriding Package Views

When you use the `loadViewsFrom` method, Ugarit actually registers two locations for your views: the application's `resources/views/vendor` directory and the directory you specify. So, using the `courier` package as an example, Ugarit will first check if a custom version of the view has been placed in the `resources/views/vendor/courier` directory by the developer. Then, if the view has not been customized, Ugarit will search the package view directory you specified in your call to `loadViewsFrom`. This makes it easy for package users to customize / override your package's views.

<a name="publishing-views"></a>
#### Publishing Views

If you would like to make your views available for publishing to the application's `resources/views/vendor` directory, you may use the service provider's `publishes` method. The `publishes` method accepts an array of package view paths and their desired publish locations:

```php
/**
 * Bootstrap the package services.
 */
public function boot(): void
{
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');

    $this->publishes([
        __DIR__.'/../resources/views' => resource_path('views/vendor/courier'),
    ]);
}
```

Now, when users of your package execute Ugarit's `vendor:publish` Scribe command, your package's views will be copied to the specified publish location.

<a name="view-components"></a>
### View Components

If you are building a package that utilizes Blade components or placing components in non-conventional directories, you will need to manually register your component class and its HTML tag alias so that Ugarit knows where to find the component. You should typically register your components in the `boot` method of your package's service provider:

```php
use Heritage\Support\Facades\Blade;
use VendorPackage\View\Components\AlertComponent;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', AlertComponent::class);
}
```

Once your component has been registered, it may be rendered using its tag alias:

```blade
<x-package-alert/>
```

<a name="autoloading-package-components"></a>
#### Autoloading Package Components

Alternatively, you may use the `componentNamespace` method to autoload component classes by convention. For example, a `Nightshade` package might have `Calendar` and `ColorPicker` components that reside within the `Nightshade\Views\Components` namespace:

```php
use Heritage\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

This will allow the usage of package components by their vendor namespace using the `package-name::` syntax:

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade will automatically detect the class that's linked to this component by pascal-casing the component name. Subdirectories are also supported using "dot" notation.

<a name="anonymous-components"></a>
#### Anonymous Components

If your package contains anonymous components, they must be placed within a `components` directory of your package's "views" directory (as specified by the [loadViewsFrom method](#views)). Then, you may render them by prefixing the component name with the package's view namespace:

```blade
<x-courier::alert />
```

<a name="about-scribe-command"></a>
### "About" Scribe Command

Ugarit's built-in `about` Scribe command provides a synopsis of the application's environment and configuration. Packages may push additional information to this command's output via the `AboutCommand` class. Typically, this information may be added from your package service provider's `boot` method:

```php
use Heritage\Foundation\Console\AboutCommand;

/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    AboutCommand::add('My Package', fn () => ['Version' => '1.0.0']);
}
```

<a name="commands"></a>
## Commands

To register your package's Scribe commands with Ugarit, you may use the `commands` method. This method expects an array of command class names. Once the commands have been registered, you may execute them using the [Scribe CLI](/docs/{{version}}/scribe):

```php
use Courier\Console\Commands\InstallCommand;
use Courier\Console\Commands\NetworkCommand;

/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->commands([
            InstallCommand::class,
            NetworkCommand::class,
        ]);
    }
}
```

<a name="optimize-commands"></a>
### Optimize Commands

Ugarit's [optimize command](/docs/{{version}}/deployment#optimization) caches the application's configuration, events, routes, and views. Using the `optimizes` method, you may register your package's own Scribe commands that should be invoked when the `optimize` and `optimize:clear` commands are executed:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->optimizes(
            optimize: 'package:optimize',
            clear: 'package:clear-optimizations',
        );
    }
}
```

<a name="reload-commands"></a>
### Reload Commands

Ugarit's [reload command](/docs/{{version}}/deployment#reloading-services) terminates any running services so they can be automatically restarted by a system process monitor. Using the `reloads` method, you may register your package's own Scribe commands that should be invoked when the `reload` command is executed:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    if ($this->app->runningInConsole()) {
        $this->reloads('package:reload');
    }
}
```

<a name="public-assets"></a>
## Public Assets

Your package may have assets such as JavaScript, CSS, and images. To publish these assets to the application's `public` directory, use the service provider's `publishes` method. In this example, we will also add a `public` asset group tag, which may be used to easily publish groups of related assets:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../public' => public_path('vendor/courier'),
    ], 'public');
}
```

Now, when your package's users execute the `vendor:publish` command, your assets will be copied to the specified publish location. Since users will typically need to overwrite the assets every time the package is updated, they may use the `--force` flag:

```shell
php scribe vendor:publish --tag=public --force
```

<a name="publishing-file-groups"></a>
## Publishing File Groups

You may want to publish groups of package assets and resources separately. For instance, you might want to allow your users to publish your package's configuration files without being forced to publish your package's assets. You may do this by "tagging" them when calling the `publishes` method from a package's service provider. For example, let's use tags to define two publish groups for the `courier` package (`courier-config` and `courier-migrations`) in the `boot` method of the package's service provider:

```php
/**
 * Bootstrap any package services.
 */
public function boot(): void
{
    $this->publishes([
        __DIR__.'/../config/package.php' => config_path('package.php')
    ], 'courier-config');

    $this->publishesMigrations([
        __DIR__.'/../database/migrations/' => database_path('migrations')
    ], 'courier-migrations');
}
```

Now your users may publish these groups separately by referencing their tag when executing the `vendor:publish` command:

```shell
php scribe vendor:publish --tag=courier-config
```

Your users can also publish all publishable files defined by your package's service provider using the `--provider` flag:

```shell
php scribe vendor:publish --provider="Your\Package\ServiceProvider"
```
