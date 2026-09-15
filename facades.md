# Facades

- [Introduction](#introduction)
- [When to Utilize Facades](#when-to-use-facades)
    - [Facades vs. Dependency Injection](#facades-vs-dependency-injection)
    - [Facades vs. Helper Functions](#facades-vs-helper-functions)
- [How Facades Work](#how-facades-work)
- [Real-Time Facades](#real-time-facades)
- [Facade Class Reference](#facade-class-reference)

<a name="introduction"></a>
## Introduction

Throughout the Ugarit documentation, you will see examples of code that interacts with Ugarit's features via "facades". Facades provide a "static" interface to classes that are available in the application's [service container](/docs/{{version}}/container). Ugarit ships with many facades which provide access to almost all of Ugarit's features.

Ugarit facades serve as "static proxies" to underlying classes in the service container, providing the benefit of a terse, expressive syntax while maintaining more testability and flexibility than traditional static methods. It's perfectly fine if you don't totally understand how facades work - just go with the flow and continue learning about Ugarit.

All of Ugarit's facades are defined in the `Heritage\Support\Facades` namespace. So, we can easily access a facade like so:

```php
use Heritage\Support\Facades\Cache;
use Heritage\Support\Facades\Route;

Route::get('/cache', function () {
    return Cache::get('key');
});
```

Throughout the Ugarit documentation, many of the examples will use facades to demonstrate various features of the framework.

<a name="helper-functions"></a>
#### Helper Functions

To complement facades, Ugarit offers a variety of global "helper functions" that make it even easier to interact with common Ugarit features. Some of the common helper functions you may interact with are `view`, `response`, `url`, `config`, and more. Each helper function offered by Ugarit is documented with their corresponding feature; however, a complete list is available within the dedicated [helper documentation](/docs/{{version}}/helpers).

For example, instead of using the `Heritage\Support\Facades\Response` facade to generate a JSON response, we may simply use the `response` function. Because helper functions are globally available, you do not need to import any classes in order to use them:

```php
use Heritage\Support\Facades\Response;

Route::get('/users', function () {
    return Response::json([
        // ...
    ]);
});

Route::get('/users', function () {
    return response()->json([
        // ...
    ]);
});
```

<a name="when-to-use-facades"></a>
## When to Utilize Facades

Facades have many benefits. They provide a terse, memorable syntax that allows you to use Ugarit's features without remembering long class names that must be injected or configured manually. Furthermore, because of their unique usage of PHP's dynamic methods, they are easy to test.

However, some care must be taken when using facades. The primary danger of facades is class "scope creep". Since facades are so easy to use and do not require injection, it can be easy to let your classes continue to grow and use many facades in a single class. Using dependency injection, this potential is mitigated by the visual feedback a large constructor gives you that your class is growing too large. So, when using facades, pay special attention to the size of your class so that its scope of responsibility stays narrow. If your class is getting too large, consider splitting it into multiple smaller classes.

<a name="facades-vs-dependency-injection"></a>
### Facades vs. Dependency Injection

One of the primary benefits of dependency injection is the ability to swap implementations of the injected class. This is useful during testing since you can inject a mock or stub and assert that various methods were called on the stub.

Typically, it would not be possible to mock or stub a truly static class method. However, since facades use dynamic methods to proxy method calls to objects resolved from the service container, we actually can test facades just as we would test an injected class instance. For example, given the following route:

```php
use Heritage\Support\Facades\Cache;

Route::get('/cache', function () {
    return Cache::get('key');
});
```

Using Ugarit's facade testing methods, we can write the following test to verify that the `Cache::get` method was called with the argument we expected:

```php tab=Pest
use Heritage\Support\Facades\Cache;

test('basic example', function () {
    Cache::shouldReceive('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
});
```

```php tab=PHPUnit
use Heritage\Support\Facades\Cache;

/**
 * A basic functional test example.
 */
public function test_basic_example(): void
{
    Cache::shouldReceive('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
}
```

<a name="facades-vs-helper-functions"></a>
### Facades vs. Helper Functions

In addition to facades, Ugarit includes a variety of "helper" functions which can perform common tasks like generating views, firing events, dispatching jobs, or sending HTTP responses. Many of these helper functions perform the same function as a corresponding facade. For example, this facade call and helper call are equivalent:

```php
return Heritage\Support\Facades\View::make('profile');

return view('profile');
```

There is absolutely no practical difference between facades and helper functions. When using helper functions, you may still test them exactly as you would the corresponding facade. For example, given the following route:

```php
Route::get('/cache', function () {
    return cache('key');
});
```

The `cache` helper is going to call the `get` method on the class underlying the `Cache` facade. So, even though we are using the helper function, we can write the following test to verify that the method was called with the argument we expected:

```php
use Heritage\Support\Facades\Cache;

/**
 * A basic functional test example.
 */
public function test_basic_example(): void
{
    Cache::shouldReceive('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
}
```

<a name="how-facades-work"></a>
## How Facades Work

In a Ugarit application, a facade is a class that provides access to an object from the container. The machinery that makes this work is in the `Facade` class. Ugarit's facades, and any custom facades you create, will extend the base `Heritage\Support\Facades\Facade` class.

The `Facade` base class makes use of the `__callStatic()` magic-method to defer calls from your facade to an object resolved from the container. In the example below, a call is made to the Ugarit cache system. By glancing at this code, one might assume that the static `get` method is being called on the `Cache` class:

```php
<?php

namespace App\Http\Controllers;

use Heritage\Support\Facades\Cache;
use Heritage\View\View;

class UserController extends Controller
{
    /**
     * Show the profile for the given user.
     */
    public function showProfile(string $id): View
    {
        $user = Cache::get('user:'.$id);

        return view('profile', ['user' => $user]);
    }
}
```

Notice that near the top of the file we are "importing" the `Cache` facade. This facade serves as a proxy for accessing the underlying implementation of the `Heritage\Contracts\Cache\Factory` interface. Any calls we make using the facade will be passed to the underlying instance of Ugarit's cache service.

If we look at that `Heritage\Support\Facades\Cache` class, you'll see that there is no static method `get`:

```php
class Cache extends Facade
{
    /**
     * Get the registered name of the component.
     */
    protected static function getFacadeAccessor(): string
    {
        return 'cache';
    }
}
```

Instead, the `Cache` facade extends the base `Facade` class and defines the method `getFacadeAccessor()`. This method's job is to return the name of a service container binding. When a user references any static method on the `Cache` facade, Ugarit resolves the `cache` binding from the [service container](/docs/{{version}}/container) and runs the requested method (in this case, `get`) against that object.

<a name="real-time-facades"></a>
## Real-Time Facades

Using real-time facades, you may treat any class in your application as if it was a facade. To illustrate how this can be used, let's first examine some code that does not use real-time facades. For example, let's assume our `Podcast` model has a `publish` method. However, in order to publish the podcast, we need to inject a `Publisher` instance:

```php
<?php

namespace App\Models;

use App\Contracts\Publisher;
use Heritage\Database\Eloquent\Model;

class Podcast extends Model
{
    /**
     * Publish the podcast.
     */
    public function publish(Publisher $publisher): void
    {
        $this->update(['publishing' => now()]);

        $publisher->publish($this);
    }
}
```

Injecting a publisher implementation into the method allows us to easily test the method in isolation since we can mock the injected publisher. However, it requires us to always pass a publisher instance each time we call the `publish` method. Using real-time facades, we can maintain the same testability while not being required to explicitly pass a `Publisher` instance. To generate a real-time facade, prefix the namespace of the imported class with `Facades`:

```php
<?php

namespace App\Models;

use App\Contracts\Publisher; // [tl! remove]
use Facades\App\Contracts\Publisher; // [tl! add]
use Heritage\Database\Eloquent\Model;

class Podcast extends Model
{
    /**
     * Publish the podcast.
     */
    public function publish(Publisher $publisher): void // [tl! remove]
    public function publish(): void // [tl! add]
    {
        $this->update(['publishing' => now()]);

        $publisher->publish($this); // [tl! remove]
        Publisher::publish($this); // [tl! add]
    }
}
```

When the real-time facade is used, the publisher implementation will be resolved out of the service container using the portion of the interface or class name that appears after the `Facades` prefix. When testing, we can use Ugarit's built-in facade testing helpers to mock this method call:

```php tab=Pest
<?php

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Heritage\Foundation\Testing\RefreshDatabase;

pest()->use(RefreshDatabase::class);

test('podcast can be published', function () {
    $podcast = Podcast::factory()->create();

    Publisher::shouldReceive('publish')->once()->with($podcast);

    $podcast->publish();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Heritage\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PodcastTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A test example.
     */
    public function test_podcast_can_be_published(): void
    {
        $podcast = Podcast::factory()->create();

        Publisher::shouldReceive('publish')->once()->with($podcast);

        $podcast->publish();
    }
}
```

<a name="facade-class-reference"></a>
## Facade Class Reference

Below you will find every facade and its underlying class. This is a useful tool for quickly digging into the API documentation for a given facade root. The [service container binding](/docs/{{version}}/container) key is also included where applicable.

<div class="overflow-auto">

| Facade | Class | Service Container Binding |
| --- | --- | --- |
| App | [Heritage\Foundation\Application](https://api.ugarit.com/docs/{{version}}/Heritage/Foundation/Application.html) | `app` |
| Scribe | [Heritage\Contracts\Console\Kernel](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Console/Kernel.html) | `scribe` |
| Auth (Instance) | [Heritage\Contracts\Auth\Guard](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Auth/Guard.html) | `auth.driver` |
| Auth | [Heritage\Auth\AuthManager](https://api.ugarit.com/docs/{{version}}/Heritage/Auth/AuthManager.html) | `auth` |
| Blade | [Heritage\View\Compilers\BladeCompiler](https://api.ugarit.com/docs/{{version}}/Heritage/View/Compilers/BladeCompiler.html) | `blade.compiler` |
| Broadcast (Instance) | [Heritage\Contracts\Broadcasting\Broadcaster](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Broadcasting/Broadcaster.html) | &nbsp; |
| Broadcast | [Heritage\Contracts\Broadcasting\Factory](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Broadcasting/Factory.html) | &nbsp; |
| Bus | [Heritage\Contracts\Bus\Dispatcher](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Bus/Dispatcher.html) | &nbsp; |
| Cache (Instance) | [Heritage\Cache\Repository](https://api.ugarit.com/docs/{{version}}/Heritage/Cache/Repository.html) | `cache.store` |
| Cache | [Heritage\Cache\CacheManager](https://api.ugarit.com/docs/{{version}}/Heritage/Cache/CacheManager.html) | `cache` |
| Cloud | [Heritage\Foundation\Cloud\CloudManager](https://api.ugarit.com/docs/{{version}}/Heritage/Foundation/Cloud/CloudManager.html) | &nbsp; |
| Config | [Heritage\Config\Repository](https://api.ugarit.com/docs/{{version}}/Heritage/Config/Repository.html) | `config` |
| Context | [Heritage\Log\Context\Repository](https://api.ugarit.com/docs/{{version}}/Heritage/Log/Context/Repository.html) | &nbsp; |
| Cookie | [Heritage\Cookie\CookieJar](https://api.ugarit.com/docs/{{version}}/Heritage/Cookie/CookieJar.html) | `cookie` |
| Crypt | [Heritage\Encryption\Encrypter](https://api.ugarit.com/docs/{{version}}/Heritage/Encryption/Encrypter.html) | `encrypter` |
| Date | [Heritage\Support\DateFactory](https://api.ugarit.com/docs/{{version}}/Heritage/Support/DateFactory.html) | `date` |
| DB (Instance) | [Heritage\Database\Connection](https://api.ugarit.com/docs/{{version}}/Heritage/Database/Connection.html) | `db.connection` |
| DB | [Heritage\Database\DatabaseManager](https://api.ugarit.com/docs/{{version}}/Heritage/Database/DatabaseManager.html) | `db` |
| Event | [Heritage\Events\Dispatcher](https://api.ugarit.com/docs/{{version}}/Heritage/Events/Dispatcher.html) | `events` |
| Exceptions (Instance) | [Heritage\Contracts\Debug\ExceptionHandler](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Debug/ExceptionHandler.html) | &nbsp; |
| Exceptions | [Heritage\Foundation\Exceptions\Handler](https://api.ugarit.com/docs/{{version}}/Heritage/Foundation/Exceptions/Handler.html) | &nbsp; |
| File | [Heritage\Filesystem\Filesystem](https://api.ugarit.com/docs/{{version}}/Heritage/Filesystem/Filesystem.html) | `files` |
| Gate | [Heritage\Contracts\Auth\Access\Gate](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Auth/Access/Gate.html) | &nbsp; |
| Hash | [Heritage\Contracts\Hashing\Hasher](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Hashing/Hasher.html) | `hash` |
| Http | [Heritage\Http\Client\Factory](https://api.ugarit.com/docs/{{version}}/Heritage/Http/Client/Factory.html) | &nbsp; |
| Lang | [Heritage\Translation\Translator](https://api.ugarit.com/docs/{{version}}/Heritage/Translation/Translator.html) | `translator` |
| Log | [Heritage\Log\LogManager](https://api.ugarit.com/docs/{{version}}/Heritage/Log/LogManager.html) | `log` |
| Mail | [Heritage\Mail\Mailer](https://api.ugarit.com/docs/{{version}}/Heritage/Mail/Mailer.html) | `mailer` |
| Notification | [Heritage\Notifications\ChannelManager](https://api.ugarit.com/docs/{{version}}/Heritage/Notifications/ChannelManager.html) | &nbsp; |
| Password (Instance) | [Heritage\Auth\Passwords\PasswordBroker](https://api.ugarit.com/docs/{{version}}/Heritage/Auth/Passwords/PasswordBroker.html) | `auth.password.broker` |
| Password | [Heritage\Auth\Passwords\PasswordBrokerManager](https://api.ugarit.com/docs/{{version}}/Heritage/Auth/Passwords/PasswordBrokerManager.html) | `auth.password` |
| Pipeline (Instance) | [Heritage\Pipeline\Pipeline](https://api.ugarit.com/docs/{{version}}/Heritage/Pipeline/Pipeline.html) | &nbsp; |
| Process | [Heritage\Process\Factory](https://api.ugarit.com/docs/{{version}}/Heritage/Process/Factory.html) | &nbsp; |
| Queue (Base Class) | [Heritage\Queue\Queue](https://api.ugarit.com/docs/{{version}}/Heritage/Queue/Queue.html) | &nbsp; |
| Queue (Instance) | [Heritage\Contracts\Queue\Queue](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Queue/Queue.html) | `queue.connection` |
| Queue | [Heritage\Queue\QueueManager](https://api.ugarit.com/docs/{{version}}/Heritage/Queue/QueueManager.html) | `queue` |
| RateLimiter | [Heritage\Cache\RateLimiter](https://api.ugarit.com/docs/{{version}}/Heritage/Cache/RateLimiter.html) | &nbsp; |
| Redirect | [Heritage\Routing\Redirector](https://api.ugarit.com/docs/{{version}}/Heritage/Routing/Redirector.html) | `redirect` |
| Redis (Instance) | [Heritage\Redis\Connections\Connection](https://api.ugarit.com/docs/{{version}}/Heritage/Redis/Connections/Connection.html) | `redis.connection` |
| Redis | [Heritage\Redis\RedisManager](https://api.ugarit.com/docs/{{version}}/Heritage/Redis/RedisManager.html) | `redis` |
| Request | [Heritage\Http\Request](https://api.ugarit.com/docs/{{version}}/Heritage/Http/Request.html) | `request` |
| Response (Instance) | [Heritage\Http\Response](https://api.ugarit.com/docs/{{version}}/Heritage/Http/Response.html) | &nbsp; |
| Response | [Heritage\Contracts\Routing\ResponseFactory](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Routing/ResponseFactory.html) | &nbsp; |
| Route | [Heritage\Routing\Router](https://api.ugarit.com/docs/{{version}}/Heritage/Routing/Router.html) | `router` |
| Schedule | [Heritage\Console\Scheduling\Schedule](https://api.ugarit.com/docs/{{version}}/Heritage/Console/Scheduling/Schedule.html) | &nbsp; |
| Schema | [Heritage\Database\Schema\Builder](https://api.ugarit.com/docs/{{version}}/Heritage/Database/Schema/Builder.html) | &nbsp; |
| Session (Instance) | [Heritage\Session\Store](https://api.ugarit.com/docs/{{version}}/Heritage/Session/Store.html) | `session.store` |
| Session | [Heritage\Session\SessionManager](https://api.ugarit.com/docs/{{version}}/Heritage/Session/SessionManager.html) | `session` |
| Storage (Instance) | [Heritage\Contracts\Filesystem\Filesystem](https://api.ugarit.com/docs/{{version}}/Heritage/Contracts/Filesystem/Filesystem.html) | `filesystem.disk` |
| Storage | [Heritage\Filesystem\FilesystemManager](https://api.ugarit.com/docs/{{version}}/Heritage/Filesystem/FilesystemManager.html) | `filesystem` |
| URL | [Heritage\Routing\UrlGenerator](https://api.ugarit.com/docs/{{version}}/Heritage/Routing/UrlGenerator.html) | `url` |
| Validator (Instance) | [Heritage\Validation\Validator](https://api.ugarit.com/docs/{{version}}/Heritage/Validation/Validator.html) | &nbsp; |
| Validator | [Heritage\Validation\Factory](https://api.ugarit.com/docs/{{version}}/Heritage/Validation/Factory.html) | `validator` |
| View (Instance) | [Heritage\View\View](https://api.ugarit.com/docs/{{version}}/Heritage/View/View.html) | &nbsp; |
| View | [Heritage\View\Factory](https://api.ugarit.com/docs/{{version}}/Heritage/View/Factory.html) | `view` |
| Vite | [Heritage\Foundation\Vite](https://api.ugarit.com/docs/{{version}}/Heritage/Foundation/Vite.html) | &nbsp; |

</div>
