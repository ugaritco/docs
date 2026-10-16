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
    - [Generating a Slice](#generating-a-slice)
    - [Generating a Model With Its Slice](#generating-a-model-with-its-slice)
    - [Adding Fields to a Model](#adding-fields-to-a-model)
    - [Generating the Parts](#generating-the-parts)
    - [Generating the UI](#generating-the-ui)

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
public function save(Actor $actor, ?Branch $branch = null): bool
{
    return $branch === null ? $this->create($actor) : $this->update($actor, $branch);
}
```

A policy decides for an `Heritage\Contracts\Auth\Actor` (authenticatable and authorizable, implemented by `Heritage\Foundation\Auth\User`) rather than a concrete user model, so an artifact's policies do not depend on the identity artifact. See [Policies and Actors](/docs/{{version}}/authorization#policy-actors).

When the policy method requires the model (a required, non-nullable second parameter, as in `view(Actor $actor, Branch $branch)`), the Parcel must carry it: a Parcel holding only an id throws a `LogicException` naming the action and the model, instead of silently falling back to a class-level check. Attach the model with `fromModel()` or `setModel()`; route model binding does this for you in the controller.

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

<a name="generating-a-slice"></a>
### Generating a Slice

`make:capabilities` generates everything a model's capabilities need, each part by its own generator:

```bash
php scribe make:capabilities Department --artifact=company
```

| Part | Class |
| --- | --- |
| Repository | `Contracts/Repositories/Department/DepartmentRepositoryInterface`, `Repositories/Department/DepartmentRepository` |
| Policy | `Policies/Department/DepartmentPolicy` (every ability is granted until restricted there) |
| Results | `Enums/Department/Capabilities/{Index…s,Create,Show,Edit,Save,Destroy}…Result` |
| Parcels | `Parcels/Department/Capabilities/{Index,Create,Show,Edit,Save,Destroy}DepartmentParcel` |
| Outcome | `Outcomes/Department/DepartmentOutcome` (offers to view a saved record) |
| Capabilities | `Capabilities/Department/IndexDepartmentsCapability`, `CreateDepartmentCapability`, … each with `#[Authorize]` |
| Requests | `Http/Requests/Department/Capabilities/{Store,Update,Destroy}DepartmentRequest` with `#[ForAction]`; the destroy request requires a `confirmation` equal to the record's slug (else its required, untranslated name, else its id), the field the generated delete page asks for |
| Resource | `Http/Resources/Department/DepartmentResource` |
| Controller | `Http/Controllers/Department/Capabilities/DepartmentController` |
| Lang | `lang/en/department.php`, unless it exists |

The fields, rules and filters are read from the model: its `$fillable` columns typed by `$casts` (a column ending in `_id` is an integer, validated with `exists` on its model's table; a date cast is validated as a date; a column nullable in the table is validated as `nullable`), and its translatable attributes, which travel in a `translations` payload and are validated from the translation table's columns (`max:255` for a `varchar`). Fill `$fillable` before generating; with `--force` the slice is regenerated. The command ends by printing the route to register:

```php
Route::resource('department', DepartmentController::class);
```

With `--routes` it registers the route itself in `routes/web.php` (the artifact's, or the application's): it imports the controller as `DepartmentCapabilitiesController` and adds the route to the file's last group, or creates the file with a `web` and `auth` group. A model whose route is already registered is left untouched.

For a translatable model, the show and edit capabilities load its `translations`, the resource returns them keyed by locale (`'translations' => $this->whenLoaded(...)`), the policy has a `translate()` ability, and the command generates its translations sub-resource too, as the `Branch` one is written:

| Part | Class |
| --- | --- |
| Repository | `Contracts/Repositories/Department/DepartmentTranslationRepositoryInterface`, `Repositories/Department/DepartmentTranslationRepository` (a `TranslationRepository`) |
| Results | `Enums/Department/Capabilities/Translation/{Save,Destroy}DepartmentTranslationResult` (`translation.saved`, `translation.deleted`) |
| Parcels | `Parcels/Department/Capabilities/Translation/{Save,Destroy}DepartmentTranslationParcel` |
| Capabilities | `Capabilities/Department/Translation/{Save,Destroy}DepartmentTranslationCapability`, with `#[Authorize('translate', Department::class)]` |
| Requests | `Http/Requests/Department/Capabilities/Translation/{Store,Update,Destroy}DepartmentTranslationRequest`: an available locale the model has no translation in yet, the translated fields (`UniqueTranslation` for a unique one), and the confirmation of the deletion |
| Resource | `Http/Resources/Department/Translation/DepartmentTranslationResource` |
| Controller | `Http/Controllers/Department/Capabilities/DepartmentTranslationController` (`store`, `update`, `destroy`) |

Its route is nested in the model's and scoped, so a translation of another record answers 404:

```php
Route::resource('{department}/department-translation', DepartmentTranslationCapabilitiesController::class)
    ->only(['store', 'update', 'destroy'])
    ->scoped(['department_translation' => 'id']);
```

The translation stubs (`stubs/translation/{repository,repository.interface,parcel.save,parcel.destroy,capability.save,capability.destroy,request.store,request.update,request.destroy,resource,controller}.stub`) may be copied to the application's `stubs/translation` folder to be replaced.

<a name="the-feature-layer"></a>
#### The Feature Layer

Each capability of a translatable model does one thing: `SaveDepartmentCapability` saves the department's own columns, and the translation capabilities save or delete one locale. Its capability Save parcel and Store / Update requests therefore carry the model's columns only, without `translations`. Saving a department with its translations is a [Feature](/docs/{{version}}/features), which the command generates too:

| Part | Class |
| --- | --- |
| Feature | `Features/Department/SaveDepartmentFeature`: a [flow](/docs/{{version}}/features#flows) saving the department, then each locale of the `translations` payload (a `null` locale deletes its translation), in one transaction |
| Parcel | `Parcels/Department/Features/SaveDepartmentParcel` (the columns and `translations`) |
| Result | `Enums/Department/Features/SaveDepartmentResult` (`SUCCESS`, `SUCCESS_NO_TRANSLATION`, `FAILED_MODEL_SAVE`, `FAILED_TRANSLATION_SAVE`, `FAILED_TRANSLATION_DESTROY`, `FAILED`) |
| Requests | `Http/Requests/Department/Features/{Store,Update}DepartmentRequest`, declaring `$translatable` |
| Controller | `Http/Controllers/Department/Features/DepartmentController` (`store`, `update`) |

Its route is registered with `--routes`, before the translation route:

```php
Route::resource('department-feature', DepartmentFeaturesController::class)
    ->only(['store', 'update'])
    ->parameters(['department-feature' => 'department']);
```

The pages generated by [`make:pages`](#generating-the-ui) carry both forms: the feature forms post to `department-feature.store` and `department-feature.update` with the translated fields, and the capability forms post to `department.store` / `department.update` with the columns alone. The stubs `stubs/feature/{feature.save,result.save,controller}.stub` may be copied to the application's `stubs/feature` folder to be replaced.

<a name="generating-a-model-with-its-slice"></a>
### Generating a Model With Its Slice

`make:model --all` generates everything at once: the model, its migration, factory and seeder, its slice and its routes. Declare the fields with `--fields`, as `name:type[:modifier...]` separated by commas:

```bash
php scribe make:model Department --artifact=company --all \
    --fields="branch_id:foreignId,slug:string:unique,name:string:translatable,active:boolean"
```

| Types | Modifiers |
| --- | --- |
| `string`, `text`, `integer`, `bigInteger`, `boolean`, `decimal`, `float`, `json`, `date`, `dateTime`, `foreignId` | `nullable`, `unique`, `index`, `translatable` |

The fields become the model's `$fillable`, `$casts` and `$translatable` (a translatable field makes the model translatable; a translatable `string` falls back to its `slug`), a `belongsTo()` relation for each foreign key with its `hasMany()` inverse on the related model when it exists (`Author::books()` for a book's `author_id`), the migration's columns (`foreignId` is `constrained()`) and the factory's state; the slice is then read from that model. A foreign key whose model is not in the artifact may name a model of another artifact: `country_id` of i18n's `Locale` relates to `Ugarit\Artifacts\Geography\Models\Country`, which the model imports (without adding the inverse to the other artifact), and the panel pages show it as a select of the countries, like a relation within the artifact. A bare model name always resolves inside the artifact, even when PHP has a global class of that name (`Locale`). Without `--fields`, the command asks for the fields one by one (type and modifiers), or uses a name, a unique slug and an active flag when none is given or the command is not interactive.

<a name="adding-fields-to-a-model"></a>
### Adding Fields to a Model

`make:field` adds fields to an existing model and to every file of its slice, with the same `name:type[:modifier...]` syntax:

```bash
php scribe make:field Department --artifact=company \
    branch_id:foreignId description:text:nullable:translatable code:string:unique featured:boolean
```

| File | What is added |
| --- | --- |
| Model | `$fillable` (or `$translatable`), `$casts`, a `belongsTo()` relation for a foreign key and its `hasMany()` inverse on the related model |
| Migration | `add_{field}_to_{table}_table` (`add_fields_to_…` for more than two): the commented columns on the table (`Schema::tableTranslatable()` with a translatable field, which goes to the translation table), and a `down()` dropping them |
| Factory | The field's state, `Related::factory()` for a foreign key |
| Requests | The rule in `Store…Request` and `Update…Request` of the capability and [Feature](#the-feature-layer) layers; a translatable field, with its `$translatable` entry, only in the Feature requests |
| Parcels | A `Save…Parcel` property in both layers (a translatable field travels in `translations`); in `Index…Parcel` a filter (boolean, foreign key) or a searchable and sortable column (string, text) |
| Resource | The attribute |
| Translation slice | For a translatable field, the `Save…TranslationParcel` property, the rule in `Store…TranslationRequest` and `Update…TranslationRequest`, and the `…TranslationResource` attribute |
| Lang | The attribute's label and placeholder, in every locale file of the model |

Each file keeps its formatting and imports; a file that does not exist (a model without a slice) or a list that was rewritten by hand is reported and left to be edited. A field the model already has, or a translatable field on a model without a translation table, stops the command before anything is written. The new columns are nullable on the table (a boolean defaults to `false`, a number to `0`) so the existing rows remain valid, while the request requires them unless they are `nullable`. Use `--without-migration` to edit the files only.

<a name="generating-the-parts"></a>
### Generating the Parts

```bash
# Every capability operation: index, create, show, edit, save, destroy (all but create need --repository)
php scribe make:capability Department/ShowDepartment --artifact=company --operation=show \
    --parcel=Department/Capabilities/ShowDepartmentParcel \
    --repository=Department/DepartmentRepositoryInterface \
    --result=Department/Capabilities/ShowDepartmentResult \
    --outcome=Department/DepartmentOutcome

# The parcel of an operation (create, show, edit, save, destroy), or a QueryParcel for a listing
php scribe make:parcel Department/Capabilities/SaveDepartmentParcel --artifact=company --operation=save --model=Department
php scribe make:parcel Department/Capabilities/IndexDepartmentParcel --artifact=company --query --model=Department

# Repository contract and implementation, outcome, policy
php scribe make:repository Department --artifact=company
php scribe make:outcome Department --artifact=company --result=Department/Capabilities/SaveDepartmentResult
php scribe make:policy DepartmentPolicy --artifact=company --model=Department

# A request whose rules come from the model (store, update, destroy) and that authorizes its action;
# --nullable lists the fields accepted empty (make:capabilities passes it on, make:model fills it from its nullable fields)
php scribe make:request StoreDepartmentRequest --artifact=company --capability --model=Department \
    --operation=store --action=Capabilities/Department/SaveDepartmentCapability --nullable=description

# Resource listing the model's fields; controller running the capabilities
php scribe make:resource DepartmentResource --artifact=company --model=Department
php scribe make:controller DepartmentController --artifact=company --capabilities --model=Department
```

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

# Result enum; FEEDBACK is guessed from the verb (Save → saved, Destroy → deleted, ...),
# and each case is commented with the feedback key it reads
php scribe make:result Branch/Capabilities/SaveBranch --artifact=company
php scribe make:result Branch/Capabilities/ArchiveBranch --artifact=company --feedback=archived

# Parcel
php scribe make:parcel Branch/Capabilities/ShowBranchParcel --artifact=company
```

The generated repository property is named after the contract (`BranchRepositoryInterface` becomes `$branches`). Results whose name contains `Translation` get a `translation.` feedback group. With `--outcome` the generated code returns `BranchOutcome::fromResult(...)` (pass the artifact Outcome); without `--result`/`--outcome` it returns `Outcome::success()` / `Outcome::failure()`. Pass the data as the second argument of `fromResult()`.

<a name="the-artifact-summary"></a>
### The Artifact Summary

An artifact has a summary: the page its panel opens on, at `panel/artifact/{artifact}`. It is generated in two parts, the backend by `make:summary` and the frontend by `make:summary-page`:

```bash
php scribe make:summary library        # ShowArtifactCapability, its result, ArtifactController and the route
php scribe make:summary-page library   # the summary page of the artifact's panel, and the types of its data
```

`make:artifact` runs `make:summary` for every artifact it creates (unless `--minimal`), and with `--panel`, or when it is confirmed at the prompt, `make:panel`, which writes the summary page with the rest of the panel:

```bash
php scribe make:artifact library --description="Books, their authors and genres" --panel
```

`make:summary {artifact} {--force}` writes:

| File | Content |
| --- | --- |
| `src/Capabilities/Artifact/ShowArtifactCapability.php` | Returns `ArtifactSummary::of('{artifact}')`; edit it to add to the summary |
| `src/Enums/Artifact/Capabilities/ShowArtifactResult.php` | Its result, with the `showed` feedback |
| `src/Http/Controllers/Artifact/ArtifactController.php` | `show()` renders `{artifact}::artifact/Show` with the summary as `data` (JSON for clients expecting JSON) |
| `routes/web.php` | `Route::get('/', [ArtifactController::class, 'show'])->name('{artifact}.artifact.show')`, in place of a `Route::inertia()` summary route, else first in the artifact's group, else in a new `panel/artifact/{artifact}` group |

A former `{artifact}.module.show` route, in whichever route file of the artifact holds it, is replaced by the new one; a `Route::controller(ModuleController::class)` group left holding only it is unwrapped, and the unused `ModuleController` import dropped. The former `Module` classes are not deleted.

Its stubs may be copied to the application's `stubs/summary/{capability,controller}.stub` to be replaced.

`Heritage\Foundation\Support\ArtifactSummary` reads a loaded artifact by convention:

| Key | Content |
| --- | --- |
| `artifact` | Its `art.php` manifest: `id`, `name`, `description`, `version`, `requires`, and the capabilities it `provides` |
| `stats` | The totals of its `entities`, `records`, records `created_this_month`, `translatable` entities, `translations`, `capabilities` and `features` |
| `entities` | A model each from `src/Models` (translation models aside): its `table`, `records`, `created_this_month`, `active` records (with an `active` column), `trashed` records (when soft deletable), `translations`, `last_updated_at`, the `capabilities` and `features` of `src/Capabilities/{Model}` and `src/Features/{Model}`, and the `url` of its `{model}.index` route. A model whose table does not match it yet (its migration not run) has `available` false and no figures, rather than failing the summary |
| `recent` | Its 8 latest updated records across its models: the `label` (the first filled of `name`, `title`, `label`, `full_name`, `email`, `slug`, `code`, translated, else its key), whether it was `created` or updated, and the `url` of its `{model}.show` route |

`make:summary-page {artifact} {--stack=} {--force}` writes, in the artifact's panel folder (created by `make:panel`):

| File | Content |
| --- | --- |
| `pages/artifact/Show.vue` | The artifact's identity and version, a card per total, a card per entity (its sidebar icon, records, badges of the records added this month, active, deleted, translations, capabilities and features, linking to its listing), the latest activity linking to each record, and the artifact's requirements and provided capabilities |
| `types/artifact/summary.d.ts` | The `ArtifactSummary` interface and its parts, the shape of `Heritage\Foundation\Support\ArtifactSummary` |
| `lang/{en,ar}/summary.php` | The page's labels, in the application, when missing |

When the artifact has no `ShowArtifactCapability`, `make:panel` and `make:summary-page` run `make:summary` first.

<a name="generating-the-ui"></a>
### Generating the UI

The frontend has its own commands, named as the backend ones are; the backend commands above write nothing for it. Once an artifact and its models exist, `make:panel` writes the artifact's panel in `resources/js/apps/panel/artifacts/{artifact}`, imported through the `@artifact/{artifact}` alias, and with `--all` the pages of its models, from what the artifact declares:

```bash
# Backend
php scribe make:artifact library --description="Books, their authors and genres"
php scribe make:model Author --artifact=library --all --fields="name:string,slug:string:unique,active:boolean"
php scribe make:model Book --artifact=library --all --fields="author_id:foreignId,title:string,price:decimal,active:boolean"

# Frontend: the panel, then every model with capabilities, in the order their tables were created
php scribe make:panel library --icon=Library --all --icons=Author:UserPen,Book:BookOpen
```

The parts can be run alone:

```bash
php scribe make:panel library --icon=Library                     # the panel
php scribe make:summary-page library                             # the summary page alone
php scribe make:pages Book --artifact=library --icon=BookOpen    # a model's pages (creates the panel when needed)
php scribe make:component BookCard --artifact=library --model=Book --type=summary   # one component
php scribe lang:sync                                             # the translations only
```

The files are written in the frontend stack of the application: `--stack=` names it, else the `ui.stack` config, else it is detected from the dependencies of `package.json` (`vue` or `@inertiajs/vue3` for Vue, the only stack supported for now). Each stack has its stubs under `stubs/{stack}`, and a stub copied to the application's `stubs/ui/{stack}` folder, under the same path, replaces the framework's.

`make:panel {artifact} {--icon=} {--group=master_data} {--all} {--icons=Model:Icon,…}` creates the artifact's panel and registers it:

| File | Content |
| --- | --- |
| `navigation.ts` | The artifact's inner sidebar: its summary, then its entities (`{artifact}Entities()`, filled by `make:pages`) |
| `layouts/{Artifact}Layout.vue`, `layouts/index.ts` | The artifact layout with its collapsible inner sidebar |
| `pages/artifact/Show.vue`, `types/artifact/summary.d.ts` | The [summary page](#the-artifact-summary) and the types of its data, as `make:summary-page` writes them; the summary capability and its route are generated first by `make:summary` when the artifact has none |
| `artifacts/{artifact}/lang/en/artifact.php` | The artifact's name and description (`{artifact}::artifact.name`, `{artifact}::artifact.description`) |
| `resources/js/app.ts` | The artifact's pages in the Inertia pages glob |
| `vite.config.ts`, `tsconfig.json` | The `@artifact/{artifact}` alias, and the artifact's `lang` folder in `additionalLangPaths` |
| `apps/panel/composables/usePanelModuleNav.ts` | The artifact's item in the `--group` of the main sidebar, with its icon |

The stubs name the artifact with the `[[ artifact ]]`, `[[ Artifact ]]` and `[[ artifactCamel ]]` placeholders; the former `[[ module ]]`, `[[ Module ]]` and `[[ moduleCamel ]]` are still filled for the stubs an application overrides.

`make:pages {model} --artifact= {--icon=} {--only=} {--except=}` creates the pages and components of a model, as the company artifact's are written, and adds the model to the artifact's inner sidebar:

| File | Content |
| --- | --- |
| `types/{model}/{model}.d.ts` | The zod schema of the model's resource, its collection and its table column metadata |
| `pages/{model}/{Index,Create,Edit,Show}.vue` | The pages the controller renders (`{artifact}::{model}/Index`, …) |
| `layouts/{Model}Layout.vue` | The show page shell: header with the update and delete dialogs, and a tab per section |
| `pages/{model}/partials/` | The table and its columns, the create, update, delete and show dialogs with their forms and zod schemas, the summary, and `index.ts` exporting them |
| `pages/{model}/partials/relations/` | A select per foreign key (`AuthorSelect.vue`) and the loader of its options (`useAuthorOptions.ts`) |

The fields are those of the model's store request, in its order: the rules give whether a field is required and its bounds (`min`, `max`), the artifact's migrations give its type (else the model's casts, else the rules), a field ending in `_id` whose model exists in the artifact is a foreign key, and json fields are left out of the forms. A record is named by its `name`, `title`, `slug` or first string field; deleting it asks for its slug (else that name, else its id).

A translatable model gets the forms and tab of the `Branch` pages:

| File | Content |
| --- | --- |
| `pages/{model}/partials/store/Store.vue`, `update/Update.vue` | The capability forms: the model's own columns alone, posted to `{model}.store` / `{model}.update` |
| `pages/{model}/partials/store/Dialog.vue`, `update/Dialog.vue` | The capability form, its own columns alone, with `in-dialog`, which adds the shortcut to the create or edit page; the create and edit pages show the feature forms, with the translations and without the shortcut |
| `pages/{model}/partials/store/feature/Store.vue`, `update/feature/Update.vue` | The feature forms, posted to `{model}-feature.store` / `{model}-feature.update`: its plain fields, then a section of translated fields per locale, sent as `translations.{locale}.{field}`: the current locale's section is always shown, the others are added and removed from a menu. In the update form, a stored locale whose section is removed is sent empty, which deletes it. A unique `slug` is generated from the current locale's name until it is typed |
| `types/{model}/{model}-translation.d.ts` | The zod schema of the translation resource |
| `pages/{model}/partials/translation/` | The Translations tab of the show page: an alert when the record has no translation or misses some locales (offering to add them), and the table of its translations with a show, update and delete dialog per locale and a create dialog, posting to the `{model}-translation` routes |

The pages offer six operations, all generated by default; `--only` and `--except` take a comma separated list of them:

| Operation | Files |
| --- | --- |
| `index` | The index page, the table and its columns, and the model's item in the artifact sidebar |
| `show` | The show page and its layout, the quick view dialog and the summary |
| `create` | The create page, the create form and its dialog |
| `edit` | The edit page, the update form and its dialog |
| `destroy` | The delete confirmation form and its dialog |
| `translations` | The Translations tab of a translatable model (needs `show`) |

```bash
# Read-only pages
php scribe make:pages Book --artifact=library --only=index,show

# Everything but the deletion
php scribe make:pages Book --artifact=library --except=destroy
```

The pages link to what is generated only: without `destroy`, neither the layout, the quick view nor the table rows offer the delete dialog; without `index`, the breadcrumbs do not link to it. An operation whose routes are not registered (`book.index`, `book-translation.store`, …), or the translations of a model without translatable fields, is left out with a warning, and is an error when asked by `--only`.

`make:component {name}` creates one component. In an application with a frontend stack (or given `--type`), it writes a component of that stack; otherwise, or with `--stack=blade`, `--view` or `--inline`, a Blade view component as before:

```bash
php scribe make:component Widgets/EmptyState                                  # resources/js/components/Widgets/EmptyState.vue
php scribe make:component ConfirmBox --artifact=library --type=dialog         # artifacts/library/components/ConfirmBox.vue
php scribe make:component BookTable --artifact=library --model=Book --type=table   # artifacts/library/pages/book/partials/BookTable.vue
```

| Type | Component |
| --- | --- |
| `blank` (default) | An empty component with its slot |
| `dialog` | A modal with a title, a trigger slot and a content slot receiving `close` |
| `form` | The model's create form (`{Name}.vue`) and its schema (`{Name}.form.ts`) |
| `table` | The model's table (`{Name}.vue`) and its columns (`{Name}.columns.ts`), without row actions |
| `select` | A select of the model's records (`{Name}.vue`) and the loader of its options (`use{Model}Options.ts`) |
| `summary` | The model's summary |

`form`, `table`, `select` and `summary` need `--model` (with `--artifact`); the relation selects and filter options they import are written when missing.

`make:panel` and `make:pages` end with `lang:sync`, then regenerate the Wayfinder routes the pages import (`wayfinder:generate --with-form`); `make:field` runs `lang:sync` too when the application has JSON translations. `lang:sync` writes every available locale's `lang/{locale}.json` from the application's `lang/{locale}/*.php` (as `{file}.{key}`) and the artifacts' (as `{artifact}::{file}.{key}`, a null value as empty): the synced keys are added or updated and the other keys kept, or with `--fresh` the file is rebuilt from the PHP translations only. Existing files are kept unless `--force` is given, and running a command again adds nothing twice; every file written or edited is reported, and a place that cannot be found (a list edited by hand) is reported to be completed by hand.
