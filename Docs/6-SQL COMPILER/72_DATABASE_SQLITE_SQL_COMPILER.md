# 72_DATABASE_SQLITE_SQL_COMPILER.md

# VoltStack Quantum Database
## SQLite SQL Compiler

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 72 — SQLite SQL Compiler  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`SQLite SQL Compiler` define la especialización del SQL Compiler de VoltStack responsable de transformar operaciones compilables en representaciones SQL válidas para un target SQLite concreto.

SQLite deberá tratarse como:

```text
First-Class Database Platform
```

y no como:

```text
Testing-Only Database
```

El compiler deberá comprender las particularidades representacionales de SQLite relacionadas con:

- dynamic typing;
- storage classes;
- type affinity;
- STRICT tables;
- parameter placeholders;
- `ROWID`;
- `WITHOUT ROWID`;
- `RETURNING`;
- UPSERT;
- `ON CONFLICT`;
- `INSERT OR ...`;
- CTE;
- recursive CTE;
- window functions;
- aggregate `FILTER`;
- generated columns;
- JSON;
- collations;
- date/time representations;
- boolean representations;
- `LIMIT`/`OFFSET`;
- compound queries;
- capabilities dependientes de versión;
- funciones integradas;
- extensiones SQLite;
- restricciones propias de DML y SQL grammar.

Principio central:

```text
SQLiteCompiler
=
Common SQL Compiler
+
SQLite Target
+
SQLite Capability Model
+
SQLite Dialect Adaptation
+
SQLite Rendering Rules
```

Nunca:

```text
SQLiteCompiler
=
SQLite Emulator
+
ORM
+
Migration Planner
+
Execution Engine
```

---

# 2. Objetivo arquitectónico

El objetivo no será hacer que SQLite parezca MySQL o PostgreSQL.

Será:

```text
Preserve VoltStack semantics
        ↓
Determine SQLite representability
        ↓
Use native SQLite representation
        ↓
Fail explicitly when semantics cannot be preserved
```

Por tanto:

```text
Portability
≠
Pretending all databases are identical
```

---

# 3. SQLite como plataforma de primera clase

VoltStack deberá soportar SQLite para escenarios como:

```text
local applications
embedded applications
development
testing
CLI applications
desktop-oriented workloads
single-node services
small services
temporary databases
in-memory databases
edge deployments
application-local persistence
```

Pero el diseño no asumirá que SQLite existe exclusivamente para tests.

---

# 4. Posición arquitectónica

```text
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
Logical Query Plan
    │
    ▼
Physical Query Plan
    │
    ▼
Execution Plan
    │
    ▼
SQL Compiler
    │
    ├── MySQL
    ├── MariaDB
    ├── PostgreSQL
    └── SQLite
          │
          ▼
SQLiteCompiledDatabaseCommand
          │
          ▼
Execution Engine
          │
          ▼
SQLite Driver
          │
          ▼
SQLite
```

---

# 5. Frontera fundamental

```text
SQLite Compiler
≠
SQLite Driver
≠
SQLite Connection
≠
SQLite Transaction Manager
≠
SQLite Schema Migration Planner
≠
SQLite Native Query Planner
```

---

# 6. Responsabilidades

El SQLite Compiler deberá:

1. validar el target SQLite;
2. consumir capabilities explícitas;
3. adaptar SQL común a SQLite;
4. representar tipos correctamente;
5. modelar type affinity;
6. respetar STRICT mode cuando sea conocido;
7. compilar parámetros;
8. generar aliases deterministas;
9. compilar expressions;
10. compilar predicates;
11. compilar SELECT;
12. compilar INSERT;
13. compilar UPDATE;
14. compilar DELETE;
15. compilar UPSERT;
16. compilar RETURNING;
17. compilar CTE;
18. compilar window functions;
19. compilar compound queries;
20. adaptar JSON;
21. preservar result contracts;
22. generar binding layouts;
23. generar source maps;
24. generar dependencies;
25. generar fingerprints.

---

# 7. No responsabilidades

El compiler no deberá:

```text
open SQLite files
create database files
query sqlite_master
query sqlite_schema
execute PRAGMA
change journal mode
change WAL mode
set busy timeout
begin transactions
commit transactions
rollback transactions
execute SQL
hydrate entities
manage UnitOfWork
choose indexes
run EXPLAIN QUERY PLAN
discover runtime extensions
```

---

# 8. SQLite target

```php
final readonly class SQLiteTarget
{
    public function __construct(
        public SQLiteVersion $version,
        public SQLiteCapabilitySnapshot $capabilities,
        public SQLiteDialectProfile $dialect,
        public SQLiteCompilationProfile $compilation,
    ) {}
}
```

---

# 9. SQLite version

```php
final readonly class SQLiteVersion
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
    ) {}
}
```

---

# 10. Version ≠ capability

Nunca:

```php
if ($version >= '3.35') {
    // assume every desired feature
}
```

Preferir:

```php
if ($capabilities->supports(
    SQLiteFeature::RETURNING
)) {
}
```

---

# 11. Capability snapshot

```php
final readonly class SQLiteCapabilitySnapshot
{
    public function __construct(
        public SQLiteVersion $version,
        public SQLiteFeatureSet $features,
        public SQLiteExtensionCapabilitySet $extensions,
        public CapabilityFingerprint $fingerprint,
    ) {}
}
```

---

# 12. Capability examples

El modelo podrá representar:

```text
supportsReturning()
supportsUpsert()
supportsWindowFunctions()
supportsAggregateFilter()
supportsRecursiveCte()
supportsGeneratedColumns()
supportsStrictTables()
supportsJson()
supportsJsonOperators()
supportsUpdateFrom()
supportsExpressionIndexes()
supportsPartialIndexes()
supportsWithoutRowId()
supportsRightJoin()
supportsFullJoin()
supportsDeleteLimit()
supportsUpdateLimit()
```

Las últimas capacidades deberán reflejar el build/target real cuando correspondan.

---

# 13. Compile-time options

SQLite puede ser construido con distintas opciones.

Por tanto:

```text
SQLite Version
≠
Complete Runtime Capability Set
```

---

# 14. Capability discovery

La discovery podrá ocurrir fuera del compiler mediante:

```text
SQLiteConnectionDiscovery
        ↓
SQLiteVersionDiscovery
        ↓
SQLiteCompileOptionDiscovery
        ↓
SQLiteExtensionDiscovery
        ↓
SQLiteCapabilityResolver
        ↓
SQLiteCapabilitySnapshot
```

---

# 15. No discovery durante compilation

El compiler no ejecutará:

```sql
SELECT sqlite_version();
```

ni:

```sql
PRAGMA compile_options;
```

ni consultas sobre:

```text
sqlite_schema
sqlite_master
```

durante `compile()`.

---

# 16. SQLite dialect profile

```php
final readonly class SQLiteDialectProfile
{
    public function __construct(
        public SQLiteIdentifierPolicy $identifiers,
        public SQLitePlaceholderPolicy $placeholders,
        public SQLiteTypeProfile $types,
        public SQLiteFunctionProfile $functions,
        public SQLiteOperatorProfile $operators,
    ) {}
}
```

---

# 17. Compilation context

```php
final readonly class SQLiteCompilationContext
{
    public function __construct(
        public SQLiteTarget $target,
        public CompilationOptions $options,
        public CompilationBudget $budget,
        public SqlRenderingProfile $rendering,
    ) {}
}
```

---

# 18. No live connection

El context no contendrá:

```text
PDO
PDOStatement
SQLite3
Connection
Transaction
Cursor
ServiceContainer
EntityManager
UnitOfWork
```

---

# 19. Compiler contract

```php
interface SQLiteCompiler
{
    public function compile(
        CompilableDatabaseOperation $operation,
        SQLiteCompilationContext $context,
    ): SQLiteCompiledDatabaseCommand;
}
```

---

# 20. Pipeline

```text
CompilableDatabaseOperation
        │
        ▼
Input Validation
        │
        ▼
SQLite Capability Preflight
        │
        ▼
Generic SQL Lowering
        │
        ▼
SQLite Dialect Adaptation
        │
        ▼
SQLite Representation Validation
        │
        ▼
Identifier/Alias Planning
        │
        ▼
Parameter Planning
        │
        ▼
Type/Affinity Planning
        │
        ▼
SQLite SQL Generation
        │
        ▼
Binding Layout Compilation
        │
        ▼
Result Contract Compilation
        │
        ▼
Dependency Collection
        │
        ▼
Source Map Finalization
        │
        ▼
Fingerprint
        │
        ▼
SQLiteCompiledDatabaseCommand
```

---

# 21. SQLite dialect adapter

```php
interface SQLiteDialectAdapter
{
    public function adapt(
        SqlEmissionTree $tree,
        SQLiteCompilationContext $context,
    ): SQLiteEmissionTree;
}
```

---

# 22. Dialect adaptation ≠ semantic rewrite

El adapter podrá cambiar:

```text
representation
syntax
type spelling
placeholder representation
supported SQL construct
```

pero no:

```text
meaning
query cardinality
join semantics
NULL semantics
mutation set
observable ordering
security predicates
```

---

# 23. SQLite representation validator

```php
interface SQLiteRepresentationValidator
{
    public function validate(
        SQLiteEmissionTree $tree,
        SQLiteCompilationContext $context,
    ): void;
}
```

---

# 24. Validation principle

```text
Semantic validity
    established earlier

SQLite representability
    validated here
```

---

# 25. Identifier quoting

La representación canónica de VoltStack para SQLite podrá utilizar:

```sql
"identifier"
```

---

# 26. Qualified identifier

```sql
"users"."id"
```

---

# 27. Alias

```sql
"users" AS "u"
```

---

# 28. Identifier safety

Nunca:

```php
$sql .= '"' . $userInput . '"';
```

---

# 29. Identifier compiler

```text
Structured Identifier
        ↓
SQLiteIdentifierCompiler
        ↓
Validated SQLite Identifier
        ↓
Renderer
```

---

# 30. SQLite parameters

SQLite soporta diferentes formas de parámetros.

Conceptualmente:

```text
?
?NNN
:name
@name
$name
```

VoltStack no deberá acoplar su Query Model a ninguna de ellas.

---

# 31. Canonical placeholder policy

El profile podrá elegir una representación canónica.

Por ejemplo:

```sql
?1
?2
?3
```

---

# 32. Parameter identities

```text
ParameterId
≠
ParameterOccurrenceId
≠
ExecutionParameterSlotId
≠
SQLitePlaceholder
≠
DriverBindingPosition
```

---

# 33. Example

Input:

```text
email = P1
AND
active = P2
```

Output conceptual:

```sql
"email" = ?1
AND "active" = ?2
```

---

# 34. Repeated parameters

Input:

```text
"x" = P1
OR
"y" = P1
```

podrá representarse como:

```sql
"x" = ?1
OR "y" = ?1
```

cuando el binding profile lo permita.

---

# 35. Parameter allocation

Será operation-scoped.

Nunca:

```php
static int $parameter = 0;
```

---

# 36. Dynamic typing

SQLite posee un modelo de tipos significativamente diferente de PostgreSQL/MySQL.

Por tanto:

```text
SQLite Type System
≠
Traditional Static SQL Type System
```

---

# 37. Storage classes

El compiler/type adapter deberá conocer conceptualmente las storage classes:

```text
NULL
INTEGER
REAL
TEXT
BLOB
```

sin confundirlas con declared column types.

---

# 38. Storage class ≠ declared type

```text
Storage Class
≠
Declared Type
≠
Type Affinity
≠
VoltStack QueryType
```

---

# 39. Type affinity

El sistema deberá modelar affinities como:

```text
INTEGER
TEXT
BLOB
REAL
NUMERIC
```

cuando sean relevantes para representación o conversiones.

---

# 40. SQLite type descriptor

```php
final readonly class SQLiteTypeDescriptor
{
    public function __construct(
        public SQLiteDeclaredType $declaredType,
        public SQLiteTypeAffinity $affinity,
        public SQLiteStorageExpectation $storage,
        public QueryType $semanticType,
    ) {}
}
```

---

# 41. QueryType ≠ SQLite affinity

Por ejemplo:

```text
QueryType::Boolean
```

podrá representarse físicamente mediante INTEGER semantics.

Esto no significa:

```text
Boolean
=
Integer
```

en el modelo semántico de VoltStack.

---

# 42. Type mapping boundary

```text
VoltStack QueryType
        ↓
SQLiteTypeResolver
        ↓
SQLiteTypeDescriptor
        ↓
SQLite SQL Representation
```

---

# 43. Type affinity must not leak upward

ORM y Query Builder no deberán razonar directamente sobre:

```text
INTEGER affinity
TEXT affinity
NUMERIC affinity
```

---

# 44. STRICT tables

SQLite targets modernos pueden soportar:

```sql
CREATE TABLE ... STRICT
```

El capability model deberá representarlo.

---

# 45. STRICT table metadata

Cuando una operación depende de la naturaleza STRICT de una tabla, esa información deberá llegar mediante metadata/snapshot explícito.

---

# 46. Compiler does not inspect table mode

Nunca:

```text
compile()
→ query sqlite_schema
→ detect STRICT
```

---

# 47. STRICT typing semantics

La existencia de STRICT tables no convierte globalmente a SQLite en un sistema de tipos equivalente a PostgreSQL.

---

# 48. STRICT awareness

```text
SQLiteTableTypePolicy
├── DYNAMIC
└── STRICT
```

podrá formar parte de metadata relevante.

---

# 49. Boolean representation

SQLite no deberá forzar al Query Model a utilizar integers para booleanos.

El semantic model continuará usando:

```text
Boolean
```

---

# 50. Boolean adaptation

Una representación podrá usar:

```sql
1
```

y:

```sql
0
```

para structural boolean literals cuando el dialect profile lo determine.

---

# 51. Boolean parameter

Un runtime boolean deberá pasar por:

```text
Boolean QueryType
        ↓
SQLite Binding Conversion
        ↓
Driver-compatible representation
```

---

# 52. TRUE/FALSE keywords

Si el target/profile utiliza keywords compatibles:

```sql
TRUE
FALSE
```

esto será una decisión representacional.

---

# 53. SQL NULL

`NULL` continuará preservando semántica SQL.

---

# 54. Three-valued logic

```text
TRUE
FALSE
UNKNOWN
```

deberán mantenerse.

---

# 55. NULL comparisons

Nunca:

```sql
"x" = NULL
```

cuando la operación semántica sea:

```text
IS NULL
```

---

# 56. Date/time model

SQLite no posee el mismo modelo de tipos temporales nativos que PostgreSQL.

VoltStack deberá mantener una separación entre:

```text
Semantic Date/Time Type
```

y:

```text
SQLite Storage Representation
```

---

# 57. Possible date/time representations

Según configuration/type mapping:

```text
TEXT
INTEGER
REAL
```

podrán utilizarse.

---

# 58. Date/time storage policy

```php
enum SQLiteDateTimeStoragePolicy
{
    case ISO8601_TEXT;
    case UNIX_SECONDS_INTEGER;
    case UNIX_MILLISECONDS_INTEGER;
    case JULIAN_DAY_REAL;
}
```

La lista final dependerá de la estrategia adoptada.

---

# 59. Storage policy explicitness

La representación temporal deberá ser explícita y fingerprinted cuando afecte SQL/bindings/result conversion.

---

# 60. No hidden application timezone

El compiler no utilizará:

```text
date_default_timezone_get()
```

para alterar la semántica de una consulta.

---

# 61. Database current time

Expresiones como:

```text
CurrentTimestamp
```

se compilarán mediante descriptors de función/keyword SQLite.

No mediante:

```php
new DateTimeImmutable()
```

---

# 62. Date/time functions

Funciones SQLite como:

```text
date
time
datetime
julianday
unixepoch
strftime
```

deberán resolverse mediante function descriptors cuando se expongan.

---

# 63. Function registry

```php
interface SQLiteFunctionRegistry
{
    public function resolve(
        SemanticFunctionId $function,
        SQLiteTarget $target,
    ): SQLiteFunctionDescriptor;
}
```

---

# 64. Built-in function ≠ extension function

```text
SQLite Built-in Function
≠
Application Registered Function
≠
Extension Function
```

---

# 65. Runtime user-defined functions

El compiler no deberá asumir que una función registrada en una conexión existe globalmente.

---

# 66. Function capability dependency

Si una consulta usa una función runtime específica, el compiled command deberá depender explícitamente de esa capability/profile.

---

# 67. SELECT

Ejemplo:

```sql
SELECT
    "u"."id",
    "u"."email"
FROM "users" AS "u"
WHERE "u"."active" = ?1
ORDER BY "u"."id" ASC
LIMIT 20
```

---

# 68. SELECT ALL

La ausencia de DISTINCT conservará bag semantics.

---

# 69. DISTINCT

```sql
SELECT DISTINCT
    "country"
FROM "users"
```

---

# 70. SQLite DISTINCT semantics

El compiler no deberá sustituir las reglas de comparación de SQLite por igualdad PHP.

---

# 71. FROM

Podrá representar:

```text
table
view
subquery
CTE
VALUES where grammar allows
table-valued function where available
extension relation
```

---

# 72. JOIN

El compiler deberá soportar únicamente las join forms que el target capability declare válidas.

---

# 73. Join types

Podrán incluir:

```text
INNER
LEFT
CROSS
RIGHT
FULL
```

según target/version/capabilities.

---

# 74. No historical assumptions

No se codificará permanentemente:

```text
SQLite never supports RIGHT JOIN
```

o:

```text
SQLite never supports FULL JOIN
```

La decisión será capability-driven.

---

# 75. Join optimizer boundary

El compiler no seleccionará:

```text
join order
join algorithm
index
loop nesting
```

---

# 76. Native SQLite planner

SQLite podrá decidir internamente su estrategia de ejecución.

```text
VoltStack Physical Intent
        ↓
SQLite SQL
        ↓
SQLite Query Planner
```

---

# 77. WHERE

```sql
WHERE "status" = ?1
```

---

# 78. Predicate preservation

El compiler preservará:

```text
AND
OR
NOT
comparison
NULL tests
IN
EXISTS
BETWEEN
LIKE
```

según semantic descriptors.

---

# 79. Operator precedence

La precedencia será descriptor-driven.

No dependerá de concatenación arbitraria.

---

# 80. Parentheses

El renderer añadirá paréntesis cuando la estructura lo requiera.

---

# 81. LIKE

`LIKE` deberá conservar su semántica target-specific respecto a collation/case behavior.

---

# 82. Case-insensitive assumptions

El compiler no deberá asumir que:

```text
LIKE
=
portable case-insensitive comparison
```

---

# 83. GLOB

Si se expone:

```text
GLOB
```

será SQLite-specific.

---

# 84. REGEXP

SQLite no deberá asumir automáticamente disponibilidad de `REGEXP` sólo porque la gramática permita un operator hook.

Su disponibilidad deberá provenir del capability/extension model.

---

# 85. GROUP BY

```sql
GROUP BY
    "department_id"
```

---

# 86. HAVING

```sql
HAVING COUNT(*) > ?1
```

---

# 87. Aggregate FILTER

Cuando el capability lo permita:

```sql
COUNT(*) FILTER (
    WHERE "active" = 1
)
```

---

# 88. Aggregate filter structured model

```text
AggregateCall
├── function
├── arguments
├── distinct
├── filter
└── internal ordering if supported
```

---

# 89. Window functions

Cuando sean soportadas:

```sql
ROW_NUMBER() OVER (
    PARTITION BY "department_id"
    ORDER BY "created_at" DESC
)
```

---

# 90. Window model

Deberá preservar:

```text
partition
ordering
frame
boundaries
exclusion
function
```

según capability.

---

# 91. Window ordering ≠ query ordering

```text
Window Ordering
≠
Final Result Ordering
```

---

# 92. Named windows

Cuando sean representables:

```sql
WINDOW "w" AS (
    PARTITION BY "department_id"
    ORDER BY "created_at"
)
```

---

# 93. ORDER BY

```sql
ORDER BY
    "created_at" DESC,
    "id" ASC
```

---

# 94. NULL ordering

El compiler deberá utilizar la representación SQLite soportada por el target.

No asumirá universalmente que todos los targets/versiones manejan la misma sintaxis.

---

# 95. Collation

Ejemplo:

```sql
ORDER BY "name" COLLATE "NOCASE"
```

cuando la collation haya sido resuelta estructuralmente.

---

# 96. Collation identity

```text
CollationId
≠
arbitrary string
```

---

# 97. Built-in collations

El target podrá describir collations conocidas como:

```text
BINARY
NOCASE
RTRIM
```

sin impedir custom collations explícitamente registradas.

---

# 98. Custom collations

Una custom collation será capability/dependency de la conexión/target.

---

# 99. LIMIT

```sql
LIMIT 20
```

---

# 100. OFFSET

SQLite presenta consideraciones sintácticas cuando existe OFFSET sin un límite ordinario.

El adapter deberá resolverlo de forma explícita cuando la operación sea representable.

---

# 101. Canonical pagination

VoltStack deberá escoger una forma canónica para:

```text
LIMIT
OFFSET
```

y evitar múltiples representaciones equivalentes dentro del mismo profile.

---

# 102. LIMIT without ORDER BY

Nunca inventará ordering.

---

# 103. CTE

```sql
WITH "active_users" AS (
    SELECT ...
)
SELECT ...
```

---

# 104. Recursive CTE

```sql
WITH RECURSIVE "tree" AS (
    ...
)
SELECT ...
```

---

# 105. Recursive model

El compiler recibirá:

```text
anchor
recursive member
recursive binding
set operation
output schema
```

ya validados.

---

# 106. CTE materialization

Las capabilities de materialization hints deberán modelarse independientemente de otros motores.

No se copiará la semántica PostgreSQL automáticamente.

---

# 107. Compound queries

SQLite compiler deberá modelar:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

según capabilities.

---

# 108. ALL semantics

```text
UNION
≠
UNION ALL
```

---

# 109. Unsupported compound multiplicity

Si una forma como:

```text
INTERSECT ALL
```

no puede representarse nativamente y no existe una estrategia exacta ya decidida por etapas anteriores:

```text
→ compilation failure
```

---

# 110. No semantic emulation in renderer

El compiler no transformará arbitrariamente:

```text
INTERSECT ALL
```

en una consulta compleja inventada durante rendering.

---

# 111. INSERT

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES (?1, ?2)
```

---

# 112. Multi-row INSERT

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES
    (?1, ?2),
    (?3, ?4)
```

---

# 113. DEFAULT VALUES

```sql
INSERT INTO "users"
DEFAULT VALUES
```

cuando sea compatible con el operation contract.

---

# 114. INSERT SELECT

```sql
INSERT INTO "archive_users" (
    "id",
    "email"
)
SELECT
    "id",
    "email"
FROM "users"
```

---

# 115. Conflict handling families

SQLite posee varias construcciones relacionadas con conflictos.

VoltStack deberá distinguir:

```text
Constraint Conflict Resolution
≠
UPSERT Conflict Handling
```

---

# 116. INSERT OR ...

SQLite puede representar:

```text
INSERT OR ROLLBACK
INSERT OR ABORT
INSERT OR FAIL
INSERT OR IGNORE
INSERT OR REPLACE
```

según la operación/capability.

---

# 117. Conflict resolution enum

```php
enum SQLiteConflictResolution
{
    case ROLLBACK;
    case ABORT;
    case FAIL;
    case IGNORE;
    case REPLACE;
}
```

---

# 118. Conflict policy must be explicit

El compiler no añadirá:

```sql
OR IGNORE
```

porque una aplicación quiera “evitar errores”.

La política debe existir en el input compilable.

---

# 119. REPLACE semantics

Especial cuidado:

```text
REPLACE
≠
generic UPDATE existing row
```

VoltStack no deberá presentar `REPLACE` como equivalente universal a un upsert convencional.

---

# 120. UPSERT

Cuando sea soportado:

```sql
INSERT INTO "users" (
    "email",
    "name"
)
VALUES (?1, ?2)
ON CONFLICT ("email")
DO UPDATE SET
    "name" = excluded."name"
```

---

# 121. Semantic upsert

```text
UpsertIntent
├── target
├── insert values
├── conflict target
├── conflict predicate
├── action
└── returning
```

---

# 122. SQLite upsert adapter

```text
UpsertIntent
        ↓
SQLiteUpsertAdapter
        ↓
SQLite ON CONFLICT representation
```

---

# 123. excluded relation

La pseudo-relación:

```text
excluded
```

deberá representarse mediante un descriptor especial.

No como arbitrary alias.

---

# 124. UPSERT ≠ INSERT OR REPLACE

```text
UPSERT
≠
INSERT OR REPLACE
```

---

# 125. Conflict target

Podrá incluir, según capability:

```text
indexed columns
expressions
collations
partial-index predicate
```

cuando el semantic/upsert model lo permita.

---

# 126. RETURNING

Targets SQLite compatibles pueden soportar:

```sql
INSERT INTO "users" (
    "name"
)
VALUES (?1)
RETURNING
    "id",
    "name"
```

---

# 127. RETURNING capability

Nunca se asumirá únicamente porque:

```text
SQLite target exists
```

---

# 128. RETURNING output contract

```text
Mutation
+
ReturningProjection
        ↓
SQLite SQL
        ↓
CompiledResultContract
```

---

# 129. RETURNING ≠ lastInsertId

```text
RETURNING
≠
lastInsertId()
```

---

# 130. UPDATE

```sql
UPDATE "users"
SET "status" = ?1
WHERE "id" = ?2
```

---

# 131. UPDATE RETURNING

Cuando sea soportado:

```sql
UPDATE "users"
SET "status" = ?1
WHERE "id" = ?2
RETURNING
    "id",
    "status"
```

---

# 132. UPDATE FROM

Cuando el capability snapshot lo permita:

```text
SQLiteUpdateFrom
```

podrá utilizar una representación nativa.

---

# 133. UPDATE FROM ≠ PostgreSQL UPDATE FROM

Incluso si la sintaxis es similar:

```text
same-looking syntax
≠
assume identical full semantics/capabilities
```

---

# 134. UPDATE LIMIT

La disponibilidad de:

```text
UPDATE ... LIMIT
```

puede depender de características concretas del build.

Por tanto deberá ser:

```text
capability-gated
```

---

# 135. DELETE

```sql
DELETE FROM "users"
WHERE "id" = ?1
```

---

# 136. DELETE RETURNING

Cuando sea soportado:

```sql
DELETE FROM "users"
WHERE "id" = ?1
RETURNING "id"
```

---

# 137. DELETE LIMIT

Al igual que UPDATE LIMIT:

```text
capability-gated
```

cuando dependa del target/build.

---

# 138. No mutation rewriting

Si un UPDATE/DELETE con limitación requerida no es representable:

```text
compiler
→ error
```

No:

```text
compiler
→ SELECT rowids
→ UPDATE another statement
```

---

# 139. Single command invariant

Por defecto:

```text
One CompilableDatabaseOperation
        ↓
One CompiledDatabaseCommand
        ↓
One SQL statement
```

---

# 140. No hidden multi-statement emulation

Nunca:

```text
unsupported operation
        ↓
compiler emits:
SELECT ...
;
UPDATE ...
;
```

---

# 141. Multi-command operations

Si una feature futura necesita varios commands:

```text
Execution Plan
```

deberá representarlos explícitamente como varias execution units.

---

# 142. ROWID

SQLite posee el concepto especial de:

```text
ROWID
```

para tablas compatibles.

VoltStack deberá modelarlo explícitamente.

---

# 143. ROWID ≠ ordinary user column

```text
RowIdentity::SQLiteRowId
≠
ResolvedColumnId("rowid")
```

---

# 144. ROWID aliases

El comportamiento de aliases relacionados con integer primary keys deberá permanecer encapsulado en metadata SQLite.

---

# 145. No universal ROWID assumption

No todas las relaciones SQLite deben asumirse compatibles con ROWID.

---

# 146. WITHOUT ROWID

El capability/schema metadata deberá poder expresar:

```text
WITHOUT ROWID
```

---

# 147. WITHOUT ROWID consequence

Una estrategia física o SQL que requiera ROWID será inválida para dicha tabla.

---

# 148. Compiler boundary

El compiler valida representabilidad.

La decisión de usar ROWID como estrategia física deberá provenir del planner.

---

# 149. Row identity descriptor

```php
enum SQLiteRowIdentityKind
{
    case ROWID;
    case PRIMARY_KEY;
    case WITHOUT_ROWID_PRIMARY_KEY;
    case UNKNOWN;
}
```

---

# 150. INTEGER PRIMARY KEY

No deberá tratarse como simple coincidencia textual.

Su metadata física/schema deberá haber sido resuelta previamente.

---

# 151. Generated columns

Cuando el target las soporte, el schema/compiler architecture podrá representar:

```text
VIRTUAL
STORED
```

según capabilities.

---

# 152. Query compiler relationship

El Query Compiler podrá leer generated columns como columnas ordinarias desde el punto de vista de consulta, conservando metadata de writability.

---

# 153. Generated column mutation

No deberá generar assignments inválidos hacia columnas no writable.

---

# 154. Generated column rules

La writability deberá provenir de schema metadata/semantic validation, no descubrirse durante rendering.

---

# 155. JSON architecture

SQLite JSON deberá ser capability-driven.

---

# 156. JSON capability

No se asumirá:

```text
SQLite
=
JSON always available in every target configuration
```

El target snapshot decidirá las operaciones disponibles.

---

# 157. JSON semantic operations

VoltStack podrá modelar:

```text
JsonExtract
JsonExtractScalar
JsonSet
JsonInsert
JsonReplace
JsonRemove
JsonType
JsonValid
JsonArrayLength
JsonEach
JsonTree
JsonPatch
```

según el sistema JSON superior.

---

# 158. JSON adapter

```text
Semantic JSON Operation
        ↓
SQLiteJsonAdapter
        ↓
SQLite JSON function/operator representation
```

---

# 159. JSON functions

No se dispersarán strings como:

```text
json_extract
json_set
json_remove
```

por todo el compiler.

---

# 160. JSON function registry

```text
SQLiteJsonFunctionRegistry
```

centralizará descriptors.

---

# 161. JSON operators

Si el target soporta operadores como:

```text
->
->>
```

se representarán mediante typed descriptors y capabilities.

---

# 162. JSON path

Un JSON path será:

```text
parameterized runtime value
```

cuando corresponda, o:

```text
validated structured path
```

cuando sea parte de estructura.

---

# 163. SQL NULL vs JSON null

Se preservará:

```text
SQL NULL
≠
JSON null
```

---

# 164. JSONB naming collision

VoltStack no deberá asumir que conceptos denominados `JSONB` en diferentes motores poseen necesariamente las mismas propiedades o representación.

---

# 165. JSON portability

```text
Semantic JSON Operation
        ↓
Platform-specific adaptation
```

No:

```text
PostgreSQL JSON syntax
        ↓
attempt to reuse in SQLite
```

---

# 166. Table-valued JSON functions

Funciones como iteradores JSON, cuando se utilicen como relation sources, deberán representarse como:

```text
Structured TableFunctionRelation
```

no como raw SQL.

---

# 167. FTS

SQLite Full-Text Search deberá ser una extensión/capability separada.

---

# 168. FTS modules

El compiler core no asumirá automáticamente:

```text
FTS3
FTS4
FTS5
```

---

# 169. Full-text abstraction

```text
FullText Semantic Model
        ↓
SQLite FullText Adapter
        ↓
FTS-specific SQL
```

---

# 170. Future integration

Se coordinará con:

```text
272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
```

---

# 171. Virtual tables

SQLite posee un modelo importante de:

```text
virtual tables
```

El compiler deberá permitir extensiones estructuradas.

---

# 172. Virtual table ≠ ordinary table

Metadata podrá indicar:

```text
relation kind
module
capabilities
writability
special operators
```

---

# 173. Extension modules

Ejemplos:

```text
FTS
RTree
custom virtual table modules
```

deberán entrar mediante el extension model.

---

# 174. No runtime module discovery in compiler

El compiler no consultará módulos cargados durante compilation.

---

# 175. Expression system

SQLite expression renderer deberá soportar:

```text
column references
parameters
structural literals
unary expressions
binary expressions
function calls
CASE
CAST
subqueries
aggregate expressions
window expressions
JSON expressions
extension expressions
raw expressions
```

---

# 176. CAST

SQLite podrá utilizar:

```sql
CAST(?1 AS TEXT)
```

cuando el type planner determine que el cast forma parte de la representación correcta.

---

# 177. No arbitrary casting

Nunca:

```text
parameter
→ always CAST to declared column type
```

---

# 178. Affinity-aware casts

La decisión podrá considerar:

```text
semantic type
expected expression type
column affinity
STRICT mode
operator semantics
result contract
```

---

# 179. Numeric semantics

SQLite numeric conversion behavior no deberá reemplazar la semántica declarada por VoltStack.

Si una operación no puede preservar el contrato requerido:

```text
→ fail or require explicit adaptation
```

---

# 180. Division semantics

Operaciones aritméticas sensibles al tipo deberán recibir type information suficiente.

El renderer no deberá decidir tipos observando literals textuales.

---

# 181. String concatenation

La operación semántica:

```text
Concat(a, b)
```

podrá renderizarse mediante la representación SQLite apropiada.

---

# 182. Semantic operation ≠ SQL token

```text
Concat
≠
"||"
```

El token pertenece al dialect adapter.

---

# 183. CASE

```sql
CASE
    WHEN "status" = ?1 THEN ?2
    ELSE ?3
END
```

---

# 184. CASE type contract

La result type ya deberá haber sido determinada antes de rendering.

---

# 185. IN

El compiler deberá soportar:

```text
IN value list
IN subquery
```

preservando NULL semantics.

---

# 186. Empty IN lists

Un empty set deberá resolverse mediante una representación semánticamente correcta definida por capas anteriores/compiler rules.

Nunca:

```sql
IN ()
```

sin validar si la representación es válida y portable.

---

# 187. Variable limits

SQLite puede imponer límites de parámetros.

El capability/resource profile deberá poder expresar:

```text
maximum host parameters
```

cuando sea conocido.

---

# 188. Compilation budget

Si una consulta excede el límite conocido:

```text
→ compilation error
```

antes de ejecución.

---

# 189. Large IN strategy

El compiler no inventará automáticamente:

```text
temporary table
multiple statements
chunked execution
```

Estas son decisiones de planning/execution.

---

# 190. SQL length limits

Si el target profile expone límites relevantes:

```text
maximum SQL length
maximum expression depth
maximum compound terms
maximum function arguments
```

podrán participar en validation/budget.

---

# 191. Limit knowledge

Un límite desconocido será:

```text
UNKNOWN
```

no:

```text
UNLIMITED
```

---

# 192. PRAGMA architecture

`PRAGMA` no deberá mezclarse con el Query Compiler ordinario.

---

# 193. PRAGMA operations

Si VoltStack necesita administrarlos:

```text
SQLitePragmaOperation
```

deberá ser una categoría explícita de command/admin operation.

---

# 194. No implicit PRAGMA

El compiler nunca deberá insertar silenciosamente:

```sql
PRAGMA foreign_keys = ON;
```

antes de una consulta.

---

# 195. Foreign key enforcement

La configuración runtime de foreign key enforcement pertenece a:

```text
connection/session initialization
```

no al Query Compiler.

---

# 196. WAL

```text
WAL mode
```

pertenece a runtime/connection/operations configuration.

No al SQL compiler.

---

# 197. Busy timeout

```text
busy_timeout
```

pertenece a connection/runtime policy.

---

# 198. Locking semantics

SQLite no deberá recibir artificialmente las cláusulas locking de PostgreSQL/MySQL.

---

# 199. FOR UPDATE

Si el Query Model contiene una locking requirement no representable en SQLite:

```text
SQLiteCapabilityValidation
→ unsupported
```

---

# 200. No fake FOR UPDATE

Nunca:

```text
FOR UPDATE requirement
→ silently drop clause
```

---

# 201. Transaction-based locking

Si una estrategia equivalente requiere modificar transaction mode:

```text
BEGIN IMMEDIATE
BEGIN EXCLUSIVE
```

esa decisión pertenece al Transaction/Execution system.

No al Query Compiler.

---

# 202. Compiler cannot widen transaction semantics

El compiler no convertirá una query lock en un cambio de transaction mode.

---

# 203. Transaction modes

Conceptos como:

```text
DEFERRED
IMMEDIATE
EXCLUSIVE
```

pertenecerán al SQLite transaction adapter.

---

# 204. Database file identity

La ruta:

```text
/path/app.sqlite
```

no formará parte del SQL Compiler.

---

# 205. In-memory databases

Targets como:

```text
:memory:
```

pertenecen a connection configuration.

---

# 206. Shared in-memory modes

Igualmente, URI/connection-level in-memory behavior no deberá contaminar el SQL AST.

---

# 207. Attached databases

SQLite permite conceptos de attached databases.

Si VoltStack los soporta, deberán modelarse explícitamente como namespace/catalog-like identities.

---

# 208. ATTACH/DETACH

`ATTACH` y `DETACH` serán admin/session commands.

No ordinary query transformations.

---

# 209. Schema qualification

Una relation podrá renderizarse conceptualmente como:

```sql
"main"."users"
```

cuando la resolved relation identity lo requiera.

---

# 210. main/temp identities

Conceptos como:

```text
main
temp
attached database
```

deberán permanecer estructurados.

---

# 211. No string-qualified relations

Nunca:

```php
$table = $database . '.' . $table;
```

sin structured identifier model.

---

# 212. Schema compiler separation

Limitaciones SQLite de:

```text
ALTER TABLE
DROP COLUMN
RENAME
constraint changes
```

pertenecen principalmente a:

```text
Schema Compiler
Migration Planner
```

no al Query Compiler.

---

# 213. Migration rebuild strategy

La conocida estrategia de reconstrucción de tabla:

```text
create temporary table
copy data
drop old
rename
```

no deberá implementarse en `SQLiteQueryCompiler`.

---

# 214. Query Compiler ≠ Migration Compiler

```text
SQLiteQueryCompiler
≠
SQLiteSchemaMigrationCompiler
```

---

# 215. Result contract

```php
final readonly class SQLiteCompiledResultContract
{
    public function __construct(
        public ResultShape $shape,
        public array $columns,
        public ResultConversionPlan $conversion,
    ) {}
}
```

---

# 216. Result type conversion

Debido al dynamic typing de SQLite, el result contract será especialmente importante.

---

# 217. Driver value ≠ semantic value

```text
SQLite Driver Value
        ↓
Result Conversion
        ↓
VoltStack Semantic Value
```

---

# 218. Hydration boundary

Result conversion podrá normalizar tipos de bajo nivel.

Pero:

```text
Result Conversion
≠
ORM Hydration
```

---

# 219. Boolean result

Por ejemplo:

```text
INTEGER 0/1
        ↓
Boolean conversion descriptor
        ↓
false/true
```

si el semantic output contract es boolean.

---

# 220. Date result

```text
TEXT/INTEGER/REAL
        ↓
DateTime conversion descriptor
```

según storage policy.

---

# 221. JSON result

```text
TEXT/BLOB-like target representation
        ↓
JSON conversion descriptor
```

según capability/type mapping.

---

# 222. Source map

```text
SQLite SQL Range
        ↕
Emission Node
        ↕
Execution Node
        ↕
Physical Node
        ↕
Logical/Semantic Origin
```

---

# 223. Source map purpose

Permitirá traducir errores posteriores sin parsear nuevamente el SQL generado.

---

# 224. No source map reconstruction

Nunca:

```text
render SQL
→ parse generated SQL
→ guess origins
```

---

# 225. Compilation dependencies

Podrán incluir:

```text
SQLite target identity
SQLite version
capability snapshot
compile-option profile
extension profile
relation identity
column identity
type mapping profile
collation identity
function identity
compiler version
renderer version
```

---

# 226. Dependency precision

No deberá invalidarse un compiled command por cambios no relacionados cuando puedan distinguirse dependencies más específicas.

---

# 227. Fingerprint

Conceptualmente:

```text
SQLiteCompiledFingerprint
=
OperationFingerprint
+
SQLiteTargetFingerprint
+
CapabilityFingerprint
+
DialectFingerprint
+
TypeMappingFingerprint
+
PlaceholderProfileFingerprint
+
CompilerVersion
+
ExtensionFingerprint
+
RepresentationSpecializationFingerprint
```

---

# 228. Runtime values excluded

No participarán:

```text
email
password
tenant runtime value
current connection id
transaction id
database file handle
```

---

# 229. Database path excluded by default

La ruta física del archivo SQLite no deberá participar en el compiled SQL fingerprint salvo que una operación estructural excepcional dependa explícitamente de esa identidad.

---

# 230. Cacheability

El compiled command podrá clasificarse como:

```text
CACHEABLE
CONTEXT_BOUND
SPECIALIZED
NON_CACHEABLE
```

con reason set.

---

# 231. Stable SQL shape

```text
same operation structure
+
same SQLite target profile
+
different runtime bindings
=
same canonical compiled SQL
```

salvo especialización explícita.

---

# 232. Persistent runtime

El compiler deberá ser seguro en:

```text
FrankenPHP
RoadRunner
OpenSwoole
CLI workers
queue workers
long-running tests
```

---

# 233. Shared state

Puede compartirse:

```text
frozen registries
immutable type descriptors
immutable function descriptors
immutable operator descriptors
renderer definitions
capability profiles
```

---

# 234. Operation-local state

Debe permanecer local:

```text
alias allocator
placeholder allocator
binding builder
source map builder
diagnostics
render buffer
temporary maps
budget counters
```

---

# 235. Connection state isolation

Especialmente importante:

```text
SQLite connection state
≠
SQLite compiler state
```

---

# 236. Per-connection functions

Una custom function registrada en Connection A no deberá asumirse disponible en Connection B.

---

# 237. Capability profile identity

Cuando una query dependa de funciones/collations/extensions per-connection, la capability identity deberá reflejarlo.

---

# 238. Persistent worker leakage

Nunca:

```text
Request A
registers SQLite function capability
        ↓
global compiler state
        ↓
Request B accidentally sees it
```

---

# 239. Extension architecture

```php
interface SQLiteCompilerExtension
{
    public function descriptor(): SQLiteCompilerExtensionDescriptor;
}
```

---

# 240. Extension categories

Podrán existir extensiones para:

```text
functions
operators
types
collations
virtual tables
JSON capabilities
full-text
spatial features
custom SQL expressions
custom statement kinds
```

---

# 241. Extension lifecycle

```text
bootstrap
→ discover
→ validate
→ dependency resolution
→ conflict resolution
→ freeze
```

---

# 242. No last-wins

```text
Extension A
and
Extension B
claim same semantic operator incompatibly
        ↓
explicit conflict
```

---

# 243. Unknown extension

```text
→ SQLiteExtensionCompilationException
```

No raw fallback.

---

# 244. Raw SQL

SQLite Compiler deberá respetar el escape-hatch architecture general.

---

# 245. Raw expression classification

```text
TrustedFrameworkRaw
TrustedExtensionRaw
ExplicitUserRaw
```

---

# 246. Raw values

Incluso raw fragments deberán utilizar explicit bindings para runtime values.

---

# 247. Security boundary

```text
Runtime Values
    → Bindings

Identifiers
    → Structured Identifier Compiler

Functions
    → Function Registry

Operators
    → Operator Registry

Collations
    → Collation Registry

JSON Paths
    → Binding / Structured Path

Raw SQL
    → Explicit Escape Hatch
```

---

# 248. Mandatory security predicates

El SQLite compiler no podrá eliminar:

```text
tenant predicates
authorization predicates
security filters
```

---

# 249. Unsupported security representation

Si una security requirement no puede representarse:

```text
→ compilation failure
```

Nunca:

```text
→ remove security restriction
```

---

# 250. Compilation budget

El compiler deberá respetar límites para:

```text
AST/emission nodes
expression depth
compound branches
CTEs
parameters
generated aliases
SQL bytes
source map entries
extension invocations
```

---

# 251. Target-specific limits

Cuando se conozcan límites SQLite, éstos podrán reducir el budget efectivo.

---

# 252. Budget exhaustion

```text
→ SQLiteCompilationBudgetExceededException
```

---

# 253. No truncated SQL

Nunca:

```text
budget exhausted
→ return partially generated SQL
```

---

# 254. Atomic compilation

```text
Compilation success
    → complete immutable artifact

Compilation failure
    → no published artifact
```

---

# 255. Diagnostics

Un diagnostic podrá incluir:

```text
phase
SQLite target
SQLite version
capability
source node
semantic origin
physical origin
execution origin
extension
type
parameter
SQL source range
```

---

# 256. Sensitive-data policy

Diagnostics no deberán incluir runtime parameter values por defecto.

---

# 257. Determinism

Dados los mismos:

```text
operation
SQLite target
capabilities
dialect profile
extensions
compiler configuration
```

se deberá obtener:

```text
same canonical SQL
same placeholder plan
same binding layout
same result contract
same dependency set
same fingerprint
```

---

# 258. No environment-dependent rendering

No deberán influir:

```text
current time
randomness
process id
memory address
global counters
current working directory
database file path
request id
```

salvo metadata explícitamente estructural.

---

# 259. SQLite native query planner

VoltStack deberá respetar que SQLite posee su propio query planner.

---

# 260. Planner boundary

```text
VoltStack Physical Planner
        ↓
determine framework-level strategy
        ↓
SQLite Compiler
        ↓
generate SQLite SQL
        ↓
SQLite Query Planner
        ↓
native execution strategy
```

---

# 261. No fake index enforcement

El compiler no afirmará que un índice será usado salvo que exista un mecanismo target-specific con garantía suficiente y haya sido solicitado por una etapa superior.

---

# 262. INDEXED BY

Si VoltStack decide soportar una construcción SQLite como:

```sql
INDEXED BY "index_name"
```

deberá ser:

```text
explicit
capability-aware
physical-planner-driven
structured
```

---

# 263. NOT INDEXED

Igualmente:

```sql
NOT INDEXED
```

será un physical representation descriptor.

No una decisión espontánea del renderer.

---

# 264. Index hint ≠ optimizer hint

No se generalizará automáticamente como una abstracción universal entre motores.

---

# 265. EXPLAIN

Una operación explícita podrá compilar:

```text
EXPLAIN
EXPLAIN QUERY PLAN
```

cuando sea parte del API.

---

# 266. No implicit EXPLAIN

Nunca:

```text
compile query
→ run EXPLAIN QUERY PLAN
→ choose SQL
```

---

# 267. EXPLAIN execution

El compiler sólo generará la representación.

El Execution Engine decidirá ejecutarla.

---

# 268. Prepared statement integration

```text
SQLiteCompiledDatabaseCommand
        ↓
Prepared Statement Compilation
        ↓
SQLite Prepared Statement
```

formalizado posteriormente en:

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

---

# 269. Compiled query cache integration

```text
SQLiteCompiledFingerprint
        ↓
Compiled Query Cache
```

formalizado en:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 270. Execution integration

```text
SQLiteCompiledDatabaseCommand
        ↓
Query Executor
        ↓
Statement Execution
        ↓
SQLite Connection
        ↓
SQLite
```

---

# 271. SQLite compiler ends before execution

La responsabilidad termina antes de:

```text
prepare live statement
bind actual runtime values
step statement
fetch rows
reset statement
finalize statement
```

---

# 272. Driver independence

El compiler no deberá depender directamente de:

```text
PDO_SQLITE
SQLite3 PHP extension
future native driver
```

---

# 273. Driver compilation profile

Si el driver afecta:

```text
placeholder style
binding types
binary handling
result conversion
```

deberá proporcionar:

```text
SQLiteDriverCompilationProfile
```

como immutable metadata.

---

# 274. Driver profile ≠ live driver

Nunca:

```text
SQLiteCompiler
→ PDO object
```

---

# 275. Testing architecture

La suite deberá incluir:

```text
unit tests
golden SQL tests
capability tests
version tests
compile-option tests
type-affinity tests
STRICT-table tests
placeholder tests
binding tests
ROWID tests
UPSERT tests
RETURNING tests
JSON tests
window tests
CTE tests
compound-query tests
extension tests
security tests
persistent-worker tests
concurrency tests
integration tests
property tests
fuzz tests
```

---

# 276. Golden SELECT test

Input:

```text
Select users.id
where users.active = P1
limit 10
```

Output canónico:

```sql
SELECT "users"."id"
FROM "users"
WHERE "users"."active" = ?1
LIMIT 10
```

según placeholder profile.

---

# 277. Type-affinity tests

Cubrir:

```text
INTEGER affinity
TEXT affinity
REAL affinity
NUMERIC affinity
BLOB affinity
NULL values
mixed storage classes
explicit casts
comparison behavior
```

---

# 278. STRICT tests

Cubrir:

```text
valid binding type
invalid binding type
NULL handling
ANY-like permissive columns where modeled
generated columns
result conversion
```

---

# 279. Boolean tests

Cubrir:

```text
structural true
structural false
runtime boolean
NULL boolean
boolean result conversion
predicate contexts
```

---

# 280. Date/time tests

Cubrir cada configured storage policy:

```text
ISO text
Unix seconds
Unix milliseconds
Julian day
NULL
ordering
comparison
binding
result conversion
```

---

# 281. ROWID tests

Cubrir:

```text
ordinary rowid table
INTEGER PRIMARY KEY alias
WITHOUT ROWID
explicit rowid-named user columns
physical row identity requirements
```

---

# 282. UPSERT tests

Cubrir:

```text
DO NOTHING
DO UPDATE
single conflict target
multi-column conflict target
partial conflict target
excluded references
WHERE
RETURNING
NULL values
```

---

# 283. Conflict-resolution tests

Separadamente:

```text
OR ROLLBACK
OR ABORT
OR FAIL
OR IGNORE
OR REPLACE
```

---

# 284. RETURNING tests

Cubrir:

```text
INSERT RETURNING
UPDATE RETURNING
DELETE RETURNING
multiple columns
expressions
multiple rows
type conversion
```

---

# 285. CTE tests

Cubrir:

```text
normal CTE
recursive CTE
multiple CTEs
nested CTEs
correlated subqueries
compound recursive members
```

---

# 286. Window tests

Cubrir:

```text
ROW_NUMBER
RANK
partition
ordering
ROWS frame
RANGE frame where supported
GROUPS frame where supported
frame exclusions where supported
named windows
```

---

# 287. JSON tests

Cubrir:

```text
JSON extraction
scalar extraction
validation
set
insert
replace
remove
array length
JSON null
SQL NULL
paths
operators
table-valued JSON functions
```

---

# 288. Compound-query tests

Cubrir:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
parentheses
ORDER BY
LIMIT
unsupported ALL variants
```

---

# 289. Capability regression tests

Para cada capability dependiente de versión/build:

```text
unsupported target
supported target
restricted target
extension-provided target
```

---

# 290. Persistent worker tests

```text
compile SQLite target A
reset session
compile SQLite target B
```

deberá demostrar ausencia de:

```text
placeholder leakage
extension leakage
collation leakage
function leakage
database identity leakage
```

---

# 291. Concurrent compilation

Múltiples compilaciones simultáneas deberán producir resultados independientes.

---

# 292. Integration tests

Deberán ejecutarse contra SQLite real para verificar:

```text
syntax
bindings
types
RETURNING
UPSERT
CTEs
window functions
JSON
ROWID
STRICT tables
result conversion
```

según capabilities del test target.

---

# 293. SQLite test matrix

Idealmente:

```text
minimum supported SQLite profile
current stable profile
feature-rich profile
restricted build profile
in-memory connection
file-backed connection
STRICT schema profile
extension-enabled profile
```

---

# 294. Compiler benchmark

Medirá:

```text
capability validation
dialect adaptation
type-affinity resolution
parameter planning
rendering
source map generation
fingerprinting
memory allocations
```

sin necesidad de ejecutar SQLite.

---

# 295. Runtime benchmark

Separadamente:

```text
compile
prepare
bind
execute
fetch
```

---

# 296. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Compiler\Platform\SQLite
```

---

# 297. Estructura propuesta

```text
SQLite/
├── SQLiteCompiler.php
├── SQLiteCompilerFactory.php
│
├── Target/
│   ├── SQLiteTarget.php
│   ├── SQLiteVersion.php
│   ├── SQLiteVersionRange.php
│   ├── SQLiteCompilationProfile.php
│   └── SQLiteTargetFingerprint.php
│
├── Capability/
│   ├── SQLiteCapabilitySnapshot.php
│   ├── SQLiteCapabilityResolver.php
│   ├── SQLiteFeature.php
│   ├── SQLiteFeatureSet.php
│   ├── SQLiteCompileOptionSet.php
│   └── SQLiteExtensionCapabilitySet.php
│
├── Dialect/
│   ├── SQLiteDialectProfile.php
│   ├── SQLiteDialectAdapter.php
│   └── SQLiteDialectRenderingContract.php
│
├── Validation/
│   ├── SQLiteRepresentationValidator.php
│   ├── SQLiteCapabilityValidator.php
│   ├── SQLiteStatementValidator.php
│   └── SQLiteLimitValidator.php
│
├── Identifier/
│   ├── SQLiteIdentifierRenderer.php
│   ├── SQLiteQualifiedIdentifierRenderer.php
│   └── SQLiteAliasAllocator.php
│
├── Placeholder/
│   ├── SQLitePlaceholderPolicy.php
│   ├── SQLitePlaceholderRenderer.php
│   └── SQLiteParameterAllocator.php
│
├── Type/
│   ├── SQLiteTypeResolver.php
│   ├── SQLiteTypeDescriptor.php
│   ├── SQLiteDeclaredType.php
│   ├── SQLiteTypeAffinity.php
│   ├── SQLiteStorageClass.php
│   ├── SQLiteCastPlanner.php
│   ├── SQLiteBooleanAdapter.php
│   └── SQLiteDateTimeStoragePolicy.php
│
├── Statement/
│   ├── SQLiteSelectRenderer.php
│   ├── SQLiteInsertRenderer.php
│   ├── SQLiteUpdateRenderer.php
│   └── SQLiteDeleteRenderer.php
│
├── Clause/
│   ├── SQLiteReturningRenderer.php
│   ├── SQLiteUpsertRenderer.php
│   ├── SQLiteConflictResolutionRenderer.php
│   ├── SQLitePaginationRenderer.php
│   ├── SQLiteCteRenderer.php
│   ├── SQLiteWindowRenderer.php
│   └── SQLiteCompoundQueryRenderer.php
│
├── Expression/
│   ├── SQLiteExpressionRenderer.php
│   ├── SQLitePredicateRenderer.php
│   ├── SQLiteCaseRenderer.php
│   ├── SQLiteCastRenderer.php
│   └── SQLiteSubqueryRenderer.php
│
├── Function/
│   ├── SQLiteFunctionRegistry.php
│   ├── SQLiteFunctionDescriptor.php
│   └── SQLiteFunctionRenderer.php
│
├── Operator/
│   ├── SQLiteOperatorRegistry.php
│   ├── SQLiteOperatorDescriptor.php
│   └── SQLiteOperatorRenderer.php
│
├── Json/
│   ├── SQLiteJsonAdapter.php
│   ├── SQLiteJsonFunctionRegistry.php
│   ├── SQLiteJsonOperatorRegistry.php
│   └── SQLiteJsonRenderer.php
│
├── RowIdentity/
│   ├── SQLiteRowIdentityKind.php
│   ├── SQLiteRowIdentityDescriptor.php
│   ├── SQLiteRowIdRenderer.php
│   └── SQLiteWithoutRowIdDescriptor.php
│
├── Conflict/
│   ├── SQLiteConflictResolution.php
│   ├── SQLiteUpsertAdapter.php
│   ├── SQLiteConflictTarget.php
│   ├── SQLiteConflictAction.php
│   └── SQLiteExcludedRelation.php
│
├── Collation/
│   ├── SQLiteCollationRegistry.php
│   └── SQLiteCollationDescriptor.php
│
├── Extension/
│   ├── SQLiteCompilerExtension.php
│   ├── SQLiteCompilerExtensionRegistry.php
│   ├── SQLiteCompilerExtensionDescriptor.php
│   └── SQLiteExtensionConflictDetector.php
│
├── Result/
│   ├── SQLiteCompiledResultContract.php
│   └── SQLiteResultConversionPlan.php
│
├── Diagnostic/
│   ├── SQLiteCompilationDiagnostic.php
│   └── SQLiteDiagnosticMetadata.php
│
└── Exception/
    ├── SQLiteCompilationException.php
    ├── SQLiteCapabilityException.php
    ├── UnsupportedSQLiteFeatureException.php
    ├── UnsupportedSQLiteVersionException.php
    ├── SQLiteRepresentationException.php
    ├── SQLiteTypeCompilationException.php
    ├── SQLitePlaceholderException.php
    ├── SQLiteExtensionCompilationException.php
    ├── SQLiteCompilationBudgetExceededException.php
    └── SQLiteInvariantException.php
```

---

# 298. Invariantes arquitectónicos

## DB-SQLITE-001

SQLite será tratado como plataforma de primera clase.

## DB-SQLITE-002

SQLite no será considerado únicamente un database para testing.

## DB-SQLITE-003

SQLite Compiler no realizará Semantic Analysis.

## DB-SQLITE-004

SQLite Compiler no realizará Query Optimization.

## DB-SQLITE-005

SQLite Compiler no realizará Physical Planning.

## DB-SQLITE-006

SQLite Compiler no ejecutará queries.

## DB-SQLITE-007

SQLite Compiler no abrirá database files.

## DB-SQLITE-008

SQLite Compiler no abrirá connections.

## DB-SQLITE-009

SQLite Compiler no ejecutará PRAGMA.

## DB-SQLITE-010

SQLite Compiler no iniciará transactions.

## DB-SQLITE-011

SQLite Compiler no hidratará entities.

## DB-SQLITE-012

SQLite Compiler no conocerá UnitOfWork.

## DB-SQLITE-013

SQLite version será explícita.

## DB-SQLITE-014

SQLite version será distinta de capabilities.

## DB-SQLITE-015

Compile options podrán participar en capabilities.

## DB-SQLITE-016

Capabilities serán immutable snapshots.

## DB-SQLITE-017

Compiler no descubrirá capabilities mediante I/O.

## DB-SQLITE-018

Compiler no ejecutará `sqlite_version()`.

## DB-SQLITE-019

Compiler no consultará `sqlite_schema`.

## DB-SQLITE-020

Compiler no consultará `sqlite_master`.

## DB-SQLITE-021

Compilation context será immutable.

## DB-SQLITE-022

Mutable compilation state será operation-scoped.

## DB-SQLITE-023

Dialect adaptation será distinta de semantic rewriting.

## DB-SQLITE-024

Representation validation será distinta de semantic validation.

## DB-SQLITE-025

Identifiers serán estructurados.

## DB-SQLITE-026

Identifiers no serán runtime values.

## DB-SQLITE-027

Parameter identity será independiente de SQLite placeholder syntax.

## DB-SQLITE-028

Binding layout será separado del SQL.

## DB-SQLITE-029

Placeholder allocation será operation-scoped.

## DB-SQLITE-030

No habrá global parameter counter.

## DB-SQLITE-031

SQLite dynamic typing no contaminará el Query Type System.

## DB-SQLITE-032

Storage class será distinta de declared type.

## DB-SQLITE-033

Declared type será distinto de type affinity.

## DB-SQLITE-034

Type affinity será distinta de semantic QueryType.

## DB-SQLITE-035

Type mappings serán explícitos.

## DB-SQLITE-036

STRICT tables serán capability/schema metadata.

## DB-SQLITE-037

Compiler no descubrirá STRICT mode mediante live schema access.

## DB-SQLITE-038

STRICT mode no cambiará globalmente el semantic type model.

## DB-SQLITE-039

Boolean seguirá siendo semantic Boolean.

## DB-SQLITE-040

Boolean storage representation pertenecerá al SQLite adapter.

## DB-SQLITE-041

Three-valued logic será preservada.

## DB-SQLITE-042

SQL NULL será preservado.

## DB-SQLITE-043

Date/time semantic types serán independientes de storage representation.

## DB-SQLITE-044

Date/time storage policy será explícita.

## DB-SQLITE-045

Date/time policy participará en fingerprints cuando corresponda.

## DB-SQLITE-046

Compiler no utilizará application current time para reemplazar database time.

## DB-SQLITE-047

Functions serán registry-driven.

## DB-SQLITE-048

Per-connection functions requerirán explicit capability metadata.

## DB-SQLITE-049

SELECT rendering será determinista.

## DB-SQLITE-050

DISTINCT preservará SQL semantics.

## DB-SQLITE-051

FROM sources serán structured.

## DB-SQLITE-052

Join support será capability-driven.

## DB-SQLITE-053

No se codificarán supuestos históricos permanentes sobre JOIN support.

## DB-SQLITE-054

Compiler no elegirá join order.

## DB-SQLITE-055

Compiler no elegirá join algorithm.

## DB-SQLITE-056

Predicate semantics serán preservadas.

## DB-SQLITE-057

Operator precedence será explícita.

## DB-SQLITE-058

LIKE no será asumido portable case-insensitive comparison.

## DB-SQLITE-059

GLOB será SQLite-specific.

## DB-SQLITE-060

REGEXP requerirá capability/extension cuando corresponda.

## DB-SQLITE-061

Aggregate FILTER será capability-driven.

## DB-SQLITE-062

Window functions serán capability-driven.

## DB-SQLITE-063

Window ordering será distinto de final ordering.

## DB-SQLITE-064

Collations serán structured.

## DB-SQLITE-065

Custom collations serán explicit capabilities.

## DB-SQLITE-066

LIMIT sin ORDER BY no inventará ordering.

## DB-SQLITE-067

Pagination tendrá representación canónica.

## DB-SQLITE-068

CTEs serán structured.

## DB-SQLITE-069

Recursive CTEs serán structured.

## DB-SQLITE-070

Compound query multiplicity será preservada.

## DB-SQLITE-071

UNION será distinto de UNION ALL.

## DB-SQLITE-072

Unsupported compound variants no serán emuladas en renderer.

## DB-SQLITE-073

INSERT será structured.

## DB-SQLITE-074

Multi-row INSERT preservará parameter identity.

## DB-SQLITE-075

DEFAULT VALUES será first-class.

## DB-SQLITE-076

Conflict resolution será explícita.

## DB-SQLITE-077

INSERT OR REPLACE será distinto de generic UPSERT.

## DB-SQLITE-078

UPSERT será structured.

## DB-SQLITE-079

UPSERT será distinto de INSERT OR REPLACE.

## DB-SQLITE-080

`excluded` será special relation typed.

## DB-SQLITE-081

RETURNING será capability-driven.

## DB-SQLITE-082

RETURNING será distinto de lastInsertId.

## DB-SQLITE-083

RETURNING producirá ResultContract.

## DB-SQLITE-084

UPDATE FROM será capability-driven.

## DB-SQLITE-085

UPDATE LIMIT será capability-driven.

## DB-SQLITE-086

DELETE LIMIT será capability-driven.

## DB-SQLITE-087

Compiler no convertirá unsupported mutation en varios statements.

## DB-SQLITE-088

One compilable operation producirá un command salvo explicit batch architecture.

## DB-SQLITE-089

No habrá hidden multi-statement emulation.

## DB-SQLITE-090

ROWID será una identidad especial.

## DB-SQLITE-091

ROWID será distinto de ordinary column identity.

## DB-SQLITE-092

No se asumirá ROWID para todas las tablas.

## DB-SQLITE-093

WITHOUT ROWID será metadata explícita.

## DB-SQLITE-094

Planner decidirá estrategias basadas en ROWID.

## DB-SQLITE-095

Compiler sólo validará/representará dichas estrategias.

## DB-SQLITE-096

Generated-column writability será respetada.

## DB-SQLITE-097

Generated-column metadata será resuelta antes de rendering.

## DB-SQLITE-098

JSON será capability-driven.

## DB-SQLITE-099

JSON functions serán registry-driven.

## DB-SQLITE-100

JSON operators serán typed descriptors.

## DB-SQLITE-101

JSON paths no serán concatenados libremente.

## DB-SQLITE-102

SQL NULL será distinto de JSON null.

## DB-SQLITE-103

PostgreSQL JSON semantics no serán copiadas automáticamente a SQLite.

## DB-SQLITE-104

JSON table functions serán structured relations.

## DB-SQLITE-105

Full-text search será extension-driven.

## DB-SQLITE-106

FTS availability no será asumida.

## DB-SQLITE-107

Virtual tables serán explicit relation kinds.

## DB-SQLITE-108

Virtual-table modules serán capabilities/extensions.

## DB-SQLITE-109

Compiler no descubrirá modules en runtime.

## DB-SQLITE-110

Expressions serán structured.

## DB-SQLITE-111

CAST será type-planner-driven.

## DB-SQLITE-112

Compiler no añadirá casts arbitrarios.

## DB-SQLITE-113

Affinity-aware conversion no cambiará semantic meaning.

## DB-SQLITE-114

Arithmetic typing no se inferirá desde rendered strings.

## DB-SQLITE-115

Semantic concatenation será independiente del SQL token.

## DB-SQLITE-116

CASE result type estará resuelto previamente.

## DB-SQLITE-117

IN preservará NULL semantics.

## DB-SQLITE-118

Empty IN handling será estructurado.

## DB-SQLITE-119

Known host-parameter limits serán respetados.

## DB-SQLITE-120

Unknown limits no significarán unlimited.

## DB-SQLITE-121

Compiler no inventará temporary-table strategies.

## DB-SQLITE-122

PRAGMA estará fuera del ordinary Query Compiler.

## DB-SQLITE-123

Compiler no insertará implicit PRAGMA commands.

## DB-SQLITE-124

Foreign-key runtime configuration pertenecerá a Connection initialization.

## DB-SQLITE-125

WAL configuration pertenecerá a runtime.

## DB-SQLITE-126

Busy timeout pertenecerá a runtime.

## DB-SQLITE-127

Unsupported locking requirement no será descartado.

## DB-SQLITE-128

Compiler no convertirá query locking en transaction mode.

## DB-SQLITE-129

Database file identity estará fuera del compiler.

## DB-SQLITE-130

`:memory:` será connection concern.

## DB-SQLITE-131

ATTACH/DETACH no serán ordinary query transformations.

## DB-SQLITE-132

Database/schema qualifiers serán structured identities.

## DB-SQLITE-133

SQLite Query Compiler será distinto del Schema Compiler.

## DB-SQLITE-134

Migration table-rebuild strategies no pertenecerán al Query Compiler.

## DB-SQLITE-135

Result contract será explícito.

## DB-SQLITE-136

Dynamic typing aumentará la importancia de result conversion.

## DB-SQLITE-137

Result conversion será distinta de ORM hydration.

## DB-SQLITE-138

Source maps serán producidos durante compilation.

## DB-SQLITE-139

Source maps no serán reconstruidos parseando SQL.

## DB-SQLITE-140

Compilation dependencies serán explícitas.

## DB-SQLITE-141

Target fingerprint participará en compiled fingerprint.

## DB-SQLITE-142

Capability fingerprint participará en compiled fingerprint.

## DB-SQLITE-143

Type-mapping fingerprint participará cuando corresponda.

## DB-SQLITE-144

Runtime values serán excluidos del compiled fingerprint.

## DB-SQLITE-145

Database file path será excluido por defecto del compiled fingerprint.

## DB-SQLITE-146

Compiled command será immutable.

## DB-SQLITE-147

Persistent shared state será immutable.

## DB-SQLITE-148

Operation state será isolated.

## DB-SQLITE-149

Per-connection functions no podrán filtrarse entre contexts.

## DB-SQLITE-150

Per-connection collations no podrán filtrarse entre contexts.

## DB-SQLITE-151

Extension registry será frozen.

## DB-SQLITE-152

No habrá extension last-wins.

## DB-SQLITE-153

Unknown extension fallará explícitamente.

## DB-SQLITE-154

Runtime values usarán bindings por defecto.

## DB-SQLITE-155

Mandatory security predicates no podrán eliminarse.

## DB-SQLITE-156

Compilation budget será bounded.

## DB-SQLITE-157

Budget exhaustion no producirá partial SQL.

## DB-SQLITE-158

Compilation será atomic.

## DB-SQLITE-159

Compiler será deterministic.

## DB-SQLITE-160

Compiler será concurrency-safe.

## DB-SQLITE-161

SQLite native query planner será distinto del VoltStack Physical Planner.

## DB-SQLITE-162

Compiler no prometerá index usage sin garantía explícita.

## DB-SQLITE-163

INDEXED BY será physical-planner-driven si se soporta.

## DB-SQLITE-164

NOT INDEXED será physical-planner-driven si se soporta.

## DB-SQLITE-165

EXPLAIN no será ejecutado durante compilation.

## DB-SQLITE-166

Prepared statement creation pertenecerá al execution layer.

## DB-SQLITE-167

Compiler será driver-independent.

## DB-SQLITE-168

Driver-specific compilation behavior requerirá immutable profile.

## DB-SQLITE-169

SQLite limitations no serán escondidas mediante semantic degradation.

## DB-SQLITE-170

Toda representación SQLite deberá preservar semántica observable.

---

# 299. Anti-patrones

## 299.1 SQLite sólo para tests

Incorrecto:

```text
if sqlite:
    relax semantics because this is only testing
```

SQLite deberá tener el mismo rigor arquitectónico que los demás targets.

---

## 299.2 Version checks dispersos

Incorrecto:

```php
if ($sqliteVersion >= '3.35.0') {
}
```

repetido por todo el compiler.

Preferir capabilities centralizadas.

---

## 299.3 Live PRAGMA discovery

Incorrecto:

```php
$pdo->query('PRAGMA compile_options');
```

dentro del compiler.

---

## 299.4 Affinity == semantic type

Incorrecto:

```text
INTEGER affinity
=
VoltStack Integer semantic type
```

---

## 299.5 Boolean == integer

Incorrecto:

```text
Boolean
=
Integer
```

La representación física puede usar integer; la semántica no cambia.

---

## 299.6 Date as arbitrary string

Incorrecto:

```php
(string) $date
```

sin storage policy.

---

## 299.7 REPLACE como generic upsert

Incorrecto:

```text
INSERT OR REPLACE
=
portable upsert
```

---

## 299.8 Silent FOR UPDATE removal

Incorrecto:

```text
FOR UPDATE unsupported
→ omit clause
```

---

## 299.9 Compiler changing transaction mode

Incorrecto:

```text
FOR UPDATE
→ BEGIN IMMEDIATE
```

dentro del SQL Compiler.

---

## 299.10 ROWID everywhere

Incorrecto:

```text
every SQLite table has usable ROWID
```

---

## 299.11 JSON always available

Incorrecto:

```text
target == SQLite
→ JSON capabilities guaranteed
```

---

## 299.12 Global custom-function registry

Incorrecto:

```text
Connection A registers custom SQL function
→ global compiler assumes it exists everywhere
```

---

## 299.13 Multi-statement fallback

Incorrecto:

```text
unsupported UPDATE LIMIT
→ SELECT rowid...
→ UPDATE...
```

dentro del compiler.

---

## 299.14 Migration logic inside Query Compiler

Incorrecto:

```text
ALTER unsupported
→ rebuild table
```

dentro de `SQLiteQueryCompiler`.

---

## 299.15 Database path in query fingerprint

Incorrecto por defecto:

```text
/tmp/test-123.sqlite
```

como parte de compiled query identity.

---

# 300. Ejemplo — SELECT

Input:

```text
Select
├── users.id
├── users.email
├── predicate
│   └── users.active = P1
└── limit
    └── 20
```

SQL:

```sql
SELECT
    "users"."id",
    "users"."email"
FROM "users"
WHERE "users"."active" = ?1
LIMIT 20
```

Bindings:

```text
?1 → P1
```

---

# 301. Ejemplo — Boolean

Semantic input:

```text
users.active = P1<Boolean>
```

El SQL podrá ser:

```sql
"users"."active" = ?1
```

Binding:

```text
P1<Boolean>
    ↓
SQLiteBooleanBindingConversion
    ↓
INTEGER-compatible driver value
```

La conversión no modifica el semantic type.

---

# 302. Ejemplo — UPSERT

Input:

```text
Upsert users
├── email = P1
├── name = P2
├── conflict
│   └── email
├── action
│   └── update name
└── returning
    └── id
```

SQLite SQL cuando las capabilities lo permitan:

```sql
INSERT INTO "users" (
    "email",
    "name"
)
VALUES (?1, ?2)
ON CONFLICT ("email")
DO UPDATE SET
    "name" = excluded."name"
RETURNING "id"
```

---

# 303. Ejemplo — INSERT OR IGNORE

Input explícito:

```text
Insert
├── conflict resolution: IGNORE
├── table: users
└── values
```

SQL:

```sql
INSERT OR IGNORE INTO "users" (
    "email"
)
VALUES (?1)
```

No deberá confundirse con un `UpsertIntent`.

---

# 304. Ejemplo — RETURNING

Input:

```text
Delete users
├── predicate
│   └── id = P1
└── returning
    ├── id
    └── email
```

SQL:

```sql
DELETE FROM "users"
WHERE "id" = ?1
RETURNING
    "id",
    "email"
```

---

# 305. Ejemplo — Recursive CTE

Semantic structure:

```text
RecursiveCte
├── anchor
├── recursive member
├── UNION ALL
└── output schema
```

Representación:

```sql
WITH RECURSIVE "tree" AS (
    SELECT ...
    UNION ALL
    SELECT ...
)
SELECT *
FROM "tree"
```

---

# 306. Ejemplo — Window function

Input:

```text
RowNumber
├── partition: department_id
└── order: created_at DESC
```

SQL:

```sql
ROW_NUMBER() OVER (
    PARTITION BY "department_id"
    ORDER BY "created_at" DESC
)
```

---

# 307. Ejemplo — JSON

Semantic input:

```text
JsonExtractScalar(
    metadata,
    P1
)
```

Posible representación:

```sql
json_extract("metadata", ?1)
```

o una representación mediante operator cuando el target/profile y semantic operation lo permitan.

La selección pertenece al `SQLiteJsonAdapter`.

---

# 308. Ejemplo — ROWID strategy

Physical Plan:

```text
PhysicalMutationTarget
├── relation: users
└── row identity strategy: SQLITE_ROWID
```

Compiler:

```text
verify relation supports ROWID
        ↓
render structured ROWID reference
```

Si la tabla es:

```text
WITHOUT ROWID
```

entonces:

```text
→ representation failure
```

---

# 309. Ejemplo — Unsupported locking

Input:

```text
Select
└── LockRequirement
    └── FOR UPDATE
```

Target:

```text
SQLite capability:
SELECT_FOR_UPDATE = UNSUPPORTED
```

Resultado:

```text
SQLiteCapabilityException
```

Nunca:

```text
SELECT ...
```

sin el locking requerido.

---

# 310. Ejemplo — Date/time storage

Semantic value:

```text
QueryType::DateTimeImmutable
```

Configured SQLite storage:

```text
ISO8601_TEXT
```

Binding plan:

```text
DateTimeImmutable
        ↓
SQLiteDateTimeBindingConversion
        ↓
Canonical ISO-8601 text
```

El compiler almacena el descriptor de conversión.

No el valor.

---

# 311. Portability model

VoltStack utilizará:

```text
Portable Semantic Core
        +
Platform Capabilities
        +
SQLite Extensions
```

---

# 312. Portable layer

Ejemplos:

```text
SELECT
INSERT
UPDATE
DELETE
JOIN
WHERE
GROUP BY
ORDER BY
CTE
```

cuando el target capability los soporte.

---

# 313. Capability layer

Ejemplos:

```text
RETURNING
UPSERT
window functions
aggregate FILTER
generated columns
RIGHT/FULL JOIN
UPDATE FROM
```

---

# 314. SQLite-specific layer

Ejemplos:

```text
ROWID
WITHOUT ROWID
INSERT OR ...
GLOB
virtual tables
FTS modules
SQLite-specific JSON functions
INDEXED BY
PRAGMA/admin operations
```

---

# 315. Portability principle

```text
Portable API
does not forbid
Platform Power
```

---

# 316. No lowest-common-denominator design

VoltStack no eliminará:

```text
UPSERT
RETURNING
window functions
JSON
FTS
```

simplemente porque otro target no las posea.

---

# 317. Capability-aware application code

Una aplicación avanzada podrá preguntar:

```php
if ($database->capabilities()->supports(
    DatabaseFeature::Returning
)) {
    // ...
}
```

sin consultar:

```php
if ($driver === 'sqlite') {
}
```

cuando la feature sea portable/capability-based.

---

# 318. SQLite extension API

Para capacidades genuinamente específicas:

```text
SQLite Extension API
```

podrá exponer descriptors explícitos sin contaminar el portable core.

---

# 319. Error hierarchy

```text
DatabaseCompilationException
└── SqlCompilationException
    └── SQLiteCompilationException
        ├── SQLiteCapabilityException
        ├── UnsupportedSQLiteFeatureException
        ├── UnsupportedSQLiteVersionException
        ├── SQLiteRepresentationException
        ├── SQLiteTypeCompilationException
        ├── SQLitePlaceholderException
        ├── SQLiteExtensionCompilationException
        ├── SQLiteCompilationBudgetExceededException
        └── SQLiteInvariantException
```

---

# 320. Error example

```text
SQLite SQL compilation failed.

Feature:
SELECT_FOR_UPDATE

Capability:
UNSUPPORTED

Operation:
SelectOperation

Phase:
SQLITE_REPRESENTATION_VALIDATION

Target:
SQLite

Reason:
The requested locking semantics cannot be represented
by the selected SQLite target.

No SQL command was published.
```

---

# 321. Explain architecture

VoltStack deberá poder mostrar:

```text
Semantic Query
        ↓
Optimized Query
        ↓
Logical Plan
        ↓
Physical Plan
        ↓
Execution Plan
        ↓
SQLite Compilation
        ↓
SQLite SQL
```

como niveles separados.

---

# 322. SQLite EXPLAIN is another layer

Posteriormente podrá añadirse:

```text
SQLite EXPLAIN QUERY PLAN
```

como observación runtime/native.

No deberá confundirse con:

```text
VoltStack PhysicalQueryPlan
```

---

# 323. Master semantic invariant

Para toda compilación válida:

```text
Semantics(
    SQLiteCompiledDatabaseCommand
)
=
Semantics(
    CompilableDatabaseOperation
)
```

dentro del execution contract delegado.

---

# 324. Type preservation invariant

```text
SQLite Dynamic Typing
```

nunca autoriza:

```text
VoltStack Semantic Type Degradation
```

---

# 325. Capability invariant

```text
Unsupported Native Feature
```

no implica:

```text
Drop Requirement
```

sino:

```text
Use already-authorized exact representation
or
Fail
```

---

# 326. Compiler invariant

```text
Compiler may adapt representation.
Compiler may not redesign execution semantics.
```

---

# 327. SQLite compiler formula

```text
SQLiteCompilation
=
CommonSqlCompilation
+
SQLiteTargetResolution
+
CapabilityValidation
+
DynamicTypingAwareness
+
TypeAffinityMapping
+
StrictTableAwareness
+
SQLiteDialectAdaptation
+
ParameterPlanning
+
ConflictResolutionCompilation
+
UpsertCompilation
+
ReturningCompilation
+
RowIdentityRepresentation
+
JsonAdaptation
+
DeterministicRendering
+
BindingCompilation
+
ResultConversionPlanning
+
SourceMapping
+
DependencyTracking
+
Fingerprinting
```

---

# 328. Final separation

```text
Semantic Engine
    determines meaning

Optimizer
    chooses equivalent logical form

Logical Planner
    lowers relational operations

Physical Planner
    selects physical strategy

Execution Planner
    builds execution topology

SQLite Compiler
    generates SQLite representation

Execution Engine
    executes commands

SQLite Query Planner
    determines native engine strategy

SQLite Engine
    performs storage/runtime execution
```

---

# 329. Resultado arquitectónico

```text
SQLiteSqlCompiler
=
GenericSqlCompilerPipeline
+
SQLiteTarget
+
SQLiteCapabilitySnapshot
+
SQLiteDialectProfile
+
SQLiteIdentifierSystem
+
SQLitePlaceholderSystem
+
SQLiteDynamicTypeAdapter
+
SQLiteAffinitySystem
+
SQLiteStrictTableAwareness
+
SQLiteBooleanAdapter
+
SQLiteDateTimeAdapter
+
SQLiteFunctionRegistry
+
SQLiteOperatorRegistry
+
SQLiteConflictResolutionSystem
+
SQLiteUpsertSystem
+
SQLiteReturningSystem
+
SQLiteRowIdentitySystem
+
SQLiteCteSystem
+
SQLiteWindowSystem
+
SQLiteCompoundQuerySystem
+
SQLiteJsonSystem
+
SQLiteVirtualTableExtensionSystem
+
SQLiteCollationSystem
+
DeterministicSqlGeneration
+
PersistentRuntimeIsolation
```

---

# 330. Arquitectura final

```text
CompilableDatabaseOperation
        │
        ▼
Generic SQL Compiler Pipeline
        │
        ▼
SQLite Target
├── Version
├── Capability Snapshot
├── Compile Options
├── Dialect Profile
├── Driver Compilation Profile
└── Extension Capability Set
        │
        ▼
SQLite Dialect Adaptation
        │
        ├── dynamic typing
        ├── type affinity
        ├── STRICT metadata
        ├── boolean representation
        ├── date/time representation
        ├── functions
        ├── operators
        ├── conflict resolution
        ├── UPSERT
        ├── RETURNING
        ├── ROWID
        ├── CTE
        ├── windows
        ├── compound queries
        ├── JSON
        ├── collations
        └── extensions
        │
        ▼
SQLite Representation Validation
        │
        ▼
Identifier/Alias Planning
        │
        ▼
Parameter Planning
        │
        ▼
Type/Affinity Planning
        │
        ▼
SQLite SQL Rendering
        │
        ├── RenderedSql
        ├── BindingLayout
        ├── ResultContract
        ├── ResultConversionPlan
        ├── SqlSourceMap
        ├── CompilationDependencies
        └── CompilationFingerprint
        │
        ▼
SQLiteCompiledDatabaseCommand
        │
        ▼
Prepared Statement / Compiled Cache
        │
        ▼
Execution Engine
        │
        ▼
SQLite Driver
        │
        ▼
SQLite
```

---

# 331. Principio final

La especialización SQLite deberá aceptar las características únicas del motor en lugar de intentar ocultarlas.

```text
SQLite is different by design.
```

Por ello:

```text
SQLite portability
=
Semantic Compatibility
+
Explicit Capability Modeling
+
SQLite-Aware Representation
```

y no:

```text
SQLite portability
=
Pretend SQLite behaves like PostgreSQL/MySQL
```

---

# 332. Fórmula arquitectónica final

```text
SQLite Compiler
=
Maximum Safe SQLite Capability
without
SQLite Semantics Leaking
into the Core Query Architecture
```

El resultado deberá permitir simultáneamente:

```text
simple portable queries
+
advanced SQLite workloads
+
embedded deployments
+
in-memory databases
+
persistent runtimes
+
strict semantic guarantees
```

manteniendo siempre:

```text
Compiler changes representation.
Compiler never changes meaning.
```

---

# 333. Estado del bloque SQL Compiler

Con este documento quedan definidos los cuatro compiladores SQL iniciales:

```text
69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
```

La arquitectura queda:

```text
                    SQL Compiler
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
      MySQL           MariaDB        PostgreSQL
        │                │                │
        └────────────────┼────────────────┘
                         │
                         ▼
                       SQLite
```

Conceptualmente todos comparten:

```text
Compiler Contracts
+
Compilation Pipeline
+
SQL Generation System
```

pero cada target mantiene:

```text
Dialect
Capabilities
Type Mapping
Representation Rules
Extensions
```

propios.

---

# 334. Siguiente documento

```text
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
```

El siguiente documento deberá formalizar el sistema mediante el cual paquetes oficiales, adapters y terceros podrán extender el SQL Compiler sin modificar su núcleo, incluyendo:

```text
compiler extension contracts
extension descriptors
extension registry
extension discovery
extension dependencies
extension conflicts
extension ordering
extension capabilities
statement extensions
expression extensions
predicate extensions
function extensions
operator extensions
type extensions
dialect extensions
emission-node extensions
platform-specific extensions
extension versioning
extension fingerprints
extension security boundaries
extension validation
persistent-runtime safety
deterministic registration
no-last-wins policy
unknown-extension failure
```

manteniendo como principio:

```text
Extensibility
≠
Arbitrary SQL Mutation
```

sino:

```text
Extensibility
=
Typed
+
Declared
+
Capability-Aware
+
Validated
+
Deterministic
+
Isolated
SQL Compiler Composition
```