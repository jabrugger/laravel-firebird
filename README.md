# DASU Laravel Firebird

Driver de Firebird para Laravel · Firebird database driver for Laravel.

**[🇪🇸 Español](#-español)** · **[🇬🇧 English](#-english)**

> ## ⭐ Créditos / Credits
>
> Este driver se basa en el **fork de [Alexandre Benson Smith](https://github.com/bensonsbc) — [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird)** (paquete `benson/laravel-firebird`, [Thor Software](https://thorsoftware.com.br)), que aporta la base y gran parte de sus funcionalidades. **`dasu/laravel-firebird` es un fork mantenido con correcciones propias del driver, pruebas de regresión y documentación en español e inglés.** Las mejoras de aplicación general también se proponen al repositorio de Benson mediante *pull requests*.
>
> This driver is based on the **fork by [Alexandre Benson Smith](https://github.com/bensonsbc) — [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird)** (package `benson/laravel-firebird`, [Thor Software](https://thorsoftware.com.br)), which provides its foundation and much of its functionality. **`dasu/laravel-firebird` is a maintained fork with its own driver fixes, regression tests, and Spanish and English documentation.** Generally applicable improvements are also proposed to Benson's repository through pull requests.
>
> 💜 Apoyá el proyecto original / support the original project: **[github.com/sponsors/bensonsbc](https://github.com/sponsors/bensonsbc)**
>
> Cadena de origen / lineage: [`jacquestvanzuydam/laravel-firebird`](https://github.com/jacquestvanzuydam/laravel-firebird) → [`harrygulliford/laravel-firebird`](https://github.com/harrygulliford/laravel-firebird) → [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) → este repositorio / this repository.

> 📖 **Documentación / Documentation:** **[Wiki](https://github.com/jabrugger/laravel-firebird/wiki)** · **[CHANGELOG](CHANGELOG.md)** · **[Issues](https://github.com/jabrugger/laravel-firebird/issues)** · **[Decisiones de diseño / Design decisions](docs/decisiones-de-diseno.md)**

---

## 🇪🇸 Español

Integración previsible de Firebird con Laravel: Query Builder, Eloquent, paginación, `insertGetId()` con `INSERT ... RETURNING`, `insertOrIgnore`, `upsert` con `MERGE`, `update`/`delete` con joins, `truncate`, columnas identity, columnas calculadas, tablas temporales e introspección de esquema, con soporte de Firebird 2.5 → 5.0 (dialectos 1 y 3).

### Instalación

```bash
composer require "dasu/laravel-firebird:~4.0.4"
```

El paquete se publica en [Packagist](https://packagist.org/packages/dasu/laravel-firebird) con el nombre **`dasu/laravel-firebird`**, con la base de Benson y las correcciones mantenidas en este fork. El espacio de nombres PHP no cambia: `Benson\LaravelFirebird`, y el proveedor de servicios se registra solo. Los cambios de cada versión y su validación están en el [CHANGELOG](CHANGELOG.md).

> **Migrar desde `benson/laravel-firebird` o `harrygulliford/laravel-firebird`:** `composer remove` del paquete anterior y `composer require "dasu/laravel-firebird:~4.0.4"`. No hay que tocar el código ni la configuración de la conexión.

### Actualizaciones recomendadas

Recomendamos **`~4.0.4`**, equivalente a `>=4.0.4 <4.1.0`: admite nuevas correcciones de la línea `4.0.x` y deja el salto a `4.1` para una actualización deliberada. Si una aplicación tiene fijado `4.0.3` o `4.0.4` exactamente, el comando de instalación anterior cambia su dependencia a este rango.

Para incorporar las nuevas versiones permitidas:

```bash
composer update dasu/laravel-firebird
```

Publicar una versión en Packagist no modifica las aplicaciones instaladas. `composer update` selecciona la versión permitida más reciente compatible con las demás dependencias y actualiza `composer.lock`; `composer install` respeta ese lock. Después de comprobar la aplicación, guardá `composer.json` y `composer.lock` en Git para desplegar la misma versión en todos los entornos.

Un parche (`4.0.x`) no obliga a nadie a cambiar ni a revisar su código; los cambios de comportamiento que sí lo harían esperan a la `4.1` (ver el [hito 4.1](https://github.com/jabrugger/laravel-firebird/milestone/1)). Revisá el changelog antes de adoptarla; según versionado semántico, un cambio incompatible corresponde a una versión mayor, como `5.0`.

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
| `function_identifiers` | `true` | Los identificadores con forma de función (`upper(nombre)`) pasan sin comillas. **Poner `false` si los nombres de columna vienen de datos del usuario** y usar expresiones *raw* para las funciones. Está previsto cambiarlo a `false` por defecto en la 4.1 ([issue #1](https://github.com/jabrugger/laravel-firebird/issues/1)). |
| `server_version` | autodetectada | Fija la versión del servidor (p. ej. `'3.0.10'`) para evitar detectarla por la conexión. |
| `index_names` | — | Plantillas de nombres para PK/FK/unique/index. |

### Destacados

- **Funciones según la versión**: columnas identity, `BOOLEAN` nativo y `ALTER COLUMN` de nulidad en Firebird 3+; tipos `WITH TIME ZONE` e identificadores de 63 caracteres en Firebird 4+.
- **Nombres configurables** de constraints e índices (`index_names`), incluida la PK en línea de las columnas identity.
- **Columnas calculadas** (`virtualAs`/`storedAs` → `COMPUTED BY`) y **tablas temporales** (`temporary()` → GTT).
- **`uniqueIndex()`** (índice único sin constraint), **`COMMENT ON`**, **`insertOrIgnore`** por constraint real, reconexión automática e introspección con `auto_increment`.
- **Enteros en columnas `DECIMAL`**: se enlazan como texto, lo que evita un error silencioso de escala (con `PDO::PARAM_INT`, `40` quedaba grabado como `0.40` en un `DECIMAL(5,2)`).
- **Fechas**: objetos de fecha con hora contra columnas `DATE` desde el query builder, y el trait `SerializesFirebirdDates` (con el modelo base) para los casts `date` de Eloquent.
- **`upsert()` grande**: se parte en varias sentencias `MERGE` dentro de una transacción.

> 📖 **Documentación completa en el [wiki](https://github.com/jabrugger/laravel-firebird/wiki):** identificadores/caja/comillas, paginación, fechas, nombres de constraints, columnas calculadas, GTT, comportamientos específicos, limitaciones y más.

### Qué cambia respecto del repositorio de Benson

Este fork mantiene correcciones y mejoras del código, con pruebas de regresión y versiones propias en Packagist. Las mejoras de aplicación general se proponen también a [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) como *pull requests*; mientras se revisan, ya están en la rama `main` de este repositorio (la única que se usa como punto de partida). El estado de cada pull request se sigue en el [issue #8](https://github.com/jabrugger/laravel-firebird/issues/8) y la lista está en el [wiki](https://github.com/jabrugger/laravel-firebird/wiki/Relacion-con-el-repositorio-de-Benson).

---

## 🇬🇧 English

A predictable Firebird integration for Laravel: Query Builder, Eloquent, pagination, `insertGetId()` via `INSERT ... RETURNING`, `insertOrIgnore`, `upsert` via `MERGE`, joined update/delete, truncate, identity columns, computed columns, temporary tables and schema introspection — supporting Firebird 2.5 → 5.0 (dialects 1 and 3).

### Installation

```bash
composer require "dasu/laravel-firebird:~4.0.4"
```

The package is published on [Packagist](https://packagist.org/packages/dasu/laravel-firebird) as **`dasu/laravel-firebird`**, combining Benson's foundation with fixes maintained in this fork. The PHP namespace is unchanged, `Benson\LaravelFirebird`, and the service provider registers itself. Changes for each version and their validation are in the [CHANGELOG](CHANGELOG.md).

> **Migrating from `benson/laravel-firebird` or `harrygulliford/laravel-firebird`:** `composer remove` the old package and `composer require "dasu/laravel-firebird:~4.0.4"`. No code or connection configuration changes are needed.

### Recommended updates

Use **`~4.0.4`**, equivalent to `>=4.0.4 <4.1.0`: it accepts future fixes in the `4.0.x` line and leaves `4.1` for an explicit upgrade. For an application pinned to exactly `4.0.3` or `4.0.4`, the installation command above changes its requirement to this range.

To adopt newly available versions within that range:

```bash
composer update dasu/laravel-firebird
```

Publishing a release on Packagist does not modify installed applications. `composer update` selects the latest allowed version compatible with other dependencies and updates `composer.lock`; `composer install` uses that lock. After checking the application, commit both `composer.json` and `composer.lock` so every environment deploys the same version.

A patch (`4.0.x`) never requires anyone to change or review their code; behavior changes that would wait for `4.1` (see the [4.1 milestone](https://github.com/jabrugger/laravel-firebird/milestone/1)). Review the changelog before adopting it; under semantic versioning, an incompatible change requires a major release such as `5.0`.

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
| `function_identifiers` | `true` | Identifiers shaped like function calls (`upper(name)`) are passed through unquoted. **Set `false` when column names come from user input** and use raw expressions for functions. The default is planned to change to `false` in 4.1 ([issue #1](https://github.com/jabrugger/laravel-firebird/issues/1)). |
| `server_version` | auto-detected | Pin the server version (e.g. `'3.0.10'`) to skip detection over the connection. |
| `index_names` | — | Naming templates for PK/FK/unique/index. |

### Highlights

- **Version-aware features**: identity columns, native `BOOLEAN` and nullability `ALTER COLUMN` on Firebird 3+; `WITH TIME ZONE` types and 63-char identifiers on Firebird 4+.
- **Configurable naming** of constraints/indexes (`index_names`), including the inline PK of identity columns.
- **Computed columns** (`virtualAs`/`storedAs` → `COMPUTED BY`) and **temporary tables** (`temporary()` → GTT).
- **`uniqueIndex()`** (unique index without a constraint), **`COMMENT ON`**, real-constraint **`insertOrIgnore`**, automatic reconnect and introspection with `auto_increment`.
- **Integers into `DECIMAL` columns**: bound as text, which avoids a silent scale error (with `PDO::PARAM_INT`, `40` was stored as `0.40` in a `DECIMAL(5,2)`).
- **Dates**: date-time objects against `DATE` columns from the query builder, and the `SerializesFirebirdDates` trait (with the base model) for Eloquent `date` casts.
- **Large `upsert()`**: split into several `MERGE` statements inside one transaction.

> 📖 **Full documentation in the [wiki](https://github.com/jabrugger/laravel-firebird/wiki):** identifiers/case/quoting, pagination, dates, constraint naming, computed columns, GTT, Firebird-specific behavior, limitations and more.

### How this differs from Benson's repository

This fork maintains code fixes and improvements, regression tests, and its own Packagist releases. Generally applicable improvements are also proposed to [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird) as pull requests; while they are reviewed, they are already in this repository's `main` branch (the only one used as a starting point). The status of each pull request is tracked in [issue #8](https://github.com/jabrugger/laravel-firebird/issues/8) and the list is in the [wiki](https://github.com/jabrugger/laravel-firebird/wiki/Relacion-con-el-repositorio-de-Benson).

---

## Pruebas / Tests

```bash
docker compose up -d        # Firebird 5 (puerto/port 3055), 4 (3054) y/and 3 (3053)
DB_PORT=3055 vendor/bin/phpunit
```

Para dialecto 1, exportá `DB_DIALECT=1` apuntando a una base en dialecto 1. / For dialect 1, export `DB_DIALECT=1` pointing at a dialect-1 database.

La integración continua corre la suite contra Firebird 3, 4 y 5 en PHP 8.3, 8.4 y 8.5, con Laravel 12 y 13. / CI runs the suite against Firebird 3, 4 and 5 on PHP 8.3, 8.4 and 8.5, with Laravel 12 and 13. Qué cubre y qué no / what it covers and what it does not: [wiki](https://github.com/jabrugger/laravel-firebird/wiki/Pruebas-y-CI).

## Para quienes revisan el código / For reviewers

Antes de informar un defecto, mirá [docs/decisiones-de-diseno.md](docs/decisiones-de-diseno.md): reúne los puntos que suelen señalarse y que son falsos positivos o decisiones conocidas, con cómo se verificó cada uno. Lo pendiente conocido está en los [issues](https://github.com/jabrugger/laravel-firebird/issues). / Before reporting a defect, see [docs/decisiones-de-diseno.md](docs/decisiones-de-diseno.md): it lists the points that are often reported but are false positives or known decisions, with how each was checked. Known pending work is in the [issues](https://github.com/jabrugger/laravel-firebird/issues).

Las regresiones de callbacks de commit manual, bindings de `exists()` (incluyendo unions) y expresiones SQL en `upsert()` también se verifican sin servidor Firebird. / Manual-commit callbacks, `exists()` bindings (including unions), and SQL expressions in `upsert()` also have regression tests that run without a Firebird server:

```bash
vendor/bin/phpunit tests/TransactionRegressionTest.php tests/ExistsRegressionTest.php tests/UpsertRegressionTest.php
```

## Créditos completos / Full credits

- **[Alexandre Benson Smith](https://github.com/bensonsbc)** ([Thor Software](https://thorsoftware.com.br)) — autor del fork [`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird), base de este driver y de gran parte de sus funcionalidades / author of the fork providing this driver's foundation and much of its functionality.
- **[jabrugger](https://github.com/jabrugger)** — mantenimiento de este fork, correcciones del driver, pruebas de regresión y documentación / maintenance of this fork, driver fixes, regression tests, and documentation.
- **[Harry Gulliford](https://github.com/harrygulliford)** — mantenedor del fork base / maintainer of the base fork.
- **[Jacques van Zuydam](https://github.com/jacquestvanzuydam)** — autor original / original author.
- **Contribuidores / contributors**: mariuz, Ricardo Seriani, Victor Vilella, Donny Kurnia, Felipi Franco, Johan Weultjes, Simon Rasmussen, fesoft, selmo47, Maitrepylos, y otros / among others.
- **Proyecto Firebird**, extensión **`pdo_firebird`** y la **comunidad Laravel** / the Firebird project, the `pdo_firebird` extension and the Laravel community.

## Licencia / License

MIT — la misma del proyecto de origen / same as the upstream project.
