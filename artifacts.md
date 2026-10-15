# Artifact-Driven Architecture (ADA) & Artifact Scaffolding

Artifacts in Ugarit represent self-contained, domain-isolated units that encapsulate business logic, models, controllers, capabilities, features, parcels, outcomes, responders, migrations, and seeders into cohesive packages within the `artifacts/` root directory.

---

## 1. Core Architectural Concepts

The Ugarit execution architecture is built upon a streamlined, decoupled pipeline:

- **Capability:** An atomic unit of execution responsible for performing a single, clearly defined operation (Single Responsibility Principle). It interacts directly with repositories and domain models, and returns an `Outcome`.
- **Feature:** A workflow coordinator positioned at the top of the domain workflow hierarchy. It arranges execution flow, coordinates across multiple Capabilities, and manages cross-domain transactions.
- **Parcel:** A strongly typed, validated data envelope that moves across boundaries (from HTTP Requests to Capabilities and Repositories). Replaces legacy DTO concepts.
- **Outcome:** The pure domain execution result, completely agnostic to presentation or transport protocols (HTTP, CLI, WebSockets). It encapsulates result status, payloads, errors, and UI flash metadata.
- **Responder:** Boundary adapter responsible for receiving an `Outcome` and converting it into the final presentation `Response` (Inertia view, JSON, HTTP Redirect) tailored to the client consumer.

---

## 2. Artifact Generation CLI Commands

Ugarit provides a dedicated suite of artifact code scaffolding commands:

### Scaffolding an Entire Artifact
To generate a brand new artifact package complete with service providers, manifests, configuration, and directory structure, use `make:artifact`:

```bash
# Scaffolding a domain artifact
php scribe make:artifact Catalog
```

### Inspecting Installed Artifacts & Models
To inspect all domain artifacts or models across your project:

```bash
# List all registered domain artifacts with their stats & capabilities
php scribe artifact:list

# List all Eloquent models across the application and domain artifacts
php scribe model:list

# Filter models by specific artifact
php scribe model:list --artifact=catalog

# Filter only translatable multilingual models
php scribe model:list --translatable
```

Options available for artifact scaffolding:
- `--description="Description"`: Provide a human-readable description for the artifact manifest.
- `--minimal`: Only scaffold the core files (`art.php`, `composer.json`, `ServiceProvider`) without subdirectories nor summary.
- `--panel`: Also create its panel module on the frontend (`make:panel`): layout, navigation, sidebar item and summary page. When it is not given, the interactive prompt asks.
- `--force`: Overwrite the artifact files even if the artifact already exists.

Unless `--minimal`, the artifact gets its [module summary](/docs/{{version}}/capabilities#the-module-summary) by `make:summary`: `ShowModuleCapability`, its result, `ModuleController` and the `{artifact}.module.show` route in `routes/web.php`.

This creates the following structure in `artifacts/catalog/`:
```text
artifacts/catalog/
├── art.php                     # Artifact manifest metadata
├── composer.json               # Isolated package dependencies & PSR-4 autoloading
├── config/                     # Artifact-specific configuration
├── database/
│   ├── factories/              # Eloquent model factories
│   ├── migrations/             # Domain database migrations
│   └── seeders/                # Domain seeders implementing ArtifactSeeder
├── routes/                     # Domain web and API routes
└── src/
    ├── Capabilities/           # Atomic domain operation executors
    ├── Contracts/Repositories/ # Repository contracts, bound to Repositories/ by convention
    ├── Enums/                  # Results of the capabilities and features
    ├── Features/               # Multi-step workflow orchestrators
    ├── Http/Controllers/       # Thin controllers executing Features/Capabilities
    ├── Http/Requests/          # FormRequests organized by layer (Capabilities / Features)
    ├── Http/Resources/         # JSON resources of the models
    ├── Models/                 # Eloquent entities & localized tables
    ├── Outcomes/               # Standardized operation outcome envelopes
    ├── Parcels/                # Strongly-typed data envelopes
    ├── Policies/               # Authorization policies
    ├── Providers/              # CatalogServiceProvider
    ├── Repositories/           # Data access: take Parcels, return Models
    └── Rules/                  # Custom domain validation rules
```

The service provider loads the artifact's migrations, its translations under the `catalog::` namespace from `lang/`, and `routes/web.php` when it exists.

---

## 3. Granular Generator Commands

All granular generator commands support the `--artifact` flag.

### A Model's Capabilities Slice
Generate a model's repository, policy, results, parcels, outcome, capabilities, form requests, resource, controller and lang file in one command; the fields and rules are read from the model's `$fillable`, `$casts` and translatable attributes:

```bash
php scribe make:capabilities Product --artifact=Catalog

# Or everything at once: the model from its fields, its migration, factory, seeder, slice and routes
php scribe make:model Product --artifact=Catalog --all --fields="category_id:foreignId,sku:string:unique,name:string:translatable,price:decimal,active:boolean"
```

See [Capabilities: Generating a Slice](/docs/{{version}}/capabilities#generating-a-slice) for the generated classes and the generator of each part.

### Parcels
Generate strongly-typed, immutable Parcels:

```bash
php scribe make:parcel ProductData --artifact=Catalog
```

### Capabilities
Generate atomic, single-responsibility domain capability executors, optionally typed to their Parcel, with an injected repository and their Result/Outcome:

```bash
php scribe make:capability PublishProduct --artifact=Catalog
php scribe make:capability PublishProduct --artifact=Catalog --parcel=PublishProductParcel --repository=ProductRepositoryInterface --result=PublishProductResult
```

### Features (Workflow Orchestrators)
Generate feature workflow orchestrators that coordinate across multiple capabilities:

```bash
php scribe make:feature CheckoutWorkflow --artifact=Catalog --parcel=CheckoutParcel
```

### Result Enums
Generate a Result enum whose feedback is resolved by convention from its `FEEDBACK` key:

```bash
php scribe make:result PublishProduct --artifact=Catalog --feedback=published
```

See [Capabilities](/docs/{{version}}/capabilities) for the shape of these classes, failure handling and the feedback convention, and [Features](/docs/{{version}}/features) for flows and transactions.

### Form Requests
Generate layer-aware, translatable FormRequests for domain artifacts:

```bash
# Generate a Capability-layer FormRequest with translatable fields
php scribe make:request StoreProductRequest --artifact=Catalog --capability --translatable --fields=name,description

# Generate a Feature-layer orchestration FormRequest
php scribe make:request CheckoutRequest --artifact=Catalog --feature

# Rules from the model's columns for an operation (store, update, destroy), authorizing its action
php scribe make:request StoreProductRequest --artifact=Catalog --capability --model=Product --operation=store --action=Capabilities/Product/SaveProductCapability

# Interactive mode (prompts for Artifact, Layer, and Translatable capabilities)
php scribe make:request StoreProductRequest
```

---

## 4. Domain Eloquent Models & Migrations

Generating models directly within an artifact:

```bash
php scribe make:model Product --artifact=Catalog --migration --factory --seed
```

### Translatable Models (`--translatable`)
To scaffold a translatable model and its dual database tables:

```bash
php scribe make:model Category --artifact=Catalog --translatable --migration

# Short flag
php scribe make:model Category --artifact=Catalog -t -m
```

This generates:
- `Category.php` declaring `$translatable` and `$translatableFallback` (translation support is built into the base `Model`; no trait is needed).
- A migration using `Schema::createTranslatable`, creating `categories` and `category_translations` with a cascading foreign key, a locale column chosen by the locale resolver, and the translation table's own timestamps.

The translation model is generated at runtime. Add `--with-translation-file` to also write a physical `CategoryTranslation.php` extending `Heritage\Database\Eloquent\ModelTranslation`. See [Eloquent Model Translations](/docs/{{version}}/eloquent-translations).

---

## 5. Automatic Discovery & Autoloading

The Ugarit Application Kernel automatically discovers all artifacts inside the `artifacts/` folder:
- **Service Providers:** The providers listed by the artifact's `art.php` (`$providers`) or its `composer.json` (`extra.ugarit.providers`) are registered.
- **PSR-4 Autoloading:** Namespaces conforming to `Ugarit\Artifacts\{Artifact}\...` are dynamically mapped to `artifacts/{artifact}/src/`.
- **Resources:** The kernel loads every artifact's resources by convention, so its service provider only binds its services (an artifact disabled in `config/artifacts.php` is skipped):

| Path | Loaded as |
| --- | --- |
| `config/*.php` | Merged under the file's name (`config/geography.php` → `config('geography')`), before the artifact's providers are registered; published with `vendor:publish --tag={artifact}-config` |
| `database/migrations` | A migration path |
| `lang` | Translations under the artifact id (`__('library::genre.attributes.name.label')`) |
| `resources/views` | Views under the artifact id (`view('library::index')`) |
| `routes/web.php`, `routes/api.php` | Route files, unless the routes are cached |

- **Database Seeders:** The artifact's `{Artifact}DatabaseSeeder` (or `DatabaseSeeder`) is discovered and orchestrated via `DatabaseSeeder`; it need not be registered by its provider.
