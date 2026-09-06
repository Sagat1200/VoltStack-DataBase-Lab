# 69_DATABASE_MYSQL_SQL_COMPILER.md

# VoltStack Quantum Database
## MySQL SQL Compiler

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 69 — MySQL SQL Compiler  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`MySQL SQL Compiler` define la especialización del SQL Compiler de VoltStack para transformar operaciones compilables en SQL válido para una familia MySQL concreta, sin introducir semántica nueva, consultar el servidor durante compilación ni mezclar generación SQL con ejecución.

Su función principal será:

```text
CompilableDatabaseOperation
        │
        ▼
Generic SQL Compiler Pipeline
        │
        ▼
MySQL Dialect Adaptation
        │
        ▼
MySQL SqlEmissionTree
        │
        ▼
MySQL SQL Generation
        │
        ▼
MySqlCompiledDatabaseCommand
```

El compilador MySQL implementa las fronteras definidas en:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
67_DATABASE_SQL_COMPILER_PIPELINE.md
68_DATABASE_SQL_GENERATION_SYSTEM.md
```

La regla fundamental será:

```text
MySQL Compiler
=
Common SQL Compiler
+
MySQL Capability Profile
+
MySQL Dialect Adaptation
+
MySQL Rendering Rules
```

y no:

```text
MySQL Compiler
=
Generic Compiler Forked And Rewritten For MySQL
```

---

# 2. Objetivo arquitectónico

El compilador deberá proporcionar:

- SQL compatible con el target MySQL declarado;
- identificación explícita de versión/capacidades;
- quoting correcto de identifiers;
- literals y placeholders compatibles;
- adaptación de funciones;
- adaptación de operadores;
- `LIMIT`/`OFFSET`;
- locking;
- CTE;
- recursive CTE;
- window functions;
- JSON;
- generated columns cuando correspondan a SQL compilable;
- `INSERT`;
- `UPDATE`;
- `DELETE`;
- UPSERT mediante las capacidades correspondientes;
- representación de RETURNING sólo si la capability real lo permite;
- tratamiento explícito de diferencias entre versiones;
- manejo seguro de features no soportadas;
- compilación determinista;
- prepared-statement compatibility;
- source mapping;
- persistent-runtime safety.

---

# 3. MySQL no es un booleano

VoltStack no modelará MySQL como:

```php
$isMysql = true;
```

Debe modelarse mediante un target descriptor.

```php
final readonly class MySqlTarget
{
    public function __construct(
        public MySqlVersion $version,
        public MySqlCapabilitySnapshot $capabilities,
        public MySqlDialectProfile $dialect,
    ) {}
}
```

---

# 4. Target explícito

Ejemplo conceptual:

```text
MySqlTarget
├── family = MYSQL
├── version = 8.x
├── capabilities
├── dialect profile
├── placeholder profile
├── identifier profile
└── compiler feature set
```

---

# 5. No consultar SELECT VERSION()

El compiler no deberá ejecutar:

```sql
SELECT VERSION()
```

durante una compilación.

La versión debe provenir del contexto/configuración/capability discovery realizado por una capa apropiada.

---

# 6. Discovery ≠ compilation

```text
Connection/Platform Discovery
        │
        ▼
MySqlCapabilitySnapshot
        │
        ▼
Compiler
```

Nunca:

```text
Compiler
  │
  ├── open connection
  ├── SELECT VERSION()
  └── compile
```

---

# 7. Versión objetivo

La versión objetivo deberá ser explícita.

Ejemplo:

```php
database:
    default: mysql

    connections:
        mysql:
            driver: mysql
            version: "8.4"
```

La configuración exacta se definirá en el Database Configuration System.

---

# 8. Version abstraction

```php
final readonly class MySqlVersion
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
    ) {}
}
```

---

# 9. Version comparisons

No utilizar comparaciones string ingenuas.

Incorrecto:

```php
if ($version >= '8.0') {
}
```

Preferir:

```php
if ($version->isAtLeast(8, 0, 0)) {
}
```

---

# 10. Version ≠ capabilities

La versión ayuda a construir capabilities.

Pero el compiler deberá consumir principalmente:

```text
CapabilitySnapshot
```

y no dispersar cientos de comparaciones de versión.

---

# 11. Fórmula

```text
Version
+
Configuration
+
Driver Knowledge
+
Explicit Overrides
        │
        ▼
Capability Resolution
        │
        ▼
MySqlCapabilitySnapshot
        │
        ▼
MySQL Compiler
```

---

# 12. Posición arquitectónica

```text
Query / Plan / Command
        │
        ▼
SQL Compiler
        │
        ├── Generic Lowering
        │
        ├── MySQL Dialect Adaptation
        │
        ├── Validation
        │
        ├── Alias Planning
        │
        ├── Placeholder Planning
        │
        ├── SQL Generation
        │
        └── Binding Compilation
        │
        ▼
MySqlCompiledDatabaseCommand
        │
        ▼
Executor
        │
        ▼
Connection
        │
        ▼
MySQL Driver
```

---

# 13. Fronteras

El MySQL Compiler:

```text
does:
    compile
    adapt
    validate
    render
```

No:

```text
execute
connect
hydrate
persist entities
manage transactions
discover schema live
optimize using EXPLAIN
```

---

# 14. Namespace

Propuesta:

```text
VoltStack\Quantum\Database\Query\Compiler\Platform\MySql
```

---

# 15. Componentes principales

```text
MySqlCompiler
MySqlCompilerFactory
MySqlCompilationContext
MySqlTarget
MySqlVersion
MySqlCapabilitySnapshot
MySqlDialectProfile
MySqlDialectAdapter
MySqlRepresentationValidator
MySqlIdentifierRenderer
MySqlPlaceholderRenderer
MySqlLiteralRenderer
MySqlOperatorRenderer
MySqlFunctionRendererRegistry
MySqlPaginationRenderer
MySqlLockingRenderer
MySqlUpsertRenderer
MySqlJsonRenderer
MySqlCteRenderer
MySqlWindowRenderer
MySqlSetOperationRenderer
MySqlTypeRenderer
MySqlMutationRenderer
MySqlFeatureRegistry
```

---

# 16. Compiler facade

```php
interface MySqlCompiler
{
    public function compile(
        CompilableDatabaseOperation $operation,
        MySqlCompilationContext $context,
    ): CompiledDatabaseCommand;
}
```

---

# 17. Prefer generic compiler orchestration

La implementación concreta debería reutilizar:

```text
DefaultSqlCompiler
```

inyectando componentes MySQL.

Ejemplo conceptual:

```php
$compiler = new DefaultSqlCompiler(
    dialectAdapter: $mysqlDialectAdapter,
    representationValidator: $mysqlValidator,
    renderer: $sqlRenderer,
    dialect: $mysqlDialect,
);
```

---

# 18. No MegaMySqlCompiler

Evitar una clase:

```text
MySqlCompiler.php
15,000 lines
```

con todas las reglas dentro.

---

# 19. MySqlCompilationContext

```php
final readonly class MySqlCompilationContext
{
    public function __construct(
        public MySqlTarget $target,
        public MySqlCapabilitySnapshot $capabilities,
        public CompilationOptions $options,
        public SqlRenderingProfile $rendering,
        public CompilationBudget $budget,
    ) {}
}
```

---

# 20. No runtime connection

No deberá contener:

```text
PDO
mysqli
Connection
Transaction
Result
```

---

# 21. Capability model

Capabilities deberán ser typed.

Ejemplo conceptual:

```php
interface MySqlCapabilities
{
    public function supportsCte(): bool;

    public function supportsRecursiveCte(): bool;

    public function supportsWindowFunctions(): bool;

    public function supportsJson(): bool;

    public function supportsSkipLocked(): bool;

    public function supportsNowait(): bool;

    public function supportsReturning(
        MutationKind $mutation,
    ): bool;
}
```

---

# 22. Capability descriptors

Para features complejas, preferir descriptors en lugar de booleanos.

```php
final readonly class MySqlUpsertCapability
{
    public function __construct(
        public bool $supported,
        public MySqlUpsertSyntax $syntax,
        public MySqlInsertedRowReferenceMode $insertedRowReferenceMode,
    ) {}
}
```

---

# 23. Boolean explosion

Evitar:

```text
supportsX
supportsXButNotY
supportsXWithZ
supportsXOnlyWhen...
```

cuando la feature requiere un descriptor estructurado.

---

# 24. Dialect profile

```php
final readonly class MySqlDialectProfile
{
    public function __construct(
        public IdentifierQuoteStyle $identifierQuotes,
        public MySqlStringLiteralPolicy $stringLiterals,
        public MySqlBooleanRenderingPolicy $booleans,
        public MySqlPlaceholderPolicy $placeholders,
        public MySqlPaginationPolicy $pagination,
    ) {}
}
```

---

# 25. SQL modes

MySQL puede alterar comportamiento mediante SQL modes.

Por tanto el target puede necesitar un snapshot explícito:

```text
MySqlSqlModeProfile
```

---

# 26. Compiler and sql_mode

El compiler no deberá asumir silenciosamente el `sql_mode` del servidor.

---

# 27. Relevant modes

Algunas configuraciones pueden afectar:

```text
identifier quoting
string literal escaping
grouping semantics
date handling
error behavior
```

---

# 28. ANSI_QUOTES

Si el entorno usa `ANSI_QUOTES`, las reglas de quoting pueden cambiar.

Sin embargo VoltStack debería utilizar una estrategia estable y compatible con el target configurado.

---

# 29. NO_BACKSLASH_ESCAPES

La política de literals deberá considerar esta capability/configuración cuando se emitan literals string.

---

# 30. ONLY_FULL_GROUP_BY

No debe utilizarse como sustituto de Semantic Validation.

VoltStack deberá generar queries semánticamente correctas independientemente de si el servidor rechaza consultas ambiguas.

---

# 31. Strict modes

El compiler no deberá confiar en que el servidor "corrija" silenciosamente valores inválidos.

---

# 32. Identifier quoting

La forma MySQL estándar de quoting será:

```sql
`identifier`
```

---

# 33. Example

```text
Identifier:
users
```

→

```sql
`users`
```

---

# 34. Qualified identifier

```text
users.id
```

→

```sql
`users`.`id`
```

---

# 35. Alias

```sql
`users` AS `u`
```

---

# 36. Embedded backtick

Un identifier que contenga:

```text
`
```

deberá ser escapado conforme a la sintaxis MySQL.

---

# 37. No manual quoting

Nunca:

```php
'`' . $identifier . '`'
```

fuera del renderer correspondiente.

---

# 38. Wildcard

```text
QualifiedWildcard(users)
```

→

```sql
`users`.*
```

---

# 39. Identifier rendering policy

Se recomienda quoting sistemático para identifiers generados por VoltStack.

Ventajas:

```text
reserved-word safety
case predictability
deterministic output
less parser ambiguity
```

---

# 40. User raw identifiers

No serán interpolados directamente.

---

# 41. Placeholders

La representación típica del driver será:

```sql
?
```

---

# 42. Positional placeholders

Ejemplo:

```sql
SELECT `id`
FROM `users`
WHERE `email` = ?
  AND `active` = ?
```

---

# 43. Placeholder identity

Aunque SQL use `?`, VoltStack mantendrá:

```text
ParameterId
ParameterOccurrenceId
BindingSlot
```

por separado.

---

# 44. Repeated parameter

Query:

```text
x = P1 OR y = P1
```

puede producir:

```sql
`x` = ? OR `y` = ?
```

Binding layout:

```text
slot 0 → P1
slot 1 → P1
```

---

# 45. Placeholder numbering

No se deduce mediante búsqueda textual.

---

# 46. Native vs emulated prepares

El compiler no deberá cambiar la semántica dependiendo accidentalmente de si PDO usa:

```text
ATTR_EMULATE_PREPARES
```

La integración del driver definirá el contrato compatible.

---

# 47. Prepared statement boundary

El documento:

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

formalizará este comportamiento.

---

# 48. Boolean literals

MySQL acepta representaciones equivalentes a:

```sql
TRUE
FALSE
```

pero VoltStack deberá definir una representación canónica.

Una opción estable:

```sql
TRUE
FALSE
```

cuando la posición sintáctica lo permita.

---

# 49. Boolean storage ≠ boolean expression

El compiler no deberá asumir que una columna:

```text
TINYINT(1)
```

es semánticamente boolean sólo por su forma física.

Eso pertenece al Type/Schema system.

---

# 50. NULL

```text
SqlNullLiteral
```

→

```sql
NULL
```

---

# 51. String literals

Runtime strings serán placeholders.

Compile-time strings explícitos usarán el literal renderer.

---

# 52. No backslash assumptions

El renderer no deberá asumir una estrategia de backslash escaping incompatible con el SQL mode configurado.

---

# 53. Numeric literals

Deberán ser:

```text
locale-independent
```

---

# 54. SELECT básico

Emission tree:

```text
Select
├── Projection
├── From
├── Where
├── Group
├── Having
├── Window
├── Order
├── Pagination
└── Lock
```

produce SQL MySQL en el orden correspondiente.

---

# 55. Example

```sql
SELECT
    `u`.`id`,
    `u`.`email`
FROM `users` AS `u`
WHERE `u`.`active` = ?
ORDER BY `u`.`created_at` DESC
LIMIT 20
```

---

# 56. SELECT DISTINCT

```sql
SELECT DISTINCT
    `u`.`email`
FROM `users` AS `u`
```

---

# 57. Projection aliases

```sql
SELECT
    COUNT(*) AS `total`
FROM `orders`
```

Los aliases serán asignados previamente por Alias Planning.

---

# 58. FROM

Sources soportadas podrán incluir:

```text
table
derived table
CTE reference
subquery
structured extension source
```

---

# 59. Derived table

```sql
FROM (
    SELECT ...
) AS `q1`
```

---

# 60. Derived-table alias

Cuando MySQL requiera alias para una estructura, el alias planner deberá haberlo asignado.

---

# 61. JOIN

Logical/SQL join forms podrán incluir:

```text
INNER
LEFT
RIGHT
CROSS
```

y otras representaciones sólo cuando la capability correspondiente exista.

---

# 62. INNER JOIN

```sql
INNER JOIN `orders` AS `o`
    ON `o`.`user_id` = `u`.`id`
```

---

# 63. LEFT JOIN

```sql
LEFT JOIN `profiles` AS `p`
    ON `p`.`user_id` = `u`.`id`
```

---

# 64. RIGHT JOIN

Si está permitido por la representation/capability:

```sql
RIGHT JOIN ...
```

---

# 65. FULL OUTER JOIN

No deberá emitirse si el target no lo soporta.

---

# 66. No silent FULL JOIN emulation

Si VoltStack decide soportar emulación, ésta pertenecerá a:

```text
Dialect Adaptation / Rewrite
```

y no al renderer.

---

# 67. CROSS JOIN

```sql
CROSS JOIN `countries` AS `c`
```

---

# 68. Semi/anti join

Los logical `SEMI_JOIN` y `ANTI_JOIN` pueden representarse mediante construcciones como:

```text
EXISTS
NOT EXISTS
```

si el lowering/dialect adaptation lo decide.

---

# 69. Renderer does not choose EXISTS

La decisión debe estar tomada antes.

---

# 70. WHERE

```sql
WHERE `u`.`status` = ?
```

---

# 71. SQL 3VL

NULL-sensitive predicates conservarán su forma.

---

# 72. IS NULL

```sql
`u`.`deleted_at` IS NULL
```

---

# 73. IS NOT NULL

```sql
`u`.`deleted_at` IS NOT NULL
```

---

# 74. NULL-safe equality

MySQL posee operador específico:

```sql
<=>
```

Si VoltStack expone una semántica de null-safe equality y el dialect adapter decide utilizarla, se modelará estructuralmente.

---

# 75. No accidental `<=>`

El renderer no transformará automáticamente:

```text
a = b
```

en:

```text
a <=> b
```

---

# 76. LIKE

```sql
`name` LIKE ?
```

---

# 77. ESCAPE

Si la representación incluye escape:

```sql
`name` LIKE ? ESCAPE ...
```

se generará según reglas MySQL soportadas.

---

# 78. REGEXP

Si se expone como feature:

```text
semantic regex operation
→ MySQL regex representation
```

deberá pasar por capability/function/operator adaptation.

---

# 79. GROUP BY

```sql
GROUP BY
    `customer_id`,
    `status`
```

---

# 80. Semantic grouping

El compiler no confiará en extensiones permisivas históricas de MySQL para seleccionar columnas no agrupadas.

---

# 81. HAVING

```sql
HAVING COUNT(*) > ?
```

---

# 82. Aggregate functions

Built-ins comunes:

```text
COUNT
SUM
AVG
MIN
MAX
```

se mapearán mediante function descriptors.

---

# 83. Aggregate DISTINCT

```sql
COUNT(DISTINCT `user_id`)
```

---

# 84. GROUP_CONCAT

Puede exponerse mediante capability/function abstraction específica.

---

# 85. No universal function leakage

El API semántico no deberá requerir que toda aplicación conozca:

```text
GROUP_CONCAT
```

como única forma portable de agregación textual.

---

# 86. ORDER BY

```sql
ORDER BY
    `created_at` DESC,
    `id` ASC
```

---

# 87. NULL ordering

MySQL no deberá recibir automáticamente:

```sql
NULLS FIRST
```

si el target no soporta esa sintaxis.

---

# 88. NULL ordering emulation

Si se requiere:

```text
ORDER BY expression IS NULL
```

u otra estrategia, deberá construirse antes como representación SQL explícita.

---

# 89. LIMIT

Forma típica:

```sql
LIMIT ?
```

o literal estructural según el placeholder policy/capabilities.

---

# 90. LIMIT literal

Canonical compilation podrá usar un literal compile-time si el límite forma parte estructural del plan:

```sql
LIMIT 20
```

---

# 91. LIMIT parameter

Si el target/driver contract permite:

```sql
LIMIT ?
```

podrá utilizarse.

La política será explícita.

---

# 92. OFFSET

Forma canónica preferible:

```sql
LIMIT 20 OFFSET 40
```

cuando corresponda.

---

# 93. Alternative MySQL syntax

MySQL también dispone de formas equivalentes como:

```text
LIMIT offset, count
```

VoltStack deberá elegir una única representación canónica salvo necesidad específica.

---

# 94. Canonical pagination

Se recomienda:

```text
LIMIT count OFFSET offset
```

por legibilidad y similitud conceptual con otros targets.

---

# 95. OFFSET without explicit limit

Si la representación semántica requiere offset sin límite, la adaptación MySQL deberá utilizar una representación válida o rechazar la operación.

El renderer no inventará números mágicos por sí mismo.

---

# 96. ORDER BY + LIMIT

No implica deterministic result si ORDER BY no define orden total.

El compiler no añadirá automáticamente primary key.

---

# 97. Window functions

Cuando la capability lo permita:

```sql
ROW_NUMBER() OVER (
    PARTITION BY `department_id`
    ORDER BY `created_at`
)
```

---

# 98. Window capability

Debe validarse antes de rendering.

---

# 99. Named windows

Si la representación MySQL objetivo las soporta:

```sql
WINDOW `w` AS (...)
```

podrán emitirse.

---

# 100. Window frame

Ejemplo:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

---

# 101. Frame semantics

No se inferirán durante rendering.

---

# 102. CTE

Cuando esté soportado:

```sql
WITH `active_users` AS (
    SELECT ...
)
SELECT ...
FROM `active_users`
```

---

# 103. Multiple CTE

```sql
WITH
    `a` AS (...),
    `b` AS (...)
SELECT ...
```

---

# 104. Recursive CTE

```sql
WITH RECURSIVE `tree` AS (
    ...
)
SELECT ...
```

---

# 105. Structured recursion

VoltStack deberá preservar:

```text
anchor
recursive member
set operation
binding
```

desde las fases anteriores.

---

# 106. No arbitrary recursive graph

El renderer no seguirá referencias cíclicas de objetos.

---

# 107. CTE materialization

No deberá asumir que una CTE implica necesariamente una materialización física concreta.

---

# 108. Physical intent ≠ SQL guarantee

Esto es especialmente importante en MySQL, donde el motor conserva su propio optimizer.

---

# 109. Subqueries

Soporte estructural para:

```text
scalar
EXISTS
IN
derived table
correlated
```

---

# 110. Scalar subquery

```sql
(
    SELECT ...
)
```

---

# 111. Scalar cardinality semantics

El compiler no deberá convertir una scalar subquery a otra construcción que cambie:

```text
0 rows
1 row
>1 rows
```

semantics.

---

# 112. Correlated subquery

Outer references estarán resueltas antes del compiler.

---

# 113. UNION

```sql
SELECT ...
UNION
SELECT ...
```

---

# 114. UNION ALL

```sql
SELECT ...
UNION ALL
SELECT ...
```

---

# 115. INTERSECT / EXCEPT

Sólo podrán emitirse cuando el target capability profile confirme la sintaxis/semántica requerida.

---

# 116. Capability-driven set operations

No usar:

```php
if ($version >= ...) {
```

en el renderer de cada set operation.

---

# 117. Set operation grouping

Paréntesis deberán preservar el árbol semántico.

---

# 118. INSERT

Forma base:

```sql
INSERT INTO `users` (
    `name`,
    `email`
)
VALUES (?, ?)
```

---

# 119. Multi-row insert

```sql
INSERT INTO `users` (
    `name`,
    `email`
)
VALUES
    (?, ?),
    (?, ?),
    (?, ?)
```

---

# 120. Binding order

Será determinado por:

```text
SqlPlaceholderPlan
+
BindingLayout
```

---

# 121. INSERT ... SELECT

```sql
INSERT INTO `archive_users` (
    `id`,
    `email`
)
SELECT
    `id`,
    `email`
FROM `users`
WHERE ...
```

---

# 122. DEFAULT

Column defaults serán representados estructuralmente.

---

# 123. DEFAULT VALUES

Si la semántica requerida no tiene la misma sintaxis que otros motores, MySQL Dialect Adaptation producirá la forma correcta.

---

# 124. UPSERT

MySQL dispone de una familia de sintaxis basada en:

```text
INSERT ... ON DUPLICATE KEY UPDATE
```

---

# 125. Semantic Upsert ≠ MySQL syntax

VoltStack deberá modelar primero:

```text
UpsertIntent
```

y luego adaptarlo.

---

# 126. Example conceptual intent

```text
Upsert
├── target users
├── insert values
├── conflict semantics
└── update assignments
```

---

# 127. MySQL lowering

Puede convertirse a una representación equivalente a:

```sql
INSERT INTO `users` (...)
VALUES (...)
ON DUPLICATE KEY UPDATE
    ...
```

---

# 128. Conflict target semantics

Debe reconocerse que:

```text
MySQL duplicate-key semantics
≠
PostgreSQL ON CONFLICT target semantics
```

en todos los casos.

---

# 129. Portability boundary

Una API portable de upsert sólo deberá prometer el subconjunto cuya equivalencia pueda garantizarse.

---

# 130. MySQL-specific upsert API

Features específicas podrán exponerse mediante:

```text
MySqlCapabilityExtension
```

sin contaminar el core portable.

---

# 131. Inserted-row references

La sintaxis para referenciar valores propuestos/insertados deberá depender del capability descriptor del target.

---

# 132. No hard-coded historical syntax

No asumir eternamente una única forma de:

```text
VALUES(column)
```

como abstracción universal.

---

# 133. Upsert renderer

```php
interface MySqlUpsertRenderer
{
    public function render(
        MySqlUpsertClause $clause,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 134. UPDATE

Forma base:

```sql
UPDATE `users` AS `u`
SET
    `u`.`status` = ?
WHERE `u`.`id` = ?
```

La forma exacta de qualification en assignments deberá respetar la gramática MySQL target.

---

# 135. Assignment target rendering

No deberá reutilizar ciegamente `QualifiedColumnRenderer` si MySQL exige una forma sintáctica diferente en cierto contexto.

---

# 136. Syntax context matters

Por eso:

```text
Semantic column identity
≠
Rendered column syntax in every clause
```

---

# 137. UPDATE JOIN

Cuando la operación lo requiera y la capability/representation lo soporte:

```sql
UPDATE ...
JOIN ...
SET ...
WHERE ...
```

---

# 138. Portable UPDATE FROM

No deberá suponerse que una forma de otro motor existe idénticamente en MySQL.

---

# 139. UPDATE ORDER BY

Si la capability específica lo permite para la operación correspondiente, podrá renderizarse.

---

# 140. UPDATE LIMIT

Igualmente deberá estar capability-gated.

---

# 141. DELETE

Forma básica:

```sql
DELETE FROM `users`
WHERE `id` = ?
```

---

# 142. DELETE aliases

La sintaxis exacta para aliases/multi-table delete deberá estar modelada específicamente.

---

# 143. Multi-table DELETE

No será tratado como simple variación textual de:

```sql
DELETE FROM table
```

---

# 144. DELETE ORDER BY/LIMIT

Sólo cuando capability y statement shape lo permitan.

---

# 145. RETURNING

El compiler no asumirá que MySQL ofrece las mismas capacidades `RETURNING` que PostgreSQL.

---

# 146. Mutation returning capability

Debe modelarse por operación:

```text
INSERT
UPDATE
DELETE
```

y por target.

---

# 147. No fake RETURNING

Nunca:

```text
unsupported RETURNING
→ silently remove clause
```

---

# 148. Framework emulation

Si VoltStack implementa en el futuro:

```text
INSERT
→ obtain generated ID
→ SELECT row
```

eso sería:

```text
Execution Plan / Persistence strategy
```

no `SQL Generation`.

---

# 149. LAST_INSERT_ID

No será usado como sustituto semántico universal de RETURNING.

---

# 150. JSON

MySQL JSON deberá modelarse mediante capabilities y function/operator adaptation.

---

# 151. JSON extraction

Semantic:

```text
JsonExtract(document, path)
```

puede convertirse a una representación MySQL específica.

---

# 152. JSON_EXTRACT

Ejemplo target representation:

```sql
JSON_EXTRACT(`payload`, ?)
```

---

# 153. JSON unquoting

No deberá confundirse:

```text
JSON value
```

con:

```text
SQL text scalar
```

---

# 154. Arrow operators

Si se utilizan operadores específicos, deberán existir como:

```text
MySqlJsonOperatorDescriptor
```

---

# 155. JSON path

Runtime paths deberán seguir la política de parametrización compatible con la operación.

---

# 156. JSON containment

Semantic capability:

```text
JsonContains
```

podrá mapearse a una función MySQL correspondiente cuando la equivalencia sea válida.

---

# 157. JSON mutation

Funciones de:

```text
set
replace
remove
merge
```

se expondrán mediante extension/capability contracts apropiados.

---

# 158. JSON portability

No todas las funciones MySQL tendrán una abstracción portable.

---

# 159. Date/time

El compiler deberá distinguir:

```text
semantic temporal operation
```

de:

```text
MySQL temporal function
```

---

# 160. Current timestamp

Podrá renderizarse como:

```sql
CURRENT_TIMESTAMP
```

según descriptor.

---

# 161. No application clock injection

El compiler no sustituirá:

```text
CURRENT_TIMESTAMP
```

por la hora PHP.

---

# 162. Date arithmetic

Deberá usar representación MySQL adaptada.

---

# 163. INTERVAL

No será tratado como simple string concatenado.

---

# 164. Interval representation

Preferir:

```text
MySqlIntervalExpression
├── value
└── unit
```

---

# 165. Interval unit

Las unidades serán enum/descriptors controlados, no runtime strings arbitrarios.

---

# 166. String functions

Funciones como:

```text
CONCAT
LOWER
UPPER
SUBSTRING
LENGTH
```

se mapearán mediante el function registry.

---

# 167. Length semantics

Debe distinguirse:

```text
bytes
characters
```

si la abstracción semántica lo requiere.

---

# 168. Concatenation

No asumir operador:

```text
||
```

como concatenación portable en MySQL.

---

# 169. Semantic concat

```text
Concat(a,b)
```

→ representación MySQL apropiada.

---

# 170. Conditional expressions

MySQL-specific functions como:

```text
IF
IFNULL
```

no reemplazarán automáticamente semánticas portables de:

```text
CASE
COALESCE
```

sin una regla de equivalencia explícita.

---

# 171. COALESCE

Preferir SQL estándar cuando la representación y capabilities lo permitan.

---

# 172. CASE

Se renderizará mediante infraestructura común.

---

# 173. CAST

La adaptación deberá mapear tipos VoltStack a targets MySQL válidos.

---

# 174. Type rendering

```text
QueryType
        │
        ▼
Platform Type Resolution
        │
        ▼
MySqlSqlType
        │
        ▼
MySqlTypeRenderer
```

---

# 175. No direct PHP type mapping

No:

```text
PHP int
→ INT
```

dentro del renderer.

---

# 176. Collation

Si una query contiene collation explícita:

```text
CollationId
```

deberá resolverse mediante metadata/capabilities.

---

# 177. User-provided collation strings

No deberán interpolarse directamente.

---

# 178. Character set operations

Igualmente deberán estar estructuradas.

---

# 179. Full-text search

MySQL-specific full-text features podrán implementarse como extension/capability.

Ejemplo conceptual:

```text
FullTextMatch
```

→

```text
MATCH(...) AGAINST(...)
```

---

# 180. Portable search boundary

El core Database Query Engine no deberá asumir que MySQL full-text representa toda semántica de búsqueda.

---

# 181. Locking

El MySQL compiler deberá modelar locking mediante un descriptor.

---

# 182. Locking modes

Ejemplo conceptual:

```text
FOR_UPDATE
FOR_SHARE
```

según capabilities.

---

# 183. Lock modifiers

Podrán incluir:

```text
NOWAIT
SKIP_LOCKED
```

cuando estén soportados.

---

# 184. Example

```sql
SELECT ...
FROM `jobs`
WHERE ...
FOR UPDATE SKIP LOCKED
```

---

# 185. Locking semantics

No deberán reducirse a strings.

---

# 186. Locking validation

Debe considerar:

```text
statement kind
query shape
target capability
transaction requirements
```

---

# 187. Transaction requirement

El compiler puede declarar metadata como:

```text
requiresTransaction
```

pero no abrirá la transacción.

---

# 188. Locking and execution plan

La ejecución será responsable de garantizar el contexto transaccional requerido.

---

# 189. Optimizer hints

MySQL posee mecanismos de hints.

VoltStack no deberá permitir strings arbitrarios como abstracción principal.

---

# 190. Structured hints

Si se implementan:

```text
PhysicalStrategyIntent
        │
        ▼
MySqlHintAdapter
        │
        ▼
MySqlStructuredHint
        │
        ▼
Renderer
```

---

# 191. Advisory nature

Una hint puede ser:

```text
advisory
```

y no garantía de que el motor utilice exactamente el algoritmo físico deseado.

---

# 192. Engine-owned physical planning

Regla crítica:

```text
VoltStack Physical Plan
≠
MySQL Runtime Execution Plan
```

---

# 193. MySQL optimizer authority

Para una query SQL delegada:

```text
VoltStack
→ emits SQL / supported hints
→ MySQL optimizer
→ chooses actual runtime plan
```

---

# 194. No pretend HashJoin control

Si VoltStack Physical Plan tiene:

```text
PhysicalJoinStrategy = HASH
```

pero el target no permite exigirlo:

```text
Enforceability = ENGINE_DECIDED / ADVISORY
```

---

# 195. Access path hints

Lo mismo para:

```text
index preference
join order
optimizer hints
```

---

# 196. Compiler requirement

Nunca deberá generar una hint que no esté soportada por:

```text
capability snapshot
```

---

# 197. EXPLAIN

El MySQL Compiler no ejecutará:

```sql
EXPLAIN ...
```

---

# 198. EXPLAIN support

Un subsystem futuro podrá compilar:

```text
ExplainOperation
```

como operación explícita.

---

# 199. EXPLAIN telemetry

La obtención del plan real pertenece a:

```text
Diagnostics / Telemetry / Profiling
```

y requerirá I/O explícito.

---

# 200. No compile-time feedback loop

Nunca:

```text
compile
→ EXPLAIN
→ modify query
→ compile again
```

como comportamiento oculto del core compiler.

---

# 201. Generated columns

Cuando schema compilation las utilice, el MySQL platform compiler podrá renderizar expresiones compatibles.

---

# 202. Schema compiler boundary

La definición completa pertenece a:

```text
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
```

Este documento sólo define reutilización de expresiones/dialect contracts cuando aplique.

---

# 203. DDL ≠ Query SQL

No deberá mezclarse todo en un único statement renderer.

---

# 204. Query compiler vs schema compiler

```text
Query SQL Compiler
├── SELECT
├── INSERT
├── UPDATE
└── DELETE

Schema SQL Compiler
├── CREATE
├── ALTER
├── DROP
└── indexes/constraints
```

pueden compartir:

```text
identifier renderer
type renderer
literal renderer
dialect descriptors
```

---

# 205. MySQL-specific raw expressions

Sólo mediante extension API explícita.

---

# 206. Compiler security

Las reglas de seguridad del documento 68 aplican completamente.

---

# 207. Injection invariant

```text
runtime data
→ placeholders
```

---

# 208. Identifier invariant

```text
dynamic identifier
→ typed identifier
→ MySqlIdentifierRenderer
```

---

# 209. Raw SQL invariant

```text
raw SQL
→ explicit trust boundary
```

---

# 210. SQL mode security

No deberá utilizarse una estrategia de escaping que dependa de un SQL mode desconocido.

---

# 211. Prepared statement security

Binding y SQL textual permanecerán separados.

---

# 212. Source map

MySQL compilation conservará:

```text
SqlSourceMap
```

igual que el compiler común.

---

# 213. MySQL diagnostic metadata

Podrá añadir:

```text
MySqlVersion
MySqlFeatureId
MySqlDialectRuleId
MySqlCapabilityId
```

---

# 214. Error translation

Cuando MySQL reporte posición/contexto útil, VoltStack podrá correlacionarlo con source map.

---

# 215. Compiler error example

```text
MySQL SQL compilation failed.

Feature:
FULL_OUTER_JOIN

Target:
MySQL

Capability:
unsupported

Source:
LogicalJoin J14

Compilation phase:
MYSQL_DIALECT_ADAPTATION
```

---

# 216. No misleading SQL syntax error

Si VoltStack sabe que una feature no existe en el target, deberá fallar antes de enviar SQL inválido.

---

# 217. MySQL representation validation

```php
interface MySqlRepresentationValidator
{
    public function validate(
        SqlEmissionTree $tree,
        MySqlCompilationContext $context,
    ): void;
}
```

---

# 218. Validation categories

```text
syntax capability
feature compatibility
statement shape
clause compatibility
type compatibility
placeholder compatibility
locking compatibility
mutation compatibility
extension compatibility
```

---

# 219. Validation ≠ semantic query validation

No vuelve a resolver:

```text
column existence
join meaning
aggregate legality
```

salvo invariantes necesarios de representación.

---

# 220. Feature registry

```php
interface MySqlFeatureRegistry
{
    public function descriptor(
        DatabaseFeatureId $feature,
    ): MySqlFeatureDescriptor;
}
```

---

# 221. Feature descriptor

```php
final readonly class MySqlFeatureDescriptor
{
    public function __construct(
        public DatabaseFeatureId $id,
        public CapabilityStatus $status,
        public ?MySqlVersionRange $versionRange,
        public MySqlFeatureRenderingStrategy $strategy,
    ) {}
}
```

---

# 222. Capability statuses

Por ejemplo:

```text
SUPPORTED
SUPPORTED_WITH_ADAPTATION
SUPPORTED_WITH_RESTRICTIONS
UNSUPPORTED
EXTENSION_REQUIRED
```

---

# 223. Emulation classification

Puede añadirse:

```text
NATIVE
COMPILER_EMULATED
EXECUTION_EMULATED
NOT_EMULATABLE
```

---

# 224. Compiler emulation

Sólo si puede transformarse a SQL equivalente sin runtime orchestration.

---

# 225. Execution emulation

Si requiere múltiples statements o feedback runtime:

```text
ExecutionPlan
```

deberá encargarse.

---

# 226. Example

Una feature:

```text
mutation returning
```

que requiera:

```text
INSERT
+
SELECT
```

no es SQL rendering de un único statement.

---

# 227. SQL statement boundaries

El MySQL compiler deberá producir explícitamente:

```text
one statement
```

o una estructura multi-command cuando el Execution Plan lo haya autorizado.

---

# 228. No hidden multi-statements

Un query compiler no convertirá silenciosamente una operación en:

```sql
statement1;
statement2;
statement3;
```

---

# 229. Multi-statement security

Debe evitarse depender de driver multi-statements para emulaciones comunes.

---

# 230. Statement terminator

Canonical compiled SQL no necesita incluir:

```text
;
```

salvo que un target contract lo requiera.

---

# 231. Recommended

```text
no trailing semicolon
```

para prepared commands.

---

# 232. MySQL canonical rendering

Ejemplo:

```sql
SELECT `u`.`id`,`u`.`email` FROM `users` AS `u` WHERE `u`.`active`=? ORDER BY `u`.`id` ASC LIMIT 20
```

---

# 233. Pretty rendering

```sql
SELECT
    `u`.`id`,
    `u`.`email`
FROM `users` AS `u`
WHERE
    `u`.`active` = ?
ORDER BY
    `u`.`id` ASC
LIMIT 20
```

---

# 234. Canonical fingerprinting

MySQL compiled artifact fingerprint deberá incluir al menos:

```text
generic compiler version
MySQL compiler version
MySQL target capability fingerprint
SQL emission fingerprint
alias plan fingerprint
placeholder plan fingerprint
extension fingerprint
rendering canonical version
```

---

# 235. Version sensitivity

Una misma semantic query puede producir compiled artifacts diferentes para targets MySQL distintos.

---

# 236. Formula

```text
CompiledQueryKey
=
Semantic/Plan Identity
+
Target Identity
+
Capability Snapshot
+
Compiler Version
+
Extension Set
+
Specialization Key
```

---

# 237. No cache across incompatible targets

No reutilizar un compiled artifact MySQL sin verificar target compatibility.

---

# 238. Schema dependencies

Compiled artifact podrá declarar:

```text
relation dependencies
column dependencies
type dependencies
index dependencies when physically relevant
function dependencies
collation dependencies
```

---

# 239. Index dependency

Una query SQL normal no necesariamente depende de un índice para ser semánticamente válida.

---

# 240. Physical plan dependency

Pero si el compiled strategy utiliza una MySQL-specific index hint, entonces:

```text
index dependency
```

sí puede ser relevante.

---

# 241. Statistics

Cambiar estadísticas normalmente no invalida el SQL semánticamente.

Puede invalidar:

```text
physical planning decisions
advisory hints
```

según el fingerprint correspondiente.

---

# 242. Parameter specialization

Generic compilation:

```text
WHERE id = ?
```

no incluirá valor en fingerprint.

---

# 243. Specialized compilation

Si una estrategia explícitamente parameter-sensitive produce SQL diferente:

```text
PhysicalPlanSpecializationKey
```

deberá formar parte del cache key.

---

# 244. No value leakage

Los valores concretos no deberán aparecer en:

```text
generic SQL fingerprint
logs
cache keys
diagnostics
```

salvo representación segura/hashing explícito cuando sea indispensable.

---

# 245. MySQL extensions

Una extensión podrá añadir:

```text
functions
operators
expressions
clauses
hints
types
compiler adaptation rules
renderers
```

---

# 246. Extension registration

```text
bootstrap
→ discover
→ validate
→ conflict resolution
→ freeze
```

---

# 247. No runtime extension mutation

Especialmente en FrankenPHP.

---

# 248. Extension compatibility

Cada extensión deberá declarar:

```text
supported MySQL version ranges
required capabilities
conflicts
compiler version compatibility
```

---

# 249. Persistent runtime

Shared:

```text
MySqlFeatureRegistry
MySqlFunctionRegistry
MySqlOperatorRegistry
MySqlDialectProfile
MySqlRendererDescriptors
MySqlCapabilityDescriptors
```

deberán ser frozen.

---

# 250. Operation-local

```text
MySqlCompilationContext
SqlRenderingSession
SqlTextWriter
SqlSourceMapBuilder
BindingLayoutBuilder
DiagnosticCollector
```

---

# 251. Worker safety

```text
FrankenPHP Worker
│
├── Frozen MySQL Compiler Services
│
├── Request A
│   └── Compilation A
│
├── Request B
│   └── Compilation B
│
└── Job C
    └── Compilation C
```

---

# 252. No target leakage

Compilar para:

```text
MySQL Target A
```

no deberá modificar registries usados después por:

```text
MySQL Target B
```

---

# 253. Multi-database applications

Una aplicación podrá tener:

```text
mysql80
mysql84
postgres
sqlite
```

simultáneamente.

El compiler target será operation-scoped.

---

# 254. No global current dialect

Incorrecto:

```php
Dialect::setCurrent('mysql');
```

---

# 255. Compiler factory

```php
interface SqlCompilerResolver
{
    public function resolve(
        DatabaseTarget $target,
    ): SqlCompiler;
}
```

---

# 256. Resolver result

```text
MYSQL
→ MySQL compiler profile

MARIADB
→ MariaDB compiler profile

POSTGRESQL
→ PostgreSQL compiler profile

SQLITE
→ SQLite compiler profile
```

---

# 257. MySQL ≠ MariaDB

Regla crítica:

```text
MySQL
≠
MariaDB
```

aunque compartan historia y gran parte de la sintaxis.

---

# 258. No MariaDB aliasing

Nunca:

```text
if mysql or mariadb:
    use MySqlCompiler
```

como solución permanente.

---

# 259. Shared family components

Sí podrán compartir:

```text
MySqlFamilyIdentifierRenderer
MySqlFamilyCommonFunctions
MySqlFamilyCommonSyntax
```

cuando las reglas sean realmente equivalentes.

---

# 260. Divergence must be explicit

Cuando MySQL y MariaDB diverjan:

```text
separate capability
separate adapter
separate renderer
```

---

# 261. Testing architecture

El compilador MySQL deberá tener:

```text
unit tests
golden SQL tests
capability tests
version-profile tests
integration tests
real-engine tests
persistent-runtime tests
security tests
```

---

# 262. Unit tests

Cubrir:

```text
identifiers
placeholders
literals
pagination
joins
CTEs
windows
JSON
locking
upsert
mutation forms
function mapping
type rendering
```

---

# 263. Golden tests

Ejemplo:

Input:

```text
Select(users.id)
```

Expected:

```sql
SELECT `users`.`id` FROM `users`
```

según alias policy.

---

# 264. Capability negative tests

Ejemplo:

```text
FULL OUTER JOIN
```

con target sin soporte deberá fallar antes de execution.

---

# 265. Version matrix tests

Se mantendrán fixtures como:

```text
mysql-profile-A
mysql-profile-B
mysql-profile-C
```

basados en las versiones oficialmente soportadas por VoltStack.

---

# 266. Do not hardcode lifecycle here

Las versiones concretas oficialmente soportadas deberán definirse en:

```text
compatibility policy
release metadata
```

no congelarse permanentemente en este documento arquitectónico.

---

# 267. Real-engine tests

Ejecutar SQL generado contra contenedores/entornos MySQL soportados.

---

# 268. Syntax-only tests insufficient

Debe comprobarse también:

```text
result semantics
NULL behavior
locking where practical
upsert behavior
JSON behavior
pagination
window semantics
```

---

# 269. Prepared statement tests

Comprobar:

```text
placeholder count
binding order
repeated parameters
NULL bindings
boolean bindings
date/time bindings
binary bindings
JSON bindings
```

---

# 270. Injection tests

Especialmente:

```text
identifier injection
raw SQL
JSON paths
collation names
ordering identifiers
function extension arguments
```

---

# 271. SQL mode tests

Cuando un SQL mode sea parte del supported target profile, deberá incluirse en integración.

---

# 272. Persistent worker tests

Compilar repetidamente:

```text
Target A
Target B
Target A
```

en el mismo proceso.

---

# 273. Determinism tests

Mismo target + mismo input:

```text
same SQL
same source map semantics
same binding layout
same fingerprint
```

---

# 274. Cross-compiler tests

La misma semantic query deberá poder compilarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando pertenezca al portable subset.

---

# 275. Expected differences

Los tests no exigirán SQL textual idéntico.

Exigirán:

```text
equivalent observable semantics
```

---

# 276. Benchmarking

Medir:

```text
compilation latency
SQL generation latency
allocation count
peak memory
source-map overhead
prepared-statement compilation overhead
cache hit/miss
```

---

# 277. No database latency in compiler benchmark

Benchmark del compiler deberá poder ejecutarse sin MySQL activo.

---

# 278. Integration benchmark

Será separado.

---

# 279. Directory structure

```text
VoltStack/Quantum/Database/Query/Compiler/Platform/MySql
├── MySqlCompiler.php
├── MySqlCompilerFactory.php
│
├── Target
│   ├── MySqlTarget.php
│   ├── MySqlVersion.php
│   ├── MySqlVersionRange.php
│   └── MySqlTargetFingerprint.php
│
├── Capability
│   ├── MySqlCapabilitySnapshot.php
│   ├── MySqlCapabilityResolver.php
│   ├── MySqlFeatureDescriptor.php
│   ├── MySqlFeatureRegistry.php
│   ├── MySqlUpsertCapability.php
│   ├── MySqlLockingCapability.php
│   ├── MySqlJsonCapability.php
│   └── MySqlReturningCapability.php
│
├── Dialect
│   ├── MySqlDialectProfile.php
│   ├── MySqlDialectAdapter.php
│   ├── MySqlSqlModeProfile.php
│   └── MySqlDialectRenderingContract.php
│
├── Validation
│   ├── MySqlRepresentationValidator.php
│   ├── MySqlFeatureValidator.php
│   ├── MySqlStatementValidator.php
│   └── MySqlCapabilityValidator.php
│
├── Identifier
│   ├── MySqlIdentifierRenderer.php
│   └── MySqlIdentifierQuoteEscaper.php
│
├── Placeholder
│   ├── MySqlPlaceholderRenderer.php
│   └── MySqlPlaceholderPolicy.php
│
├── Literal
│   ├── MySqlLiteralRenderer.php
│   ├── MySqlBooleanRenderer.php
│   ├── MySqlStringLiteralRenderer.php
│   └── MySqlNumericLiteralRenderer.php
│
├── Statement
│   ├── MySqlSelectRenderer.php
│   ├── MySqlInsertRenderer.php
│   ├── MySqlUpdateRenderer.php
│   └── MySqlDeleteRenderer.php
│
├── Clause
│   ├── MySqlPaginationRenderer.php
│   ├── MySqlLockingRenderer.php
│   ├── MySqlUpsertRenderer.php
│   ├── MySqlCteRenderer.php
│   ├── MySqlWindowRenderer.php
│   └── MySqlSetOperationRenderer.php
│
├── Expression
│   ├── MySqlJsonRenderer.php
│   ├── MySqlIntervalRenderer.php
│   └── MySqlCastRenderer.php
│
├── Function
│   ├── MySqlFunctionRendererRegistry.php
│   ├── MySqlFunctionDescriptor.php
│   └── BuiltIn
│
├── Operator
│   ├── MySqlOperatorRegistry.php
│   └── MySqlOperatorDescriptor.php
│
├── Type
│   ├── MySqlTypeRenderer.php
│   └── MySqlTypeDescriptor.php
│
├── Hint
│   ├── MySqlHintAdapter.php
│   ├── MySqlHintRenderer.php
│   └── MySqlStructuredHint.php
│
├── Upsert
│   ├── MySqlUpsertAdapter.php
│   ├── MySqlUpsertClause.php
│   └── MySqlInsertedRowReferenceMode.php
│
├── Json
│   ├── MySqlJsonCapability.php
│   ├── MySqlJsonFunctionRegistry.php
│   └── MySqlJsonOperatorRegistry.php
│
├── Diagnostic
│   ├── MySqlCompilationDiagnostic.php
│   └── MySqlDiagnosticMetadata.php
│
├── Extension
│   ├── MySqlCompilerExtension.php
│   ├── MySqlCompilerExtensionRegistry.php
│   └── MySqlExtensionDescriptor.php
│
└── Exception
    ├── MySqlCompilationException.php
    ├── UnsupportedMySqlFeatureException.php
    ├── UnsupportedMySqlVersionException.php
    ├── MySqlCapabilityException.php
    ├── MySqlDialectAdaptationException.php
    └── MySqlRepresentationException.php
```

---

# 280. Invariantes arquitectónicos

## DB-MYSQL-001

MySQL Compiler será una especialización del SQL Compiler común.

## DB-MYSQL-002

MySQL Compiler no duplicará innecesariamente el Compiler Pipeline.

## DB-MYSQL-003

MySQL Compiler no ejecutará SQL.

## DB-MYSQL-004

MySQL Compiler no abrirá conexiones.

## DB-MYSQL-005

MySQL Compiler no ejecutará `SELECT VERSION()`.

## DB-MYSQL-006

MySQL Compiler no ejecutará `EXPLAIN`.

## DB-MYSQL-007

Target MySQL será explícito.

## DB-MYSQL-008

Versión MySQL será un value object.

## DB-MYSQL-009

Version comparison no utilizará strings ingenuos.

## DB-MYSQL-010

Version será distinta de capabilities.

## DB-MYSQL-011

Compiler consumirá capability snapshots.

## DB-MYSQL-012

Capability snapshot será immutable.

## DB-MYSQL-013

Capability resolution ocurrirá fuera del rendering.

## DB-MYSQL-014

SQL mode relevante será explícito cuando afecte compilación.

## DB-MYSQL-015

Compiler no asumirá SQL mode desconocido.

## DB-MYSQL-016

`ONLY_FULL_GROUP_BY` no sustituirá Semantic Validation.

## DB-MYSQL-017

Strict mode no será usado para reparar compiler errors.

## DB-MYSQL-018

Identifiers MySQL tendrán renderer especializado.

## DB-MYSQL-019

Identifiers no se interpolarán.

## DB-MYSQL-020

Qualified identifiers renderizarán componentes por separado.

## DB-MYSQL-021

Wildcard no será quoted como identifier normal.

## DB-MYSQL-022

Backticks serán escapados centralmente.

## DB-MYSQL-023

Aliases provendrán del Alias Plan.

## DB-MYSQL-024

Renderer no generará aliases.

## DB-MYSQL-025

Placeholders provendrán del Placeholder Plan.

## DB-MYSQL-026

Runtime values no serán interpolados.

## DB-MYSQL-027

Repeated parameters respetarán Binding Layout.

## DB-MYSQL-028

Compiler no descubrirá placeholders mediante regex.

## DB-MYSQL-029

Prepared statement mode no alterará accidentalmente query semantics.

## DB-MYSQL-030

Boolean storage no será inferido sólo desde `TINYINT(1)`.

## DB-MYSQL-031

Runtime strings usarán bindings.

## DB-MYSQL-032

String literal escaping respetará target profile.

## DB-MYSQL-033

Numeric rendering será locale-independent.

## DB-MYSQL-034

SELECT clause order será determinista.

## DB-MYSQL-035

Derived tables tendrán aliases cuando el target lo requiera.

## DB-MYSQL-036

JOIN rendering preservará join semantics.

## DB-MYSQL-037

Unsupported FULL JOIN no será emitido.

## DB-MYSQL-038

FULL JOIN emulation no ocurrirá en renderer.

## DB-MYSQL-039

SEMI/ANTI lowering no será decidido por renderer.

## DB-MYSQL-040

Predicate rendering preservará SQL 3VL.

## DB-MYSQL-041

Null-safe equality será distinta de ordinary equality.

## DB-MYSQL-042

Regex será capability-gated.

## DB-MYSQL-043

Grouping semantics no dependerán de comportamiento permisivo del servidor.

## DB-MYSQL-044

Aggregate DISTINCT será preservado.

## DB-MYSQL-045

MySQL-specific aggregate functions no contaminarán el portable core.

## DB-MYSQL-046

ORDER BY sequence será preservada.

## DB-MYSQL-047

Unsupported NULL ordering syntax no será emitida.

## DB-MYSQL-048

NULL ordering emulation ocurrirá antes del renderer.

## DB-MYSQL-049

Pagination tendrá estrategia canónica.

## DB-MYSQL-050

Renderer no inventará magic LIMIT values.

## DB-MYSQL-051

LIMIT sin ORDER BY no inventará ordering.

## DB-MYSQL-052

Compiler no añadirá primary key automáticamente a ORDER BY.

## DB-MYSQL-053

Window functions serán capability-gated.

## DB-MYSQL-054

Window frame será semánticamente resuelto antes del compiler.

## DB-MYSQL-055

CTEs serán capability-gated.

## DB-MYSQL-056

Recursive CTE será first-class.

## DB-MYSQL-057

CTE materialization no será asumida.

## DB-MYSQL-058

Subquery correlation estará resuelta previamente.

## DB-MYSQL-059

Scalar subquery cardinality semantics serán preservadas.

## DB-MYSQL-060

Set operations serán capability-gated.

## DB-MYSQL-061

Set operation grouping será preservado.

## DB-MYSQL-062

UNION y UNION ALL serán distintos.

## DB-MYSQL-063

INSERT binding order será determinista.

## DB-MYSQL-064

Multi-row INSERT no romperá parameter identity.

## DB-MYSQL-065

INSERT SELECT será estructurado.

## DB-MYSQL-066

Upsert semantic model será distinto de MySQL syntax.

## DB-MYSQL-067

MySQL duplicate-key semantics no serán consideradas idénticas a PostgreSQL conflict-target semantics.

## DB-MYSQL-068

Portable upsert sólo prometerá equivalencias demostrables.

## DB-MYSQL-069

Inserted-row references serán capability-aware.

## DB-MYSQL-070

Compiler no dependerá de sintaxis histórica deprecable como abstracción universal.

## DB-MYSQL-071

UPDATE syntax será context-aware.

## DB-MYSQL-072

Assignment target syntax no reutilizará renderers incompatibles.

## DB-MYSQL-073

UPDATE JOIN será capability/representation-gated.

## DB-MYSQL-074

UPDATE LIMIT será capability-gated.

## DB-MYSQL-075

DELETE multi-table será modelado explícitamente.

## DB-MYSQL-076

DELETE LIMIT será capability-gated.

## DB-MYSQL-077

RETURNING será capability-gated por mutation kind.

## DB-MYSQL-078

Unsupported RETURNING no será omitido.

## DB-MYSQL-079

RETURNING emulation multi-statement pertenecerá al Execution Plan.

## DB-MYSQL-080

`LAST_INSERT_ID` no será sustituto universal de RETURNING.

## DB-MYSQL-081

JSON será capability-aware.

## DB-MYSQL-082

JSON value y SQL text serán tipos semánticos distintos cuando corresponda.

## DB-MYSQL-083

JSON paths no serán interpolados inseguramente.

## DB-MYSQL-084

JSON-specific operations podrán ser extensiones.

## DB-MYSQL-085

Current database time no será sustituido por PHP time.

## DB-MYSQL-086

INTERVAL units serán structured descriptors.

## DB-MYSQL-087

String concatenation no asumirá `||`.

## DB-MYSQL-088

Function adaptation será explícita.

## DB-MYSQL-089

CAST utilizará platform type resolution.

## DB-MYSQL-090

PHP types no se convertirán directamente a SQL types dentro del renderer.

## DB-MYSQL-091

Collations serán structured metadata.

## DB-MYSQL-092

Collation names no serán interpolados.

## DB-MYSQL-093

Full-text search será capability/extension-based.

## DB-MYSQL-094

Locking será structured.

## DB-MYSQL-095

Locking modifiers serán capability-gated.

## DB-MYSQL-096

Compiler podrá declarar transaction requirements pero no abrirá transacciones.

## DB-MYSQL-097

Optimizer hints serán structured si se soportan.

## DB-MYSQL-098

User hints no podrán violar semantic/security requirements.

## DB-MYSQL-099

VoltStack Physical Plan será distinto del runtime plan real de MySQL.

## DB-MYSQL-100

MySQL conservará autoridad sobre engine-internal planning cuando la estrategia sea delegada.

## DB-MYSQL-101

Compiler no fingirá poder forzar Hash Join si el target no lo permite.

## DB-MYSQL-102

Access-path intent tendrá enforceability explícita.

## DB-MYSQL-103

Unsupported hints no serán emitidas.

## DB-MYSQL-104

EXPLAIN será operación/diagnóstico explícito, no compiler side effect.

## DB-MYSQL-105

DDL compiler será distinto del query compiler.

## DB-MYSQL-106

Query y Schema compilers podrán compartir low-level dialect services.

## DB-MYSQL-107

Raw SQL será explicit escape hatch.

## DB-MYSQL-108

SQL injection boundaries del compiler común se mantendrán.

## DB-MYSQL-109

SQL mode desconocido no permitirá escaping ambiguo.

## DB-MYSQL-110

SQL y bindings permanecerán separados.

## DB-MYSQL-111

Source map se conservará.

## DB-MYSQL-112

Diagnostics incluirán target/capability cuando sea útil.

## DB-MYSQL-113

Known unsupported features fallarán antes de execution.

## DB-MYSQL-114

Representation validation será distinta de semantic validation.

## DB-MYSQL-115

Feature descriptors serán typed.

## DB-MYSQL-116

Capabilities complejas usarán descriptors, no boolean explosion.

## DB-MYSQL-117

Emulation tendrá clasificación explícita.

## DB-MYSQL-118

Compiler emulation no requerirá runtime I/O.

## DB-MYSQL-119

Execution emulation pertenecerá al Execution Plan.

## DB-MYSQL-120

Compiler no introducirá hidden multi-statements.

## DB-MYSQL-121

Prepared SQL no requerirá trailing semicolon.

## DB-MYSQL-122

Canonical rendering será determinista.

## DB-MYSQL-123

Target identity participará en compiled fingerprint.

## DB-MYSQL-124

Capability snapshot participará en compiled fingerprint.

## DB-MYSQL-125

Artifacts no se compartirán entre targets incompatibles.

## DB-MYSQL-126

Index dependencies sólo se incluirán cuando sean realmente relevantes.

## DB-MYSQL-127

Statistics no alterarán semantic SQL por sí mismas.

## DB-MYSQL-128

Physical hint decisions podrán depender de planning snapshots explícitos.

## DB-MYSQL-129

Generic plan fingerprints no incluirán runtime values.

## DB-MYSQL-130

Parameter-sensitive specialization será explícita.

## DB-MYSQL-131

Extensions declararán compatibilidad.

## DB-MYSQL-132

Extension registry será frozen.

## DB-MYSQL-133

No habrá last-write-wins silencioso.

## DB-MYSQL-134

Compiler shared services serán immutable.

## DB-MYSQL-135

Compilation state será operation-scoped.

## DB-MYSQL-136

No habrá leakage entre persistent requests.

## DB-MYSQL-137

Multi-target applications estarán soportadas.

## DB-MYSQL-138

No existirá global mutable `current dialect`.

## DB-MYSQL-139

MySQL y MariaDB serán plataformas distintas.

## DB-MYSQL-140

Shared MySQL-family components sólo se usarán para comportamiento realmente común.

## DB-MYSQL-141

Divergencias MySQL/MariaDB serán explícitas.

## DB-MYSQL-142

Testing incluirá capability-negative cases.

## DB-MYSQL-143

Testing incluirá version profiles.

## DB-MYSQL-144

Testing incluirá real MySQL execution.

## DB-MYSQL-145

Testing incluirá prepared statements.

## DB-MYSQL-146

Testing incluirá SQL mode profiles soportados.

## DB-MYSQL-147

Testing incluirá injection attempts.

## DB-MYSQL-148

Testing incluirá persistent worker isolation.

## DB-MYSQL-149

Testing incluirá deterministic compilation.

## DB-MYSQL-150

Portable queries serán verificadas cross-platform cuando sea posible.

## DB-MYSQL-151

Compiler benchmarks no requerirán servidor MySQL.

## DB-MYSQL-152

Integration benchmarks serán separados.

## DB-MYSQL-153

Renderer no reparará semántica inválida.

## DB-MYSQL-154

Compiler no utilizará server permissiveness como feature.

## DB-MYSQL-155

Compiler no dependerá de undocumented server behavior.

## DB-MYSQL-156

Compiler no convertirá warnings del servidor en parte de su estrategia normal.

## DB-MYSQL-157

Toda sintaxis MySQL-specific deberá tener ownership arquitectónico explícito.

## DB-MYSQL-158

Toda capability seleccionada deberá ser verificable contra el target snapshot.

## DB-MYSQL-159

Toda divergencia de versión deberá resolverse mediante capability/dialect descriptors.

## DB-MYSQL-160

MySQL Compiler preservará el significado observable de la operación compilada.

---

# 281. Anti-patrones

## 281.1 `if ($driver === 'mysql')` disperso

Incorrecto:

```php
if ($connection->driver() === 'mysql') {
    // special SQL
}
```

en Query Builder, ORM o Executor.

---

## 281.2 MySQL == MariaDB

Incorrecto:

```text
mysql/mariadb
→ same compiler forever
```

---

## 281.3 Compiler consultando versión

Incorrecto:

```php
$version = $connection->query('SELECT VERSION()');
```

---

## 281.4 Compiler usando EXPLAIN

Incorrecto:

```text
compile
→ EXPLAIN
→ inspect
→ change SQL
```

---

## 281.5 Values concatenados

Incorrecto:

```php
"WHERE email = '$email'"
```

---

## 281.6 Identifiers concatenados

Incorrecto:

```php
"ORDER BY `$column`"
```

---

## 281.7 Upsert universalizado

Incorrecto:

```text
MySQL ON DUPLICATE KEY
=
PostgreSQL ON CONFLICT
=
SQLite UPSERT
```

sin analizar diferencias.

---

## 281.8 Fake RETURNING

Incorrecto:

```text
RETURNING unsupported
→ remove clause
```

---

## 281.9 Multi-statement hidden emulation

Incorrecto:

```text
one semantic command
→ compiler silently emits three SQL statements
```

---

## 281.10 Assuming MySQL runtime plan

Incorrecto:

```text
PhysicalHashJoin
→ MySQL definitely uses hash join
```

---

## 281.11 SQL-mode assumptions

Incorrecto:

```text
string escaping
→ assume backslash behavior
```

sin target profile.

---

## 281.12 Version conditionals everywhere

Incorrecto:

```php
if ($version->major === 8) { ... }
```

disperso por decenas de renderers.

---

# 282. Ejemplo completo

Consulta conceptual:

```php
$query
    ->from('users', 'u')
    ->select('u.id', 'u.email')
    ->where('u.status', '=', $status)
    ->whereNull('u.deleted_at')
    ->orderBy('u.created_at', 'desc')
    ->limit(25);
```

---

# 283. Semantic representation

```text
SelectQuery
├── Relation users [R1]
├── Projection
│   ├── users.id
│   └── users.email
├── Predicate
│   ├── users.status = P1
│   └── users.deleted_at IS NULL
├── Ordering
│   └── users.created_at DESC
└── Limit 25
```

---

# 284. Logical plan

```text
Limit(25)
└── Sort(created_at DESC)
    └── Project(id,email)
        └── Filter(
            status = P1
            AND deleted_at IS NULL
        )
            └── Scan(users)
```

---

# 285. Physical planning

Para un SQL pushdown target:

```text
DatabasePushdownRegion
└── ENGINE_DECIDED relational execution
```

El planner puede conservar access-path preferences, pero no fingirá controlar internals no enforceables.

---

# 286. Execution plan

```text
DatabaseStatementExecution
├── target: mysql
├── command: C1
├── binding requirements
└── result contract
```

---

# 287. MySQL emission representation

```text
MySqlSelectStatement
├── projection
│   ├── Column(R1,id)
│   └── Column(R1,email)
├── from
│   └── Table(users,R1)
├── where
│   └── AND
│       ├── Eq(Column(R1,status),Parameter(P1))
│       └── IsNull(Column(R1,deleted_at))
├── order
│   └── Column(R1,created_at) DESC
└── limit
    └── 25
```

---

# 288. Alias plan

```text
R1 → u
```

---

# 289. Placeholder plan

```text
P1 occurrence 1
→ ?
→ binding slot 0
```

---

# 290. Generated SQL

Canonical:

```sql
SELECT `u`.`id`,`u`.`email` FROM `users` AS `u` WHERE `u`.`status`=? AND `u`.`deleted_at` IS NULL ORDER BY `u`.`created_at` DESC LIMIT 25
```

---

# 291. Pretty SQL

```sql
SELECT
    `u`.`id`,
    `u`.`email`
FROM `users` AS `u`
WHERE
    `u`.`status` = ?
    AND `u`.`deleted_at` IS NULL
ORDER BY
    `u`.`created_at` DESC
LIMIT 25
```

---

# 292. Binding layout

```text
slot 0
├── ParameterId: P1
├── QueryType: STRING
├── nullable: false
└── runtime value: supplied later
```

---

# 293. Runtime

Sólo entonces:

```text
P1 → "active"
```

---

# 294. Final flow

```text
Developer Query
        │
        ▼
Query Builder
        │
        ▼
Query AST
        │
        ▼
Semantic Engine
        │
        ▼
Optimizer
        │
        ▼
Logical Plan
        │
        ▼
Physical Plan
        │
        ▼
Execution Plan
        │
        ▼
SQL Compiler
        │
        ▼
MySQL Dialect Adapter
        │
        ▼
MySQL SqlEmissionTree
        │
        ├── capability validation
        ├── aliases
        ├── placeholders
        ├── MySQL syntax
        ├── MySQL functions
        ├── MySQL operators
        ├── pagination
        ├── locking
        ├── CTE
        ├── windows
        ├── JSON
        └── upsert
        │
        ▼
MySQL SQL Generator
        │
        ▼
CompiledDatabaseCommand
        │
        ▼
Executor
        │
        ▼
MySQL Connection
        │
        ▼
MySQL Server
```

---

# 295. Arquitectura de compatibilidad

```text
                    Common SQL Compiler
                           │
                           ▼
                  Platform Compiler API
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
           MySQL         MariaDB      PostgreSQL ...
             │
             ▼
       MySqlTarget
             │
      ┌──────┼─────────┐
      ▼      ▼         ▼
 Version  Capabilities SQL Modes
      │      │         │
      └──────┼─────────┘
             ▼
      Dialect Adaptation
             │
             ▼
      MySQL Representation
             │
             ▼
        SQL Generation
```

---

# 296. Principio sobre portabilidad

VoltStack no buscará portabilidad mediante el mínimo común denominador absoluto.

Buscará:

```text
Portable Semantic Core
+
Explicit Platform Capabilities
+
Safe Platform Extensions
```

---

# 297. Ejemplo

Portable:

```php
$query->limit(20);
```

Platform capability:

```php
$query->forUpdate();
```

MySQL-specific extension:

```text
MySQL optimizer/full-text/special JSON feature
```

cuando no exista semántica portable suficiente.

---

# 298. Regla sobre fallback

Cuando una feature no esté soportada:

```text
unsupported
        │
        ├── equivalent compiler adaptation exists
        │       ↓
        │    adapt
        │
        ├── explicit execution emulation exists
        │       ↓
        │    Execution Plan
        │
        └── neither
                ↓
             fail
```

Nunca:

```text
unsupported
→ approximate silently
```

---

# 299. Regla sobre MySQL optimizer

El compiler deberá respetar:

```text
VoltStack decides:
    query semantics
    SQL representation
    enforceable requirements
    supported hints

MySQL decides:
    actual internal runtime plan
```

cuando la ejecución haya sido delegada al motor.

---

# 300. Resultado arquitectónico

El compilador MySQL queda definido como:

```text
MySqlSqlCompiler
=
GenericCompilerPipeline
+
MySqlTargetModel
+
MySqlCapabilitySystem
+
MySqlDialectAdapter
+
MySqlRepresentationValidation
+
MySqlIdentifierRules
+
MySqlPlaceholderRules
+
MySqlFunctionAndOperatorMapping
+
MySqlMutationRules
+
MySqlPagination
+
MySqlLocking
+
MySqlCteAndWindowSupport
+
MySqlJsonSupport
+
MySqlUpsertSupport
+
MySqlExtensionModel
+
DeterministicGeneration
```

---

# 301. Principio final

La regla más importante será:

```text
MySQL is a compilation target,
not a special case scattered through VoltStack.
```

Por tanto:

```text
Query Builder
does not know MySQL

ORM
does not know MySQL syntax

Optimizer
does not generate MySQL SQL

Physical Planner
does not assume MySQL obeys every strategy

Executor
does not build SQL

MySQL Compiler
does not execute SQL
```

---

# 302. Fórmula final

```text
CorrectMySqlCompilation
=
PortableSemanticMeaning
+
ExplicitTarget
+
CapabilityAwareAdaptation
+
VersionAwareDescriptors
+
StructuredMySqlRepresentation
+
SafeIdentifierRendering
+
PlannedParameterization
+
DeterministicSqlGeneration
+
ExplicitFeatureFailure
+
SourceMapping
+
PersistentRuntimeIsolation
```

Y:

```text
CompilableDatabaseOperation
        │
        ▼
Common SQL Compiler
        │
        ▼
MySqlTarget
        │
        ├── Version
        ├── Capabilities
        ├── SQL Mode Profile
        └── Extensions
        │
        ▼
MySQL Dialect Adaptation
        │
        ▼
Validated MySQL SqlEmissionTree
        │
        ▼
Alias + Placeholder Planning
        │
        ▼
MySQL SQL Generation
        │
        ▼
RenderedSql
+
BindingLayout
+
ResultContract
+
SourceMap
+
Dependencies
        │
        ▼
MySqlCompiledDatabaseCommand
```

> En VoltStack, MySQL no será tratado como un conjunto de `if ($driver === 'mysql')`. Será un target formal de compilación con versión, capabilities, dialecto, reglas de representación y extensiones explícitas. Esto permitirá mantener un Query Engine independiente del proveedor sin renunciar a utilizar de forma segura las capacidades específicas de MySQL.

---

# 303. Siguiente documento

```text
70_DATABASE_MARIADB_SQL_COMPILER.md
```

El siguiente documento deberá formalizar especialmente la frontera:

```text
MySQL
≠
MariaDB
```

definiendo qué infraestructura puede compartirse mediante una familia común y qué diferencias deberán permanecer aisladas mediante capabilities, adapters y renderers propios.