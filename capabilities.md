# Capabilities

- [Introduction](#introduction)
    - [Roles](#roles)
- [Writing a Capability](#writing-a-capability)
    - [The Accepted Parcel](#the-accepted-parcel)
    - [Injecting Repositories](#injecting-repositories)
    - [Authorization](#authorization)
    - [Handling Failures](#handling-failures)
- [Repositories](#repositories)
    - [Provided Keys & Partial Updates](#provided-keys)
    - [Listing With Query Parcels](#query-parcels)
- [Results & Outcomes](#results-and-outcomes)
    - [Artifact Outcomes](#artifact-outcomes)
    - [The Feedback Convention](#feedback-convention)
    - [Overriding a Case](#overriding-a-case)
    - [Displayable Errors](#displayable-errors)
- [Running Actions From Controllers](#running-actions)
    - [Inertia or Blade Pages](#blade-pages)
    - [Failed Outcomes](#failed-outcomes)
    - [Lower-Level API](#lower-level-api)
- [CLI Scaffolding](#cli-scaffolding)

<a name="introduction"></a>
## Introduction

Business logic in an artifact lives in **Capabilities** (one operation each) and [Features](/docs/{{version}}/features) (workflows coordinating several Capabilities). Both receive a typed **Parcel** and return an **Outcome**; the Controller hands that Outcome to a **Responder** that renders it for the client.

```text
Request ─► Controller ─► Feature ─► Capability ─► Repository ─► Model
                │            └──────► Capability ─┘
                ▼
           Responder ◄── Outcome (Result enum + data + errors)
```

<a name="roles"></a>
### Roles

| Piece | Responsibility |
| --- | --- |
| Parcel | Carries the validated input; remembers which keys were provided. |
| Capability | Performs one operation through repositories; turns its failures into a failed Outcome. |
| Feature | Orders several Capabilities as a [flow](/docs/{{version}}/features#flows): named steps in one transaction. |
| Repository | Reads and writes models; takes Parcels, returns models. |
| Result enum | Names every possible result and its feedback (title, message, variant, status). |
| Outcome | Carries the Result, the domain data (models) and displayable `errors` back to the Controller. |
| Controller | Runs the action and chooses the Resource presenting its data. |
| Responder | Renders an Outcome as Inertia, JSON or a redirect. |

<a name="writing-a-capability"></a>
## Writing a Capability

A Capability extends `Heritage\Capabilities\Capability` and implements two methods: `perform()`, the operation, and `failed()`, the Outcome to return when `perform()` throws. The public `handle()` is final: it checks the input, calls `perform()`, and handles exceptions.

```php
use Heritage\Capabilities\Capability;
use Heritage\Outcomes\Outcome;
use Throwable;

class ShowBranchCapability extends Capability
{
    public function __construct(protected BranchRepositoryInterface $branches)
    {
    }

    protected function perform(ShowBranchParcel $data): Outcome
    {
        $branch = $data->getModel() ?? ($data->id ? $this->branches->find($data->id) : null);

        if (! $branch) {
            return BranchOutcome::fromResult(ShowBranchResult::FAILED);
        }

        return BranchOutcome::fromResult(ShowBranchResult::SUCCESS, $branch->loadMissing('translations'));
    }

    protected function failed(Throwable $e): Outcome
    {
        return BranchOutcome::fromResult(ShowBranchResult::FAILED);
    }
}
```

A Capability returns domain data (a model, a collection, a paginator), never a Resource: how the data is presented is the Controller's decision, so the same Capability serves a page and an API.

<a name="the-accepted-parcel"></a>
### The Accepted Parcel

The type of `perform()`'s first parameter declares what the Capability accepts, and PHP enforces it:

| Signature | Accepts |
| --- | --- |
| `perform(ShowBranchParcel $data)` | a `ShowBranchParcel` only |
| `perform(?IndexBranchParcel $data = null)` | an `IndexBranchParcel`, or no input |
| `perform()` | no input (a given Parcel is ignored) |

A wrong Parcel is a programming error: PHP's `TypeError` is rethrown as it is, not reported nor turned into a failed Outcome:

```text
TypeError: SaveBranchCapability::perform(): Argument #1 ($data) must be of type SaveBranchParcel, ShowBranchParcel given
```

So is the `Error` of a Capability that does not define `perform()`. A `TypeError` raised **inside** `perform()` is handled like any other exception ([Handling Failures](#handling-failures)).

<a name="injecting-repositories"></a>
### Injecting Repositories

Declare the repository contracts a Capability needs in its constructor. Capabilities are resolved from the container, and an unbound repository interface is resolved by convention: `Contracts\Repositories\Branch\BranchRepositoryInterface` maps to `Repositories\Branch\BranchRepository` (as a singleton). Bind the interface yourself to override it.

A Feature's [flow](/docs/{{version}}/features#flows) resolves the Capabilities it runs from the container by class name, so a Feature needs no constructor.

<a name="authorization"></a>
### Authorization

A Capability or Feature declares the ability it requires with the `Heritage\Capabilities\Attributes\Authorize` attribute. The rules themselves stay in the model's [policy](/docs/{{version}}/authorization#creating-policies):

```php
use Heritage\Capabilities\Attributes\Authorize;

#[Authorize('delete', Branch::class)]
class DestroyBranchCapability extends Capability
{
    // ...
}
```

The controller checks the ability before running the action ([Running Actions From Controllers](#running-actions)), so a denied action never runs and the request ends with `403`. The ability is checked against the Parcel's attached model when it is an instance of the given class, otherwise against the class itself, so one declaration serves both creating and updating:

```php
#[Authorize('save', Branch::class)]
class SaveBranchFeature extends Feature { /* ... */ }

// Policies\Branch\BranchPolicy
public function save(User $user, ?Branch $branch = null): bool
{
    return $branch === null ? $this->create($user) : $this->update($user, $branch);
}
```

When the policy method requires the model (a required, non-nullable second parameter, as in `view(User $user, Branch $branch)`), the Parcel must carry it: a Parcel holding only an id throws a `LogicException` naming the action and the model, instead of silently falling back to a class-level check. Attach the model with `fromModel()` or `setModel()`; route model binding does this for you in the controller.

Without a model class, the ability is checked as a plain gate (`#[Authorize('manage-settings')]`). An action without the attribute is not checked.

#### Authorizing Before Validation

The controller resolves a [form request](/docs/{{version}}/validation#form-request-validation) before its method runs, so on its own the ability would be checked after validation, and a user without it would see `422` errors before `403`. A form request names the action it feeds with `#[ForAction]`. It then checks that action's `#[Authorize]` ability before validating, against the route model of the ability's class, and the controller does not check it again:

```php
use Heritage\Foundation\Http\Attributes\ForAction;

#[ForAction(SaveBranchCapability::class)]
class UpdateBranchRequest extends FormRequest
{
    public function rules(): array { /* ... */ }
}
```

The ability stays declared once, on the action. Requests without a body (index, show, edit) have no form request and are checked by the controller. A condition that depends on the request input belongs in the request's own `authorize()`, which runs after the declared ability and must also pass:

```php
public function authorize(): bool
{
    return $this->user()->can('update', Branch::find($this->input('branch_id')));
}
```

Conditions on the user and the record belong in the policy method, and global exceptions such as administrators in `Gate::before`. `php scribe make:request UpdateBranchRequest --artifact=company --capability --action=Capabilities/Branch/SaveBranchCapability` generates the attribute.

The check belongs to the entry point, not to the action: the Capabilities a Feature runs as [flow steps](/docs/{{version}}/features#flows) are covered by the Feature's ability and are not checked again, and calling `handle()` directly (in a command, a job or a test) runs without a check.

A policy is found by convention at `Policies\BranchPolicy` or, like repositories, in a folder named after the model: `Policies\Branch\BranchPolicy`.

<a name="handling-failures"></a>
### Handling Failures

When `perform()` throws, `handle()` decides what happens:

- Exceptions listed in `$propagate` are rethrown so the framework handles them: HTTP exceptions, `AuthorizationException`, `ValidationException` and `ModelNotFoundException`. Extend the list in a Capability when needed.
- A `Heritage\Capabilities\CapabilityException` is an expected failure: it is **not** reported, and the Outcome from `failed()` receives its errors (`->withErrors($e->errors())`).
- Any other exception is reported, then `failed()` builds the Outcome. The exception's message is never shown to the user.

```php
use Heritage\Capabilities\CapabilityException;

if ($data->slug !== $branch->slug) {
    throw new CapabilityException(['slug' => __('The slug does not match.')]);
}
```

<a name="repositories"></a>
## Repositories

Repositories extend `Heritage\Repositories\ModelRepository` and implement `ModelRepositoryInterface`; a concrete repository usually needs only its constructor. They take Parcels and return models:

| Method | Returns |
| --- | --- |
| `all()` | `Collection` of models |
| `paginate($perPage, $columns)` | `LengthAwarePaginator` |
| `browse(QueryParcel $parcel)` | filtered, searched and sorted `Collection`, or `LengthAwarePaginator` when the Parcel has `per_page` |
| `find($id)` | the model or `null` |
| `findOrFail($id)` | the model, or throws `ModelNotFoundException` |
| `save(Parcel $parcel)` | the created or updated model |
| `destroy(Parcel\|Model\|int\|string $target)` | `bool` |

`save()` updates the Parcel's attached model (`$parcel->getModel()`), or the model with the Parcel's `id`, and creates one otherwise. `destroy()` honours a `force` key on the Parcel for soft-deleting models, and does not swallow exceptions. Translation repositories follow the same rules; see [Translation Repositories](/docs/{{version}}/eloquent-translations#translation-repositories).

<a name="provided-keys"></a>
### Provided Keys & Partial Updates

A Parcel built with `fromArray()` or `fromRequest()` remembers which keys were present in its input. `toAttributes()`, used by `save()`, returns only those keys:

```php
$parcel = SaveBranchParcel::fromArray(['description' => null]);

$parcel->toAttributes(); // ['description' => null]: clears the column, leaves every other column untouched
```

`fromParcel($source, $overrides)` builds one Parcel from another, for example a Capability's Parcel from a Feature's: it keeps the source's provided keys that the new Parcel declares, and the source's attached model, so an update stays an update. Build a Parcel for another model with `fromArray()`.

A Parcel created with `new` has no record of provided keys; `toAttributes()` then returns all fields, dropping nulls when `shouldFilterNulls()` is true.

<a name="query-parcels"></a>
### Listing With Query Parcels

A listing takes a `Heritage\Parcels\QueryParcel`. It carries the common fields `search`, `sort` (a column, `-column` for descending), `page` and `per_page` (`null` returns every record); filters are its typed constructor parameters, and `searchable()` / `sortable()` whitelist the columns a client may reach:

```php
use Heritage\Parcels\QueryParcel;

class IndexBranchParcel extends QueryParcel
{
    public function __construct(
        public ?bool $active = null,
    ) {}

    public function searchable(): array
    {
        return ['slug', 'name'];
    }

    public function sortable(): array
    {
        return ['slug', 'name', 'created_at'];
    }
}
```

`fromQuery()` builds the Parcel from a query string. It validates first, throwing a `ValidationException` (a redirect back with errors, or `422` for JSON clients), then casts each filter to its declared type, so `?active=1` arrives as `true`. Empty values are ignored, a `sort` outside `sortable()` is rejected, and `per_page` is capped by `maxPerPage()` (100 by default):

```php
// Controller
public function index(Request $request)
{
    return $this->page(IndexBranchesCapability::class, IndexBranchParcel::fromQuery($request->query()), 'company::branch/Index', BranchResource::class);
}

// Capability
protected function perform(?IndexBranchParcel $data = null): BranchOutcome
{
    return BranchOutcome::fromResult(IndexBranchesResult::SUCCESS, $this->branches->browse($data ?? new IndexBranchParcel));
}
```

`browse()` applies each provided filter as an equality match (an array value matches any of its values), searches the `searchable()` columns with `LIKE`, and sorts by `sort`. Translated attributes are filtered, searched and sorted in the current locale. To give a filter its own meaning, define `filter{Name}` on the repository; it receives the query builder and the value:

```php
protected function filterActive(Builder $query, bool $active): void
{
    $query->where('active', $active)->whereNotNull('opened_at');
}
```

<a name="results-and-outcomes"></a>
## Results & Outcomes

Each action has a Result enum listing its results. The `ResolvesResultFeedback` trait derives every case's feedback from one `FEEDBACK` constant:

```php
use Heritage\Contracts\Result\ResultContract;
use Heritage\Enums\Concerns\ResolvesResultFeedback;

enum SaveBranchResult: int implements ResultContract
{
    use ResolvesResultFeedback;

    const FEEDBACK = 'saved';

    case SUCCESS = 1;
    case FAILED = 2;
}
```

The Outcome is built from a case with `fromResult($result, $data, $context)`. Its title, message, variant and status code are read from the Result **when requested**, with the context at that time, so a copy with more context (`withMergedContext()`) renders the new message.

<a name="artifact-outcomes"></a>
### Artifact Outcomes

One Outcome subclass per domain supplies what all its actions share; there is no Outcome class per action:

```php
use Heritage\Outcomes\Outcome;

readonly class BranchOutcome extends Outcome
{
    // Placeholders of every Branch message; the context given to fromResult() overrides them.
    protected function defaultContext(): array
    {
        $subject = $this->subject(); // the data, unwrapped from a Resource

        return [
            'model' => trans('company::branch.name'),
            'entity' => $subject instanceof Branch ? $subject->slug : null,
        ];
    }

    // Follow-up actions offered in the toast.
    public function actions(): array
    {
        return $this->result === SaveBranchResult::SUCCESS && $this->subject() instanceof Branch
            ? [Action::redirect(route('branch.show', $this->subject()->id), __('actions.view'))]
            : parent::actions();
    }
}
```

<a name="feedback-convention"></a>
### The Feedback Convention

Keys are read from `lang/{locale}/responses.php`, within the group of `FEEDBACK` (`'translation.saved'` reads under `responses.translation.`):

| Case | Key (`FEEDBACK = 'saved'`) | Variant | Status |
| --- | --- | --- | --- |
| `SUCCESS` | `responses.saved` | success | 200 |
| `SUCCESS_NO_TRANSLATION` | `responses.saved_no_translation` | success | 200 |
| `FAILED` | `responses.failed_saved` | error | 400 |
| `FAILED_SLUG_MISMATCH` | `responses.failed_slug_mismatch` | error | 400 |

Each key holds a `title` and a `message`; placeholders (`:entity`, `:locale`, ...) are filled from the Outcome's context.

<a name="overriding-a-case"></a>
### Overriding a Case

Override `feedbackKey()`, `feedbackContext()`, `variant()` or `statusCode()` for the cases that differ, and fall back to the `conventional*()` methods for the rest:

```php
protected function feedbackKey(): string
{
    return match ($this) {
        self::FAILED_SLUG_MISMATCH => 'failed_attribute_mismatch',
        default => $this->conventionalFeedbackKey(),
    };
}

public function variant(): Variant
{
    return $this === self::SUCCESS_NO_TRANSLATION ? Variant::INFO : $this->conventionalVariant();
}
```

<a name="displayable-errors"></a>
### Displayable Errors

An Outcome's `errors()` holds the reasons shown to the user, keyed by field. Add them with `withErrors()`, which merges and returns a copy:

```php
return BranchOutcome::fromResult(SaveBranchResult::FAILED)
    ->withErrors(['translations.en' => $reason]);
```

The redirect responder flashes them as validation errors, together with the Outcome's message under `form`. Never put an exception message in `errors`.

<a name="running-actions"></a>
## Running Actions From Controllers

Controllers resolve Capabilities and Features from the container, so their dependencies are injected, and choose the Resource presenting the result. Four shortcuts cover the usual responses:

```php
// An Inertia page
return $this->page(ShowBranchCapability::class, ShowBranchParcel::fromModel($branch, true), 'company::branch/Show', BranchDetailsResource::class);

// Redirect back with the feedback (forms)
return $this->back(SaveBranchCapability::class, SaveBranchParcel::fromArray($request->validated()), BranchResource::class);

// Redirect to a route once it succeeds
return $this->toRoute(DestroyBranchCapability::class, $parcel, 'branch.index');

// Always JSON
return $this->json(IndexBranchesCapability::class, null, BranchResource::class);
```

`page()`, `back()` and `toRoute()` use `ResponderType::AUTO`: a client expecting JSON (and not an Inertia visit) gets JSON with the presented data, so one endpoint serves the panel and an API.

<a name="blade-pages"></a>
### Inertia or Blade Pages

`page()` renders an Inertia component by default. Set `view.pages` to `blade` (`VIEW_PAGES=blade`) to render Blade views instead, or choose per controller:

```php
use Heritage\Enums\Responders\ResponderType;

class BranchController extends Controller
{
    protected ?ResponderType $pages = ResponderType::VIEW;

    public function edit(Branch $branch)
    {
        return $this->page(EditBranchCapability::class, EditBranchParcel::fromModel($branch, true), 'company::branch.edit');
    }
}
```

The view receives `$data` (the model as it is, or the resolved array when a Resource is given), `$outcome` (`title()`, `message()`, `errors()`, `actions()`) and the extra variables. The toast is in `session('toast')` for the current request, as a redirect flashes it for the next one; forms read `$errors` and `old()` after a failed redirect back. A missing view throws an `InvalidArgumentException`.

`resource` wraps a model with `make()` and a collection, paginator or array with `collection()`; data that is `null` or already a Resource is left as it is. Without it the model is serialized as it is (`toArray()`). The Responder sends the resolved `title`, `message`, `variant` and `statusCode` along with the data.

<a name="failed-outcomes"></a>
### Failed Outcomes

A failed outcome bound for a page or a route redirect does not render the page nor follow the route: it redirects back with its toast and `errors`, and aborts with 404 when its status code is 404 (override `statusCode()` on the Result case). Override `respondToFailure()` to change this for a controller. JSON responses and redirects back carry the failure as they are.

<a name="lower-level-api"></a>
### Lower-Level API

`run()` returns the Outcome; `respond()` and `execute()` take every option (`responderType`, `component`, `route`, `parameters`, `extra`, `resource`):

```php
$outcome = $this->run(ShowBranchCapability::class, new ShowBranchParcel(id: $id));

return $this->respond($outcome, ResponderType::JSON, resource: BranchDetailsResource::class);
```

An Inertia response without a component, or a route redirect without a route name, throws an `InvalidArgumentException`. Responders are resolved from the container; replace one with `app(ResponderFactory::class)->extend(ResponderType::JSON, ApiResponder::class)`.

<a name="cli-scaffolding"></a>
## CLI Scaffolding

Every generator accepts `--artifact`. Companion classes are given relative to their conventional folder (`Parcels/`, `Contracts/Repositories/`, `Enums/`, `Outcomes/`) or fully-qualified; a class that does not exist yet produces a warning.

```bash
# Capability typed to its Parcel, with an injected repository and its Result/Outcome
php scribe make:capability Branch/ShowBranch --artifact=company \
    --parcel=Branch/Capabilities/ShowBranchParcel \
    --repository=Branch/BranchRepositoryInterface \
    --result=Branch/Capabilities/ShowBranchResult \
    --outcome=Branch/BranchOutcome

# With the ability it requires: #[Authorize('view', Branch::class)] (--model is relative to Models/)
php scribe make:capability Branch/ShowBranch --artifact=company --authorize=view --model=Branch

# Without --parcel the capability accepts any Parcel or none; --nullable makes a given Parcel optional
php scribe make:capability Branch/IndexBranches --artifact=company --parcel=Branch/Capabilities/IndexBranchParcel --nullable

# Feature with Parcel check and a flow (see Features: Generating Features)
php scribe make:feature Branch/SaveBranch --artifact=company --parcel=Branch/Features/SaveBranchParcel --result=Branch/Features/SaveBranchResult

# Result enum; FEEDBACK is guessed from the verb (Save → saved, Destroy → deleted, ...)
php scribe make:result Branch/Capabilities/SaveBranch --artifact=company
php scribe make:result Branch/Capabilities/ArchiveBranch --artifact=company --feedback=archived

# Parcel
php scribe make:parcel Branch/Capabilities/ShowBranchParcel --artifact=company
```

The generated repository property is named after the contract (`BranchRepositoryInterface` becomes `$branches`). Results whose name contains `Translation` get a `translation.` feedback group. With `--outcome` the generated code returns `BranchOutcome::fromResult(...)` (pass the artifact Outcome); without `--result`/`--outcome` it returns `Outcome::success()` / `Outcome::failure()`. Pass the data as the second argument of `fromResult()`.
