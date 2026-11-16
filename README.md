# DASU Laravel Firebird

Driver de Firebird para Laravel · Firebird database driver for Laravel.

**[🇪🇸 Español](#-español)** · **[🇬🇧 English](#-english)**

> ## ⭐ Créditos / Credits
>
> Este trabajo se apoya enteramente en el **fork de [Alexandre Benson Smith](https://github.com/bensonsbc) — [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird)** (paquete `benson/laravel-firebird`, [Thor Software](https://thorsoftware.com.br)), que llevó el driver de ~51 % a ~96 % de paridad funcional con los drivers oficiales. **Todo el mérito del driver es de Benson**; este repositorio solo mantiene la documentación en español e inglés y reúne propuestas que se envían a su repositorio como *pull requests*.
>
> This work stands entirely on the **fork by [Alexandre Benson Smith](https://github.com/bensonsbc) — [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird)** (package `benson/laravel-firebird`, [Thor Software](https://thorsoftware.com.br)), which took the driver from ~51% to ~96% functional parity with the first-party drivers. **All credit for the driver goes to Benson**; this repository only keeps the Spanish and English documentation and gathers proposals that are sent to his repository as pull requests.
>
> 💜 Apoyá el proyecto original / support the original project: **[github.com/sponsors/bensonsbc](https://github.com/sponsors/bensonsbc)**
>
> Cadena de origen / lineage: [`jacquestvanzuydam/laravel-firebird`](https://github.com/jacquestvanzuydam/laravel-firebird) → [`harrygulliford/laravel-firebird`](https://github.com/harrygulliford/laravel-firebird) → [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) → este repositorio / this repository.

---

## 🇪🇸 Español

Integración previsible de Firebird con Laravel: Query Builder, Eloquent, paginación, `insertGetId()` con `INSERT ... RETURNING`, `insertOrIgnore`, `upsert` con `MERGE`, `update`/`delete` con joins, `truncate`, columnas identity, columnas calculadas, tablas temporales e introspección de esquema, con soporte de Firebird 2.5 → 5.0 (dialectos 1 y 3).

### Instalación

El paquete **no está publicado en Packagist**: hay que declarar el repositorio de GitHub en el `composer.json` de la aplicación y fijar la versión:

```json
{
    "require": { "benson/laravel-firebird": "4.0.2" },
    "repositories": [
        { "type": "vcs", "url": "https://github.com/bensonsbc/laravel-firebird" }
    ]
}
```

Configurá la conexión en `config/database.php`:

```php
'firebird' => [
    'driver' => 'firebird',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3050'),
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME', 'sysdba'),
    'password' => env('DB_PASSWORD', 'masterkey'),
    'charset' => env('DB_CHARSET', 'UTF8'),

    'quote_identifiers' => true,
    'uppercase_identifiers' => false,
    'pagination_mode' => 'first_skip',
],
```

### Opciones específicas del driver

| Opción | Valor por defecto | Descripción |
| --- | --- | --- |
| `quote_identifiers` | `true` | Envuelve los identificadores entre comillas dobles. Desactivar para bases legadas sin comillas. |
| `uppercase_identifiers` | `false` | Convierte a mayúsculas los identificadores sin comillas (estilo legado). |
| `pagination_mode` | `first_skip` | `first_skip` = `SELECT FIRST n SKIP m`; `offset_fetch` = `OFFSET ... FETCH`. |
| `dialect` | — | Poner `1` para bases en dialecto 1 (ajusta tipos y fechas, y desactiva identity). |
| `role` | — | Rol SQL que se envía al conectar. |
| `column_case` | `natural` | Caja de los nombres de columna en los resultados: `natural` (como los devuelve Firebird: mayúsculas para identificadores sin comillas), `lower` o `upper`. |
| `function_identifiers` | `true` | Los identificadores con forma de función (`upper(nombre)`) pasan sin comillas. Poner `false` para entrecomillar siempre (recomendado si los nombres de columna vienen de datos del usuario) y usar expresiones *raw* para las funciones. *(Propuesta pendiente de aceptación en el repositorio de origen.)* |
| `server_version` | autodetectada | Fija la versión del servidor (p. ej. `'3.0.10'`) para evitar detectarla por la conexión. |
| `index_names` | — | Plantillas de nombres para PK/FK/unique/index. |

### Destacados

- **Funciones según la versión**: columnas identity, `BOOLEAN` nativo y `ALTER COLUMN` de nulidad en Firebird 3+; tipos `WITH TIME ZONE` e identificadores de 63 caracteres en Firebird 4+.
- **Nombres configurables** de constraints e índices (`index_names`), incluida la PK en línea de las columnas identity.
- **Columnas calculadas** (`virtualAs`/`storedAs` → `COMPUTED BY`) y **tablas temporales** (`temporary()` → GTT).
- **`uniqueIndex()`** (índice único sin constraint), **`COMMENT ON`**, **`insertOrIgnore`** por constraint real, reconexión automática e introspección con `auto_increment`.
- **Enteros en columnas `DECIMAL`**: se enlazan como texto, lo que evita un error silencioso de escala (con `PDO::PARAM_INT`, `40` quedaba grabado como `0.40` en un `DECIMAL(5,2)`).
- **Fechas**: el trait `SerializesFirebirdDates` (y el modelo base) graba los casts `date` sin la parte horaria, evitando el error de conversión de Firebird.

> 📖 **Documentación completa de las funcionalidades: [docs/funcionalidades.md](docs/funcionalidades.md)** — identificadores/caja/comillas, paginación, fechas, nombres, columnas calculadas, GTT, limitaciones y más.

### Qué cambia respecto del repositorio de Benson

Esta copia no modifica el código del driver por su cuenta: lo que se corrige o se agrega se propone primero a [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) como *pull request*. Mientras se revisan, las ramas viven acá:

| Rama / PR | Qué hace |
| --- | --- |
| [#3](https://github.com/bensonsbc/laravel-firebird/pull/3) `fix/tests-column-case-natural` | Actualiza tres pruebas que esperaban nombres de columna en minúsculas y documenta `column_case`. |
| [#4](https://github.com/bensonsbc/laravel-firebird/pull/4) `fix/dialect1-metadata-keys` | La introspección de esquema fallaba en dialecto 1 (los alias de metadatos vuelven en mayúsculas). |
| [#5](https://github.com/bensonsbc/laravel-firebird/pull/5) `fix/truncate-shared-generators` | `truncate()` ya no pone en cero un generador compartido con otras tablas, ni `dropAllTables()` intenta borrarlo. |
| [#6](https://github.com/bensonsbc/laravel-firebird/pull/6) `feat/function-identifiers-option` | Opción `function_identifiers` para entrecomillar siempre los identificadores. |

La rama `main` de este repositorio (la única que se usa como punto de partida) reúne esas cuatro ramas sobre la `4.x` de Benson, más PHP 8.5 en la matriz de CI ([#7](https://github.com/bensonsbc/laravel-firebird/pull/7), también propuesto a Benson, rama `ci/php-8.5`). Las ramas de los PR se conservan solo mientras el PR siga abierto.

---

## 🇬🇧 English

A predictable Firebird integration for Laravel: Query Builder, Eloquent, pagination, `insertGetId()` via `INSERT ... RETURNING`, `insertOrIgnore`, `upsert` via `MERGE`, joined update/delete, truncate, identity columns, computed columns, temporary tables and schema introspection — supporting Firebird 2.5 → 5.0 (dialects 1 and 3).

### Installation

The package is **not published on Packagist**: declare the GitHub repository in the application's `composer.json` and pin the version:

```json
{
    "require": { "benson/laravel-firebird": "4.0.2" },
    "repositories": [
        { "type": "vcs", "url": "https://github.com/bensonsbc/laravel-firebird" }
    ]
}
```

Configure the connection in `config/database.php`:

```php
'firebird' => [
    'driver' => 'firebird',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3050'),
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME', 'sysdba'),
    'password' => env('DB_PASSWORD', 'masterkey'),
    'charset' => env('DB_CHARSET', 'UTF8'),

    'quote_identifiers' => true,
    'uppercase_identifiers' => false,
    'pagination_mode' => 'first_skip',
],
```

### Driver-specific options

| Option | Default | Description |
| --- | --- | --- |
| `quote_identifiers` | `true` | Wraps identifiers in double quotes. Disable for legacy unquoted schemas. |
| `uppercase_identifiers` | `false` | Uppercases unquoted identifiers (legacy style). |
| `pagination_mode` | `first_skip` | `first_skip` = `SELECT FIRST n SKIP m`; `offset_fetch` = `OFFSET ... FETCH`. |
| `dialect` | — | Set `1` for dialect-1 databases (adjusts types, dates, disables identity). |
| `role` | — | SQL role sent on connect. |
| `column_case` | `natural` | Case of result column names: `natural` (as Firebird returns them: uppercase for unquoted identifiers), `lower` or `upper`. |
| `function_identifiers` | `true` | Identifiers shaped like function calls (`upper(name)`) are passed through unquoted. Set `false` to always quote (recommended when column names come from user input) and use raw expressions for functions. *(Proposal pending acceptance upstream.)* |
| `server_version` | auto-detected | Pin the server version (e.g. `'3.0.10'`) to skip detection over the connection. |
| `index_names` | — | Naming templates for PK/FK/unique/index. |

### Highlights

- **Version-aware features**: identity columns, native `BOOLEAN` and nullability `ALTER COLUMN` on Firebird 3+; `WITH TIME ZONE` types and 63-char identifiers on Firebird 4+.
- **Configurable naming** of constraints/indexes (`index_names`), including the inline PK of identity columns.
- **Computed columns** (`virtualAs`/`storedAs` → `COMPUTED BY`) and **temporary tables** (`temporary()` → GTT).
- **`uniqueIndex()`** (unique index without a constraint), **`COMMENT ON`**, real-constraint **`insertOrIgnore`**, automatic reconnect and introspection with `auto_increment`.
- **Integers into `DECIMAL` columns**: bound as text, which avoids a silent scale error (with `PDO::PARAM_INT`, `40` was stored as `0.40` in a `DECIMAL(5,2)`).
- **Dates**: the `SerializesFirebirdDates` trait (and base model) stores `date` casts without the time component, avoiding Firebird's conversion error.

> 📖 **Full feature documentation: [docs/funcionalidades.md](docs/funcionalidades.md)** (in Spanish) — identifiers/case/quoting, pagination, dates, naming, computed columns, GTT, limitations and more.

### How this differs from Benson's repository

This copy does not change the driver code on its own: whatever is fixed or added is first proposed to [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) as a pull request. While they are under review, the branches live here:

| Branch / PR | What it does |
| --- | --- |
| [#3](https://github.com/bensonsbc/laravel-firebird/pull/3) `fix/tests-column-case-natural` | Updates three tests that expected lowercase column names and documents `column_case`. |
| [#4](https://github.com/bensonsbc/laravel-firebird/pull/4) `fix/dialect1-metadata-keys` | Schema introspection failed in dialect 1 (metadata aliases come back in uppercase). |
| [#5](https://github.com/bensonsbc/laravel-firebird/pull/5) `fix/truncate-shared-generators` | `truncate()` no longer zeroes a generator shared with other tables, and `dropAllTables()` no longer tries to drop it. |
| [#6](https://github.com/bensonsbc/laravel-firebird/pull/6) `feat/function-identifiers-option` | `function_identifiers` option to always quote identifiers. |

This repository's `main` branch (the only one used as a starting point) gathers those four branches on top of Benson's `4.x`, plus PHP 8.5 in the CI matrix ([#7](https://github.com/bensonsbc/laravel-firebird/pull/7), also proposed to Benson, branch `ci/php-8.5`). The PR branches are kept only while their PR is open.

---

## Antes × Después / Before × After

Paridad funcional estimada con los drivers oficiales (MySQL/PostgreSQL/SQL Server), según el fork de Benson / estimated functional parity with the first-party drivers, per Benson's fork:

| | |
| --- | --- |
| **Antes / Before** | **~51 %** |
| **Después / After** | **~96 %** |

Leyenda / Legend: ✅ completo/full · ⚠️ parcial/partial · ❌ ausente/missing · 🚫 limitación de Firebird / Firebird limitation

| Funcionalidad / Feature | Antes / Before | Después / After |
|---|:---:|:---:|
| Query builder (select/where/join/group/order/union) | ✅ | ✅ |
| Paginación / Pagination (limit/offset, paginate, cursor) | ✅ | ✅ |
| `whereDate/Time/...` | ✅ | ✅ |
| Locks `forUpdate` | ✅ | ✅ |
| Stored procedures | ✅ | ✅ |
| `insertGetId` / `insertUsing` | ✅ | ✅ |
| `upsert` (MERGE) | ✅ | ✅ |
| Update/Delete + join | ✅ | ✅ |
| `truncate` | ✅ | ✅ |
| Schema CRUD (columnas/índices/FK · columns/indexes/FK) | ✅ | ✅ |
| Introspección / Introspection (tables/columns/indexes/FKs/views) | ✅ | ✅ |
| Transacciones / Transactions, savepoints | ✅ | ✅ |
| `exists` + union | ⚠️ | ✅ |
| `insertOrIgnore` (cualquier unique / any unique) | ⚠️ | ✅ |
| Identity columns (FB3+) | ❌ | ✅ |
| Identificadores de 63 caracteres / 63-char identifiers | ⚠️ | ✅ |
| `change()` nulidad según versión / version-aware nullability | ⚠️ | ✅ |
| Tipos con zona horaria / time-zone types (FB4+) | ❌ | ✅ |
| Comentarios / Comments (tabla/columna · table/column) | ❌ | ✅ |
| `whereLike(caseSensitive)` | ❌ | ⚠️ |
| `whereIn` > 1499 | ❌ | ✅ |
| `groupLimit` (eager load) | ❌ | ✅ |
| `json()` (almacenamiento / storage) | ⚠️ | ⚠️ |
| Reconexión / Lost connection | ❌ | ✅ |
| `UniqueConstraintViolationException` | ❌ | ✅ |
| Cast `date` sin hora / without time | ❌ | ✅ |
| Enteros en `DECIMAL` / Integer bind into `DECIMAL` | ❌ | ✅ |
| `dropAllTables` (caja mixta / mixed case) | ⚠️ | ✅ |
| `auto_increment` en la introspección / in introspection | ❌ | ✅ |
| `server_version` / feature detection | ❌ | ✅ |
| Nombres personalizados / custom naming (`index_names`) | ❌ | ✅ |
| `uniqueIndex` sin constraint / without constraint | ❌ | ✅ |
| Columnas calculadas / computed columns (`COMPUTED BY`) | ❌ | ✅ |
| Tablas temporales / temporary tables (GTT) | ❌ | ✅ |
| JSON ops / full-text / spatial / rename table | 🚫 | 🚫 |

## Pruebas / Tests

```bash
docker compose up -d        # Firebird 5 (puerto/port 3055) y/and Firebird 4 (3054)
DB_PORT=3055 vendor/bin/phpunit
```

Para dialecto 1, exportá `DB_DIALECT=1` apuntando a una base en dialecto 1. / For dialect 1, export `DB_DIALECT=1` pointing at a dialect-1 database.

La integración continua corre la suite contra Firebird 3, 4 y 5 en PHP 8.3, 8.4 y 8.5. / CI runs the suite against Firebird 3, 4 and 5 on PHP 8.3, 8.4 and 8.5.

## Créditos completos / Full credits

- **[Alexandre Benson Smith](https://github.com/bensonsbc)** ([Thor Software](https://thorsoftware.com.br)) — autor del fork [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird), del que sale todo el driver actual / author of the fork the current driver comes from.
- **[Harry Gulliford](https://github.com/harrygulliford)** — mantenedor del fork base / maintainer of the base fork.
- **[Jacques van Zuydam](https://github.com/jacquestvanzuydam)** — autor original / original author.
- **Contribuidores / contributors**: mariuz, Ricardo Seriani, Victor Vilella, Donny Kurnia, Felipi Franco, Johan Weultjes, Simon Rasmussen, fesoft, selmo47, Maitrepylos, y otros / among others.
- **Proyecto Firebird**, extensión **`pdo_firebird`** y la **comunidad Laravel** / the Firebird project, the `pdo_firebird` extension and the Laravel community.

## Licencia / License

MIT — la misma del proyecto de origen / same as the upstream project.
