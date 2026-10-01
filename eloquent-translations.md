# Eloquent Model Translations

Ugarit provides first-class, database-level localization support for Eloquent models via the `Translatable` concern and the intelligent **Virtual Translation Model** engine.

---

## 1. Overview & Architecture

Instead of storing localized strings inside fragile JSON columns or requiring cumbersome companion translation classes, Ugarit combines the power of the **Dual-Table pattern** with a zero-boilerplate **Virtual Translation Engine**:

1. **Dual-Table Database Design:**
   - **Primary Entity Table:** Stores non-translatable attributes (e.g. `slug`, `active`, `price`, `timestamps`).
   - **Translation Entity Table:** Stores localized attributes (`branch_id`, `locale_id` / `locale`, `name`, `description`).
2. **Virtual Translation Engine:**
   - Developers only define **a single model file** (e.g., `Branch.php`).
   - If no physical `BranchTranslation.php` file exists, Ugarit dynamically generates and binds a `VirtualTranslation` instance extending `ModelTranslation` at runtime with zero performance overhead.
   - Physical `*Translation.php` files remain 100% optional for advanced use cases where custom relations or scopes are required on the translation table.

---

## 2. CLI Scaffolding (`make:model`)

You can generate translatable models across applications or modular domain artifacts using `php scribe make:model`:

```bash
# Generate a translatable model (Single file + Virtual Translation Engine)
php scribe make:model Branch --translatable

# With short flag alias (-t)
php scribe make:model Branch -t

# Inside a Modular Domain Artifact (artifacts/company/src/Models)
php scribe make:model Branch --artifact=company --translatable

# Generate everything (Migration, Factory, Seeder, Controller, Requests)
php scribe make:model Branch --artifact=company --translatable --all

# (Optional) Generate a physical companion Translation class file
php scribe make:model Branch --artifact=company --translatable --with-translation-file

# Generate a standalone translation model extending ModelTranslation
php scribe make:model BranchTranslation --artifact=company --translation-model
```

### Command Flags Summary:

| Option Flag | Description | Output |
| :--- | :--- | :--- |
| `-t`, `--trans`, `--translatable`, `--translation` | Configures model with virtual multilingual translations | Creates a single `Model.php` file with `Translatable` |
| `--with-translation-file` | Instructs the generator to create a physical companion translation file | Creates `Model.php` and `ModelTranslation.php` |
| `--translation-model` | Creates a standalone translation class extending `ModelTranslation` | Creates `ModelTranslation.php` only |
| `--artifact=slug` | Targets a specific modular domain artifact | Scaffolds into `artifacts/{slug}/...` |

---

## 3. Defining a Translatable Model

To enable translations, simply attach `Translatable` and declare your `$translatable` attributes:

```php
namespace App\Models;

use Heritage\Database\Eloquent\Concerns\Translatable;
use Heritage\Database\Eloquent\Factories\HasFactory;
use Heritage\Database\Eloquent\Model;
use Heritage\Database\Eloquent\SoftDeletes;

class Branch extends Model
{
    use Translatable, HasFactory, SoftDeletes;

    /**
     * Primary table fillable attributes.
     *
     * @var list<string>
     */
    protected $fillable = [
        'slug',
        'active',
    ];

    /**
     * Translatable attributes stored in branch_translations.
     *
     * @var list<string>
     */
    protected array $translatable = [
        'name',
        'description',
    ];

    /**
     * Optional fallback mapping when a translation is missing.
     *
     * @var array<string, string>
     */
    protected array $translatableFallback = [
        'name' => 'slug',
    ];
}
```

> [!TIP]
> **Trait Ordering Standard:**
> In translatable models, the `Translatable` trait is always placed **first** in the `use` statement (`use Translatable, HasFactory, SoftDeletes;`), highlighting the entity's multilingual nature and ensuring early trait booting priority.

---

## 4. Optional Custom Translation Model

If you ever need custom methods, additional relationships, or model events specifically on the translation table, you can create a physical class extending `Heritage\Database\Eloquent\ModelTranslation`:

```php
namespace App\Models;

use Heritage\Database\Eloquent\ModelTranslation;

class BranchTranslation extends ModelTranslation
{
    /**
     * Mass-assignable attributes.
     *
     * Note: 'branch_id', 'locale_id', and 'locale' are
     * automatically merged into $fillable by ModelTranslation.
     *
     * @var list<string>
     */
    protected $fillable = [
        'name',
        'description',
    ];
}
```

---

## 5. Unified Translatable Migrations (`Schema::createTranslatable`)

With Ugarit's unified schema engine, both the primary table and its corresponding translations table are defined in **a single migration file** using `Schema::createTranslatable`:

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

            // Marking columns as translatable routes them to branch_translations
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

---

## 6. Translation API & Queries

### Automatic Locale-Aware Access
```php
// Automatically reads the attribute based on app()->getLocale()
echo $branch->name;

// Read for a specific locale
echo $branch->translate('en')->name;
echo $branch->translateOrDefault('fr')->name;
```

### Saving & Mutating Translations
```php
// Single attribute setting
$branch->setTranslation('name', 'فرع غزة الرئيسي', 'ar');
$branch->setTranslation('name', 'Gaza Main Branch', 'en');
$branch->save();

// Fill multiple attributes for a locale
$branch->fillTranslation('ar', [
    'name' => 'فرع القدس',
    'description' => 'الفرع الإقليمي لمدينة القدس',
]);
$branch->save();
```

### Eager Loading & Query Scopes
```php
// Eager load translations for current locale (prevents N+1 queries)
$branches = Branch::withTranslation()->get();

// Eager load translations for specific locale
$branches = Branch::withTranslation('en')->get();

// Query models by translated attribute value
$results = Branch::whereTranslation('name', 'فرع غزة الرئيسي')->get();

// Query models using LIKE pattern
$results = Branch::whereTranslationLike('name', '%غزة%')->get();
```

### Inverse & Locale Relations
```php
// Access parent entity from translation
$parentBranch = $translation->branch;

// Access associated Locale entity
$locale = $translation->locale;
```

---

## 7. Unified Translatable Factories

Ugarit eliminates companion factory files. You define **a single factory** (`BranchFactory.php`) where translatable attributes are declared directly inside `definition()`:

```php
namespace App\Database\Factories;

use App\Models\Branch;
use Heritage\Database\Eloquent\Factories\Factory;

class BranchFactory extends Factory
{
    protected $model = Branch::class;

    public function definition(): array
    {
        return [
            // Translatable attributes are auto-populated across active locales (e.g. ar, en)
            'name' => $this->faker->company(),

            'slug' => $this->faker->unique()->slug(),
            'active' => true,
        ];
    }
}
```

### 1. Zero-Boilerplate Auto-Translation Seeding
When calling `Branch::factory()->create()`, the framework detects all `$translatable` attributes and automatically creates localized translation records for all configured active locales (`ar` via Arabic Faker and `en` via English Faker) with zero extra code:

```php
$branch = Branch::factory()->create();

echo $branch->translate('ar')->name; // "شركة الأمل للتجارة"
echo $branch->translate('en')->name; // "Global Logistics Corp"
```

### 2. Fluent Chaining States for Tests & Seeders
Override or customize translations on the fly using fluent chaining methods:

```php
// Define translations for all locales at once
$branch = Branch::factory()->withTranslations([
    'ar' => ['name' => 'فرع دمشق الرئيسي'],
    'en' => ['name' => 'Damascus Main Branch'],
])->create();

// Define translations for a specific locale
$branch = Branch::factory()
    ->inLocale('ar', ['name' => 'فرع القدس الشريف'])
    ->create();

// Set a single translatable attribute for a locale
$branch = Branch::factory()
    ->withTranslation('name', 'ar', 'فرع بيروت')
    ->create();
```

---

## 8. Unified Translatable Seeders

With the translatable factory engine, seeders for translatable entities are defined in **a single concise seeder file** (`BranchSeeder.php`), eliminating redundant translation seeders:

```php
namespace App\Database\Seeders;

use App\Models\Branch;
use Heritage\Database\Seeder;

class BranchSeeder extends Seeder
{
    public function run(): void
    {
        // 1. Seed primary entity with explicit multilingual translations
        Branch::factory()->withTranslations([
            'ar' => ['name' => 'المقر الرئيسي'],
            'en' => ['name' => 'Headquarters'],
        ])->create([
            'slug' => 'headquarters',
            'active' => true,
        ]);

        // 2. Generate random sample entities (automatically localized across active locales)
        Branch::factory()->count(10)->create();
    }
}
```



