# 70_DATABASE_MARIADB_SQL_COMPILER.md

# VoltStack Quantum Database
## MariaDB SQL Compiler

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 70 — MariaDB SQL Compiler  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`MariaDB SQL Compiler` define la especialización del SQL Compiler de VoltStack encargada de transformar operaciones compilables en SQL compatible con un target MariaDB concreto.

Su responsabilidad será adaptar la representación SQL común a:

- gramática MariaDB;
- capabilities MariaDB;
- versión objetivo;
- operadores;
- funciones;
- tipos;
- JSON;
- CTE;
- window functions;
- locking;
- pagination;
- mutations;
- UPSERT;
- `RETURNING` cuando la capability concreta lo permita;
- extensiones específicas de MariaDB.

El principio central será:

```text
MariaDB Compiler
=
Common SQL Compiler
+
MariaDB Target
+
MariaDB Capabilities
+
MariaDB Dialect Adaptation
+
MariaDB Rendering Rules
```

y nunca:

```text
MariaDB Compiler
=
MySQL Compiler
+
A Few Conditionals
```

---

# 2. Principio arquitectónico fundamental

Aunque MySQL y MariaDB comparten origen histórico y gran cantidad de sintaxis:

```text
MySQL ≠ MariaDB
```

VoltStack deberá tratarlos como plataformas distintas.

La arquitectura correcta será:

```text
                    Common SQL Compiler
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       MySQL Compiler             MariaDB Compiler
              │                         │
              ▼                         ▼
        MySqlTarget               MariaDbTarget
```

No:

```text
             MySqlCompiler
                   │
          if ($isMariaDb)
                   │
                   ▼
             special cases
```

---

# 3. Objetivos

El sistema deberá permitir:

1. reutilizar infraestructura realmente común entre MySQL y MariaDB;
2. aislar divergencias;
3. evitar condicionales por vendor dispersos;
4. modelar capabilities por versión;
5. detectar features incompatibles antes de ejecución;
6. generar SQL determinista;
7. mantener separación entre SQL y bindings;
8. soportar persistent runtimes;
9. permitir extensiones MariaDB-specific;
10. preservar portabilidad del Query Engine.

---

# 4. No existe un "MySQL-compatible compiler"

VoltStack no deberá construir una abstracción vaga:

```text
MysqlCompatibleCompiler
```

que termine absorbiendo:

```text
MySQL
MariaDB
Percona-specific behavior
vendor forks
```

sin fronteras claras.

En su lugar:

```text
Shared MySQL-family primitives
        │
        ├── MySQL specialization
        └── MariaDB specialization
```

---

# 5. Infraestructura compartible

Podrán compartirse componentes cuando exista equivalencia demostrable.

Ejemplos:

```text
backtick identifier quoting
common SELECT grammar
common basic JOIN forms
basic INSERT syntax
basic UPDATE syntax
basic DELETE syntax
common placeholder behavior
common CASE syntax
common aggregate syntax
common LIMIT syntax
```

---

# 6. Infraestructura no compartible automáticamente

No deberá asumirse equivalencia para:

```text
RETURNING
JSON semantics
UPSERT details
sequence support
generated columns
temporal/versioning features
optimizer hints
system-versioned tables
type system details
functions
window features
CTE details
DDL
replication-related syntax
locking variants
version-specific features
```

---

# 7. Shared family layer

Se propone una capa interna:

```text
VoltStack\Quantum\Database\Query\Compiler\Platform\MySqlFamily
```

No será una plataforma pública.

Será infraestructura reusable.

---

# 8. Estructura conceptual

```text
SQL Compiler
    │
    ▼
MySqlFamily
├── common identifiers
├── common literals
├── common basic clauses
├── common expression renderers
└── common utility contracts
    │
    ├───────────────┐
    ▼               ▼
MySQL            MariaDB
```

---

# 9. Regla de dependencia

Permitido:

```text
MySQL Compiler
    ↓
MySqlFamily

MariaDB Compiler
    ↓
MySqlFamily
```

No permitido:

```text
MariaDB Compiler
    ↓
MySQL Compiler
```

---

# 10. Target MariaDB

```php
final readonly class MariaDbTarget
{
    public function __construct(
        public MariaDbVersion $version,
        public MariaDbCapabilitySnapshot $capabilities,
        public MariaDbDialectProfile $dialect,
    ) {}
}
```

---

# 11. Target identity

El target deberá distinguir explícitamente:

```text
family = MARIADB
```

de:

```text
family = MYSQL
```

aunque compartan driver protocol o infraestructura PDO.

---

# 12. Driver ≠ platform

Una conexión puede usar:

```text
PDO MySQL driver
```

para conectarse a MariaDB.

Eso no significa:

```text
Platform = MySQL
```

Por tanto:

```text
Driver
≠
Database Platform
≠
SQL Dialect
```

---

# 13. Ejemplo

```text
Driver:
PDO_MYSQL

Server:
MariaDB

Platform:
MARIADB

Compiler:
MariaDbSqlCompiler
```

---

# 14. MariaDbVersion

```php
final readonly class MariaDbVersion
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
    ) {}
}
```

---

# 15. Version comparisons

Las comparaciones serán estructuradas:

```php
$version->isAtLeast(11, 0, 0);
```

No:

```php
$version >= '11.0';
```

---

# 16. Version ≠ capability

El compiler no deberá basarse principalmente en:

```text
if version >= X
```

sino en:

```text
MariaDbCapabilitySnapshot
```

---

# 17. Capability resolution

```text
MariaDB Version
+
Configuration
+
Server Discovery
+
Driver Knowledge
+
Explicit Overrides
        │
        ▼
MariaDbCapabilityResolver
        │
        ▼
MariaDbCapabilitySnapshot
```

---

# 18. No server discovery durante compilación

El compiler no deberá ejecutar:

```sql
SELECT VERSION()
```

ni consultar:

```text
information_schema
performance_schema
server variables
```

durante compilación.

---

# 19. Discovery boundary

```text
Connection / Platform Discovery
        │
        ▼
Capability Snapshot
        │
        ▼
Compiler
```

---

# 20. MariaDbCapabilitySnapshot

Conceptualmente:

```php
final readonly class MariaDbCapabilitySnapshot
{
    public function __construct(
        public MariaDbVersion $version,
        public MariaDbFeatureSet $features,
        public MariaDbSqlModeProfile $sqlModes,
        public CapabilityFingerprint $fingerprint,
    ) {}
}
```

---

# 21. Capability model

Deberá responder preguntas como:

```text
supports CTE?
supports recursive CTE?
supports window functions?
supports RETURNING for INSERT?
supports RETURNING for DELETE?
supports RETURNING for UPDATE?
supports sequences?
supports system-versioned tables?
supports JSON operation X?
supports SKIP LOCKED?
supports NOWAIT?
supports INTERSECT?
supports EXCEPT?
supports specific UPSERT representation?
```

---

# 22. Structured capabilities

Para features complejas:

```php
final readonly class MariaDbReturningCapability
{
    public function __construct(
        public CapabilityStatus $insert,
        public CapabilityStatus $update,
        public CapabilityStatus $delete,
    ) {}
}
```

---

# 23. Capability granularity

Evitar:

```text
supportsReturning = true
```

si el soporte difiere por statement kind.

Preferir:

```text
INSERT → supported
UPDATE → unsupported/restricted
DELETE → supported
```

según el target real.

---

# 24. Capability status

```text
SUPPORTED
SUPPORTED_WITH_RESTRICTIONS
SUPPORTED_WITH_ADAPTATION
UNSUPPORTED
EXTENSION_REQUIRED
```

---

# 25. Emulation classification

Cada feature puede declarar:

```text
NATIVE
COMPILER_EMULATED
EXECUTION_EMULATED
NOT_EMULATABLE
```

---

# 26. Compiler emulation

Sólo cuando:

```text
one semantic operation
→ one equivalent SQL representation
```

sin runtime orchestration.

---

# 27. Execution emulation

Cuando requiera:

```text
multiple statements
runtime feedback
temporary state
transaction orchestration
```

pertenecerá a:

```text
Execution Plan
```

---

# 28. MariaDB dialect profile

```php
final readonly class MariaDbDialectProfile
{
    public function __construct(
        public IdentifierQuoteStyle $identifierQuotes,
        public MariaDbPlaceholderPolicy $placeholders,
        public MariaDbStringLiteralPolicy $stringLiterals,
        public MariaDbPaginationPolicy $pagination,
        public MariaDbFunctionProfile $functions,
    ) {}
}
```

---

# 29. SQL modes

MariaDB también posee modos SQL capaces de afectar interpretación.

VoltStack deberá modelar los relevantes mediante:

```text
MariaDbSqlModeProfile
```

---

# 30. No global sql_mode

Incorrecto:

```php
MariaDbDialect::$sqlMode = ...;
```

Correcto:

```text
MariaDbCompilationContext
    └── MariaDbTarget
        └── MariaDbSqlModeProfile
```

---

# 31. Compilation context

```php
final readonly class MariaDbCompilationContext
{
    public function __construct(
        public MariaDbTarget $target,
        public MariaDbCapabilitySnapshot $capabilities,
        public CompilationOptions $options,
        public SqlRenderingProfile $rendering,
        public CompilationBudget $budget,
    ) {}
}
```

---

# 32. No runtime resources

No contendrá:

```text
PDO
Connection
Transaction
Statement
Cursor
Result
EntityManager
```

---

# 33. Compiler specialization

```php
interface MariaDbCompiler
{
    public function compile(
        CompilableDatabaseOperation $operation,
        MariaDbCompilationContext $context,
    ): CompiledDatabaseCommand;
}
```

---

# 34. Common pipeline

El MariaDB Compiler deberá reutilizar:

```text
67_DATABASE_SQL_COMPILER_PIPELINE
```

mediante componentes especializados.

---

# 35. Pipeline

```text
CompilableDatabaseOperation
        │
        ▼
Input Validation
        │
        ▼
Generic SQL Lowering
        │
        ▼
MariaDB Dialect Adaptation
        │
        ▼
MariaDB Representation Validation
        │
        ▼
Alias Planning
        │
        ▼
Placeholder Planning
        │
        ▼
MariaDB SQL Generation
        │
        ▼
Binding Compilation
        │
        ▼
Source Map
        │
        ▼
CompiledDatabaseCommand
```

---

# 36. No compiler fork

El pipeline MariaDB no deberá ser una copia del pipeline MySQL.

---

# 37. Dialect adapter

```php
interface MariaDbDialectAdapter
{
    public function adapt(
        SqlEmissionTree $tree,
        MariaDbCompilationContext $context,
    ): SqlEmissionTree;
}
```

---

# 38. Adapter responsibilities

Podrá adaptar:

```text
functions
operators
pagination
RETURNING
UPSERT
JSON
locking
set operations
temporal constructs
MariaDB extensions
```

---

# 39. Adapter ≠ renderer

El adapter decide:

```text
what MariaDB representation should be used
```

El renderer decide:

```text
how that representation is serialized
```

---

# 40. Identifier quoting

MariaDB soporta normalmente:

```sql
`identifier`
```

---

# 41. Qualified identifiers

```sql
`users`.`id`
```

---

# 42. Aliases

```sql
`users` AS `u`
```

---

# 43. Shared identifier renderer

Si las reglas son equivalentes, podrá reutilizarse:

```text
MySqlFamilyIdentifierRenderer
```

---

# 44. Platform ownership

Aunque se comparta implementación:

```text
MariaDB compiler
```

seguirá siendo propietario del comportamiento mediante su dialect profile.

---

# 45. No implementation identity assumption

```text
same implementation today
```

no implica:

```text
same contract forever
```

---

# 46. Identifier escaping

Los backticks internos serán escapados mediante un componente central.

---

# 47. Reserved words

La estrategia preferida será quoting sistemático de identifiers generados.

---

# 48. Placeholder rendering

La representación típica será:

```sql
?
```

---

# 49. Parameter identity

Se mantendrá separada:

```text
ParameterId
ParameterOccurrenceId
BindingSlot
```

---

# 50. Repeated parameter

```text
x = P1 OR y = P1
```

puede producir:

```sql
`x` = ? OR `y` = ?
```

con:

```text
slot 0 → P1
slot 1 → P1
```

---

# 51. SQL ≠ bindings

```text
RenderedSql
+
BindingLayout
```

serán artifacts distintos.

---

# 52. Literal rendering

El compiler deberá proporcionar renderers para:

```text
NULL
boolean
numeric
string compile-time literals
binary literals where applicable
temporal literals where appropriate
```

---

# 53. Runtime values

Runtime values deberán utilizar bindings salvo excepción estructural documentada.

---

# 54. String escaping

No deberá depender de supuestos no declarados sobre SQL mode.

---

# 55. SELECT

Ejemplo:

```sql
SELECT
    `u`.`id`,
    `u`.`email`
FROM `users` AS `u`
WHERE `u`.`active` = ?
ORDER BY `u`.`id` ASC
LIMIT 20
```

---

# 56. SELECT DISTINCT

```sql
SELECT DISTINCT
    `country`
FROM `users`
```

---

# 57. FROM

Sources:

```text
table
derived table
CTE
subquery
extension source
```

---

# 58. Derived tables

```sql
FROM (
    SELECT ...
) AS `q1`
```

---

# 59. Alias planning

Alias assignment ocurrirá antes del rendering.

---

# 60. JOIN

Soporte básico:

```text
INNER
LEFT
RIGHT
CROSS
```

según capabilities.

---

# 61. INNER JOIN

```sql
INNER JOIN `orders` AS `o`
    ON `o`.`user_id` = `u`.`id`
```

---

# 62. LEFT JOIN

```sql
LEFT JOIN `profiles` AS `p`
    ON `p`.`user_id` = `u`.`id`
```

---

# 63. FULL OUTER JOIN

No deberá emitirse salvo que el target capability lo permita.

---

# 64. No MySQL assumption

El hecho de que MySQL no soporte determinada construcción no será usado como prueba de que MariaDB tampoco la soporte, ni viceversa.

---

# 65. Semi/anti joins

Las formas internas:

```text
SEMI_JOIN
ANTI_JOIN
```

podrán adaptarse mediante:

```text
EXISTS
NOT EXISTS
```

u otra representación semánticamente equivalente.

---

# 66. No renderer optimization

El renderer no decidirá transformar joins.

---

# 67. WHERE

```sql
WHERE `status` = ?
```

---

# 68. SQL three-valued logic

Se preservará:

```text
TRUE
FALSE
UNKNOWN
```

---

# 69. NULL

```sql
IS NULL
IS NOT NULL
```

serán representaciones explícitas.

---

# 70. Null-safe equality

Si MariaDB proporciona una operación compatible y VoltStack posee una operación semántica equivalente:

```text
NullSafeEqual
```

podrá adaptarse explícitamente.

---

# 71. Ordinary equality

Nunca se cambiará automáticamente:

```text
=
```

por una variante null-safe.

---

# 72. GROUP BY

```sql
GROUP BY
    `department_id`,
    `status`
```

---

# 73. Grouping semantics

VoltStack no dependerá de modos permisivos del servidor para aceptar queries semánticamente ambiguas.

---

# 74. HAVING

```sql
HAVING COUNT(*) > ?
```

---

# 75. Aggregate functions

Funciones comunes podrán compartir descriptors con `MySqlFamily` cuando exista equivalencia.

---

# 76. MariaDB-specific functions

Deberán registrarse mediante:

```text
MariaDbFunctionRegistry
```

---

# 77. Function registry

```php
interface MariaDbFunctionRegistry
{
    public function resolve(
        SemanticFunctionId $function,
        MariaDbTarget $target,
    ): MariaDbFunctionDescriptor;
}
```

---

# 78. Function portability

```text
semantic function
≠
vendor function name
```

---

# 79. ORDER BY

```sql
ORDER BY
    `created_at` DESC,
    `id` ASC
```

---

# 80. NULL ordering

El compiler sólo emitirá sintaxis nativa cuando el target la soporte.

---

# 81. Emulated ordering

Si se requiere emulación:

```text
semantic ordering
→ dialect adaptation
→ explicit SQL expression ordering
```

---

# 82. LIMIT

Forma típica:

```sql
LIMIT 20
```

---

# 83. OFFSET

Representación canónica:

```sql
LIMIT 20 OFFSET 40
```

cuando sea compatible.

---

# 84. Pagination strategy

La estrategia será definida mediante:

```text
MariaDbPaginationPolicy
```

---

# 85. No magic unlimited value

El renderer no inventará automáticamente un número máximo para representar OFFSET-only.

---

# 86. LIMIT without ORDER

No implica ordering.

---

# 87. Window functions

Cuando capabilities lo permitan:

```sql
ROW_NUMBER() OVER (
    PARTITION BY `department_id`
    ORDER BY `created_at`
)
```

---

# 88. Window representation

Deberá conservar:

```text
partitioning
ordering
frame
exclusion when supported
function semantics
```

---

# 89. Window order ≠ query order

```text
Window ordering
≠
Observable final ordering
```

---

# 90. CTE

```sql
WITH `active_users` AS (
    SELECT ...
)
SELECT ...
```

cuando sea soportado.

---

# 91. Recursive CTE

```sql
WITH RECURSIVE `tree` AS (
    ...
)
SELECT ...
```

---

# 92. Structured recursion

Se preservarán:

```text
anchor
recursive member
set operation
binding
termination semantics
```

---

# 93. CTE materialization

No se asumirá que:

```text
CTE
=
materialized temporary relation
```

---

# 94. Subqueries

Soporte:

```text
scalar
EXISTS
IN
derived
correlated
```

---

# 95. Scalar cardinality

La semántica:

```text
0 rows → NULL
1 row → scalar
>1 rows → error
```

deberá preservarse cuando corresponda.

---

# 96. Correlation

Outer references estarán resueltas antes del compiler.

---

# 97. Set operations

Las capabilities deberán determinar soporte para:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

y variantes disponibles.

---

# 98. UNION

```sql
SELECT ...
UNION
SELECT ...
```

---

# 99. UNION ALL

```sql
SELECT ...
UNION ALL
SELECT ...
```

---

# 100. INTERSECT/EXCEPT

Una diferencia de soporte respecto a MySQL deberá ser expresada mediante:

```text
MariaDbCapabilitySnapshot
```

y no mediante una excepción dispersa en Query Builder.

---

# 101. Set-operation multiplicity

Se preservará:

```text
ALL
DISTINCT
```

---

# 102. INSERT

```sql
INSERT INTO `users` (
    `name`,
    `email`
)
VALUES (?, ?)
```

---

# 103. Multi-row insert

```sql
INSERT INTO `users` (
    `name`,
    `email`
)
VALUES
    (?, ?),
    (?, ?)
```

---

# 104. INSERT SELECT

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

# 105. MariaDB RETURNING

MariaDB-specific `RETURNING` capabilities deberán modelarse independientemente de MySQL.

Este punto constituye una de las razones principales para separar ambos compilers.

---

# 106. Mutation capability matrix

Conceptualmente:

```text
                     MariaDB Target
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
INSERT RETURNING    UPDATE RETURNING    DELETE RETURNING
        │                 │                 │
   capability         capability         capability
```

---

# 107. No universal RETURNING boolean

Incorrecto:

```text
supportsReturning = true
```

Correcto:

```text
ReturningCapability
├── INSERT
├── UPDATE
└── DELETE
```

---

# 108. RETURNING output contract

Cuando sea soportado:

```text
Mutation
+
ReturningProjection
        │
        ▼
MariaDB Mutation Statement
        │
        ▼
Result Contract
```

---

# 109. Result contract

Deberá conservar:

```text
columns
types
nullability
ordering when observable
result shape
```

---

# 110. Unsupported RETURNING

Debe producir:

```text
UnsupportedMariaDbFeatureException
```

o activar una estrategia de ejecución explícitamente definida.

Nunca se eliminará silenciosamente.

---

# 111. RETURNING emulation

Si requiere:

```text
mutation
+
follow-up query
```

será responsabilidad del:

```text
Execution Plan
```

---

# 112. INSERT generated identifiers

La obtención de generated IDs no será considerada automáticamente equivalente a `RETURNING`.

---

# 113. UPSERT

MariaDB comparte sintaxis histórica de:

```sql
INSERT ... ON DUPLICATE KEY UPDATE
```

pero sus capacidades concretas deberán modelarse por separado.

---

# 114. Semantic upsert

VoltStack modelará:

```text
UpsertIntent
├── target
├── insert values
├── conflict semantics
└── update behavior
```

---

# 115. MariaDB adaptation

```text
UpsertIntent
        │
        ▼
MariaDbUpsertAdapter
        │
        ▼
MariaDbUpsertClause
```

---

# 116. No cross-platform equivalence assumption

```text
ON DUPLICATE KEY UPDATE
≠
ON CONFLICT
```

como abstracciones universales.

---

# 117. Inserted-row references

La forma concreta deberá provenir de:

```text
MariaDbUpsertCapability
```

---

# 118. UPDATE

```sql
UPDATE `users`
SET `status` = ?
WHERE `id` = ?
```

---

# 119. UPDATE joins

Las formas MariaDB-specific deberán modelarse mediante statement descriptors propios.

---

# 120. UPDATE ORDER/LIMIT

Serán capability-gated.

---

# 121. DELETE

```sql
DELETE FROM `users`
WHERE `id` = ?
```

---

# 122. DELETE extensions

Multi-table delete, aliases, ordering y limits deberán modelarse según el target.

---

# 123. DELETE RETURNING

Si la versión/capability soporta la semántica requerida:

```text
DeleteMutation
+
ReturningProjection
        │
        ▼
MariaDbDeleteReturningRepresentation
```

---

# 124. Mutation renderer separation

Se recomienda:

```text
MariaDbInsertRenderer
MariaDbUpdateRenderer
MariaDbDeleteRenderer
MariaDbReturningRenderer
MariaDbUpsertRenderer
```

---

# 125. JSON

MariaDB deberá tener un subsystem de adaptación JSON independiente del de MySQL.

---

# 126. Razón

Aunque ambos sistemas proporcionen APIs SQL similares:

```text
storage semantics
type behavior
functions
operators
validation
comparison
return types
```

pueden diferir.

---

# 127. Semantic JSON model

```text
JsonExtract
JsonContains
JsonSet
JsonRemove
JsonMerge
JsonScalar
JsonDocument
```

serán conceptos VoltStack.

---

# 128. MariaDB JSON adaptation

```text
Semantic JSON Operation
        │
        ▼
MariaDbJsonAdapter
        │
        ▼
MariaDbJsonExpression
```

---

# 129. JSON ≠ string

VoltStack no deberá reducir automáticamente:

```text
JSON
```

a:

```text
STRING
```

en el Query Type System sólo porque cierta implementación física utilice representación textual.

---

# 130. JSON path security

Paths dinámicos deberán pasar por parameterization/structured validation según el contrato de la función.

---

# 131. Temporal capabilities

MariaDB posee características temporales que pueden diferir significativamente de MySQL.

Deberán exponerse mediante:

```text
MariaDbTemporalCapability
```

---

# 132. System-versioned tables

Si VoltStack soporta consultas temporales específicas de MariaDB, deberán entrar mediante:

```text
Database Temporal Extension
        │
        ▼
MariaDbTemporalAdapter
```

---

# 133. Portable temporal core

El futuro:

```text
267_DATABASE_TEMPORAL_DATA_SYSTEM.md
```

definirá la semántica portable/general.

El compiler sólo traducirá la representación que le corresponda.

---

# 134. No temporal semantics in renderer

El renderer no decidirá:

```text
which historical version should be queried
```

---

# 135. Sequences

Si el target MariaDB ofrece sequence capabilities relevantes, deberán modelarse explícitamente.

---

# 136. Sequence abstraction

```text
SequenceReference
SequenceNextValue
SequenceCurrentValue
```

podrán formar parte de una extensión/capability.

---

# 137. Sequence ≠ auto increment

```text
SEQUENCE
≠
AUTO_INCREMENT
```

---

# 138. Portable generated value model

El ORM/Persistence System deberá trabajar con abstracciones superiores.

---

# 139. Date/time functions

Funciones específicas deberán mapearse mediante:

```text
MariaDbFunctionRegistry
```

---

# 140. Current time

No sustituir:

```text
database current timestamp
```

por:

```text
PHP current timestamp
```

durante compilación.

---

# 141. Interval expressions

Serán structured:

```text
MariaDbIntervalExpression
├── value
└── unit
```

---

# 142. Units

Las unidades serán typed.

---

# 143. String concatenation

La abstracción semántica:

```text
Concat(a,b)
```

se adaptará a una representación válida MariaDB.

---

# 144. No operator assumption

No asumir que:

```text
||
```

tiene universalmente la semántica de concatenación bajo cualquier SQL mode.

---

# 145. CAST

```text
QueryType
        │
        ▼
MariaDbTypeResolution
        │
        ▼
MariaDbSqlType
        │
        ▼
MariaDbTypeRenderer
```

---

# 146. Type system divergence

MariaDB types deberán tener descriptors propios cuando diverjan de MySQL.

---

# 147. Shared type descriptor

Sólo se compartirá un type descriptor si:

```text
syntax
+
conversion
+
comparison
+
binding expectations
```

son compatibles para el uso relevante.

---

# 148. Collation

Las collations serán metadata estructurada.

---

# 149. No arbitrary collation interpolation

Incorrecto:

```php
$sql .= ' COLLATE ' . $userInput;
```

---

# 150. Locking

El sistema deberá modelar:

```text
lock mode
lock modifiers
statement compatibility
transaction requirement
```

---

# 151. Lock representation

Conceptualmente:

```text
MariaDbLockClause
├── mode
├── wait policy
└── target restrictions
```

---

# 152. NOWAIT/SKIP LOCKED

Serán capability-gated.

---

# 153. Compiler does not transact

El compiler no:

```text
BEGIN
COMMIT
ROLLBACK
```

como side effect.

---

# 154. Transaction requirements

Puede producir metadata:

```text
ExecutionTransactionRequirement
```

para la capa posterior.

---

# 155. Optimizer hints

Las hints MariaDB no deberán reutilizarse automáticamente desde MySQL.

---

# 156. Structured MariaDB hints

Si se soportan:

```text
MariaDbStructuredHint
```

---

# 157. Physical intent

```text
Physical Planning Intent
        │
        ▼
MariaDbHintAdapter
        │
        ▼
MariaDb Hint Representation
```

---

# 158. Hint enforceability

Toda hint deberá declarar:

```text
GUARANTEED
REQUESTED
ADVISORY
OBSERVABLE_ONLY
UNSUPPORTED
```

según el capability model correspondiente.

---

# 159. Database optimizer authority

Para operaciones delegadas:

```text
VoltStack
    │
    ▼
MariaDB SQL
    │
    ▼
MariaDB Optimizer
    │
    ▼
Actual Runtime Plan
```

---

# 160. VoltStack plan ≠ native plan

```text
VoltStack PhysicalQueryPlan
≠
MariaDB native execution plan
```

---

# 161. EXPLAIN

El compiler no ejecutará `EXPLAIN`.

---

# 162. Explain operation

Puede existir:

```text
ExplainQueryOperation
```

como operación explícita compilable.

---

# 163. Runtime explain

Obtener el plan real requerirá I/O y pertenecerá a diagnostics/telemetry.

---

# 164. No optimizer feedback hidden in compiler

Nunca:

```text
compile
→ EXPLAIN
→ alter SQL
→ compile again
```

como comportamiento implícito.

---

# 165. MariaDB representation validator

```php
interface MariaDbRepresentationValidator
{
    public function validate(
        SqlEmissionTree $tree,
        MariaDbCompilationContext $context,
    ): void;
}
```

---

# 166. Validation

Debe verificar:

```text
feature support
statement shape
clause compatibility
type representation
placeholder compatibility
locking
RETURNING
UPSERT
JSON
temporal features
extensions
```

---

# 167. Validation boundary

No volverá a realizar Semantic Analysis.

---

# 168. Known unsupported feature

Debe fallar durante:

```text
DIALECT_VALIDATION
```

y no esperar al servidor.

---

# 169. Diagnostic example

```text
MariaDB SQL compilation failed.

Feature:
MUTATION_RETURNING

Statement:
UPDATE

Target:
MariaDB

Capability:
UNSUPPORTED

Source:
MutationNode M17

Phase:
MARIADB_REPRESENTATION_VALIDATION
```

---

# 170. MariaDbFeatureRegistry

```php
interface MariaDbFeatureRegistry
{
    public function descriptor(
        DatabaseFeatureId $feature,
    ): MariaDbFeatureDescriptor;
}
```

---

# 171. Feature descriptor

```php
final readonly class MariaDbFeatureDescriptor
{
    public function __construct(
        public DatabaseFeatureId $id,
        public CapabilityStatus $status,
        public ?MariaDbVersionRange $versions,
        public MariaDbRenderingStrategy $strategy,
    ) {}
}
```

---

# 172. No switch gigante

Evitar:

```php
switch ($feature) {
    // hundreds of cases
}
```

como arquitectura central.

---

# 173. Registry lifecycle

```text
bootstrap
→ discover
→ validate
→ resolve conflicts
→ freeze
```

---

# 174. Extension model

MariaDB extensions podrán registrar:

```text
functions
operators
types
expressions
clauses
hints
temporal constructs
JSON operations
compiler adapters
renderers
```

---

# 175. Extension contract

```php
interface MariaDbCompilerExtension
{
    public function descriptor(): MariaDbExtensionDescriptor;
}
```

---

# 176. Extension compatibility

Descriptor deberá declarar:

```text
MariaDB version range
required capabilities
compiler API version
dependencies
conflicts
priority/order where meaningful
```

---

# 177. No last-wins

Dos extensiones que reclamen la misma feature incompatible deberán producir conflicto explícito.

---

# 178. No raw fallback

Unknown extension node:

```text
→ compilation error
```

No:

```text
→ stringify object
```

---

# 179. Source mapping

El compiler conservará:

```text
SqlSourceMap
```

---

# 180. Source mapping metadata

Podrá incluir:

```text
MariaDbFeatureId
MariaDbDialectRuleId
MariaDbCapabilityId
MariaDbExtensionId
```

---

# 181. Compiled artifact

Resultado conceptual:

```php
final readonly class MariaDbCompiledDatabaseCommand
{
    public function __construct(
        public RenderedSql $sql,
        public BindingLayout $bindings,
        public ResultContract $result,
        public SqlSourceMap $sourceMap,
        public CompilationDependencies $dependencies,
        public CompilationFingerprint $fingerprint,
    ) {}
}
```

---

# 182. No runtime values

El artifact compilado no incluirá runtime parameter values.

---

# 183. Fingerprint

Deberá considerar:

```text
input/plan identity
MariaDB target identity
capability fingerprint
dialect profile
compiler version
extension set
alias plan
placeholder plan
canonical rendering version
specialization key
```

---

# 184. Formula

```text
MariaDbCompiledFingerprint
=
InputFingerprint
+
MariaDbTargetFingerprint
+
CapabilityFingerprint
+
CompilerVersion
+
DialectVersion
+
ExtensionFingerprint
+
SpecializationFingerprint
```

---

# 185. Runtime values excluded

No incluir:

```text
email value
tenant id value
timestamp now
current connection id
transaction id
PDO handle
```

---

# 186. Parameter-sensitive planning

Si el Physical Plan fue especializado:

```text
PhysicalPlanSpecializationKey
```

deberá propagarse.

---

# 187. Dependencies

El artifact podrá depender de:

```text
relations
columns
types
functions
collations
extensions
platform capabilities
physical access descriptors
```

---

# 188. Hard vs soft dependencies

Ejemplo:

```text
removed column
→ hard invalidation

statistics changed
→ usually physical-plan reconsideration

compiler version changed
→ compiled artifact invalidation
```

---

# 189. Compiled query cache

El documento:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

formalizará cache/invalidation.

---

# 190. MariaDB/MySQL cache isolation

Nunca:

```text
same SQL text
→ same cache artifact
```

si targets diferentes participaron.

---

# 191. Reason

Aunque el SQL textual coincida:

```text
target capabilities
binding behavior
result semantics
dependencies
compiler version
```

pueden diferir.

---

# 192. Persistent runtime safety

El MariaDB compiler deberá ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
long-running CLI workers
queue workers
```

---

# 193. Shared state

Puede compartirse:

```text
frozen registries
immutable descriptors
dialect definitions
renderer definitions
feature metadata
```

---

# 194. Operation-local state

Debe permanecer local:

```text
compilation context
rendering session
alias planner state
placeholder state
binding builder
source map builder
diagnostics
temporary buffers
```

---

# 195. No static current target

Prohibido:

```php
MariaDbCompiler::$currentVersion
```

---

# 196. Multi-target process

Debe ser seguro:

```text
Request A
→ MariaDB target A

Request B
→ MySQL target

Request C
→ MariaDB target B
```

en el mismo worker.

---

# 197. Determinism

Dados:

```text
same input
same target
same capabilities
same extensions
same compiler version
same options
```

deberán producirse:

```text
same canonical SQL
same binding layout
same semantic source map
same fingerprint
```

---

# 198. Serialization

Los descriptors cacheables no deberán contener:

```text
closures
connections
PDO
statements
cursors
services
fibers
promises
live schema objects
```

---

# 199. Compiler factory

```php
interface SqlCompilerResolver
{
    public function resolve(
        DatabaseTarget $target,
    ): SqlCompiler;
}
```

---

# 200. Resolution

```text
MYSQL
    → MySqlCompiler

MARIADB
    → MariaDbCompiler

POSTGRESQL
    → PostgreSqlCompiler

SQLITE
    → SqliteCompiler
```

---

# 201. Driver resolution separate

```text
Compiler resolution
≠
Connection driver resolution
```

---

# 202. Example

```text
MariaDB
├── connection protocol → PDO_MYSQL
└── SQL compiler        → MARIADB
```

---

# 203. Testing strategy

Debe incluir:

```text
unit
golden SQL
capability matrix
version profiles
integration
real MariaDB server
cross-platform
security
persistent worker
determinism
serialization
```

---

# 204. Golden tests

Ejemplo:

Input:

```text
Select users.id where users.active = P1
```

Expected:

```sql
SELECT `users`.`id` FROM `users` WHERE `users`.`active`=?
```

según canonical alias policy.

---

# 205. RETURNING tests

Deberán ser especialmente exhaustivos.

Matriz:

```text
target version
×
statement kind
×
returning projection
×
prepared mode
```

---

# 206. UPSERT tests

Cubrir:

```text
single row
multi row
duplicate path
non-duplicate path
inserted value references
NULL values
unique keys
generated values
RETURNING interaction where supported
```

---

# 207. JSON tests

Cubrir:

```text
extraction
scalar conversion
containment
mutation
NULL
JSON null
SQL NULL
paths
comparison
binding
```

---

# 208. Critical JSON distinction

Tests deberán distinguir:

```text
SQL NULL
≠
JSON null
```

---

# 209. Temporal tests

Cuando features temporales sean soportadas:

```text
current data
historical data
time ranges
system-versioned syntax
parameter binding
```

---

# 210. Sequence tests

Si están soportadas por el target profile:

```text
next value
current value
multiple references
transaction behavior where relevant
```

---

# 211. Locking tests

Cubrir:

```text
FOR UPDATE
share modes
NOWAIT
SKIP LOCKED
unsupported combinations
transaction requirement metadata
```

---

# 212. Capability-negative tests

Cada feature importante deberá probar:

```text
supported target
unsupported target
restricted target
```

---

# 213. Cross-family tests

Cuando MySQL y MariaDB compartan comportamiento:

```text
same semantic input
→ equivalent result semantics
```

---

# 214. Divergence tests

Cuando difieran:

```text
same semantic input
→ different valid SQL
```

o:

```text
MariaDB → supported
MySQL   → unsupported
```

deberá ser una expectativa explícita.

---

# 215. No shared golden-file assumption

No mantener un único archivo:

```text
mysql_family_expected.sql
```

para features que pueden divergir.

---

# 216. Real server testing

El CI deberá poder ejecutar suites contra las versiones MariaDB oficialmente soportadas.

---

# 217. Version policy

Las versiones soportadas concretas pertenecerán a:

```text
compatibility metadata
release policy
CI matrix
```

y no quedarán eternamente fijadas en este documento.

---

# 218. Security testing

Cubrir:

```text
value injection
identifier injection
collation injection
JSON path injection
raw expressions
extension renderers
optimizer hints
temporal syntax
RETURNING expressions
```

---

# 219. Benchmarking

Medir:

```text
compilation latency
dialect adaptation
SQL rendering
binding layout
source map generation
memory allocation
cache lookup
```

---

# 220. No live server benchmark for compiler core

El compiler benchmark puro no requerirá MariaDB activo.

---

# 221. Integration benchmark

Separadamente:

```text
compile
prepare
bind
execute
fetch
```

podrá medirse end-to-end.

---

# 222. Directory structure

```text
VoltStack/Quantum/Database/Query/Compiler/Platform
├── MySqlFamily
│   ├── Identifier
│   ├── Literal
│   ├── Placeholder
│   ├── Expression
│   ├── Clause
│   └── Support
│
├── MySql
│   └── ...
│
└── MariaDb
    ├── MariaDbCompiler.php
    ├── MariaDbCompilerFactory.php
    │
    ├── Target
    │   ├── MariaDbTarget.php
    │   ├── MariaDbVersion.php
    │   ├── MariaDbVersionRange.php
    │   └── MariaDbTargetFingerprint.php
    │
    ├── Capability
    │   ├── MariaDbCapabilitySnapshot.php
    │   ├── MariaDbCapabilityResolver.php
    │   ├── MariaDbFeatureDescriptor.php
    │   ├── MariaDbFeatureRegistry.php
    │   ├── MariaDbReturningCapability.php
    │   ├── MariaDbUpsertCapability.php
    │   ├── MariaDbJsonCapability.php
    │   ├── MariaDbTemporalCapability.php
    │   ├── MariaDbSequenceCapability.php
    │   └── MariaDbLockingCapability.php
    │
    ├── Dialect
    │   ├── MariaDbDialectProfile.php
    │   ├── MariaDbDialectAdapter.php
    │   ├── MariaDbSqlModeProfile.php
    │   └── MariaDbDialectRenderingContract.php
    │
    ├── Validation
    │   ├── MariaDbRepresentationValidator.php
    │   ├── MariaDbFeatureValidator.php
    │   ├── MariaDbStatementValidator.php
    │   └── MariaDbCapabilityValidator.php
    │
    ├── Identifier
    │   └── MariaDbIdentifierRenderer.php
    │
    ├── Placeholder
    │   ├── MariaDbPlaceholderRenderer.php
    │   └── MariaDbPlaceholderPolicy.php
    │
    ├── Literal
    │   ├── MariaDbLiteralRenderer.php
    │   ├── MariaDbBooleanRenderer.php
    │   └── MariaDbStringLiteralRenderer.php
    │
    ├── Statement
    │   ├── MariaDbSelectRenderer.php
    │   ├── MariaDbInsertRenderer.php
    │   ├── MariaDbUpdateRenderer.php
    │   └── MariaDbDeleteRenderer.php
    │
    ├── Clause
    │   ├── MariaDbPaginationRenderer.php
    │   ├── MariaDbLockingRenderer.php
    │   ├── MariaDbReturningRenderer.php
    │   ├── MariaDbUpsertRenderer.php
    │   ├── MariaDbCteRenderer.php
    │   ├── MariaDbWindowRenderer.php
    │   └── MariaDbSetOperationRenderer.php
    │
    ├── Function
    │   ├── MariaDbFunctionRegistry.php
    │   └── MariaDbFunctionDescriptor.php
    │
    ├── Operator
    │   ├── MariaDbOperatorRegistry.php
    │   └── MariaDbOperatorDescriptor.php
    │
    ├── Json
    │   ├── MariaDbJsonAdapter.php
    │   ├── MariaDbJsonRenderer.php
    │   └── MariaDbJsonFunctionRegistry.php
    │
    ├── Temporal
    │   ├── MariaDbTemporalAdapter.php
    │   ├── MariaDbTemporalRenderer.php
    │   └── MariaDbTemporalDescriptor.php
    │
    ├── Sequence
    │   ├── MariaDbSequenceRenderer.php
    │   └── MariaDbSequenceDescriptor.php
    │
    ├── Type
    │   ├── MariaDbTypeResolver.php
    │   ├── MariaDbTypeRenderer.php
    │   └── MariaDbTypeDescriptor.php
    │
    ├── Hint
    │   ├── MariaDbHintAdapter.php
    │   ├── MariaDbHintRenderer.php
    │   └── MariaDbStructuredHint.php
    │
    ├── Extension
    │   ├── MariaDbCompilerExtension.php
    │   ├── MariaDbCompilerExtensionRegistry.php
    │   └── MariaDbExtensionDescriptor.php
    │
    ├── Diagnostic
    │   ├── MariaDbCompilationDiagnostic.php
    │   └── MariaDbDiagnosticMetadata.php
    │
    └── Exception
        ├── MariaDbCompilationException.php
        ├── UnsupportedMariaDbFeatureException.php
        ├── UnsupportedMariaDbVersionException.php
        ├── MariaDbCapabilityException.php
        ├── MariaDbDialectAdaptationException.php
        └── MariaDbRepresentationException.php
```

---

# 223. Invariantes arquitectónicos

## DB-MARIADB-001
MariaDB será una plataforma distinta de MySQL.

## DB-MARIADB-002
MariaDB Compiler no heredará semánticamente del MySQL Compiler.

## DB-MARIADB-003
MySQL y MariaDB podrán compartir infraestructura de familia únicamente cuando exista equivalencia real.

## DB-MARIADB-004
Shared implementation no implicará shared platform contract.

## DB-MARIADB-005
Driver PDO MySQL no implicará plataforma MySQL.

## DB-MARIADB-006
MariaDB target será explícito.

## DB-MARIADB-007
MariaDB version será immutable value object.

## DB-MARIADB-008
Version comparison será estructurada.

## DB-MARIADB-009
Version será distinta de capability.

## DB-MARIADB-010
Compiler consumirá capability snapshots.

## DB-MARIADB-011
Capability snapshot será immutable.

## DB-MARIADB-012
Compiler no realizará server discovery.

## DB-MARIADB-013
Compiler no ejecutará `SELECT VERSION()`.

## DB-MARIADB-014
Compiler no consultará `information_schema`.

## DB-MARIADB-015
Compiler no ejecutará SQL.

## DB-MARIADB-016
Compiler no abrirá Connection.

## DB-MARIADB-017
Compiler no manejará Transaction.

## DB-MARIADB-018
Compiler no hidratará entities.

## DB-MARIADB-019
Compiler no conocerá UnitOfWork.

## DB-MARIADB-020
SQL modes relevantes serán explícitos.

## DB-MARIADB-021
No existirá global mutable SQL mode.

## DB-MARIADB-022
Common SQL Compiler Pipeline será reutilizado.

## DB-MARIADB-023
MariaDB pipeline no será copia del MySQL pipeline.

## DB-MARIADB-024
Dialect adaptation será distinta de rendering.

## DB-MARIADB-025
Representation validation será distinta de semantic validation.

## DB-MARIADB-026
Identifiers serán rendered mediante componente especializado.

## DB-MARIADB-027
Identifiers dinámicos no serán interpolados.

## DB-MARIADB-028
Backtick escaping será centralizado.

## DB-MARIADB-029
Aliases serán planificados previamente.

## DB-MARIADB-030
Placeholders serán planificados previamente.

## DB-MARIADB-031
Runtime values no se interpolarán.

## DB-MARIADB-032
Repeated parameters conservarán identity.

## DB-MARIADB-033
Binding layout será separado de SQL.

## DB-MARIADB-034
String escaping respetará dialect profile.

## DB-MARIADB-035
Numeric rendering será locale-independent.

## DB-MARIADB-036
SELECT rendering será determinista.

## DB-MARIADB-037
JOIN semantics serán preservadas.

## DB-MARIADB-038
Unsupported joins no serán emitidos.

## DB-MARIADB-039
SEMI/ANTI adaptation ocurrirá antes del renderer.

## DB-MARIADB-040
SQL three-valued logic será preservada.

## DB-MARIADB-041
Ordinary equality será distinta de null-safe equality.

## DB-MARIADB-042
Grouping correctness no dependerá de permissive server modes.

## DB-MARIADB-043
Aggregate semantics serán preservadas.

## DB-MARIADB-044
Vendor functions no contaminarán el semantic core.

## DB-MARIADB-045
ORDER BY será preservado.

## DB-MARIADB-046
NULL ordering será capability-aware.

## DB-MARIADB-047
Pagination será target-aware.

## DB-MARIADB-048
Renderer no inventará unlimited magic constants.

## DB-MARIADB-049
LIMIT sin ORDER BY no inventará ordering.

## DB-MARIADB-050
Window functions serán capability-gated.

## DB-MARIADB-051
Window ordering será distinta de final ordering.

## DB-MARIADB-052
CTEs serán capability-gated.

## DB-MARIADB-053
Recursive CTE será structured.

## DB-MARIADB-054
CTE no implicará materialization automáticamente.

## DB-MARIADB-055
Subquery correlation estará resuelta previamente.

## DB-MARIADB-056
Scalar cardinality semantics serán preservadas.

## DB-MARIADB-057
Set operations serán capability-aware.

## DB-MARIADB-058
UNION y UNION ALL serán distintos.

## DB-MARIADB-059
INTERSECT/EXCEPT support no se inferirá desde MySQL.

## DB-MARIADB-060
INSERT bindings serán deterministas.

## DB-MARIADB-061
Multi-row INSERT preservará parameter identity.

## DB-MARIADB-062
INSERT SELECT será first-class.

## DB-MARIADB-063
RETURNING capability será modelada por mutation kind.

## DB-MARIADB-064
No existirá universal `supportsReturning` cuando la semántica difiera por statement.

## DB-MARIADB-065
Unsupported RETURNING no será eliminado.

## DB-MARIADB-066
Multi-statement RETURNING emulation pertenecerá al Execution Plan.

## DB-MARIADB-067
Generated identifier retrieval no será considerado universalmente equivalente a RETURNING.

## DB-MARIADB-068
Upsert semantic intent será independiente de MariaDB syntax.

## DB-MARIADB-069
MariaDB upsert no será considerado idéntico a PostgreSQL ON CONFLICT.

## DB-MARIADB-070
Inserted-row references serán capability-aware.

## DB-MARIADB-071
UPDATE syntax será context-aware.

## DB-MARIADB-072
UPDATE joins serán explicit representations.

## DB-MARIADB-073
UPDATE ordering/limit serán capability-gated.

## DB-MARIADB-074
DELETE variants serán explicit representations.

## DB-MARIADB-075
DELETE RETURNING será capability-aware.

## DB-MARIADB-076
JSON adaptation será independiente de MySQL.

## DB-MARIADB-077
JSON semantic type no será reducido automáticamente a string.

## DB-MARIADB-078
SQL NULL será distinto de JSON null.

## DB-MARIADB-079
JSON paths serán seguros.

## DB-MARIADB-080
Temporal features serán capability-based.

## DB-MARIADB-081
System-versioned features no contaminarán portable core.

## DB-MARIADB-082
Sequence será distinta de auto increment.

## DB-MARIADB-083
Sequence support será capability-based.

## DB-MARIADB-084
Database current time no será sustituido por PHP current time.

## DB-MARIADB-085
Intervals serán structured.

## DB-MARIADB-086
Concat semantics no dependerán de SQL-mode-sensitive operator assumptions.

## DB-MARIADB-087
CAST utilizará type resolution.

## DB-MARIADB-088
MariaDB-specific types tendrán descriptors propios.

## DB-MARIADB-089
Shared type descriptors requerirán equivalencia real.

## DB-MARIADB-090
Collations serán structured metadata.

## DB-MARIADB-091
Collation input no será interpolado.

## DB-MARIADB-092
Locking será structured.

## DB-MARIADB-093
Lock modifiers serán capability-gated.

## DB-MARIADB-094
Compiler declarará transaction requirements sin iniciar transactions.

## DB-MARIADB-095
Optimizer hints serán platform-specific.

## DB-MARIADB-096
MySQL hints no se reutilizarán automáticamente.

## DB-MARIADB-097
Hint enforceability será explícita.

## DB-MARIADB-098
VoltStack Physical Plan será distinto del MariaDB native plan.

## DB-MARIADB-099
MariaDB optimizer conservará autoridad sobre estrategias delegadas.

## DB-MARIADB-100
Compiler no ejecutará EXPLAIN.

## DB-MARIADB-101
Runtime EXPLAIN pertenecerá a diagnostics/telemetry.

## DB-MARIADB-102
Compiler no tendrá hidden optimizer feedback loop.

## DB-MARIADB-103
Known unsupported features fallarán antes de execution.

## DB-MARIADB-104
Feature registry será typed.

## DB-MARIADB-105
Complex capabilities utilizarán descriptors.

## DB-MARIADB-106
Compiler emulation no requerirá I/O.

## DB-MARIADB-107
Execution emulation no será responsabilidad del renderer.

## DB-MARIADB-108
Extensions declararán compatibilidad.

## DB-MARIADB-109
Extension registries serán frozen.

## DB-MARIADB-110
No habrá silent last-wins.

## DB-MARIADB-111
Unknown extension no tendrá raw fallback.

## DB-MARIADB-112
Source map será preservado.

## DB-MARIADB-113
Target metadata podrá formar parte de diagnostics.

## DB-MARIADB-114
Compiled artifacts no contendrán runtime values.

## DB-MARIADB-115
MariaDB target identity participará en fingerprint.

## DB-MARIADB-116
Capability fingerprint participará en compiled fingerprint.

## DB-MARIADB-117
MariaDB y MySQL compiled caches estarán aislados por target identity.

## DB-MARIADB-118
SQL textual idéntico no implicará compiled artifact idéntico.

## DB-MARIADB-119
Parameter-sensitive specialization será explícita.

## DB-MARIADB-120
Hard y soft dependencies serán diferenciadas.

## DB-MARIADB-121
Compiler shared services serán immutable.

## DB-MARIADB-122
Compilation state será operation-scoped.

## DB-MARIADB-123
No existirá static current target.

## DB-MARIADB-124
No habrá request-state leakage.

## DB-MARIADB-125
Multi-target workers estarán soportados.

## DB-MARIADB-126
Compilation será determinista.

## DB-MARIADB-127
Serializable descriptors no contendrán live resources.

## DB-MARIADB-128
Compiler resolution será distinta de driver resolution.

## DB-MARIADB-129
PDO_MYSQL podrá resolver a MariaDB platform.

## DB-MARIADB-130
Tests cubrirán version profiles.

## DB-MARIADB-131
Tests cubrirán capability-negative cases.

## DB-MARIADB-132
Tests cubrirán RETURNING por mutation kind.

## DB-MARIADB-133
Tests cubrirán UPSERT.

## DB-MARIADB-134
Tests cubrirán SQL NULL vs JSON null.

## DB-MARIADB-135
Tests cubrirán temporal features cuando estén soportadas.

## DB-MARIADB-136
Tests cubrirán sequences cuando estén soportadas.

## DB-MARIADB-137
Tests cubrirán locking.

## DB-MARIADB-138
Tests cubrirán injection boundaries.

## DB-MARIADB-139
Tests cubrirán persistent-runtime isolation.

## DB-MARIADB-140
Tests cubrirán deterministic compilation.

## DB-MARIADB-141
Cross-family tests verificarán comportamiento realmente común.

## DB-MARIADB-142
Divergence tests verificarán diferencias MySQL/MariaDB.

## DB-MARIADB-143
No se utilizarán golden files compartidos para features divergentes.

## DB-MARIADB-144
Compiler benchmark no requerirá live database.

## DB-MARIADB-145
Integration benchmark será separado.

## DB-MARIADB-146
Compiler no reparará semántica inválida.

## DB-MARIADB-147
Compiler no dependerá de permissive server behavior.

## DB-MARIADB-148
Compiler no convertirá warnings en mecanismo de compatibilidad.

## DB-MARIADB-149
Toda MariaDB-specific syntax tendrá ownership explícito.

## DB-MARIADB-150
Toda divergencia con MySQL será capability/dialect-driven.

## DB-MARIADB-151
Toda feature nativa tendrá version/capability provenance.

## DB-MARIADB-152
Shared MySqlFamily components no podrán conocer el target global mutable.

## DB-MARIADB-153
Shared family components deberán ser stateless o immutable.

## DB-MARIADB-154
MariaDB compiler no podrá mutar el input SQL tree recibido.

## DB-MARIADB-155
Dialect adaptation producirá artifacts nuevos/immutables.

## DB-MARIADB-156
Renderer no decidirá query optimization.

## DB-MARIADB-157
Renderer no decidirá physical access paths.

## DB-MARIADB-158
Renderer no resolverá ORM metadata.

## DB-MARIADB-159
Renderer no resolverá tenant global state.

## DB-MARIADB-160
MariaDB Compiler preservará observable semantics.

---

# 224. Anti-patrones

## 224.1 MariaDB como alias de MySQL

Incorrecto:

```php
case 'mysql':
case 'mariadb':
    return new MySqlCompiler();
```

---

## 224.2 MariaDB Compiler heredando MySQL Compiler

Evitar:

```php
class MariaDbCompiler extends MySqlCompiler
{
    // dozens of overrides
}
```

si esto implica heredar accidentalmente decisiones específicas de MySQL.

Preferir composición mediante `MySqlFamily`.

---

## 224.3 Version checks dispersos

Incorrecto:

```php
if ($version >= '10.x') {
}
```

por todo el código.

---

## 224.4 Capability boolean demasiado amplio

Incorrecto:

```text
supportsReturning = true
```

cuando existen diferencias por statement.

---

## 224.5 JSON == MySQL JSON

Incorrecto:

```text
MariaDB JSON
=
MySQL JSON
```

por nombre.

---

## 224.6 RETURNING eliminado

Incorrecto:

```text
unsupported
→ ignore
```

---

## 224.7 Hidden multi-query emulation

Incorrecto:

```text
DELETE RETURNING
→ DELETE
→ SELECT
```

generado silenciosamente por renderer.

---

## 224.8 SQL-mode assumptions

Incorrecto:

```text
||
always means concatenation
```

sin capability/dialect profile.

---

## 224.9 Driver determines compiler

Incorrecto:

```text
PDO_MYSQL
→ MySqlCompiler
```

---

## 224.10 Shared family becomes platform

Incorrecto:

```text
MySqlFamily
=
database target
```

`MySqlFamily` es infraestructura interna, no plataforma.

---

# 225. Ejemplo de divergencia MySQL/MariaDB

Supongamos una operación semántica:

```text
Delete
├── target: users
├── predicate: inactive = true
└── returning:
    ├── id
    └── email
```

El Query Engine no pregunta:

```text
isMariaDb?
```

---

# 226. Logical plan

```text
Returning(id,email)
└── Delete(users)
    └── Filter(inactive = TRUE)
```

---

# 227. Physical plan

```text
DatabaseDelegatedMutation
├── mutation: DELETE
├── returning requirement
└── target capability requirement
```

---

# 228. MariaDB compilation

El MariaDB capability profile determina si puede producirse:

```text
native single-statement representation
```

---

# 229. MySQL compilation

El MySQL capability profile se evalúa independientemente.

Puede resultar:

```text
unsupported as native representation
```

---

# 230. Resultado

```text
Same Semantic Operation
        │
        ├── MariaDB
        │      │
        │      ▼
        │  Native SQL representation
        │
        └── MySQL
               │
               ▼
        Different capability result
```

Esta diferencia nunca debe filtrarse hacia:

```text
Query Builder
ORM
Repository
Entity
```

---

# 231. Ejemplo de infraestructura compartida

Identifier:

```text
users.email
```

puede utilizar:

```text
MySqlFamilyIdentifierRenderer
```

para ambos targets:

```sql
`users`.`email`
```

---

# 232. Pero ownership sigue separado

```text
MariaDbDialectProfile
        │
        ▼
shared renderer implementation
```

y:

```text
MySqlDialectProfile
        │
        ▼
shared renderer implementation
```

---

# 233. Cambio futuro

Si MariaDB cambiara una regla:

```text
MariaDB
→ specialized renderer
```

sin modificar:

```text
MySQL compiler
```

---

# 234. Ejemplo completo

Query:

```php
$query
    ->from('orders', 'o')
    ->select('o.customer_id')
    ->selectRawAggregate('SUM', 'o.total', 'amount')
    ->where('o.status', '=', $status)
    ->groupBy('o.customer_id')
    ->havingAggregate('SUM', 'o.total', '>', $minimum)
    ->orderBy('amount', 'desc')
    ->limit(10);
```

---

# 235. Semantic representation

```text
SelectQuery
├── Relation orders [R1]
├── Projection
│   ├── customer_id
│   └── SUM(total) AS amount
├── Filter
│   └── status = P1
├── Group
│   └── customer_id
├── Having
│   └── SUM(total) > P2
├── Order
│   └── amount DESC
└── Limit
    └── 10
```

---

# 236. Logical plan

```text
Limit(10)
└── Sort(amount DESC)
    └── Project(customer_id, amount)
        └── Filter[HAVING](SUM(total) > P2)
            └── Aggregate
                ├── group: customer_id
                ├── SUM(total) → amount
                └── Filter[WHERE](status = P1)
                    └── Scan(orders)
```

---

# 237. Execution plan

Para delegación completa:

```text
DatabaseExecutionUnit E1
├── owner: DATABASE
├── target: MARIADB
├── delegated physical region
├── parameters:
│   ├── P1
│   └── P2
└── output:
    ├── customer_id
    └── amount
```

---

# 238. MariaDB emission tree

```text
MariaDbSelect
├── Projection
│   ├── Column(R1, customer_id)
│   └── Sum(Column(R1,total)) AS amount
├── From
│   └── orders AS o
├── Where
│   └── status = P1
├── GroupBy
│   └── customer_id
├── Having
│   └── SUM(total) > P2
├── OrderBy
│   └── amount DESC
└── Limit
    └── 10
```

---

# 239. Generated SQL

```sql
SELECT
    `o`.`customer_id`,
    SUM(`o`.`total`) AS `amount`
FROM `orders` AS `o`
WHERE `o`.`status` = ?
GROUP BY `o`.`customer_id`
HAVING SUM(`o`.`total`) > ?
ORDER BY `amount` DESC
LIMIT 10
```

---

# 240. Binding layout

```text
slot 0 → P1
slot 1 → P2
```

Runtime values permanecen fuera.

---

# 241. Architecture formula

```text
MariaDbCompilation
=
CommonSqlCompilation
+
MariaDbTargetResolution
+
MariaDbCapabilityValidation
+
MariaDbDialectAdaptation
+
MariaDbRepresentationValidation
+
MariaDbRendering
+
BindingCompilation
+
SourceMapping
```

---

# 242. MySQL-family formula

```text
MySqlFamily
=
Only Proven Common Syntax/Behavior
```

No:

```text
MySqlFamily
=
Assume MariaDB Is MySQL
```

---

# 243. Divergence formula

```text
Shared Feature
        │
        ├── semantics equivalent
        │      ↓
        │   shared component
        │
        └── semantics/syntax/capability diverge
               ↓
          platform specialization
```

---

# 244. Architectural decision

VoltStack adoptará:

```text
Composition
>
Compiler inheritance
```

para compartir comportamiento MySQL/MariaDB.

---

# 245. Compiler hierarchy

Preferido:

```text
SqlCompiler
    │
    ├── DefaultSqlCompiler orchestration
    │
    ├── MySql compiler profile
    │      └── uses MySqlFamily components
    │
    └── MariaDb compiler profile
           └── uses MySqlFamily components
```

No:

```text
SqlCompiler
    ↓
MySqlCompiler
    ↓
MariaDbCompiler
```

---

# 246. Integration with Platform Capability System

```text
Database Platform
        │
        ▼
MariaDbPlatform
        │
        ▼
Capability Snapshot
        │
        ▼
MariaDb Compiler
```

---

# 247. Integration with Execution Plan

```text
PhysicalQueryPlan
        │
        ▼
ExecutionPlan
        │
        ▼
DatabaseExecutionUnit
        │
        ▼
CompilationRequest
        │
        ▼
MariaDbSqlCompiler
```

---

# 248. Integration with Executor

```text
MariaDbCompiledDatabaseCommand
        │
        ▼
Query Executor
        │
        ▼
Connection Manager
        │
        ▼
MariaDB Connection
```

---

# 249. ORM isolation

```text
Entity
Repository
UnitOfWork
IdentityMap
        │
        ▼
Persistence / Query Engine
        │
        ▼
...
        │
        ▼
MariaDB Compiler
```

El compiler nunca deberá recibir un `Entity`.

---

# 250. Portability model

VoltStack deberá seguir:

```text
Portable Semantic Core
+
Capability-driven Platform Adaptation
+
Explicit Vendor Extensions
```

---

# 251. No lowest-common-denominator design

No se eliminarán capabilities útiles de MariaDB sólo porque otro motor no las soporte.

En su lugar:

```text
portable operation
vendor capability
vendor extension
```

serán capas distintas.

---

# 252. Example public architecture

Portable:

```php
$query->limit(20);
```

Capability-sensitive:

```php
$query->forUpdate();
```

MariaDB-specific extension:

```text
temporal/system-versioned feature
```

si no existe todavía una abstracción portable equivalente.

---

# 253. Failure model

```text
Requested Feature
        │
        ▼
MariaDbCapabilitySnapshot
        │
        ├── Native
        │      ↓
        │   compile
        │
        ├── Compiler-equivalent adaptation
        │      ↓
        │   adapt + compile
        │
        ├── Execution emulation available
        │      ↓
        │   ExecutionPlan strategy
        │
        └── Unsupported
               ↓
             fail
```

---

# 254. Principle: fail explicitly

Nunca:

```text
unsupported
→ approximate silently
```

---

# 255. Principle: no vendor leakage

```text
Query Builder
does not know MariaDB syntax

ORM
does not know MariaDB syntax

Semantic Engine
does not know MariaDB grammar

Optimizer
does not render MariaDB SQL

Execution Engine
does not construct MariaDB SQL
```

---

# 256. Principle: compiler purity

```text
MariaDbSqlCompiler
does:
    adapt
    validate
    render
    compile bindings
    build diagnostics
```

No:

```text
connect
execute
fetch
hydrate
persist
begin transaction
discover live schema
query server version
run EXPLAIN
```

---

# 257. Resultado arquitectónico final

El subsistema queda definido como:

```text
MariaDbSqlCompiler
=
GenericSqlCompilerPipeline
+
MariaDbTarget
+
MariaDbVersionModel
+
MariaDbCapabilitySnapshot
+
MariaDbDialectProfile
+
MySqlFamilySharedPrimitives
+
MariaDbSpecificAdapters
+
MariaDbSpecificRenderers
+
MariaDbReturningModel
+
MariaDbUpsertModel
+
MariaDbJsonModel
+
MariaDbTemporalCapabilities
+
MariaDbSequenceCapabilities
+
MariaDbLockingModel
+
MariaDbExtensionRegistry
+
DeterministicGeneration
+
PersistentRuntimeIsolation
```

---

# 258. Principio final

La decisión arquitectónica esencial será:

```text
MySQL and MariaDB may share syntax,
but they do not share identity.
```

Por ello VoltStack deberá aplicar:

```text
share implementation
when equivalent

separate capability
when behavior differs

separate renderer
when syntax differs

separate target
always
```

---

# 259. Fórmula final

```text
CorrectMariaDbCompilation
=
PortableSemanticMeaning
+
ExplicitMariaDbTarget
+
CapabilityAwareAdaptation
+
VersionAwareDescriptors
+
StructuredMariaDbRepresentation
+
SafeIdentifierRendering
+
PlannedParameterization
+
ExplicitMySqlFamilyReuse
+
ExplicitMariaDbDivergence
+
DeterministicSqlGeneration
+
SourceMapping
+
PersistentRuntimeIsolation
```

Y el pipeline completo:

```text
CompilableDatabaseOperation
        │
        ▼
Common SQL Compiler Pipeline
        │
        ▼
MariaDbTarget
├── MariaDbVersion
├── MariaDbCapabilitySnapshot
├── MariaDbSqlModeProfile
└── MariaDbExtensions
        │
        ▼
MariaDB Dialect Adaptation
        │
        ├── common MySqlFamily primitives
        └── MariaDB-specific rules
        │
        ▼
MariaDB Representation Validation
        │
        ▼
Alias Planning
        │
        ▼
Placeholder Planning
        │
        ▼
MariaDB SQL Generation
        │
        ▼
RenderedSql
+
BindingLayout
+
ResultContract
+
SqlSourceMap
+
CompilationDependencies
+
CompilationFingerprint
        │
        ▼
MariaDbCompiledDatabaseCommand
```

> MariaDB será para VoltStack un target de compilación independiente y de primera clase. La compatibilidad histórica con MySQL se aprovechará mediante composición de componentes comunes, pero nunca se utilizará para ocultar diferencias de sintaxis, capacidades, tipos, JSON, `RETURNING`, funciones, características temporales o evolución futura de ambas plataformas.

---

# 260. Siguiente documento

```text
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
```

El siguiente documento deberá formalizar el target PostgreSQL, incluyendo especialmente:

```text
$1, $2, ... parameterization
RETURNING
ON CONFLICT
DISTINCT ON
FILTER
LATERAL
CTE
recursive CTE
window functions
arrays
JSON / JSONB
operators
casts
collations
locking
set operations
native PostgreSQL capabilities
```

manteniendo la misma frontera:

```text
PostgreSQL Compiler
=
SQL target specialization

not:
Query Optimizer
Execution Engine
ORM
Connection Driver
```