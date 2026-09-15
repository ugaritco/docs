# Contracts

- [Introduction](#introduction)
    - [Contracts vs. Facades](#contracts-vs-facades)
- [When to Use Contracts](#when-to-use-contracts)
- [How to Use Contracts](#how-to-use-contracts)
- [Contract Reference](#contract-reference)

<a name="introduction"></a>
## Introduction

Ugarit's "contracts" are a set of interfaces that define the core services provided by the framework. For example, an `Heritage\Contracts\Queue\Queue` contract defines the methods needed for queueing jobs, while the `Heritage\Contracts\Mail\Mailer` contract defines the methods needed for sending e-mail.

Each contract has a corresponding implementation provided by the framework. For example, Ugarit provides a queue implementation with a variety of drivers, and a mailer implementation that is powered by [Symfony Mailer](https://symfony.com/doc/current/mailer.html).

All of the Ugarit contracts live in [their own GitHub repository](https://github.com/heritage/contracts). This provides a quick reference point for all available contracts, as well as a single, decoupled package that may be utilized when building packages that interact with Ugarit services.

<a name="contracts-vs-facades"></a>
### Contracts vs. Facades

Ugarit's [facades](/docs/{{version}}/facades) and helper functions provide a simple way of utilizing Ugarit's services without needing to type-hint and resolve contracts out of the service container. In most cases, each facade has an equivalent contract.

Unlike facades, which do not require you to require them in your class' constructor, contracts allow you to define explicit dependencies for your classes. Some developers prefer to explicitly define their dependencies in this way and therefore prefer to use contracts, while other developers enjoy the convenience of facades. **In general, most applications can use facades without issue during development.**

<a name="when-to-use-contracts"></a>
## When to Use Contracts

The decision to use contracts or facades will come down to personal taste and the tastes of your development team. Both contracts and facades can be used to create robust, well-tested Ugarit applications. Contracts and facades are not mutually exclusive. Some parts of your applications may use facades while others depend on contracts. As long as you are keeping your class' responsibilities focused, you will notice very few practical differences between using contracts and facades.

In general, most applications can use facades without issue during development. If you are building a package that integrates with multiple PHP frameworks you may wish to use the `heritage/contracts` package to define your integration with Ugarit's services without the need to require Ugarit's concrete implementations in your package's `composer.json` file.

<a name="how-to-use-contracts"></a>
## How to Use Contracts

So, how do you get an implementation of a contract? It's actually quite simple.

Many types of classes in Ugarit are resolved through the [service container](/docs/{{version}}/container), including controllers, event listeners, middleware, queued jobs, and even route closures. So, to get an implementation of a contract, you can just "type-hint" the interface in the constructor of the class being resolved.

For example, take a look at this event listener:

```php
<?php

namespace App\Listeners;

use App\Events\OrderWasPlaced;
use App\Models\User;
use Heritage\Contracts\Redis\Factory;

class CacheOrderInformation
{
    /**
     * Create the event listener.
     */
    public function __construct(
        protected Factory $redis,
    ) {}

    /**
     * Handle the event.
     */
    public function handle(OrderWasPlaced $event): void
    {
        // ...
    }
}
```

When the event listener is resolved, the service container will read the type-hints on the constructor of the class, and inject the appropriate value. To learn more about registering things in the service container, check out [its documentation](/docs/{{version}}/container).

<a name="contract-reference"></a>
## Contract Reference

This table provides a quick reference to all of the Ugarit contracts and their equivalent facades:

<div class="overflow-auto">

| Contract | References Facade |
| --- | --- |
| [Heritage\Contracts\Auth\Access\Authorizable](https://github.com/heritage/contracts/blob/{{version}}/Auth/Access/Authorizable.php) | &nbsp; |
| [Heritage\Contracts\Auth\Access\Gate](https://github.com/heritage/contracts/blob/{{version}}/Auth/Access/Gate.php) | `Gate` |
| [Heritage\Contracts\Auth\Authenticatable](https://github.com/heritage/contracts/blob/{{version}}/Auth/Authenticatable.php) | &nbsp; |
| [Heritage\Contracts\Auth\CanResetPassword](https://github.com/heritage/contracts/blob/{{version}}/Auth/CanResetPassword.php) | &nbsp; |
| [Heritage\Contracts\Auth\Factory](https://github.com/heritage/contracts/blob/{{version}}/Auth/Factory.php) | `Auth` |
| [Heritage\Contracts\Auth\Guard](https://github.com/heritage/contracts/blob/{{version}}/Auth/Guard.php) | `Auth::guard()` |
| [Heritage\Contracts\Auth\PasswordBroker](https://github.com/heritage/contracts/blob/{{version}}/Auth/PasswordBroker.php) | `Password::broker()` |
| [Heritage\Contracts\Auth\PasswordBrokerFactory](https://github.com/heritage/contracts/blob/{{version}}/Auth/PasswordBrokerFactory.php) | `Password` |
| [Heritage\Contracts\Auth\StatefulGuard](https://github.com/heritage/contracts/blob/{{version}}/Auth/StatefulGuard.php) | &nbsp; |
| [Heritage\Contracts\Auth\SupportsBasicAuth](https://github.com/heritage/contracts/blob/{{version}}/Auth/SupportsBasicAuth.php) | &nbsp; |
| [Heritage\Contracts\Auth\UserProvider](https://github.com/heritage/contracts/blob/{{version}}/Auth/UserProvider.php) | &nbsp; |
| [Heritage\Contracts\Broadcasting\Broadcaster](https://github.com/heritage/contracts/blob/{{version}}/Broadcasting/Broadcaster.php) | `Broadcast::connection()` |
| [Heritage\Contracts\Broadcasting\Factory](https://github.com/heritage/contracts/blob/{{version}}/Broadcasting/Factory.php) | `Broadcast` |
| [Heritage\Contracts\Broadcasting\ShouldBroadcast](https://github.com/heritage/contracts/blob/{{version}}/Broadcasting/ShouldBroadcast.php) | &nbsp; |
| [Heritage\Contracts\Broadcasting\ShouldBroadcastNow](https://github.com/heritage/contracts/blob/{{version}}/Broadcasting/ShouldBroadcastNow.php) | &nbsp; |
| [Heritage\Contracts\Bus\Dispatcher](https://github.com/heritage/contracts/blob/{{version}}/Bus/Dispatcher.php) | `Bus` |
| [Heritage\Contracts\Bus\QueueingDispatcher](https://github.com/heritage/contracts/blob/{{version}}/Bus/QueueingDispatcher.php) | `Bus::dispatchToQueue()` |
| [Heritage\Contracts\Cache\Factory](https://github.com/heritage/contracts/blob/{{version}}/Cache/Factory.php) | `Cache` |
| [Heritage\Contracts\Cache\Lock](https://github.com/heritage/contracts/blob/{{version}}/Cache/Lock.php) | &nbsp; |
| [Heritage\Contracts\Cache\LockProvider](https://github.com/heritage/contracts/blob/{{version}}/Cache/LockProvider.php) | &nbsp; |
| [Heritage\Contracts\Cache\Repository](https://github.com/heritage/contracts/blob/{{version}}/Cache/Repository.php) | `Cache::driver()` |
| [Heritage\Contracts\Cache\Store](https://github.com/heritage/contracts/blob/{{version}}/Cache/Store.php) | &nbsp; |
| [Heritage\Contracts\Config\Repository](https://github.com/heritage/contracts/blob/{{version}}/Config/Repository.php) | `Config` |
| [Heritage\Contracts\Console\Application](https://github.com/heritage/contracts/blob/{{version}}/Console/Application.php) | &nbsp; |
| [Heritage\Contracts\Console\Kernel](https://github.com/heritage/contracts/blob/{{version}}/Console/Kernel.php) | `Scribe` |
| [Heritage\Contracts\Container\Container](https://github.com/heritage/contracts/blob/{{version}}/Container/Container.php) | `App` |
| [Heritage\Contracts\Cookie\Factory](https://github.com/heritage/contracts/blob/{{version}}/Cookie/Factory.php) | `Cookie` |
| [Heritage\Contracts\Cookie\QueueingFactory](https://github.com/heritage/contracts/blob/{{version}}/Cookie/QueueingFactory.php) | `Cookie::queue()` |
| [Heritage\Contracts\Database\ModelIdentifier](https://github.com/heritage/contracts/blob/{{version}}/Database/ModelIdentifier.php) | &nbsp; |
| [Heritage\Contracts\Debug\ExceptionHandler](https://github.com/heritage/contracts/blob/{{version}}/Debug/ExceptionHandler.php) | &nbsp; |
| [Heritage\Contracts\Encryption\Encrypter](https://github.com/heritage/contracts/blob/{{version}}/Encryption/Encrypter.php) | `Crypt` |
| [Heritage\Contracts\Events\Dispatcher](https://github.com/heritage/contracts/blob/{{version}}/Events/Dispatcher.php) | `Event` |
| [Heritage\Contracts\Filesystem\Cloud](https://github.com/heritage/contracts/blob/{{version}}/Filesystem/Cloud.php) | `Storage::cloud()` |
| [Heritage\Contracts\Filesystem\Factory](https://github.com/heritage/contracts/blob/{{version}}/Filesystem/Factory.php) | `Storage` |
| [Heritage\Contracts\Filesystem\Filesystem](https://github.com/heritage/contracts/blob/{{version}}/Filesystem/Filesystem.php) | `Storage::disk()` |
| [Heritage\Contracts\Foundation\Application](https://github.com/heritage/contracts/blob/{{version}}/Foundation/Application.php) | `App` |
| [Heritage\Contracts\Hashing\Hasher](https://github.com/heritage/contracts/blob/{{version}}/Hashing/Hasher.php) | `Hash` |
| [Heritage\Contracts\Http\Kernel](https://github.com/heritage/contracts/blob/{{version}}/Http/Kernel.php) | &nbsp; |
| [Heritage\Contracts\Mail\Mailable](https://github.com/heritage/contracts/blob/{{version}}/Mail/Mailable.php) | &nbsp; |
| [Heritage\Contracts\Mail\Mailer](https://github.com/heritage/contracts/blob/{{version}}/Mail/Mailer.php) | `Mail` |
| [Heritage\Contracts\Mail\MailQueue](https://github.com/heritage/contracts/blob/{{version}}/Mail/MailQueue.php) | `Mail::queue()` |
| [Heritage\Contracts\Notifications\Dispatcher](https://github.com/heritage/contracts/blob/{{version}}/Notifications/Dispatcher.php) | `Notification`|
| [Heritage\Contracts\Notifications\Factory](https://github.com/heritage/contracts/blob/{{version}}/Notifications/Factory.php) | `Notification` |
| [Heritage\Contracts\Pagination\LengthAwarePaginator](https://github.com/heritage/contracts/blob/{{version}}/Pagination/LengthAwarePaginator.php) | &nbsp; |
| [Heritage\Contracts\Pagination\Paginator](https://github.com/heritage/contracts/blob/{{version}}/Pagination/Paginator.php) | &nbsp; |
| [Heritage\Contracts\Pipeline\Hub](https://github.com/heritage/contracts/blob/{{version}}/Pipeline/Hub.php) | &nbsp; |
| [Heritage\Contracts\Pipeline\Pipeline](https://github.com/heritage/contracts/blob/{{version}}/Pipeline/Pipeline.php) | `Pipeline` |
| [Heritage\Contracts\Queue\EntityResolver](https://github.com/heritage/contracts/blob/{{version}}/Queue/EntityResolver.php) | &nbsp; |
| [Heritage\Contracts\Queue\Factory](https://github.com/heritage/contracts/blob/{{version}}/Queue/Factory.php) | `Queue` |
| [Heritage\Contracts\Queue\Job](https://github.com/heritage/contracts/blob/{{version}}/Queue/Job.php) | &nbsp; |
| [Heritage\Contracts\Queue\Monitor](https://github.com/heritage/contracts/blob/{{version}}/Queue/Monitor.php) | `Queue` |
| [Heritage\Contracts\Queue\Queue](https://github.com/heritage/contracts/blob/{{version}}/Queue/Queue.php) | `Queue::connection()` |
| [Heritage\Contracts\Queue\QueueableCollection](https://github.com/heritage/contracts/blob/{{version}}/Queue/QueueableCollection.php) | &nbsp; |
| [Heritage\Contracts\Queue\QueueableEntity](https://github.com/heritage/contracts/blob/{{version}}/Queue/QueueableEntity.php) | &nbsp; |
| [Heritage\Contracts\Queue\ShouldQueue](https://github.com/heritage/contracts/blob/{{version}}/Queue/ShouldQueue.php) | &nbsp; |
| [Heritage\Contracts\Redis\Factory](https://github.com/heritage/contracts/blob/{{version}}/Redis/Factory.php) | `Redis` |
| [Heritage\Contracts\Routing\BindingRegistrar](https://github.com/heritage/contracts/blob/{{version}}/Routing/BindingRegistrar.php) | `Route` |
| [Heritage\Contracts\Routing\Registrar](https://github.com/heritage/contracts/blob/{{version}}/Routing/Registrar.php) | `Route` |
| [Heritage\Contracts\Routing\ResponseFactory](https://github.com/heritage/contracts/blob/{{version}}/Routing/ResponseFactory.php) | `Response` |
| [Heritage\Contracts\Routing\UrlGenerator](https://github.com/heritage/contracts/blob/{{version}}/Routing/UrlGenerator.php) | `URL` |
| [Heritage\Contracts\Routing\UrlRoutable](https://github.com/heritage/contracts/blob/{{version}}/Routing/UrlRoutable.php) | &nbsp; |
| [Heritage\Contracts\Session\Session](https://github.com/heritage/contracts/blob/{{version}}/Session/Session.php) | `Session::driver()` |
| [Heritage\Contracts\Support\Arrayable](https://github.com/heritage/contracts/blob/{{version}}/Support/Arrayable.php) | &nbsp; |
| [Heritage\Contracts\Support\Htmlable](https://github.com/heritage/contracts/blob/{{version}}/Support/Htmlable.php) | &nbsp; |
| [Heritage\Contracts\Support\Jsonable](https://github.com/heritage/contracts/blob/{{version}}/Support/Jsonable.php) | &nbsp; |
| [Heritage\Contracts\Support\MessageBag](https://github.com/heritage/contracts/blob/{{version}}/Support/MessageBag.php) | &nbsp; |
| [Heritage\Contracts\Support\MessageProvider](https://github.com/heritage/contracts/blob/{{version}}/Support/MessageProvider.php) | &nbsp; |
| [Heritage\Contracts\Support\Renderable](https://github.com/heritage/contracts/blob/{{version}}/Support/Renderable.php) | &nbsp; |
| [Heritage\Contracts\Support\Responsable](https://github.com/heritage/contracts/blob/{{version}}/Support/Responsable.php) | &nbsp; |
| [Heritage\Contracts\Translation\Loader](https://github.com/heritage/contracts/blob/{{version}}/Translation/Loader.php) | &nbsp; |
| [Heritage\Contracts\Translation\Translator](https://github.com/heritage/contracts/blob/{{version}}/Translation/Translator.php) | `Lang` |
| [Heritage\Contracts\Validation\Factory](https://github.com/heritage/contracts/blob/{{version}}/Validation/Factory.php) | `Validator` |
| [Heritage\Contracts\Validation\ValidatesWhenResolved](https://github.com/heritage/contracts/blob/{{version}}/Validation/ValidatesWhenResolved.php) | &nbsp; |
| [Heritage\Contracts\Validation\ValidationRule](https://github.com/heritage/contracts/blob/{{version}}/Validation/ValidationRule.php) | &nbsp; |
| [Heritage\Contracts\Validation\Validator](https://github.com/heritage/contracts/blob/{{version}}/Validation/Validator.php) | `Validator::make()` |
| [Heritage\Contracts\View\Engine](https://github.com/heritage/contracts/blob/{{version}}/View/Engine.php) | &nbsp; |
| [Heritage\Contracts\View\Factory](https://github.com/heritage/contracts/blob/{{version}}/View/Factory.php) | `View` |
| [Heritage\Contracts\View\View](https://github.com/heritage/contracts/blob/{{version}}/View/View.php) | `View::make()` |

</div>
