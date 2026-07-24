# Repository Guidelines

## Project Structure & Module Organization

This is a ThinkPHP 5.1 payment-notification application. Code lives in `application/`: `index/controller` exposes payment and monitor endpoints, `admin/controller` serves administration APIs, and `service` contains shared services. Define routes in `route/route.php` and settings in `config/`. Static pages and assets are under `public/`; use it as the web document root. `vmq.sql` is the MySQL bootstrap schema. Treat `thinkphp/` and `vendor/` as dependency code; avoid editing them for application features.

## Build, Test, and Development Commands

- `composer install` installs the PHP dependencies from `composer.lock` (PHP 5.6 or newer is required).
- `mysql -u root -p vmq < vmq.sql` initializes a local database; adjust credentials as needed.
- `php -S 127.0.0.1:8000 -t public public/router.php` runs the application locally with `public/` as the document root.
- `php -l application/index/controller/Index.php` syntax-checks an edited PHP file. Run it for every changed PHP file.

There is no asset compilation step; frontend assets are committed directly under `public/`.

## Coding Style & Naming Conventions

Follow the PSR-4 namespace mapping (`app\` to `application/`) and use four-space indentation. Name classes in PascalCase, methods and variables in camelCase, and configuration keys in snake_case. Place reusable QR-code or domain logic in `application/service`. Preserve the encoding of legacy Chinese text. No formatter is configured, so match surrounding brace and array style.

## Testing Guidelines

The repository currently has no application-level automated test suite. `thinkphp/phpunit.xml.dist` covers upstream framework tests and is not evidence of application coverage. For each change, syntax-check modified PHP files and manually exercise affected routes against a disposable MySQL database. Verify both success and failure responses, authentication checks, order state transitions, and callback signatures. Document exact commands and endpoints tested in the pull request.

## Commit & Pull Request Guidelines

History favors short Chinese summaries and release labels such as `V1.13`. Keep commits concise, imperative, and limited to one concern. Pull requests should explain behavior changes, list affected routes or configuration, note any `vmq.sql` migration steps, and include screenshots for changes under `public/`. Link issues and call out deployment impacts.

## Security & Configuration

Never commit production database credentials, signing keys, callback URLs, payment data, or `.env` files. Replace the development defaults in `config/database.php` during deployment, keep `public/` as the only exposed document root, and disable debug output in production.
