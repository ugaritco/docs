# Middleware

- [Introduction](#introduction)
- [Defining Middleware](#defining-middleware)
- [Registering Middleware](#registering-middleware)
    - [Global Middleware](#global-middleware)
    - [Assigning Middleware to Routes](#assigning-middleware-to-routes)
    - [Middleware Groups](#middleware-groups)
    - [Middleware Aliases](#middleware-aliases)
    - [Sorting Middleware](#sorting-middleware)
- [Middleware Parameters](#middleware-parameters)
- [Terminable Middleware](#terminable-middleware)

<a name="introduction"></a>
## Introduction

Middleware provide a convenient mechanism for inspecting and filtering HTTP requests entering your application. For example, Ugarit includes a middleware that verifies the user of your application is authenticated. If the user is not authenticated, the middleware will redirect the user to your application's login screen. However, if the user is authenticated, the middleware will allow the request to proceed further into the application.

Additional middleware can be written to perform a variety of tasks besides authentication. For example, a logging middleware might log all incoming requests to your application. A variety of middleware are included in Ugarit, including middleware for authentication and CSRF protection; however, all user-defined middleware are typically located in your application's `app/Http/Middleware` directory.

<a name="defining-middleware"></a>
## Defining Middleware

To create a new middleware, use the `make:middleware` Scribe command:

```shell
php scribe make:middleware EnsureTokenIsValid
```

This command will place a new `EnsureTokenIsValid` class within your `app/Http/Middleware` directory. In this middleware, we will only allow access to the route if the supplied `token` input matches a specified value. Otherwise, we will redirect the users back to the `/home` URI:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Heritage\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Heritage\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('/home');
        }

        return $next($request);
    }
}
```

As you can see, if the given `token` does not match our secret token, the middleware will return an HTTP redirect to the client; otherwise, the request will be passed further into the application. To pass the request deeper into the application (allowing the middleware to "pass"), you should call the `$next` callback with the `$request`.

It's best to envision middleware as a series of "layers" HTTP requests must pass through before they hit your application. Each layer can examine the request and even reject it entirely.

> [!NOTE]
> All middleware are resolved via the [service container](/docs/{{version}}/container), so you may type-hint any dependencies you need within a middleware's constructor.

<a name="middleware-and-responses"></a>
#### Middleware and Responses

Of course, a middleware can perform tasks before or after passing the request deeper into the application. For example, the following middleware would perform some task **before** the request is handled by the application:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Heritage\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class BeforeMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        // Perform action

        return $next($request);
    }
}
```

However, this middleware would perform its task **after** the request is handled by the application:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Heritage\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AfterMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Perform action

        return $response;
    }
}
```

<a name="registering-middleware"></a>
## Registering Middleware

<a name="global-middleware"></a>
### Global Middleware

If you want a middleware to run during every HTTP request to your application, you may append it to the global middleware stack in your application's `bootstrap/app.php` file:

```php
use App\Http\Middleware\EnsureTokenIsValid;

->withMiddleware(function (Middleware $middleware): void {
     $middleware->append(EnsureTokenIsValid::class);
})
```

The `$middleware` object provided to the `withMiddleware` closure is an instance of `Heritage\Foundation\Configuration\Middleware` and is responsible for managing the middleware assigned to your application's routes. The `append` method adds the middleware to the end of the list of global middleware. If you would like to add a middleware to the beginning of the list, you should use the `prepend` method.

<a name="manually-managing-ugarits-default-global-middleware"></a>
#### Manually Managing Ugarit's Default Global Middleware

If you would like to manage Ugarit's global middleware stack manually, you may provide Ugarit's default stack of global middleware to the `use` method. Then, you may adjust the default middleware stack as necessary:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->use([
        \Heritage\Foundation\Http\Middleware\InvokeDeferredCallbacks::class,
        // \Heritage\Http\Middleware\TrustHosts::class,
        \Heritage\Http\Middleware\TrustProxies::class,
        \Heritage\Http\Middleware\HandleCors::class,
        \Heritage\Foundation\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Heritage\Http\Middleware\ValidatePostSize::class,
        \Heritage\Foundation\Http\Middleware\TrimStrings::class,
        \Heritage\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ]);
})
```

<a name="assigning-middleware-to-routes"></a>
### Assigning Middleware to Routes

If you would like to assign middleware to specific routes, you may invoke the `middleware` method when defining the route:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::get('/profile', function () {
    // ...
})->middleware(EnsureTokenIsValid::class);
```

You may assign multiple middleware to the route by passing an array of middleware names to the `middleware` method:

```php
Route::get('/', function () {
    // ...
})->middleware([First::class, Second::class]);
```

<a name="excluding-middleware"></a>
#### Excluding Middleware

When assigning middleware to a group of routes, you may occasionally need to prevent the middleware from being applied to an individual route within the group. You may accomplish this using the `withoutMiddleware` method:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', function () {
        // ...
    });

    Route::get('/profile', function () {
        // ...
    })->withoutMiddleware([EnsureTokenIsValid::class]);
});
```

You may also exclude a given set of middleware from an entire [group](/docs/{{version}}/routing#route-groups) of route definitions:

```php
use App\Http\Middleware\EnsureTokenIsValid;

Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () {
        // ...
    });
});
```

The `withoutMiddleware` method can only remove route middleware and does not apply to [global middleware](#global-middleware).

<a name="middleware-groups"></a>
### Middleware Groups

Sometimes you may want to group several middleware under a single key to make them easier to assign to routes. You may accomplish this using the `appendToGroup` method within your application's `bootstrap/app.php` file:

```php
use App\Http\Middleware\First;
use App\Http\Middleware\Second;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->appendToGroup('group-name', [
        First::class,
        Second::class,
    ]);

    $middleware->prependToGroup('group-name', [
        First::class,
        Second::class,
    ]);
})
```

Middleware groups may be assigned to routes and controller actions using the same syntax as individual middleware:

```php
Route::get('/', function () {
    // ...
})->middleware('group-name');

Route::middleware(['group-name'])->group(function () {
    // ...
});
```

<a name="ugarits-default-middleware-groups"></a>
#### Ugarit's Default Middleware Groups

Ugarit includes predefined `web` and `api` middleware groups that contain common middleware you may want to apply to your web and API routes. Remember, Ugarit automatically applies these middleware groups to the corresponding `routes/web.php` and `routes/api.php` files:

<div class="overflow-auto">

| The `web` Middleware Group                                |
| --------------------------------------------------------- |
| `Heritage\Cookie\Middleware\EncryptCookies`             |
| `Heritage\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Heritage\Session\Middleware\StartSession`              |
| `Heritage\View\Middleware\ShareErrorsFromSession`       |
| `Heritage\Foundation\Http\Middleware\PreventRequestForgery` |
| `Heritage\Routing\Middleware\SubstituteBindings`        |

</div>

<div class="overflow-auto">

| The `api` Middleware Group                         |
| -------------------------------------------------- |
| `Heritage\Routing\Middleware\SubstituteBindings` |

</div>

If you would like to append or prepend middleware to these groups, you may use the `web` and `api` methods within your application's `bootstrap/app.php` file. The `web` and `api` methods are convenient alternatives to the `appendToGroup` method:

```php
use App\Http\Middleware\EnsureTokenIsValid;
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        EnsureUserIsSubscribed::class,
    ]);

    $middleware->api(prepend: [
        EnsureTokenIsValid::class,
    ]);
})
```

You may even replace one of Ugarit's default middleware group entries with a custom middleware of your own:

```php
use App\Http\Middleware\StartCustomSession;
use Heritage\Session\Middleware\StartSession;

$middleware->web(replace: [
    StartSession::class => StartCustomSession::class,
]);
```

Or, you may remove a middleware entirely:

```php
$middleware->web(remove: [
    StartSession::class,
]);
```

<a name="manually-managing-ugarits-default-middleware-groups"></a>
#### Manually Managing Ugarit's Default Middleware Groups

If you would like to manually manage all of the middleware within Ugarit's default `web` and `api` middleware groups, you may redefine the groups entirely. The example below will define the `web` and `api` middleware groups with their default middleware, allowing you to customize them as necessary:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->group('web', [
        \Heritage\Cookie\Middleware\EncryptCookies::class,
        \Heritage\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Heritage\Session\Middleware\StartSession::class,
        \Heritage\View\Middleware\ShareErrorsFromSession::class,
        \Heritage\Foundation\Http\Middleware\PreventRequestForgery::class,
        \Heritage\Routing\Middleware\SubstituteBindings::class,
        // \Heritage\Session\Middleware\AuthenticateSession::class,
    ]);

    $middleware->group('api', [
        // \Ugarit\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        // 'throttle:api',
        \Heritage\Routing\Middleware\SubstituteBindings::class,
    ]);
})
```

> [!NOTE]
> By default, the `web` and `api` middleware groups are automatically applied to your application's corresponding `routes/web.php` and `routes/api.php` files by the `bootstrap/app.php` file.

<a name="middleware-aliases"></a>
### Middleware Aliases

You may assign aliases to middleware in your application's `bootstrap/app.php` file. Middleware aliases allow you to define a short alias for a given middleware class, which can be especially useful for middleware with long class names:

```php
use App\Http\Middleware\EnsureUserIsSubscribed;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'subscribed' => EnsureUserIsSubscribed::class
    ]);
})
```

Once the middleware alias has been defined in your application's `bootstrap/app.php` file, you may use the alias when assigning the middleware to routes:

```php
Route::get('/profile', function () {
    // ...
})->middleware('subscribed');
```

For convenience, some of Ugarit's built-in middleware are aliased by default. For example, the `auth` middleware is an alias for the `Heritage\Auth\Middleware\Authenticate` middleware. Below is a list of the default middleware aliases:

<div class="overflow-auto">

| Alias              | Middleware                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| `auth`             | `Heritage\Auth\Middleware\Authenticate`                                                                     |
| `auth.basic`       | `Heritage\Auth\Middleware\AuthenticateWithBasicAuth`                                                        |
| `auth.session`     | `Heritage\Session\Middleware\AuthenticateSession`                                                           |
| `cache.headers`    | `Heritage\Http\Middleware\SetCacheHeaders`                                                                  |
| `can`              | `Heritage\Auth\Middleware\Authorize`                                                                        |
| `guest`            | `Heritage\Auth\Middleware\RedirectIfAuthenticated`                                                          |
| `password.confirm` | `Heritage\Auth\Middleware\RequirePassword`                                                                  |
| `precognitive`     | `Heritage\Foundation\Http\Middleware\HandlePrecognitiveRequests`                                            |
| `signed`           | `Heritage\Routing\Middleware\ValidateSignature`                                                             |
| `subscribed`       | `\Spark\Http\Middleware\VerifyBillableIsSubscribed`                                                           |
| `throttle`         | `Heritage\Routing\Middleware\ThrottleRequests` or `Heritage\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified`         | `Heritage\Auth\Middleware\EnsureEmailIsVerified`                                                            |

</div>

<a name="sorting-middleware"></a>
### Sorting Middleware

Rarely, you may need your middleware to execute in a specific order but not have control over their order when they are assigned to the route. In these situations, you may specify your middleware priority using the `priority` method in your application's `bootstrap/app.php` file:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->priority([
        \Heritage\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
        \Heritage\Cookie\Middleware\EncryptCookies::class,
        \Heritage\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Heritage\Session\Middleware\StartSession::class,
        \Heritage\View\Middleware\ShareErrorsFromSession::class,
        \Heritage\Foundation\Http\Middleware\PreventRequestForgery::class,
        \Ugarit\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        \Heritage\Routing\Middleware\ThrottleRequests::class,
        \Heritage\Routing\Middleware\ThrottleRequestsWithRedis::class,
        \Heritage\Routing\Middleware\SubstituteBindings::class,
        \Heritage\Contracts\Auth\Middleware\AuthenticatesRequests::class,
        \Heritage\Auth\Middleware\Authorize::class,
    ]);
})
```

If you would like to add middleware to the existing priority list without replacing it, you may use the `prependToPriorityList` or `appendToPriorityList` methods. The `prependToPriorityList` method inserts the given middleware before another middleware, while the `appendToPriorityList` method inserts it after another middleware:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->prependToPriorityList(
        before: \Heritage\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\EnsureTokenIsValid::class,
    );

    $middleware->appendToPriorityList(
        after: \Heritage\Routing\Middleware\SubstituteBindings::class,
        append: \App\Http\Middleware\EnsureUserIsSubscribed::class,
    );
})
```

The `before` and `after` arguments may also be an array of middleware classes.

<a name="middleware-parameters"></a>
## Middleware Parameters

Middleware can also receive additional parameters. For example, if your application needs to verify that the authenticated user has a given "role" before performing a given action, you could create an `EnsureUserHasRole` middleware that receives a role name as an additional argument.

Additional middleware parameters will be passed to the middleware after the `$next` argument:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Heritage\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Heritage\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next, string $role): Response
    {
        if (! $request->user()->hasRole($role)) {
            // Redirect...
        }

        return $next($request);
    }
}
```

Middleware parameters may be specified when defining the route by separating the middleware name and parameters with a `:`:

```php
use App\Http\Middleware\EnsureUserHasRole;

Route::put('/post/{id}', function (string $id) {
    // ...
})->middleware(EnsureUserHasRole::class.':editor');
```

Multiple parameters may be delimited by commas:

```php
Route::put('/post/{id}', function (string $id) {
    // ...
})->middleware(EnsureUserHasRole::class.':editor,publisher');
```

<a name="terminable-middleware"></a>
## Terminable Middleware

Sometimes a middleware may need to do some work after the HTTP response has been sent to the browser. If you define a `terminate` method on your middleware and your web server is using [FastCGI](https://www.php.net/manual/en/install.fpm.php), the `terminate` method will automatically be called after the response is sent to the browser:

```php
<?php

namespace Heritage\Session\Middleware;

use Closure;
use Heritage\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class TerminatingMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Heritage\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    /**
     * Handle tasks after the response has been sent to the browser.
     */
    public function terminate(Request $request, Response $response): void
    {
        // ...
    }
}
```

The `terminate` method should receive both the request and the response. Once you have defined a terminable middleware, you should add it to the list of routes or global middleware in your application's `bootstrap/app.php` file.

When calling the `terminate` method on your middleware, Ugarit will resolve a fresh instance of the middleware from the [service container](/docs/{{version}}/container). If you would like to use the same middleware instance when the `handle` and `terminate` methods are called, register the middleware with the container using the container's `singleton` method. Typically this should be done in the `register` method of your `AppServiceProvider`:

```php
use App\Http\Middleware\TerminatingMiddleware;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->singleton(TerminatingMiddleware::class);
}
```
