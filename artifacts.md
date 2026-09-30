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
- `--minimal`: Only scaffold the core files (`art.php`, `composer.json`, `ServiceProvider`) without subdirectories.
- `--force`: Overwrite the artifact files even if the artifact already exists.

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
    ├── Features/               # Multi-step workflow orchestrators
    ├── Http/Controllers/       # Thin controllers executing Features/Capabilities
    ├── Http/Requests/          # FormRequests organized by layer (Capabilities / Features)
    ├── Models/                 # Eloquent entities & localized tables
    ├── Outcomes/               # Standardized operation outcome envelopes
    ├── Parcels/                # Strongly-typed data envelopes
    ├── Policies/               # Authorization policies
    ├── Providers/              # CatalogServiceProvider
    ├── Repositories/           # Data access boundary mapping Models to Parcels
    └── Rules/                  # Custom domain validation rules
```

---

## 3. Granular Generator Commands

All granular generator commands support the `--artifact` flag:

### Parcels
Generate strongly-typed, immutable Parcels:

```bash
php scribe make:parcel ProductData --artifact=Catalog
```

### Capabilities
Generate atomic, single-responsibility domain capability executors:

```bash
php scribe make:capability PublishProduct --artifact=Catalog
```

### Features (Workflow Orchestrators)
Generate feature workflow orchestrators that coordinate across multiple capabilities:

```bash
php scribe make:feature CheckoutWorkflow --artifact=Catalog
```

### Form Requests
Generate layer-aware, translatable FormRequests for domain artifacts:

```bash
# Generate a Capability-layer FormRequest with translatable fields
php scribe make:request StoreProductRequest --artifact=Catalog --capability --translatable --fields=name,description

# Generate a Feature-layer orchestration FormRequest
php scribe make:request CheckoutRequest --artifact=Catalog --feature

# Interactive mode (prompts for Artifact, Layer, and Translatable capabilities)
php scribe make:request StoreProductRequest
```

---

## 4. Domain Eloquent Models & Migrations

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

## 5. Automatic Discovery & Autoloading

The Ugarit Application Kernel automatically discovers all artifacts inside the `artifacts/` folder:
- **Service Providers:** Providers matching `artifacts/{artifact}/src/Providers/*ServiceProvider.php` are auto-registered.
- **PSR-4 Autoloading:** Namespaces conforming to `Ugarit\Artifacts\{Artifact}\...` are dynamically mapped to `artifacts/{artifact}/src/`.
- **Database Migrations & Seeders:** Seeders inheriting from `Heritage\Database\Seeder\ArtifactSeeder` are discovered and orchestrated via `DatabaseSeeder`.
