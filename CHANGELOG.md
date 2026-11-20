# Historial de cambios / Changelog

## v4.0.8 — 2026-11-20

### Correcciones / Fixes

- `upsert()` con muchas filas ya no falla: el origen del `MERGE` tiene un `SELECT` por fila y Firebird limita una sentencia a 256 contextos, así que desde unas 250 filas fallaba con `-902 Too many Contexts`. Ahora las filas se parten en lotes de hasta 200 filas y 30.000 parámetros, dentro de una transacción; una llamada que cabe en una sola sentencia no cambia. / `upsert()` with many rows no longer fails: the `MERGE` source has one `SELECT` per row and Firebird limits a statement to 256 contexts, so from about 250 rows it failed with `-902 Too many Contexts`. Rows are now split into chunks of up to 200 rows and 30,000 parameters inside one transaction; a call that fits in one statement is unchanged. [Upstream PR #18](https://github.com/bensonsbc/laravel-firebird/pull/18).

### Documentación / Documentation

- Se agregan a `docs/decisiones-de-diseno.md` los riesgos y decisiones de `function_identifiers`, la detección de versión, el costo de `insert()` por lote y el límite de `upsert()`. / `docs/decisiones-de-diseno.md` now covers `function_identifiers`, version detection, the per-row `insert()` cost and the `upsert()` limit.

### Pruebas / Validation

- Una prueba nueva (`it_upserts_more_rows_than_a_single_merge_statement_can_hold`); falla sin la corrección. / One new test; it fails without the fix.
- PHP 8.5.0, Laravel 13.42.0, Firebird 5.0.3: dialecto 3, 269 pruebas y 725 aserciones aprobadas (1 omitida); dialecto 1, 269 pruebas, 708 aserciones y 6 omitidas, sin fallas. / Dialect 3: 269 tests and 725 assertions passing (1 skipped); dialect 1: 269 tests, 708 assertions and 6 skipped, with no failures.

## v4.0.7 — 2026-11-17

### Correcciones / Fixes

- En dialecto 3, `insert()`, `update()`, `where()`, `whereBetween()` y `upsert()` aceptan un `DateTimeInterface` (por ejemplo `now()`) contra una columna `DATE`; antes fallaban con -303 porque Laravel envía la fecha como texto con hora. Ahora el parámetro se declara `TIMESTAMP` y Firebird descarta la hora al guardarlo en la `DATE`. Un `where('columna_date', $carbon)` con hora compara contra la medianoche guardada, así que no coincide; para comparar por día se usa `whereDate()`. Dialecto 1 no cambia. / In dialect 3, `insert()`, `update()`, `where()`, `whereBetween()` and `upsert()` accept a `DateTimeInterface` (for example `now()`) against a `DATE` column; they used to fail with -303 because Laravel sends the date as a string with a time. The parameter is now declared as `TIMESTAMP` and Firebird drops the time when storing it in the `DATE`. A `where('date_column', $carbon)` with a time compares against the stored midnight, so it does not match; use `whereDate()` to compare by day. Dialect 1 is unchanged. [Upstream PR #17](https://github.com/bensonsbc/laravel-firebird/pull/17).

### Pruebas / Validation

- Una prueba nueva (`it_writes_and_compares_datetime_objects_against_a_date_column`); falla sin la corrección. / One new test; it fails without the fix.
- PHP 8.5.0, Laravel 13.42.0, Firebird 5.0.3: dialecto 3, 266 pruebas y 712 aserciones aprobadas (1 omitida); dialecto 1, 266 pruebas, 695 aserciones y 6 omitidas, sin fallas. / Dialect 3: 266 tests and 712 assertions passing (1 skipped); dialect 1: 266 tests, 695 assertions and 6 skipped, with no failures.

## v4.0.6 — 2026-11-16

### Correcciones / Fixes

- En dialecto 1, `avg()` con `distinct()` conserva el `distinct` en lugar de devolver el promedio simple. / In dialect 1, `avg()` with `distinct()` keeps the `distinct` instead of returning the plain average. [Upstream PR #13](https://github.com/bensonsbc/laravel-firebird/pull/13).
- En dialecto 1, `whereDate()` con un texto de fecha y hora (`2026-11-16 10:30:00`) compara sólo la fecha; antes nunca coincidía. / In dialect 1, `whereDate()` with a date-time string compares only the date; it used to never match. [Upstream PR #14](https://github.com/bensonsbc/laravel-firebird/pull/14).
- `Schema::getViews()` lee vistas con definiciones de más de 8.191 caracteres; antes fallaba con -802 por la conversión a `varchar(8191)`. / `Schema::getViews()` reads view definitions longer than 8191 characters; it used to fail with -802 because of the cast to `varchar(8191)`. [Upstream PR #15](https://github.com/bensonsbc/laravel-firebird/pull/15).
- `Schema::getColumns()` informa las columnas `COMPUTED BY` con `generation` virtual y su expresión; antes devolvía siempre `null`. / `Schema::getColumns()` reports `COMPUTED BY` columns with a virtual `generation` and their expression; it used to always return `null`. [Upstream PR #16](https://github.com/bensonsbc/laravel-firebird/pull/16).

### Pruebas / Validation

- Cuatro pruebas nuevas, una por corrección (la de `whereDate()` sólo corre en dialecto 1). / Four new tests, one per fix (the `whereDate()` one only runs in dialect 1).
- PHP 8.5.0, Laravel 13.42.0, Firebird 5.0.3: dialecto 3, 265 pruebas y 709 aserciones aprobadas (1 omitida); dialecto 1, 265 pruebas, 692 aserciones y 6 omitidas, sin fallas. / Dialect 3: 265 tests and 709 assertions passing (1 skipped); dialect 1: 265 tests, 692 assertions and 6 skipped, with no failures.

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
