# Historial de cambios / Changelog

## v4.0.5 — 2026-11-16

### Correcciones / Fixes

- `dropAllTables()` borra las claves foráneas por su nombre real en el catálogo. Antes generaba `DROP CONSTRAINT "nombre_en_minúsculas"` y Firebird lo rechazaba (-607) con restricciones creadas sin comillas (guardadas en mayúsculas) o con mayúsculas y minúsculas mezcladas. / `dropAllTables()` drops foreign keys by their real catalog name. It used to generate `DROP CONSTRAINT "lowercase_name"`, which Firebird rejected (-607) for constraints created without quotes (stored uppercase) or with mixed case. [Upstream PR #12](https://github.com/bensonsbc/laravel-firebird/pull/12).
- La conexión rechaza con `InvalidArgumentException` los valores de `database`, `host`, `port`, `charset`, `role` y `dialect` que contengan `;` o caracteres de control, porque el DSN no admite escape y un `;` agregaba parámetros en silencio. / The connector rejects `database`, `host`, `port`, `charset`, `role` and `dialect` values containing `;` or control characters with an `InvalidArgumentException`: the DSN has no escaping, so a `;` silently added parameters. [Upstream PR #8](https://github.com/bensonsbc/laravel-firebird/pull/8).

### Pruebas / Validation

- Cuatro pruebas nuevas: dos del error en `commit()`, una del DSN y una de `dropAllTables()` con claves foráneas legacy. / Four new tests: two for a failing `commit()`, one for the DSN and one for `dropAllTables()` with legacy foreign keys.
- PHP 8.5.0, Laravel 13.42.0, Firebird 5.0.3: dialecto 3, 261 pruebas y 701 aserciones aprobadas; dialecto 1, 261 pruebas, 683 aserciones y 6 omitidas, sin fallas. / Dialect 3: 261 passing tests and 701 assertions; dialect 1: 261 tests, 683 assertions and 6 skipped, with no failures.
- La CI del repositorio cubre PHP 8.3 a 8.5, Firebird 3, 4 y 5, y Laravel 12 y 13 (dialecto 3). / CI covers PHP 8.3 to 8.5, Firebird 3, 4 and 5, and Laravel 12 and 13 (dialect 3).

## v4.0.4 — 2026-11-16

### Correcciones / Fixes

- Restaura autocommit antes de ejecutar los callbacks `afterCommit` y los eventos de un commit manual. Conserva los savepoints y permite que un callback abra otra transacción. / Restores autocommit before manual-commit callbacks and events, preserving savepoints and transactions started by callbacks. [Upstream PR #9](https://github.com/bensonsbc/laravel-firebird/pull/9).
- Elimina los bindings de SELECT y ORDER que `exists()` descarta en consultas sin UNION; conserva los parámetros de las unions y la consulta original. / Removes bindings for projection and ordering discarded by non-union `exists()` while retaining union parameters and the original query. [Upstream PR #10](https://github.com/bensonsbc/laravel-firebird/pull/10).
- Admite `Expression` en los valores de origen de `upsert()` sin convertir el objeto a string ni agregar bindings para SQL embebido. / Supports expressions in upsert source values without string-conversion errors or extra bindings. [Upstream PR #11](https://github.com/bensonsbc/laravel-firebird/pull/11).

### Pruebas / Validation

- Seis regresiones offline y tres pruebas de integración nuevas. / Six new offline regressions and three integration tests.
- PHP 8.5.0, Laravel 13.42.0, Firebird 5.0.3: dialecto 3, 257 pruebas y 685 aserciones aprobadas; dialecto 1, 257 pruebas, 667 aserciones y 6 omitidas, sin fallas. / Dialect 3: 257 passing tests and 685 assertions; dialect 1: 257 tests, 667 assertions and 6 skipped, with no failures.
- Firebird 3/4 y Laravel 12 no se validaron localmente para esta entrega. / Firebird 3/4 and Laravel 12 were not validated locally for this release.

### Documentación / Documentation

- Actualiza los créditos y la descripción del paquete para reflejar el mantenimiento del fork, sus correcciones y sus pruebas, conservando el reconocimiento a Benson y a los autores anteriores. / Updates credits and package metadata to describe the maintained fork, its fixes and tests, preserving attribution to Benson and earlier authors.
