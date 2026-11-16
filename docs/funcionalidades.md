# Documentación del driver `benson/laravel-firebird`

Referencia de las funcionalidades del driver de Firebird para Laravel. Se mantiene
al día a medida que se agregan recursos.

> **Créditos:** el driver es obra de [Alexandre Benson Smith](https://github.com/bensonsbc)
> ([`bensonsbc/laravel-firebird`](https://github.com/bensonsbc/laravel-firebird)).
> Este documento es la traducción al español de su documentación original (en
> portugués), con ajustes menores marcados como *(actualizado)*.
>
> La visión general y la instalación están en el [README](../README.md). Este
> documento detalla el uso de cada recurso.

## Sumario

- [Configuración de la conexión](#configuración-de-la-conexión)
- [Identificadores, caja y comillas](#identificadores-caja-y-comillas)
- [Recursos según la versión del servidor](#recursos-según-la-versión-del-servidor)
- [Paginación](#paginación)
- [Fechas: cast `date` sin hora](#fechas-cast-date-sin-hora)
- [Nombres de constraints e índices](#nombres-de-constraints-e-índices)
- [Índice único sin constraint](#índice-único-sin-constraint)
- [Columnas calculadas](#columnas-calculadas)
- [Tablas temporales](#tablas-temporales)
- [Comportamientos específicos de Firebird](#comportamientos-específicos-de-firebird)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Pruebas](#pruebas)

---

## Configuración de la conexión

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

| Opción | Valor por defecto | Descripción |
| --- | --- | --- |
| `quote_identifiers` | `true` | Envuelve los identificadores entre comillas dobles. Desactivar para bases legadas con objetos sin comillas. |
| `uppercase_identifiers` | `false` | Convierte a mayúsculas los identificadores sin comillas (estilo legado). |
| `pagination_mode` | `first_skip` | `first_skip` usa `SELECT FIRST n SKIP m`; `offset_fetch` usa `OFFSET ... FETCH`. |
| `dialect` | — | Poner `1` para bases en dialecto 1 (ajusta tipos y fechas, y desactiva identity). |
| `role` | — | Rol SQL que se envía al conectar. |
| `column_case` | `natural` | *(actualizado)* Caja de los nombres de columna en los resultados: `natural` (como los devuelve Firebird), `lower` o `upper`. Ver [Caja en el resultado](#caja-en-el-resultado-pdoattr_case). |
| `function_identifiers` | `true` | *(propuesta pendiente de aceptación)* Los identificadores con forma de función (`upper(nombre)`) pasan sin comillas. Poner `false` para entrecomillar siempre. |
| `server_version` | autodetectada | Fija la versión del servidor (p. ej. `'3.0.10'`) y evita detectarla por la conexión. |
| `index_names` | — | Plantillas para los nombres por defecto de PK/FK/unique/index. Ver la sección propia. |

---

## Identificadores, caja y comillas

Esta es el área más específica (y más delicada) de Firebird. Entender cómo trata
el servidor los identificadores evita la mayoría de los problemas de "la tabla o
columna no existe".

### Cómo trata Firebird los identificadores

- **Sin comillas** (`select * from CLIENTE`): no distingue mayúsculas de
  minúsculas; el servidor **lo pasa a MAYÚSCULAS** y así queda el nombre en el
  catálogo (`rdb$relation_name = 'CLIENTE'`).
- **Con comillas** (`select * from "Cliente"`): distingue mayúsculas; el nombre se
  guarda **exactamente** como se escribió. `"Cliente"`, `"cliente"` y `CLIENTE`
  son tres objetos distintos.

Consecuencia práctica: una tabla creada por una herramienta legada **sin
comillas** queda como `AUDITORIA` en el catálogo; una tabla creada por Laravel
(que por defecto usa comillas) queda como `auditoria`. Las dos conviven en la
misma base.

### Los dos parámetros del driver

| Configuración | Evaluación interna | Efecto |
| --- | --- | --- |
| `quote_identifiers` | verdadero salvo que sea exactamente `false` (por defecto: con comillas) | Activado, envuelve los identificadores entre comillas dobles y preserva la caja. Desactivado, emite los identificadores **sin** comillas. |
| `uppercase_identifiers` | verdadero solo cuando es exactamente `true` (por defecto: `false`) | Activado, convierte a MAYÚSCULAS los identificadores sin comillas antes de emitirlos. |

### Matriz de combinaciones

Para `DB::table('cliente')->select('nombre')`:

| `quote_identifiers` | `uppercase_identifiers` | SQL generado | Usar cuando |
| --- | --- | --- | --- |
| `true` (defecto) | `false` (defecto) | `select "nombre" from "cliente"` | Base nueva, creada por el propio Laravel (nombres en minúsculas). |
| `false` | `true` | `select NOMBRE from CLIENTE` | Base legada, creada sin comillas (nombres en mayúsculas). |
| `false` | `false` | `select nombre from cliente` | Raro: el servidor lo pasa a `NOMBRE`/`CLIENTE` al resolver. |
| `true` | `true` | `select "NOMBRE" from "CLIENTE"` | Legado que se quiere referenciar entre comillas y en mayúsculas. |

> Regla práctica: **base nueva** → dejar los valores por defecto
> (`quote_identifiers = true`, `uppercase_identifiers = false`). **Base legada en
> mayúsculas** → `quote_identifiers = false` + `uppercase_identifiers = true`.

### Identificadores reservados siempre entrecomillados

Aun con `quote_identifiers = false`, algunos nombres se entrecomillan **siempre**
porque chocan con palabras reservadas de Firebird:

```
KEY, TIMESTAMP, VALUE
```

Así, `select key, value from cache` en modo legado genera
`select "KEY", "VALUE" from CACHE`: las columnas reservadas quedan entre comillas
(y en mayúsculas); el resto sigue el modo configurado.

### Alias

Fuera del dialecto 1, los alias de columna se entrecomillan **siempre**, para que
el nombre de la columna en el resultado preserve la caja (`select X as "userName"`).
En el dialecto 1 el alias sigue la normalización de identificadores (mayúsculas
cuando está configurado).

### Caja en el resultado (`PDO::ATTR_CASE`)

*(actualizado)* Desde la v4.0.1 la caja de los nombres de columna del resultado la
controla la opción **`column_case`**:

- `natural` (**por defecto**): se devuelve como la entrega Firebird. Los
  identificadores sin comillas (por ejemplo `select nombre from cliente` en modo
  legado) vuelven en **MAYÚSCULAS** y los alias entrecomillados vuelven **tal como
  se escribieron** (`select X as "userName"` → clave `userName`).
- `lower`: todos los nombres de columna vuelven en minúsculas (el comportamiento
  fijo de las versiones anteriores).
- `upper`: todos vuelven en mayúsculas.

Los **valores** no se ven afectados, solo los nombres de las columnas. Una
aplicación que lea las propiedades de los resultados debe usar la caja que
corresponda a esta opción.

> En el dialecto 1 los alias no se pueden entrecomillar, por lo que con `natural`
> los alias vuelven en mayúsculas. Si el código espera minúsculas, usar
> `'column_case' => 'lower'`.

### Caja en la introspección del esquema

Métodos como `Schema::getTables()`, `getColumns()`, `getIndexes()` y
`getForeignKeys()` devuelven los nombres **en minúsculas** (normalización del
*processor*), para coincidir con lo que espera Laravel. Internamente, las
consultas de metadatos normalizan el nombre buscado según `uppercase_identifiers`
(mayúsculas cuando está activado), de modo que `Schema::hasTable('cliente')`
encuentra `CLIENTE` en una base legada.

> Atención en bases **mixtas** (tablas legadas en mayúsculas + tablas de Laravel
> en minúsculas): las operaciones que toman el nombre en minúsculas de la
> introspección y vuelven a emitir un identificador entre comillas pueden no
> coincidir con el nombre real. El driver ya lo resuelve en
> `dropAllTables`/`dropAllViews` (usando el nombre real del catálogo); si se
> encuentra otro punto sensible, reportarlo.

### Caja en los nombres de constraints/índices

El nombre generado por las plantillas de [`index_names`](#nombres-de-constraints-e-índices)
es literal: la caja que se escribe en la plantilla es el punto de partida. Luego
pasa por el mismo tratamiento de arriba:

- Con los valores por defecto (`quote_identifiers = true`,
  `uppercase_identifiers = false`): `'PK_{table}'` en una tabla `clientes` queda
  como la constraint `"PK_clientes"` (caja preservada).
- Con `uppercase_identifiers = true`: la misma plantilla queda `"PK_CLIENTES"`
  (forzada a mayúsculas).
- Con `quote_identifiers = false`: sale sin comillas y el servidor lo pasa a
  mayúsculas en el catálogo.

Conviene elegir la caja de la plantilla de acuerdo con la convención del proyecto
**y** con el modo de identificadores de la conexión.

### Dialecto 1

Con `'dialect' => '1'`, además de los ajustes de tipos y fechas, el autoincremento
por identity se desactiva (vuelve al camino generador + trigger) y la
normalización de alias sigue el modo de identificadores en lugar de entrecomillar
siempre.

---

## Recursos según la versión del servidor

La versión se detecta automáticamente (o se fija con `server_version`) y habilita
recursos:

- **Firebird 3+** — autoincremento con `GENERATED BY DEFAULT AS IDENTITY` (sin
  generador/trigger); `ALTER COLUMN SET/DROP NOT NULL` en `->change()`;
  `boolean()` genera `BOOLEAN` nativo (dialecto 3; `default(true)` pasa a
  `DEFAULT TRUE`); `auto_increment` correcto en la introspección; límites de
  *eager load* (`->limit()` en relaciones) con `ROW_NUMBER()`. En servidores 2.5
  (y en dialecto 1) el driver usa el camino legado: generador + trigger,
  nulabilidad por la tabla de sistema y `boolean()` como `CHAR(1)`.
- **Firebird 4+** — identificadores de hasta 63 caracteres; `timestampTz()`,
  `timeTz()` y `dateTimeTz()` generan tipos `WITH TIME ZONE`; `joinLateral()`
  (tablas derivadas laterales; en servidores más viejos lanza `RuntimeException`).

Los nombres de objetos por encima del límite de identificadores del servidor
reciben un sufijo de hash automático para evitar colisiones silenciosas.

---

## Paginación

Por defecto (`first_skip`):

```sql
select first 10 skip 20 * from CLIENTE
```

Con `'pagination_mode' => 'offset_fetch'`:

```sql
select * from CLIENTE offset 20 rows fetch first 10 rows only
```

---

## Fechas: cast `date` sin hora

En el dialecto 3 el tipo `DATE` de Firebird **no tiene componente de hora**.
Laravel formatea todo valor temporal con un único formato (`Y-m-d H:i:s`), lo que
hace que Firebird rechace la grabación en una columna `DATE` con
`-413 conversion error from string`.

La solución está en la capa del modelo, usando el cast que ya se declara. Hay dos
formas:

**1. Trait `SerializesFirebirdDates`:**

```php
use Benson\LaravelFirebird\Eloquent\Concerns\SerializesFirebirdDates;
use Illuminate\Database\Eloquent\Model;

class Pedido extends Model
{
    use SerializesFirebirdDates;

    protected $casts = [
        'fecha_emision' => 'date',      // se graba como 2026-06-18
        'creado_en'     => 'datetime',  // se graba como 2026-06-18 13:45:00
    ];
}
```

**2. Modelo base `Benson\LaravelFirebird\Eloquent\Model`** (ya incluye el trait):

```php
use Benson\LaravelFirebird\Eloquent\Model;

class Pedido extends Model
{
    protected $casts = ['fecha_emision' => 'date'];
}
```

Comportamiento:

| Cast | Se asigna | Va a la base |
| --- | --- | --- |
| `date` | `'2026-06-18'` | `2026-06-18` |
| `date` | `Carbon` con hora | `2026-06-18` (se descarta la hora) |
| `datetime` | `Carbon` con hora | `2026-06-18 13:45:00` |

La lectura sigue devolviendo instancias de `Carbon`. Vale para el acceso por
Eloquent; un `DB::table()->insert(['fecha' => $carbon])` crudo sigue usando el
formato global: en ese caso hay que pasar la fecha como texto `Y-m-d`.

---

## Nombres de constraints e índices

Por defecto Laravel nombra constraints e índices como `{tabla}_{columnas}_{tipo}`.
Para seguir la convención del proyecto, configurar `index_names` en la conexión,
con los marcadores `{table}` y `{columns}`:

```php
'firebird' => [
    // ...
    'index_names' => [
        'primary'     => 'PK_{table}',
        'foreign'     => 'FK_{table}_{columns}',
        'unique'      => 'UQ_{table}_{columns}',
        'index'       => 'IX_{table}_{columns}',
        'uniqueIndex' => 'UX_{table}_{columns}',
    ],
],
```

A partir de ahí, las constraints creadas **sin nombre explícito** siguen el patrón:

```php
Schema::create('clientes', function (Blueprint $table) {
    $table->integer('id');
    $table->string('email');
    $table->integer('user_id');

    $table->primary('id');                                      // PK_clientes
    $table->unique('email');                                    // UQ_clientes_email
    $table->unique(['cuit', 'nombre']);                         // UQ_clientes_cuit_nombre
    $table->index('nombre');                                    // IX_clientes_nombre
    $table->foreign('user_id')->references('id')->on('users');  // FK_clientes_user_id
});
```

Detalles:

- **Cada tipo es opcional**: los que no se definen mantienen el valor por defecto de Laravel.
- **El nombre explícito** (2.º argumento) siempre tiene prioridad: `$table->unique('email', 'UQ_LOGIN')`.
- **El borrado por columnas** reutiliza el mismo generador de nombres, así que `$table->dropUnique(['email'])` encuentra el nombre correcto.
- **`unique()` crea constraint + índice único** con el mismo nombre (en Firebird una constraint UNIQUE genera por debajo un índice único).
- **Las foreign keys crean un índice automáticamente.** En Firebird, al crear una FK el servidor crea un índice sobre sus columnas, con el **mismo nombre** que la constraint. O sea, con `'foreign' => 'FK_{table}_{columns}'` existe una constraint `FK_...` y un índice `FK_...`. No hace falta (ni conviene) crear un `index()` adicional para la columna de la FK: generaría un índice duplicado.
- **La PK de las columnas autoincrementales (`id()`, `increments()`) también sigue la plantilla `primary`.** La clave primaria de esas columnas se emite en línea en la definición de la columna (`... AS IDENTITY PRIMARY KEY`), fuera de `compilePrimary()`. El driver aplica ahí la plantilla `primary` como una constraint nombrada en línea; por ejemplo: `id BIGINT GENERATED BY DEFAULT AS IDENTITY CONSTRAINT PK_usuario PRIMARY KEY`. Sin plantilla, la PK queda sin nombre (el servidor genera un `INTEG_nnn`).
- **Límite de tamaño**: los nombres largos reciben un sufijo de hash respetando el límite del servidor (31/63 caracteres).

---

## Índice único sin constraint

`unique()` crea una **constraint** UNIQUE. Para un índice único **puro**
(`CREATE UNIQUE INDEX`, sin constraint: no referenciable por una FK y que se borra
con `DROP INDEX`) se usa `uniqueIndex()`:

```php
Schema::table('clientes', function (Blueprint $table) {
    $table->uniqueIndex('email');                 // UX_clientes_email (plantilla uniqueIndex)
    $table->uniqueIndex(['cuit', 'nombre']);      // UX_clientes_cuit_nombre
    $table->uniqueIndex('cuit', 'UX_ESPECIAL');   // nombre explícito
});
```

Se compila a `CREATE UNIQUE INDEX`. El nombre automático sigue la plantilla
`uniqueIndex` de `index_names` (con el valor por defecto de Laravel si no está definida).

Para quitarlo:

```php
$table->dropUniqueIndex('UX_clientes_email'); // por nombre
$table->dropUniqueIndex(['email']);           // por columnas (reconstruye el nombre con la plantilla)
```

Diferencia entre los dos:

| | `unique()` | `uniqueIndex()` |
| --- | --- | --- |
| Objeto creado | constraint UNIQUE (+ índice) | solo índice único |
| Aparece en `rdb$relation_constraints` | sí | no |
| Puede ser destino de una FK | sí | no |
| Se quita con | `dropUnique()` | `dropUniqueIndex()` |

---

## Columnas calculadas

Las columnas `virtualAs()`/`storedAs()` (y el tipo `computed`) se compilan a
`COMPUTED BY` de Firebird:

```php
Schema::create('items', function (Blueprint $table) {
    $table->integer('cantidad');
    $table->decimal('precio', 10, 2);
    $table->decimal('total', 12, 2)->virtualAs('"cantidad" * "precio"');
});
```

La expresión es SQL crudo: hay que referenciar las columnas con la caja y las
comillas que usa la conexión (en el dialecto 1, sin comillas y en mayúsculas). En
Firebird toda columna calculada es **virtual** (se calcula al leer); `storedAs()`
se trata como `virtualAs()`.

## Tablas temporales

`$table->temporary()` crea una `GLOBAL TEMPORARY TABLE ... ON COMMIT PRESERVE ROWS`
(alcance de conexión, similar a las tablas temporales de sesión de otras bases):

```php
Schema::create('calc_temp', function (Blueprint $table) {
    $table->temporary();
    $table->integer('id');
    $table->decimal('valor', 12, 2);
});
```

> La definición de la GTT es permanente en el esquema; solo los datos son
> temporales. `getTables()`/`dropAllTables()` listan solo las tablas persistentes
> (`relation_type = 0`), por lo que `dropAllTables()` no elimina las GTT.

## Comportamientos específicos de Firebird

- **`insertOrIgnore`** inserta fila por fila dentro de una transacción e ignora las violaciones de cualquier constraint única. `insertOrIgnoreUsing` resuelve los índices únicos de la tabla y genera un `NOT EXISTS` por índice.
- **`firstOrCreate`/`createOrFirst`** funcionan ante condiciones de carrera (las violaciones de unicidad se convierten en `UniqueConstraintViolationException`).
- **`truncate()`** emula `TRUNCATE` con `DELETE FROM` y reinicia los generadores y las columnas identity.
- **`upsert()`** se compila a `MERGE`. Los valores de la fuente derivada reciben un `CAST` tipado (Firebird no infiere los tipos de los parámetros en tablas derivadas); los textos de más de 8191 caracteres (límite de `VARCHAR` en UTF8) usan `CAST(? AS BLOB SUB_TYPE TEXT)`.
- **`->change()`** inspecciona la definición actual de la columna y omite el `ALTER TYPE` o los cambios de nulabilidad redundantes: una migración que solo cambia el `default` no dispara `ALTER TYPE` (que Firebird rechaza en varias conversiones). Sin una conexión inspeccionable se emiten todas las sentencias (semántica estándar de Laravel de redefinición completa).
- **`update`/`delete` con joins** ubican las filas destino mediante `RDB$DB_KEY`.
- **Las listas `IN` de más de 1499 elementos** se dividen automáticamente en varios grupos `IN`.
- **`whereLike`** sigue el comportamiento de la base: `LIKE` respeta la collation de la columna (no fuerza `UPPER()`, que rompería los índices e ignoraría la collation elegida). Para una búsqueda insensible a mayúsculas y acentos, definir la columna con una collation CI/AI (p. ej. `UNICODE_CI` / `UNICODE_CI_AI`).
- **`dropAllTables`/`dropAllViews`** usan el nombre real del catálogo (con la caja preservada), por lo que funcionan tanto con tablas en minúsculas (creadas con comillas) como en mayúsculas (legadas, creadas sin comillas).
- **`->from(n)` / `->startingValue(n)`** en columnas autoincrementales: en servidores con identity, `ALTER ... RESTART WITH` (con el ajuste del desfase de uno de Firebird 3, donde `RESTART WITH n` genera `n+1`); en el camino legado, `SET GENERATOR ... TO n-1`. El primer valor generado es siempre `n`, en cualquier versión.
- **`tinyText()`** pasa a `VARCHAR(255)` y **`year()`** pasa a `INTEGER`.
- **`DB::escape($valor, binary: true)`** genera un literal hexadecimal `x'...'`.
- **`db:show` / `db:monitor`** informan las conexiones abiertas mediante `MON$ATTACHMENTS`.
- **Enteros en columnas `DECIMAL`/`NUMERIC` con escala.** `pdo_firebird` aplica una escala equivocada cuando un entero se enlaza con `PDO::PARAM_INT` (p. ej., `40` pasa a `0.40` en un `DECIMAL(5,2)`). El driver sobrescribe `bindValues()` para enlazar los enteros como texto (`PARAM_STR`), que Firebird convierte con la escala correcta. Vale para cualquier columna; los enteros normales, los `where` y los booleanos siguen correctos.
- **Transacciones y autocommit.** `pdo_firebird` en autocommit graba cada comando al instante (un rollback no desharía nada); por eso el driver desactiva el autocommit al abrir la transacción y lo reactiva cuando termina la transacción externa (commit o rollback). En `DB::transaction(fn …)`, el autocommit vuelve **justo después del commit**, antes de los callbacks "después del commit" (`DB::afterCommit`, jobs y eventos con `afterCommit`): antes de la v4.0.2 volvía recién después de ellos y cualquier comando dentro de un callback fallaba con `-901 invalid transaction handle (expecting explicit transaction start)`. Los savepoints (transacciones anidadas), el rollback y el reintento ante conflicto (`DB::transaction($fn, $intentos)`) siguen como antes, cubiertos en `tests/TransactionTest.php`, verificando por una segunda conexión lo que realmente se grabó.

---

## Limitaciones conocidas

- *(actualizado)* La caja de los nombres de columna del resultado depende de `column_case` (por defecto `natural`: tal como la devuelve Firebird). Con `lower` todos vuelven en minúsculas, incluidos los alias entrecomillados.
- `disableForeignKeyConstraints()`/`enableForeignKeyConstraints()` no hacen nada (Firebird no tiene un interruptor de FK por conexión). `dropAllTables()` elimina las FK antes que las tablas.
- Las operaciones JSON (`whereJsonContains`, selectores `->`) no están soportadas; las columnas `json()` pasan a `BLOB SUB_TYPE TEXT`.
- `Schema::rename()` (renombrar tabla) y `renameIndex()` no están soportados por Firebird: ambos lanzan una excepción (en lugar del no-op silencioso del framework para los métodos ausentes).
- La opción `isolation_level` de la conexión no está soportada (en Firebird el aislamiento es por transacción; `pdo_firebird` solo expone control desde PHP 8.4).
- `schema:dump` no está soportado (requeriría `isql`/`gbak` en el host).
- `sharedLock()`/`lock(false)` lanza `RuntimeException`: Firebird solo tiene bloqueo exclusivo de fila (`WITH LOCK`); ignorarlo en silencio quitaría la garantía de concurrencia pedida. Usar `lockForUpdate()`.
- En servidores 2.5, volver `NOT NULL` una columna con `->change()` escribe el indicador directamente en la tabla de sistema y **no valida las filas existentes**: los NULL ya grabados permanecen y solo aparecen tras un ciclo de backup/restore. Hay que asegurarse de que la columna no tenga NULL antes de modificarla.
- Las columnas booleanas legadas (`CHAR(1)` creadas por versiones viejas del driver) siguen funcionando para `insert`/`where` con `'0'`/`'1'`; pero en un servidor 3+, `upsert()` con un `bool` de PHP castea la fuente como `BOOLEAN`: para columnas `CHAR(1)` legadas hay que pasar `'0'`/`'1'` (o enteros) en el `upsert`.
- `orderByRandom()` ignora la semilla.
- Las constraints CHECK no tienen nombre automático (Laravel no trae un `$table->check()` nativo); el único CHECK que se emite es el que lleva `enum()` en línea.
- El cliente `fbclient` debe ser compatible con el servidor: un cliente 3.0 contra un servidor 4+/5 falla (`-204 Data type unknown`) al leer tipos nuevos (`TIMESTAMP WITH TIME ZONE`, `DECFLOAT`, `INT128`) o un `CURRENT_TIMESTAMP` crudo.

---

## Pruebas

```bash
docker compose up -d        # Firebird 5 (puerto 3055) y Firebird 4 (puerto 3054)
DB_PORT=3055 vendor/bin/phpunit
```

Para el dialecto 1, exportar `DB_DIALECT=1` apuntando a una base en dialecto 1.

La integración continua (GitHub Actions) corre la suite contra Firebird 3, 4 y 5 en PHP 8.3 y 8.4.
