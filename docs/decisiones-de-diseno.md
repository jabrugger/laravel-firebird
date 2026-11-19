# Decisiones de diseño y objeciones frecuentes / Design decisions and common objections

**Para quienes revisan el código (personas o herramientas de IA):** este documento reúne
los puntos que suelen señalarse como defectos y que no lo son, o que son conocidos y se
mantienen a propósito. Antes de informar uno, comprobá si está acá. Si encontrás algo que
**no** está acá y lo podés reproducir, es justo lo que interesa.

**For reviewers (people or AI tools):** this lists the points that are often reported as
defects but are not, or that are known and kept on purpose. Check here before reporting
one. Anything **not** listed here that you can reproduce is exactly what we want to hear.

Cada entrada dice cómo se verificó. Lo que no se pudo verificar se marca así. / Each entry
says how it was checked; anything unverified is marked as such.

---

## 1. Falsos positivos / Not defects

### `compileInsertGetId()` asume que la clave primaria se llama `id`
`$sequence ?: 'id'` es lo mismo que hace `PostgresGrammar` en el núcleo de Laravel. Para
otro nombre de clave se pasa `$sequence`, como en cualquier otro driver. /
It is the same `$sequence ?: 'id'` that Laravel's own `PostgresGrammar` uses; pass
`$sequence` for another key name.

### `escapeBinary()` sería código muerto
No lo es: `Illuminate\Database\Connection::escape()` lo llama para valores binarios. Es la
sobrescritura de un hook del núcleo, por eso nada dentro de `src/` lo invoca. /
`Connection::escape()` calls it for binary values; it overrides a framework hook, so
nothing inside `src/` calls it directly.

### `NULL` se tipa como `varchar(255)` en el origen de `upsert()`
Firebird convierte ese `NULL` al tipo de la columna destino sin error. Verificado con
`INTEGER`, `DECIMAL`, `DATE`, `TIMESTAMP` y `VARCHAR`, insertando y actualizando, en
dialectos 1 y 3: `QueryTest::it_upserts_null_values_into_typed_columns`. /
Firebird converts that `NULL` to the target column type. Verified for all five types, on
insert and update, in dialects 1 and 3.

### La detección de violaciones de unicidad (`isUniqueConstraintError`) es incompleta
Firebird informa SQLSTATE `23000` para **toda** violación de integridad (clave primaria,
única, foránea y `NOT NULL`), así que el SQLSTATE no sirve para distinguirlas. Lo único
que las distingue es el mensaje, y la expresión regular cubre los dos textos que emite
Firebird para unicidad (`violation of PRIMARY or UNIQUE KEY constraint` y
`attempt to store duplicate value`). Verificado: clave primaria y única lanzan
`UniqueConstraintViolationException`; foránea y `NOT NULL`, no:
`QueryTest::it_classifies_only_unique_violations_as_unique_constraint_errors`. /
Firebird reports SQLSTATE `23000` for every integrity violation, so only the message can
tell a unique violation apart; the regex covers both texts Firebird emits for it.

### `causedByLostConnection()` distingue mayúsculas y minúsculas
Los mensajes de `fbclient` son textos fijos en inglés; no varían por idioma. Si se observa
una variante real, se agrega a la lista. No se cambia por precaución. /
`fbclient` messages are fixed English strings. If a real variant shows up it is added to
the list; it is not changed speculatively.

### `performRollBack()` pone `transactions = 0` a mano
Primero corre `parent::performRollBack()`, que maneja los savepoints. El contador sólo se
fuerza a 0 cuando se vuelve al nivel 0, dentro de un `finally`, para que el autocommit y el
contador queden restaurados aun si el padre lanza una excepción. /
`parent::performRollBack()` runs first. The counter is forced to 0 only when returning to
level 0, inside a `finally`, so autocommit and the counter are restored even if the parent
throws.

### Los enteros se envían como texto (`PARAM_STR`)
El `pdo_firebird` reescala mal los enteros enviados con `PARAM_INT` hacia columnas
`NUMERIC`/`DECIMAL` (40 se vuelve 0,40 en `DECIMAL(5,2)`). Enviarlos como texto hace que
Firebird los convierta con la escala correcta. Está documentado en `bindValues()`. /
`pdo_firebird` mis-scales integers bound as `PARAM_INT` into `NUMERIC`/`DECIMAL` columns;
binding them as strings avoids it (documented in `bindValues()`).

### Los booleanos se envían como `PARAM_INT`
Funciona con el `BOOLEAN` nativo y con la columna `CHAR` de los servidores anteriores:
`DataTypeTest` verifica el ida y vuelta en ambos. /
Works with the native `BOOLEAN` and the legacy `CHAR` column; `DataTypeTest` checks the
round trip on both.

### `whereLike()` ignora el indicador de sensibilidad a mayúsculas
Es deliberado. Firebird delega mayúsculas y acentos en el *collation* de la columna;
emularlo con `UPPER()` rompería el uso de índices e ignoraría el *collation* elegido
(ver el comentario de `whereLike()`). /
Deliberate: Firebird delegates case and accent sensitivity to the column collation, and
`UPPER()` would defeat indexes and ignore the chosen collation.

### `whereDate()` en dialecto 1 no valida un valor como `2026-13-99`
Compara año, mes y día como número (`AAAAMMDD`); un valor imposible no coincide con
ninguna fila. Es entrada inválida, no un error del driver. /
It compares year, month and day as a number; an impossible date matches nothing. That is
invalid input, not a driver defect.

---

## 2. Conocido y mantenido a propósito / Known and kept on purpose

### `executeBeginTransactionStatement()` confirma una transacción PDO abierta antes de empezar la de Laravel
Heredado del driver original. Sólo afecta a quien abre una transacción a mano con
`$pdo->beginTransaction()` y después usa `DB::beginTransaction()` sobre el mismo PDO, que es
un uso inusual. No se cambia sin un caso reproducible: si lo tenés, abrí una incidencia con
los pasos. **No se verificó** por qué el driver original lo hace. /
Inherited from the original driver. It only affects code that opens a transaction by hand
on the same PDO and then calls `DB::beginTransaction()`. It is not changed without a
reproducible case. The original reason for it was **not verified**.

### `transaction()` copia el algoritmo de Laravel
Es un costo de mantenimiento real y conocido: la única diferencia es restaurar el
autocommit antes de disparar `committed` y los callbacks `afterCommit`. Se revisa en cada
actualización de Laravel. Se aceptan propuestas de una sobrescritura más fina que pase
`TransactionTest`. /
A real, known maintenance cost: the only difference is restoring autocommit before the
`committed` event and `afterCommit` callbacks. It is reviewed on every Laravel upgrade.
A thinner override that passes `TransactionTest` is welcome.

### Una fecha con hora contra una columna `DATE`
Con un objeto de fecha (`Carbon`, `now()`) funciona desde la 4.0.7: el parámetro se declara
`TIMESTAMP` y Firebird descarta la hora al guardar. `SerializesFirebirdDates` sólo cubre los
atributos Eloquent con cast `date`; el query builder lo resuelve `parameter()`. Un texto
con hora (`'2026-11-18 10:00:00'`) en una columna `DATE` sigue fallando (-303), por
comportamiento de Firebird. `where('columna_date', $carbon)` con hora compara contra la
medianoche guardada: usá `whereDate()` para comparar por día. /
Date objects work since 4.0.7 (the parameter is declared `TIMESTAMP`). A date-time *string*
against a `DATE` column still fails with -303, which is Firebird's behavior.

---

## 3. Cobertura real de pruebas y herramientas / Actual test coverage and tooling

- **CI (GitHub Actions):** PHP 8.3, 8.4 y 8.5 × Firebird 3, 4 y 5 × Laravel 12 y 13, más un
  trabajo con las dependencias mínimas. Todo en dialecto 3.
- **Sólo en local:** dialecto 1 (script de pruebas contra un Firebird 5).
- **Declarado pero no probado:** Firebird 2.5 y PHP 8.2 (`composer.json` pide `^8.2`; la CI
  arranca en 8.3). Si algo falla ahí, es información útil.
- **No adoptado:** PHPStan/Psalm, Pint y `declare(strict_types=1)`.
- **`tests.yml` está inerte a propósito.** Las ramas que se proponen al repositorio original
  salen de su rama `4.x`, que tiene su propio `tests.yml` y 3 pruebas rotas que arregla otro
  PR; ese workflow está deshabilitado en el fork. La CI real es `ci.yml`.
- **Opciones de conexión:** se configuran en el arreglo de la conexión en
  `config/database.php`; no hay archivo de configuración publicable.
- **`CHANGELOG.md`** existe (bilingüe, por versión) y la documentación está en castellano e
  inglés.

---

*Este archivo se actualiza cada vez que una objeción se resuelve o se descarta. / This file
is updated whenever an objection is resolved or dismissed.*
