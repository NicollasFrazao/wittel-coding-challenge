# Wittel Challenge - Customer management with Filament

A Laravel 10 application that manages customers (name, CPF, birth date, income) through a Filament admin panel and a token-authenticated REST API.

## Context

Solution to the hiring challenge of Wittel (2025). The challenge statement is not included in this repository.

## What it does

**Admin panel (Filament, `/admin`)**

- `/` redirects to `/admin`, which uses Filament's own login page.
- Customer resource with list, create, view and edit pages.
- List: search and sort on id, name, CPF, birth date and income; `created_at`, `updated_at` and `deleted_at` columns can be toggled on.
- Soft delete: a "trashed" filter plus bulk delete, force delete and restore.
- Form rules: name required (max 150), CPF required (max 11 characters, unique at the database level), birth date required, income optional and numeric.

**REST API**

Authentication uses Laravel Sanctum personal access tokens (`app/Http/Controllers/AuthController.php`):

| Method | Path | Description |
| --- | --- | --- |
| POST | `/api/auth/login` | `email`, `password`, `device_name`; returns a token and the user |
| GET | `/api/auth`, `/api/auth/me` | Current user (Bearer token) |
| DELETE | `/api/auth/logout` | Revokes the current token |

The customer endpoints come from [`rupadana/filament-api-service`](https://github.com/rupadana/filament-api-service). Its handlers live in `app/Filament/Resources/CustomerResource/Api`. With `route.panel_prefix = false` in `config/api-service.php`, they are mounted under `/api/customers`:

| Method | Path | Handler |
| --- | --- | --- |
| GET | `/api/customers` | Paginated list (`per_page`), via `spatie/laravel-query-builder` |
| GET | `/api/customers/{id}` | Detail |
| POST | `/api/customers` | Create |
| PUT | `/api/customers/{id}` | Update |
| DELETE | `/api/customers/{id}` | Soft delete |

These routes need `auth:sanctum` and the package's `ability` middleware. Tokens issued by `/api/auth/login` carry no ability list, so Sanctum gives them `*`.

**Data**

- Migration `create_customers_table`: `name`, `cpf` (unique), `birth_date`, `income` (nullable decimal), soft deletes, timestamps.
- `users` also has soft deletes. `spatie/laravel-permission` tables are migrated (a dependency of the API package), and `User` uses `HasRoles`.
- `DatabaseSeeder` creates one admin user and 100 fake customers (`CustomerFactory`).

## Tech stack

Versions are the ones locked in `composer.lock` / `package.json`.

- PHP `^8.1` (the Docker image installs PHP 8.4)
- Laravel 10.48, Sanctum 3.3
- Filament 3.3, Livewire 3.6
- rupadana/filament-api-service 3.4.9, spatie/laravel-permission 6.20
- MySQL 8.0 and Redis (Laravel Sail, `docker-compose.yml`)
- PHPUnit 10.5
- Vite 5, Bootstrap 5, Sass (`package.json`; no Blade views use them)

## Running locally

The project runs on Laravel Sail with a custom image in `docker/`. The `Makefile` wraps the Sail commands and reads `.env`, so create that file first.

```bash
cp .env.example .env
make install                       # composer install (needs PHP >= 8.1 on the host)
make build                         # sail up --build: app on http://localhost, MySQL on port 33066
./vendor/bin/sail artisan key:generate
make seed                          # migrate --seed
```

When the container starts, `docker/setup-container` runs `optimize:clear`, `migrate`, `npm install` and `npm run dev`.

Log in at `http://localhost/admin` with the user created in `database/seeders/DatabaseSeeder.php`.

Other targets: `make up`, `make start`, `make stop`, `make down`, `make sh`, `make sh:mysql`, `make migrate`, `make clear`.

## Tests

```bash
make test        # sail test
```

Only the default Laravel example tests exist (`tests/Unit/ExampleTest.php`, `tests/Feature/ExampleTest.php`).

## Project structure

```
app/Filament/Resources/CustomerResource.php      Filament resource (form, table, pages)
app/Filament/Resources/CustomerResource/Api/     API service, handlers, requests, transformer
app/Http/Controllers/AuthController.php          Sanctum login / me / logout
app/Models/Customer.php                          Eloquent model (soft deletes)
app/Providers/Filament/AdminPanelProvider.php    Panel config (/admin, SPA mode, API plugin)
database/migrations/                             customers + permission tables
docker/, docker-compose.yml, Makefile            Sail environment
```

## Notes

- `vendor/` is not committed. About 2,100 Filament translation files are committed under `lang/vendor/`, along with Filament's published assets under `public/css/filament` and `public/js/filament`.
- The API create and update requests (`CreateCustomerRequest`, `UpdateCustomerRequest`) are close to the package's generated code. Both require a `deleted_at` field, and both make `income` required even though it is optional in the panel. Neither these requests nor the panel form check that the CPF is unique, so a duplicate CPF fails at the database.
- The app (`routes/api.php`) and the API package both register `POST /api/auth/login`; only one of them answers, depending on the order in which the routes are registered.
- `tests/Feature/ExampleTest.php` expects HTTP 200 from `/`, but `/` now redirects to `/admin`.
- `AuthController` imports `App\Models\CompanyScheduling` and `App\Services\GoogleService`, which do not exist in this project. They are never used.
- `laravel/ui` and `laracasts/flash` remain in `composer.json` from an AdminLTE setup that was later removed (commit `a2532be`).
- The seeder creates the admin user with a trivial password, which is only meant for local use.
