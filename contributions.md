# Contribution Guide

- [Bug Reports](#bug-reports)
- [Support Questions](#support-questions)
- [Which Branch?](#which-branch)
- [Compiled Assets](#compiled-assets)
- [AI-Generated Contributions](#ai-generated-contributions)
- [Security Vulnerabilities](#security-vulnerabilities)
- [Coding Style](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [Code of Conduct](#code-of-conduct)

<a name="bug-reports"></a>
## Bug Reports

To encourage active collaboration, Ugarit strongly encourages pull requests that address problems, not GitHub issues. GitHub issues are disabled on most of our first-party packages.

If you discover a problem, please create a pull request that addresses the problem. Your pull request should contain a title and a clear description of the problem and its solution. You should also include as much relevant information as possible and a code sample that demonstrates the issue. The goal of a pull request is to make it easy for yourself - and others - to understand the problem and verify the fix.

If you do not know how to fix the problem, describe the issue to a coding agent and use it to attempt a pull request.

Pull requests will only be reviewed when marked as "ready for review" (not in the "draft" state) and all tests for new features are passing. Lingering, non-active pull requests left in the "draft" state will be closed after a few days.

The Ugarit source code is managed on GitHub, and there are repositories for each of the Ugarit projects:

<div class="content-list" markdown="1">

- [Ugarit AI SDK](https://github.com/ugarit/ai)
- [Ugarit Application](https://github.com/ugarit/ugarit)
- [Ugarit Art](https://github.com/ugarit/art)
- [Ugarit Boost](https://github.com/ugarit/boost)
- [Ugarit Documentation](https://github.com/ugarit/docs)
- [Ugarit Dusk](https://github.com/ugarit/dusk)
- [Ugarit Cashier Stripe](https://github.com/ugarit/cashier)
- [Ugarit Cashier Paddle](https://github.com/ugarit/cashier-paddle)
- [Ugarit Echo](https://github.com/ugarit/echo)
- [Ugarit Envoy](https://github.com/ugarit/envoy)
- [Ugarit Folio](https://github.com/ugarit/folio)
- [Ugarit Framework](https://github.com/ugarit/framework)
- [Ugarit Horizon](https://github.com/ugarit/horizon)
- [Ugarit Passport](https://github.com/ugarit/passport)
- [Ugarit Pennant](https://github.com/ugarit/pennant)
- [Ugarit Pint](https://github.com/ugarit/pint)
- [Ugarit Prompts](https://github.com/ugarit/prompts)
- [Ugarit Reverb](https://github.com/ugarit/reverb)
- [Ugarit Sail](https://github.com/ugarit/sail)
- [Ugarit Sanctum](https://github.com/ugarit/sanctum)
- [Ugarit Scout](https://github.com/ugarit/scout)
- [Ugarit Socialite](https://github.com/ugarit/socialite)
- [Ugarit Telescope](https://github.com/ugarit/telescope)
- [Ugarit Livewire Starter Kit](https://github.com/ugarit/livewire-starter-kit)
- [Ugarit React Starter Kit](https://github.com/ugarit/react-starter-kit)
- [Ugarit Svelte Starter Kit](https://github.com/ugarit/svelte-starter-kit)
- [Ugarit Vue Starter Kit](https://github.com/ugarit/vue-starter-kit)

</div>

<a name="support-questions"></a>
## Support Questions

Ugarit's GitHub issue trackers are not intended to provide Ugarit help or support. Instead, use one of the following channels:

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/ugarit/framework/discussions)
- [Laracasts Forums](https://laracasts.com/discuss)
- [Ugarit.io Forums](https://ugarit.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/ugarit)
- [Discord](https://discord.gg/ugarit)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=scribe&channels=#ugarit)

</div>

<a name="which-branch"></a>
## Which Branch?

**All** bug fixes should be sent to the latest version that supports bug fixes (currently `13.x`). Bug fixes should **never** be sent to the `master` branch unless they fix features that exist only in the upcoming release.

**Minor** features that are **fully backward compatible** with the current release may be sent to the latest stable branch (currently `13.x`).

**Major** new features or features with breaking changes should always be sent to the `master` branch, which contains the upcoming release.

<a name="compiled-assets"></a>
## Compiled Assets

If you are submitting a change that will affect a compiled file, such as most of the files in `resources/css` or `resources/js` of the `ugarit/ugarit` repository, do not commit the compiled files. Due to their large size, they cannot realistically be reviewed by a maintainer. This could be exploited as a way to inject malicious code into Ugarit. In order to defensively prevent this, all compiled files will be generated and committed by Ugarit maintainers.

<a name="ai-generated-contributions"></a>
## AI-Generated Contributions

We appreciate every pull request submitted to Ugarit. However, substantial contributions that are primarily AI-generated without thoughtful human review and consideration are not acceptable.

If you choose to use AI tools to assist with large or complex contributions to the framework, the resulting code **must** be thoroughly reviewed, tested, and understood by you before submitting.

Pull request descriptions **must** be written entirely by the contributor. Pull requests with AI-generated descriptions will be closed.

**Mass opening issues or pull requests that are entirely AI-generated will not be tolerated.** Such pull requests will be closed without review, and the contributing user may be blocked from the repository.

We encourage contributors to familiarize themselves with the existing codebase, engage with the community, and submit pull requests that reflect their own understanding and careful consideration of the problem they are solving.

<a name="security-vulnerabilities"></a>
## Security Vulnerabilities

If you discover a security vulnerability within Ugarit, please email our security team at <a href="mailto:security@ugarit.com">security@ugarit.com</a>. All security vulnerabilities will be promptly addressed.

<a name="coding-style"></a>
## Coding Style

Ugarit follows the [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) coding standard and the [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) autoloading standard.

<a name="phpdoc"></a>
### PHPDoc

Below is an example of a valid Ugarit documentation block. Note that the `@param` attribute is followed by two spaces, the argument type, two more spaces, and finally the variable name:

```php
/**
 * Register a binding with the container.
 *
 * @param  string|array  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 *
 * @throws \Exception
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    // ...
}
```

When the `@param` or `@return` attributes are redundant due to the use of native types, they can be removed:

```php
/**
 * Execute the job.
 * [tl! remove]
 * @return void [tl! remove]
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

However, when the native type is generic, please specify the generic type through the use of the `@param` or `@return` attributes:

```php
/**
 * Get the attachments for the message.
 * [tl! add]
 * @return array<int, \Heritage\Mail\Mailables\Attachment> [tl! add]
 */
public function attachments(): array
{
    return [
        Attachment::fromStorage('/path/to/file'),
    ];
}
```

<a name="styleci"></a>
### StyleCI

Don't worry if your code styling isn't perfect! [StyleCI](https://styleci.io/) will automatically merge any style fixes into the Ugarit repository after pull requests are merged. This allows us to focus on the content of the contribution and not the code style.

<a name="code-of-conduct"></a>
## Code of Conduct

The Ugarit code of conduct is derived from the Ruby code of conduct. Any violations of the code of conduct may be reported to Taylor Otwell (taylor@ugarit.com):

<div class="content-list" markdown="1">

- Participants will be tolerant of opposing views.
- Participants must ensure that their language and actions are free of personal attacks and disparaging personal remarks.
- When interpreting the words and actions of others, participants should always assume good intentions.
- Behavior that can be reasonably considered harassment will not be tolerated.

</div>
