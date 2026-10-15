# Eloquent Model Translations

- [Introduction](#introduction)
    - [Terminology: Translatable vs. Translations](#terminology)
- [Architecture](#architecture)
    - [Dual-Table Design](#dual-table-design)
    - [Locale Resolver](#locale-resolver)
    - [The Canonical Translations Payload](#canonical-payload)
- [Defining a Translatable Model](#defining-a-translatable-model)
    - [Optional Translation Model Class](#optional-translation-model-class)
- [Translatable Migrations](#translatable-migrations)
- [Writing Translations](#writing-translations)
    - [Mass Assignment With `translations`](#mass-assignment)
    - [Merging & Deleting Locales](#merging-and-deleting-locales)
    - [Setting Individual Translations](#setting-individual-translations)
- [Reading Translations](#reading-translations)
    - [Fallback Chain](#fallback-chain)
    - [Eager Loading](#eager-loading)
    - [Query Scopes](#query-scopes)
    - [Translations in API Resources](#translations-in-resources)
- [Timestamps & Touching the Parent](#timestamps)
- [Deleting & Soft Deleting](#deleting)
    - [Soft Deleting Translations](#soft-deleting-translations)
- [Validating Translatable Input](#validating-translatable-input)
- [Translation Repositories & Capabilities](#translation-repositories)
- [Converting Locale Storage](#converting-locale-storage)
- [Translatable Factories](#translatable-factories)
- [Translatable Seeders](#translatable-seeders)
- [CLI Scaffolding](#cli-scaffolding)
- [API Reference](#api-reference)

<a name="introduction"></a>
## Introduction

Ugarit provides first-class, database-level localization for Eloquent models. Translation support is built into the base `Heritage\Database\Eloquent\Model` class: a model becomes translatable simply by declaring a `$translatable` property. Translated values are stored in a companion translation table, and the framework handles reading, writing, validation, eager loading and timestamps for you.

Translations work **with or without** the i18n artifact:

- **Without i18n**, the list of locales comes from `config/locale.php` and translation tables store the locale code itself (`ar`, `en`) in a `locale` column.
- **With the i18n artifact installed**, locales come from its `locales` table and translation tables store a `locale_id` foreign key.

Your code is identical in both cases, because every public API speaks in **locale codes** (`'ar'`, `'en'`) and a single service, the [locale resolver](#locale-resolver), converts codes to whatever the database stores.

<a name="terminology"></a>
### Terminology: Translatable vs. Translations

Two related words are used with distinct meanings throughout the framework:

| Term | Meaning | Examples |
| :--- | :--- | :--- |
| **translatable** (adjective) | The *declaration* of what can be translated | `$translatable` property, `->translatable()` column modifier (moves a column to the translation table; `->withTranslations()` keeps it on both), `Schema::createTranslatable()`, `isTranslatable()`, `--translatable` CLI flag |
| **translation / translations** (noun) | The translated *data* itself | `translations` payload key, `translations()` relation, `branch_translations` table, `BranchTranslation` model |

The singular `translation` refers to one locale (the `translation()` relation, `fillTranslation()`), and the plural `translations` to several locales (`translations()`, `fillTranslations()`, the `translations` payload).

<a name="architecture"></a>
## Architecture

<a name="dual-table-design"></a>
### Dual-Table Design

- **Primary table** (`branches`): non-translatable attributes such as `slug`, `active`, `timestamps`.
- **Translation table** (`branch_translations`): one row per locale holding the translated attributes (`name`, `description`), the parent foreign key (`branch_id`), the locale column (`locale` or `locale_id`) and the row's own `created_at` / `updated_at`.

You only write the primary model class. When no physical `BranchTranslation.php` exists next to it, the framework uses `Heritage\Database\Eloquent\RuntimeTranslation`: a single real class extending `ModelTranslation`, configured per instance with the model's translation table, parent class and foreign key (no class is generated at runtime).

<a name="locale-resolver"></a>
### Locale Resolver

The `Heritage\Contracts\Translation\LocaleResolver` contract is the single source of truth for locales used by translatable models:

| Method | Returns |
| :--- | :--- |
| `current()` | The active locale code |
| `fallback()` | The fallback locale code |
| `available()` | Codes of every locale the application accepts |
| `isAvailable(string $code)` | Whether a code is accepted |
| `default()` | The application's default locale code |
| `storageColumn()` | `'locale'` or `'locale_id'` |
| `storageTable()` | The table the locale column references (`locales` with i18n), or `null` when codes are stored |
| `toStorage(string $code)` | The stored value for a code, or `null` when unknown |
| `fromStorage(string|int $value)` | The code for a stored value |
| `label(string $code)` | A human-readable name (e.g. `العربية`), used in validation messages |

The active resolver is obtained through `Heritage\Translation\Locales::resolver()`, in this order:

1. The class named in `config('locale.resolver')`, when set.
2. The resolver an artifact bound to the contract. The i18n artifact binds `Ugarit\Artifacts\I18n\Localization\DatabaseLocaleResolver` (storage: `locale_id`; only *active* locales are accepted for input, every stored locale can still be read).
3. `Heritage\Translation\ConfigLocaleResolver` (storage: `locale`; locales from `config('locale.available_locales')`).

```php
use Heritage\Translation\Locales;

$resolver = Locales::resolver();

$resolver->available();        // ['ar', 'en']
$resolver->toStorage('ar');    // 'ar' (config) or 2 (i18n locale_id)
$resolver->label('ar');        // 'العربية'
```

The resolver is resolved once per application container. If you rebind the contract or change `locale.resolver` at runtime (for example in a test), call `Locales::flush()`.

To use your own strategy, implement the contract and set `resolver` in `config/locale.php` (see [Localization](/docs/{{version}}/localization#locale-resolver)).

<a name="canonical-payload"></a>
### The Canonical Translations Payload

Every layer exchanges translations in one shape, keyed by locale code:

```php
'translations' => [
    'ar' => ['name' => 'المقر الرئيسي', 'description' => '...'],
    'en' => ['name' => 'Headquarters', 'description' => '...'],
]
```

- Form requests **output** it from `validated()`.
- Models **accept** it in `fill()`, `create()` and `update()`.
- Resources **return** it with `translationsToArray()` (each entry also carries its `id`, `locale` and timestamps), so an edit form can be filled from the same shape it submits.
- Factories, seeders and frontend forms use it as well.

A locale given as `null` (`'en' => null`) means *delete this locale's translation*.

A flat translatable attribute (`'name' => '...'`) always means *the current locale*.

<a name="defining-a-translatable-model"></a>
## Defining a Translatable Model

Declare the `$translatable` attributes. No trait is required, because the base `Model` already provides translation support:

```php
namespace Ugarit\Artifacts\Company\Models;

use Heritage\Database\Eloquent\Factories\HasFactory;
use Heritage\Database\Eloquent\Model;
use Heritage\Database\Eloquent\SoftDeletes;

class Branch extends Model
{
    use HasFactory, SoftDeletes;

    /**
     * Primary table attributes.
     *
     * @var list<string>
     */
    protected $fillable = ['slug', 'active'];

    /**
     * Attributes stored in branch_translations.
     *
     * @var list<string>
     */
    protected array $translatable = ['name', 'description'];

    /**
     * Column used when no translation exists in the current or fallback locale.
     *
     * @var array<string, string>
     */
    protected array $translatableFallback = ['name' => 'slug'];
}
```

Translatable attributes are mass assignable automatically; you do not need to repeat them in `$fillable`.

Optional properties:

| Property | Default | Purpose |
| :--- | :--- | :--- |
| `$translatableFallback` | `[]` | Column used for an attribute when no translation exists (`['name' => 'slug']`) |
| `$eagerLoadTranslations` | `true` | Eager-load the current and fallback translations on every query ([details](#eager-loading)) |
| `$loadTranslationOnDemand` | `true` | Query a missing translation lazily when it is not loaded |
| `$translationModel` | conventional | Class of the translation model |
| `$translationTable` | `{singular}_translations` | Translation table name |
| `$translationForeignKey` | `{singular}_id` | Foreign key on the translation table |

> [!NOTE]
> Artifacts do not ship their own translation traits: every model, including the i18n artifact's own models, uses this kernel implementation, which reaches the i18n artifact only through the locale resolver.

<a name="optional-translation-model-class"></a>
### Optional Translation Model Class

When you need custom relations, scopes or events on the translation table, create a class extending `Heritage\Database\Eloquent\ModelTranslation` next to the model (`BranchTranslation` for `Branch`), or point `$translationModel` to it:

```php
namespace Ugarit\Artifacts\Company\Models;

use Heritage\Database\Eloquent\ModelTranslation;

class BranchTranslation extends ModelTranslation
{
    //
}
```

`ModelTranslation` provides:

- the parent foreign key, `locale_id` / `locale` and the parent's translatable attributes as fillable attributes;
- the table name by convention (`BranchTranslation` → `branch_translations`);
- the `parent()` relation and a named alias (`$translation->branch`);
- `getLocaleColumn()` (the resolver's column unless `$localeColumn` is set) and `getLocaleCode()` (the row's locale code whatever the storage);
- `locale()`: the i18n `Locale` model (`config('i18n.models.locale')`) when storage is `locale_id`, otherwise the locale code;
- its own timestamps when the table has the columns, and [touching the parent](#timestamps).

<a name="translatable-migrations"></a>
## Translatable Migrations

Define the primary table and its translation table in a single migration with `Schema::createTranslatable`, marking translated columns with `->translatable()`:

```php
use Heritage\Database\Migrations\Migration;
use Heritage\Database\Schema\Blueprint;
use Heritage\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::createTranslatable('branches', function (Blueprint $table) {
            $table->id();
            $table->string('slug')->unique();
            $table->string('name')->translatable()->unique();
            $table->boolean('active')->default(true);
            $table->softDeletes();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExistsTranslatable('branches');
    }
};
```

The translation table receives:

- `id` and the parent foreign key (`branch_id`) with cascading deletes, matching the parent key type (`id`, `uuid`, `ulid`, `increments`);
- the locale column from the resolver: `locale_id` (foreign key to `locales`) with the i18n artifact, otherwise `string('locale', 10)`;
- every `->translatable()` column; a translatable column marked `unique()` is unique **per locale** (`unique(locale, name)`), not across the whole table;
- a compound unique index on `[branch_id, locale]` (one row per locale);
- every column marked `->withTranslations()`, which stays on the primary table as well; `$table->softDeletes()->withTranslations()` [soft deletes translations](#soft-deleting-translations);
- its own `created_at` / `updated_at`.

The signature is `Schema::createTranslatable(string $table, Closure $callback, array $options = [])`; see [Migrations](/docs/{{version}}/migrations#creating-translatable-tables) for the options (`table`, `foreign_key`, `locale_column`, `locale_length`, `timestamps`, `translation`). Unknown options throw an `InvalidArgumentException`.

<a name="writing-translations"></a>
## Writing Translations

<a name="mass-assignment"></a>
### Mass Assignment With `translations`

`fill()`, `create()` and `update()` accept the canonical payload:

```php
$branch = Branch::create([
    'slug' => 'headquarters',
    'active' => true,
    'translations' => [
        'ar' => ['name' => 'المقر الرئيسي'],
        'en' => ['name' => 'Headquarters'],
    ],
]);

$branch->update([
    'translations' => ['en' => ['name' => 'Main Office']],
]);
```

While filling, the model skips:

- keys that are not translatable attributes;
- locales the resolver does not know;
- a locale whose values are all empty (`null` or `''`), such as an untouched language tab.

Within a locale that has at least one value, an empty value **clears** its attribute: `['ar' => ['name' => 'المقر', 'description' => '']]` sets the Arabic description to `null`.

A flat translatable attribute sets the current locale: `$branch->update(['name' => 'المقر'])`.

<a name="merging-and-deleting-locales"></a>
### Merging & Deleting Locales

A `translations` payload is **merged** into the stored translations: submitted locales are created or updated, and locales the payload omits are left untouched. In the update above, the Arabic translation is kept and only English changes. This makes partial updates (an API call or a form editing one locale) safe by default.

To delete a locale, send it explicitly as `null`:

```php
$branch->update([
    'translations' => [
        'ar' => ['name' => 'المقر'],   // created or updated
        'en' => null,                  // deleted
    ],                                 // other locales: untouched
]);
```

From a form, submit an empty value for the locale (`translations[en]=` or the field `translations.en` with an empty value): the `ConvertEmptyStringsToNull` middleware turns it into `null`, and the [form request](#validating-translatable-input) passes it through to the model.

Deletions are queued and applied on save. Setting a value for the same locale afterwards cancels the deletion.

<a name="setting-individual-translations"></a>
### Setting Individual Translations

```php
// One attribute, one locale (defaults to the current locale)
$branch->setTranslation('name', 'فرع دمشق', 'ar');

// One attribute, several locales
$branch->setTranslation('name', ['ar' => 'فرع دمشق', 'en' => 'Damascus Branch']);

// Several attributes, one locale
$branch->fillTranslation('ar', ['name' => 'فرع القدس', 'description' => '...']);

// Canonical payload (merged; a null locale is deleted)
$branch->fillTranslations(['ar' => ['name' => 'فرع القدس'], 'en' => null]);

// Delete one locale
$branch->forgetTranslation('en');

$branch->save();
```

`setTranslation()` throws an `InvalidArgumentException` for a locale the resolver does not accept.

Translations are queued and written when the model is saved, including quiet saves (`saveQuietly()`, `withoutEvents()`), because they are persisted by `Model::finishSave()` rather than by an event listener.

To create a translation row yourself, use `newTranslation()`, which returns an unsaved instance of the translation model:

```php
$translation = $branch->newTranslation(['name' => 'Branch']);
```

<a name="reading-translations"></a>
## Reading Translations

```php
echo $branch->name;                              // current locale, with fallbacks
echo $branch->getTranslatedAttribute('name', 'en');

$branch->translate('en');                        // translation model or null
$branch->translateOrDefault('fr');               // falls back to the fallback locale
$branch->hasTranslation('ar');                   // bool
```

`toArray()` / JSON include each translatable attribute resolved for the current locale.

<a name="fallback-chain"></a>
### Fallback Chain

Reading `$branch->name` resolves, in order:

1. the translation of the **current** locale;
2. the translation of the **fallback** locale (`config('locale.fallback_locale')`);
3. the column named in `$translatableFallback` (e.g. `slug`).

<a name="eager-loading"></a>
### Eager Loading

Every query on a translatable model eager-loads the `localizedTranslations` relation, which holds the current and fallback translations only. Listing 50 branches and reading their names therefore runs **two** queries, not 51:

```php
$names = Branch::all()->pluck('name'); // 2 queries
```

- Single-record queries (`find()`, `first()`, route model binding) skip the eager load: the translation is loaded lazily, with the same single query, only when an attribute is read.
- Load every locale explicitly with `Branch::with('translations')`.
- `Branch::withTranslation('fr')` loads every locale when the requested locale is neither current nor fallback.
- Disable the automatic eager load with `protected bool $eagerLoadTranslations = false;` or `Branch::withoutGlobalScope('localizedTranslations')`.

<a name="query-scopes"></a>
### Query Scopes

```php
// Records whose translation (current locale by default) matches
Branch::whereTranslation('name', 'فرع غزة')->get();
Branch::whereTranslation('name', 'Gaza Branch', 'en')->get();

// LIKE pattern
Branch::whereTranslationLike('name', '%غزة%')->get();

// Order by a translated attribute (current locale by default); records without that translation sort as null
Branch::orderByTranslation('name')->get();
Branch::orderByTranslation('name', 'desc', 'en')->get();

// Eager load translations for a locale
Branch::withTranslation('en')->get();
```

### Relations

```php
$branch->translations;            // HasMany: every locale
$branch->localizedTranslations;   // HasMany: current + fallback locales
$branch->translation('en');       // HasOne: one locale (current by default)

$translation->branch;             // parent (also $translation->parent)
$translation->getLocaleCode();    // 'en'
$translation->locale;             // i18n Locale model, or the code without i18n
```

<a name="translations-in-resources"></a>
### Translations in API Resources

Return translations in the same shape forms submit, keyed by locale code, with `translationsToArray()`:

```php
class BranchDetailsResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'slug' => $this->slug,
            'name' => $this->name,
            'translations' => $this->whenLoaded('translations', fn () => (object) $this->translationsToArray()),
        ];
    }
}
```

```json
"translations": {
    "ar": {"id": 5, "branch_id": 1, "locale_id": 2, "name": "المقر الرئيسي", "locale": "ar", "created_at": "...", "updated_at": "..."},
    "en": {"id": 6, "branch_id": 1, "locale_id": 1, "name": "Headquarters", "locale": "en", "created_at": "...", "updated_at": "..."}
}
```

Each entry holds the translation row's attributes plus its `locale` code. `translationsToArray()` loads the `translations` relation when needed; cast it to `object` so an empty result is encoded as `{}` rather than `[]`. A frontend can fill an edit form with `Object.entries(branch.translations)` and submit the result unchanged.

<a name="timestamps"></a>
## Timestamps & Touching the Parent

Translation rows keep their **own** `created_at` / `updated_at`, independent of the primary table. A change to translations is reflected on the parent's `updated_at`:

- When only translations change (`$branch->update(['translations' => [...]])`), the parent's `updated_at` is touched once.
- When the parent's own attributes change in the same save, its normal update already sets `updated_at`, so no extra query runs.
- Saving a translation row directly (`$translation->save()`) touches the parent through `ModelTranslation::$touches = ['parent']`, provided the parent uses timestamps.

Tables created before translation timestamps existed keep working: `ModelTranslation` only maintains timestamps when the table has the columns.

<a name="deleting"></a>
## Deleting & Soft Deleting

- Deleting a model deletes its translations (the foreign key also cascades).
- **Soft deleting** a model keeps its translations, so a restored record keeps its names. Translations are removed only on `forceDelete()`.

<a name="soft-deleting-translations"></a>
### Soft Deleting Translations

By default a deleted translation (one locale removed from a record) is deleted for good. Translation rows may instead be soft deleted, to audit or recover a locale's values.

#### Enabling

Two steps, one in the migration and one in the translation model.

1. Mark the soft delete column of the primary table with `withTranslations()`: the column stays on `branches` and is added to `branch_translations` too ([Columns on Both Tables](/docs/{{version}}/migrations#columns-on-both-tables)):

```php
Schema::createTranslatable('branches', function (Blueprint $table) {
    $table->id();
    $table->string('slug')->unique();
    $table->string('name')->translatable()->unique();
    $table->boolean('active')->default(true);
    $table->softDeletes()->withTranslations();
    $table->timestamps();
});
```

2. Use `SoftDeletes` on the translation model. This requires a physical translation class ([Optional Translation Model Class](#optional-translation-model-class)); a runtime translation model is never soft deleted.

```php
use Heritage\Database\Eloquent\ModelTranslation;
use Heritage\Database\Eloquent\SoftDeletes;

class BranchTranslation extends ModelTranslation
{
    use SoftDeletes;
}
```

`$branch->translationUsesSoftDeletes()` tells whether a model's translations are soft deleted.

#### Behavior

| Operation | Result |
| :--- | :--- |
| `forgetTranslation('en')`, `fillTranslations(['en' => null])`, an empty locale in a request | the `en` row is soft deleted on save |
| `$translation->delete()`, `TranslationRepository::destroyForLocale()`, `DestroyBranchTranslationCapability` | the row is soft deleted |
| Reading: `translations()`, `translate()`, `translationsToArray()`, translated attributes, eager loading, route bindings | trashed rows are hidden; a scoped route to a trashed translation is a 404 |
| Setting the locale again: `fillTranslations()`, `setTranslation()`, `fillTranslation()`, `TranslationRepository::save()` | the trashed row is **restored** (same `id`) with the new values; no second row is created |
| `$branch->delete()` (the parent is soft deleted) | translations are kept as they are, so `restore()` brings the names back |
| `$branch->forceDelete()` | every translation is deleted for good, trashed ones included |
| `$translation->restore()` / `$translation->forceDelete()` | the usual `SoftDeletes` methods on one row |

Restoring instead of inserting keeps the `unique([branch_id, locale])` index valid on every database: there is exactly one row per locale, live or trashed.

#### Querying Trashed Translations

The `SoftDeletes` scopes work on the translation relations:

```php
$branch->translations()->withTrashed()->get();   // every locale, trashed included
$branch->translations()->onlyTrashed()->get();   // trashed locales only

$branch->translations()->onlyTrashed()
    ->where('locale_id', Locales::resolver()->toStorage('en'))
    ->first()?->restore();                        // restore the English row as it was
```

<a name="unique-translated-values"></a>
#### Unique Translated Values

A translatable column marked `unique()` is unique per locale. With soft deleted translations, whether a trashed value still holds that uniqueness depends on the database:

| Database | Index | Trashed value | `UniqueTranslation` |
| :--- | :--- | :--- | :--- |
| PostgreSQL, SQLite, SQL Server | partial unique, `where deleted_at is null` | free again: another record may use it | ignores trashed rows |
| MySQL, MariaDB | unique (no partial indexes) | stays reserved until force deleted | counts trashed rows, so the request fails validation instead of the database |

The validation rule always matches the index, so a valid request never hits a database unique error.

#### Existing Tables

`createTranslatable()` builds the column and the indexes. For a translation table that already exists, write a migration adding the column (and, on PostgreSQL / SQLite / SQL Server, the partial index); see the note in [Soft Deleted Translations](/docs/{{version}}/migrations#soft-deleted-translations).

<a name="validating-translatable-input"></a>
## Validating Translatable Input

Declare the same `$translatable` property on the form request and write rules for the attribute as if it were a plain field:

```php
class StoreBranchRequest extends FormRequest
{
    protected array $translatable = ['name'];

    public function rules(): array
    {
        return [
            'slug' => 'required|string|unique:branches,slug',
            'name' => 'required|string|min:2|max:64',
        ];
    }
}
```

The request then:

- accepts a flat `name` (current locale), `translations[ar][name]`, or FormData keys `translations.ar.name`, and normalizes them into the canonical payload;
- validates every locale with the `name` rules (`translations.*.name`) and requires `translations` when the rules contain `required`;
- with `protected string|array $requiredLocales`, decides which locales a required field needs: `'any'` (default, at least one), `'default'` (the default locale), `'all'` (every available locale) or a list of codes such as `['ar']`; other locales stay optional;
- passes a locale sent as `null` / empty through to `validated()` as `null`, so the model [deletes it](#merging-and-deleting-locales);
- rejects locales the resolver does not accept (`translations[xx]` fails validation);
- returns translatable attributes **only** under `translations` from `validated()`, ready for `Branch::create($request->validated())`.

A translatable column marked `unique()` is unique per locale in the database. Validate it the same way with `Heritage\Validation\Rules\UniqueTranslation`, which reads the locale from the `translations.{locale}.{field}` attribute (or takes it as `locale:`) and ignores the record's own translations on update:

```php
use Heritage\Validation\Rules\UniqueTranslation;

'name' => ['required', 'string', new UniqueTranslation(Branch::class, ignore: $branch?->id)],
```

With [soft deleted translations](#unique-translated-values), the rule ignores trashed values only where the database index does (PostgreSQL, SQLite, SQL Server).

Fields are translatable only when the request declares them. See [Validation](/docs/{{version}}/validation#multilingual-and-translatable-form-requests) for details and error labels.

<a name="translation-repositories"></a>
## Translation Repositories & Capabilities

In the Capability / Feature architecture, a translatable model and its translation model each have their own capabilities: `SaveBranchCapability` saves the branch's columns, while `SaveBranchTranslationCapability` and `DestroyBranchTranslationCapability` save or delete one locale. A Feature (`SaveBranchFeature`) runs them for each locale of the `translations` payload as one [flow](/docs/{{version}}/features#flows), so a failed translation rolls the whole save back.

The translation capabilities use a repository extending `Heritage\Repositories\TranslationRepository`, which speaks locale codes and converts them through the [locale resolver](#locale-resolver):

```php
class BranchTranslationRepository extends TranslationRepository implements BranchTranslationRepositoryInterface
{
    public function __construct(?Model $model = null)
    {
        parent::__construct($model ?? (new Branch)->newTranslation());
    }
}
```

Like every `ModelRepository`, it takes Parcels and returns models:

| Method | Description |
| :--- | :--- |
| `save(Parcel $parcel)` | Upsert on (parent, locale): the parcel carries the parent key, a `locale` code and the translated attributes; an attached translation model is updated in place, and a [soft deleted](#soft-deleting-translations) one is restored. Returns the saved translation model |
| `findForLocale($parentId, string $locale)` | The translation model of a parent for a locale code, or `null` (trashed rows are not found) |
| `destroyForLocale($parentId, string $locale)` | Delete (or soft delete) the translation of a parent for a locale code; `false` when the locale had none |

Only the attributes the Parcel was built with are written (`Parcel::toAttributes()`): a parcel built by `fromArray(['locale' => 'en', 'branch_id' => 1, 'description' => null])` clears `description` and leaves `name` untouched.

An unknown locale throws an `InvalidArgumentException`. `Parcel::fromModel()` fills a `translations` property with `translationsToArray()` (keyed by locale code), and translation models expose `$translation->locale_code`.

#### Nested Translation Routes

Scope nested translation routes so a translation is only resolved through its own parent; another parent's translation is a 404:

```php
Route::resource('{branch}/branch-translation', BranchTranslationController::class)
    ->only(['store', 'update', 'destroy'])
    ->scoped(['branch_translation' => 'id']);
```

A translatable model resolves a child parameter named `translation`, `{model}_translation` or after its translation class through `translations()`, unless it defines the conventional relation (`branchTranslations()`). The controller needs no ownership check.

<a name="converting-locale-storage"></a>
## Converting Locale Storage

The storage mode is chosen when translation tables are created: a `locale` code column without the i18n artifact, a `locale_id` foreign key with it. Adopting or removing the i18n artifact in an existing database therefore requires converting the translation tables. The `translations:convert-locale-storage` command does it for every table ending with `_translations` that has a locale column:

```bash
# Adopting i18n: locale codes -> locale_id (install and seed i18n locales first)
php scribe translations:convert-locale-storage --to=locale_id --dry-run
php scribe translations:convert-locale-storage --to=locale_id

# Leaving i18n: locale_id -> locale codes (run while i18n is still installed)
php scribe translations:convert-locale-storage --to=locale

# Only some tables, another connection
php scribe translations:convert-locale-storage --to=locale_id --table=branch_translations --database=tenant
```

For each table the command adds the new column, fills it through the active locale resolver, drops the indexes and foreign key that use the old column, drops the old column, makes the new one required, adds the foreign key to the resolver's `storageTable()` (or an index on `locale`), and recreates the former indexes on the new column (e.g. `unique(branch_id, locale_id)`).

- The mapping always goes through the i18n resolver, so the command refuses to run when the active resolver does not store `locale_id`.
- `--dry-run` prints each table's row count and value mapping without changing anything; values that cannot be mapped are listed and stop the real conversion before any table is altered.
- Back up the database first. The command asks for confirmation in production unless `--force` is given.

The conversion logic is also available as `Heritage\Translation\LocaleStorageConverter` (`translationTables()`, `convert($table, $to, $dryRun)`).

<a name="translatable-factories"></a>
## Translatable Factories

Declare translatable attributes directly in the factory's `definition()`; the factory creates a translation for every locale the resolver makes available:

```php
class BranchFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name' => $this->faker->company(),
            'slug' => $this->faker->unique()->slug(),
            'active' => true,
        ];
    }
}
```

```php
$branch = Branch::factory()->create();

$branch->getTranslatedAttribute('name', 'ar');
$branch->getTranslatedAttribute('name', 'en');
```

With the i18n artifact, "available" means the active locales in the `locales` table, so seed locales before translatable models.

Fluent states override the generated values:

```php
Branch::factory()->withTranslations([
    'ar' => ['name' => 'فرع دمشق الرئيسي'],
    'en' => ['name' => 'Damascus Main Branch'],
])->create();

Branch::factory()->inLocale('ar', ['name' => 'فرع القدس'])->create();

Branch::factory()->withTranslation('name', 'ar', 'فرع بيروت')->create();
```

<a name="translatable-seeders"></a>
## Translatable Seeders

```php
class BranchSeeder extends Seeder
{
    public function run(): void
    {
        Branch::factory()->withTranslations([
            'ar' => ['name' => 'المقر الرئيسي'],
            'en' => ['name' => 'Headquarters'],
        ])->create(['slug' => 'headquarters', 'active' => true]);

        // Or without the factory
        Branch::create([
            'slug' => 'north',
            'active' => true,
            'translations' => ['ar' => ['name' => 'الشمال'], 'en' => ['name' => 'North']],
        ]);

        Branch::factory()->count(10)->create();
    }
}
```

<a name="cli-scaffolding"></a>
## CLI Scaffolding

```bash
# Translatable model (single model file, translation model generated at runtime)
php scribe make:model Branch --translatable
php scribe make:model Branch -t

# Inside an artifact, with migration, factory, seeder and the capabilities slice
php scribe make:model Branch --artifact=company --translatable --all

# Also generate a physical BranchTranslation class
php scribe make:model Branch --artifact=company --translatable --with-translation-file

# A standalone translation model extending ModelTranslation
php scribe make:model BranchTranslation --artifact=company --translation-model

# A translatable table migration (Schema::createTranslatable)
php scribe make:migration create_branches_table --translatable

# A form request declaring translatable fields
php scribe make:request StoreBranchRequest --artifact=company --translatable --fields=name,description
```

| Flag | Command | Effect |
| :--- | :--- | :--- |
| `-t`, `--translatable` | `make:model` | Model with `$translatable` / `$translatableFallback` and, with `--migration`, a translatable migration |
| `--with-translation-file` | `make:model` | Also writes `{Model}Translation.php` |
| `--translation-model` | `make:model` | Writes only a `ModelTranslation` subclass |
| `-t`, `--translatable` | `make:migration` | Uses `Schema::createTranslatable` / `dropIfExistsTranslatable` |
| `-t`, `--translatable`, `--fields=` | `make:request` | Declares `$translatable` in the request |
| `--translatable` | `model:list` | Lists only translatable models |
| `--to=locale_id\|locale`, `--table=`, `--dry-run` | `translations:convert-locale-storage` | [Converts](#converting-locale-storage) translation tables between storage modes |

<a name="api-reference"></a>
## API Reference

### Model (`Heritage\Database\Eloquent\Concerns\Translatable`)

| Method | Description |
| :--- | :--- |
| `isTranslatable()` | Whether `$translatable` is non-empty |
| `getTranslatable()` / `isTranslatableAttribute($key)` | Declared attributes |
| `fillTranslations(array $translations)` | Merge the canonical payload; a `null` locale is deleted on save |
| `forgetTranslation(string $locale)` | Queue the deletion of a locale's translation |
| `translationsToArray()` | Stored translations keyed by locale code (for resources and forms) |
| `fillTranslation(string $locale, array $attributes)` | Fill one locale |
| `setTranslation(string $key, mixed $value, ?string $locale = null)` | Set one attribute (array value = several locales) |
| `translate(?string $locale = null)` | Translation model for a locale, or `null` |
| `translateOrDefault(?string $locale = null)` | Same, falling back to the fallback locale |
| `hasTranslation(?string $locale = null)` | Whether a translation exists |
| `getTranslatedAttribute(string $key, ?string $locale = null)` | Translated value, or `null` |
| `translations()` / `localizedTranslations()` / `translation(?string $locale)` | Relations |
| `newTranslation(array $attributes = [])` | New unsaved translation model (a configured `RuntimeTranslation` when the model has no translation class) |
| `getTranslationModel()` / `getTranslationTable()` / `getTranslationForeignKey()` / `getTranslationLocaleColumn()` | Conventions |
| `savePendingTranslations()` | Persist queued translations and deletions (called on save); restores a soft deleted locale that is set again |
| `translationUsesSoftDeletes()` | Whether the translation model uses `SoftDeletes` |
| `scopeWhereTranslation()` / `scopeWhereTranslationLike()` / `scopeOrderByTranslation()` / `scopeWithTranslation()` | Query scopes |
| `static localeResolver()` | The active `LocaleResolver` |

### Schema

| Method | Description |
| :--- | :--- |
| `Schema::createTranslatable($table, Closure $callback, array $options = [])` | Create the primary and translation tables |
| `Schema::tableTranslatable($table, Closure $callback, array $options = [])` | Alter the primary and translation tables (see [Updating Translatable Tables](/docs/{{version}}/migrations#updating-translatable-tables)) |
| `Schema::dropTranslatable($table, ?string $translationTable = null)` | Drop both tables |
| `Schema::dropIfExistsTranslatable($table, ?string $translationTable = null)` | Drop both tables if they exist |
| `$column->translatable()` / `$column->isTranslatable()` | Mark / check a translatable column (moved to the translation table) |
| `$column->withTranslations()` / `$column->isWithTranslations()` | Mark / check a column kept on the primary table and added to the translation table |
| `$column->isSoftDeletes()` | Whether the column was created by `softDeletes()`, `softDeletesTz()` or `softDeletesDatetime()` |
| `Schema::supportsPartialIndexes()` | Whether the connection supports partial indexes (not MySQL / MariaDB) |
| `$table->translatable(Closure $callback)` | Mark every column defined in the callback as translatable |
