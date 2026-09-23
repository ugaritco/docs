# Artifact-Driven Architecture (ADA) & Modular Scaffolding

Artifacts in Ugarit represent self-contained, domain-isolated modules that encapsulate business logic, models, controllers, services, use cases, migrations, and seeders into cohesive packages within the `artifacts/` root directory.

---

## 1. Modular Generation CLI Commands

Ugarit provides a dedicated suite of modular code scaffolding commands:

### Scaffolding an Entire Artifact
To generate a brand new artifact package complete with service providers, manifests, configuration, and directory structure:

```bash
php scribe make:artifact Catalog
```

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
    ├── DTOs/                   # Strongly-typed Data Transfer Objects
    ├── Http/Controllers/       # Modular controllers
    ├── Models/                 # Eloquent entities & localized tables
    ├── Providers/              # CatalogServiceProvider
    ├── Services/               # Domain service orchestrators
    └── UseCases/               # Single-action application interactors
```

---

## 2. Granular Generator Commands

### Data Transfer Objects (DTOs)
Generate strongly-typed, immutable PHP 8.4 DTOs:

```bash
php scribe make:dto ProductData --artifact=Catalog
```

### Domain Services
Generate multi-action business service orchestrators:

```bash
php scribe make:service OrderProcessor --artifact=Catalog
```

### Use Cases
Generate single-action invokable application workflows:

```bash
php scribe make:usecase PublishProduct --artifact=Catalog
```

### Features (Vertical Slices)
Generate feature slice orchestrators:

```bash
php scribe make:feature CheckoutWorkflow --artifact=Catalog
```

---

## 3. Modular Eloquent Models & Migrations

Generating models directly within an artifact:

```bash
php scribe make:model Product --artifact=Catalog --migration --factory --seed
```

### Localized Models (`--translation`)
To automatically scaffold dual database tables for localized models:

```bash
php scribe make:model Category --artifact=Catalog --translation
```

This generates:
- `Category.php` using the `Heritage\Database\Eloquent\Concerns\HasTranslation` trait.
- `CategoryTranslation.php` extending `Heritage\Database\Eloquent\ModelTranslation`.
- Dual migration creating `categories` and `category_translations` tables with cascade foreign key relations.

---

## 4. Automatic Discovery & Autoloading

The Ugarit Application Kernel automatically discovers all artifacts inside the `artifacts/` folder:
- **Service Providers:** Providers matching `artifacts/{artifact}/src/Providers/*ServiceProvider.php` are auto-registered.
- **PSR-4 Autoloading:** Namespaces conforming to `Ugarit\Artifacts\{Artifact}\...` are dynamically mapped to `artifacts/{artifact}/src/`.
- **Database Migrations & Seeders:** Seeders inheriting from `Heritage\Database\Seeder\ArtifactSeeder` are discovered and orchestrated via `ModuleSeeder`.
