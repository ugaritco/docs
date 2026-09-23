# Eloquent Model Translations

Ugarit provides first-class, database-level localization support for Eloquent models via the `HasTranslation` concern and the dedicated `ModelTranslation` entity.

---

## 1. Overview & Architecture

Instead of storing localized strings inside fragile JSON columns or separate third-party packages, Ugarit implements the dual-table pattern natively:
1. **Primary Entity Table:** Stores non-translatable attributes (IDs, foreign keys, timestamps, pricing, metrics).
2. **Translation Entity Table:** Stores locale-specific attributes (`locale`, `title`, `description`, `slug`) indexed with unique composite constraints.

---

## 2. Using `HasTranslation`

To make an Eloquent model translatable, import and attach the `HasTranslation` trait:

```php
namespace App\Models;

use Heritage\Database\Eloquent\Concerns\HasTranslation;
use Heritage\Database\Eloquent\Model;

class Article extends Model
{
    use HasTranslation;

    /**
     * The attributes that are translatable.
     *
     * @var list<string>
     */
    protected $translatable = [
        'title',
        'content',
        'slug',
    ];
}
```

---

## 3. Creating the Translation Model

The translation model extends `Heritage\Database\Eloquent\ModelTranslation`:

```php
namespace App\Models;

use Heritage\Database\Eloquent\ModelTranslation;

class ArticleTranslation extends ModelTranslation
{
    /**
     * The attributes that are mass assignable.
     *
     * @var list<string>
     */
    protected $fillable = [
        'title',
        'content',
        'slug',
    ];
}
```

---

## 4. Dual-Table Migrations

In your database migration, define the primary table and its corresponding translations table:

```php
use Heritage\Database\Migrations\Migration;
use Heritage\Database\Schema\Blueprint;
use Heritage\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('articles', function (Blueprint $table) {
            $table->id();
            $table->decimal('price', 10, 2)->default(0);
            $table->boolean('is_published')->default(false);
            $table->timestamps();
        });

        Schema::create('article_translations', function (Blueprint $table) {
            $table->id();
            $table->foreignId('article_id')->constrained()->cascadeOnDelete();
            $table->string('locale', 10)->index();
            $table->string('title');
            $table->text('content')->nullable();
            $table->string('slug')->nullable();
            $table->timestamps();

            $table->unique(['article_id', 'locale']);
        });
    }
};
```

---

## 5. Reading and Writing Localized Attributes

### Reading Attributes
When accessing a translatable attribute, Ugarit resolves the value matching the current application locale (`app()->getLocale()`), automatically falling back to the configured fallback locale (`config('app.fallback_locale')`):

```php
// If locale is 'ar'
echo $article->title; // Returns Arabic title

// Access explicit locale
echo $article->translate('en')->title; // Returns English title
```

### Writing Attributes
Setting translatable attributes modifies or creates translation records dynamically:

```php
$article = Article::create([
    'price' => 49.99,
    'en' => [
        'title' => 'Getting Started with Ugarit',
        'content' => 'Comprehensive guide...',
    ],
    'ar' => [
        'title' => 'البدء مع أوغاريت',
        'content' => 'دليل شامل...',
    ],
]);
```
