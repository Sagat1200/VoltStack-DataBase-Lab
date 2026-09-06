# 71_DATABASE_POSTGRESQL_SQL_COMPILER.md

# VoltStack Quantum Database
## PostgreSQL SQL Compiler

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 71 — PostgreSQL SQL Compiler  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`PostgreSQL SQL Compiler` define la especialización del SQL Compiler de VoltStack responsable de transformar operaciones compilables en una representación SQL válida para un target PostgreSQL concreto.

Su responsabilidad será adaptar el modelo SQL común de VoltStack a las características propias de PostgreSQL:

- quoting de identifiers;
- placeholders;
- tipos y casts;
- operadores;
- funciones;
- `RETURNING`;
- `ON CONFLICT`;
- `DISTINCT ON`;
- `FILTER`;
- `LATERAL`;
- CTE;
- CTE recursivos;
- window functions;
- arrays;
- JSON/JSONB;
- locking;
- set operations;
- collations;
- capabilities específicas por versión;
- extensiones PostgreSQL.

El principio central será:

```text
PostgreSqlCompiler
=
Common SQL Compiler
+
PostgreSQL Target
+
PostgreSQL Capability Model
+
PostgreSQL Dialect Adaptation
+
PostgreSQL Rendering Rules
```

El compiler no será:

```text
PostgreSqlCompiler
=
ORM
+
Query Optimizer
+
Execution Engine
+
Connection Driver
```

---

# 2. Posición arquitectónica

```text
Query Builder
    │
    ▼
Query AST
    │
    ▼
Semantic Analysis
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
    ├── MySQL Compiler
    ├── MariaDB Compiler
    ├── PostgreSQL Compiler
    └── SQLite Compiler
            │
            ▼
     Compiled Query
            │
            ▼
     Execution Engine
```

PostgreSQL Compiler ocupa exclusivamente:

```text
Execution/Compilation Input
        ↓
PostgreSQL SQL Representation
```

---

# 3. Frontera fundamental

```text
Semantic Query
≠
Logical Plan
≠
Physical Plan
≠
Execution Plan
≠
PostgreSQL SQL
≠
PostgreSQL native execution plan
```

El compiler recibe una operación cuya semántica ya fue determinada.

No decide qué significa la consulta.

---

# 4. Responsabilidades

El sistema deberá:

1. validar que el target sea PostgreSQL;
2. consumir un capability snapshot explícito;
3. adaptar construcciones SQL comunes;
4. seleccionar representaciones PostgreSQL válidas;
5. generar SQL determinista;
6. producir layout de parámetros;
7. preservar tipos;
8. preservar output contracts;
9. producir source maps;
10. registrar dependencies;
11. producir fingerprints;
12. detectar features no soportadas;
13. soportar extensiones;
14. ser seguro en persistent runtimes.

---

# 5. No responsabilidades

No deberá:

```text
optimize joins
choose indexes
estimate cardinality
open connections
query pg_catalog
execute EXPLAIN
bind actual values
execute statements
fetch rows
hydrate entities
manage UnitOfWork
begin transactions
commit transactions
rollback transactions
resolve current tenant globally
```

---

# 6. Target PostgreSQL

Se define:

```php
final readonly class PostgreSqlTarget
{
    public function __construct(
        public PostgreSqlVersion $version,
        public PostgreSqlCapabilitySnapshot $capabilities,
        public PostgreSqlDialectProfile $dialect,
    ) {}
}
```

---

# 7. Target identity

```text
DatabaseTargetFamily::POSTGRESQL
```

deberá ser diferente de:

```text
MYSQL
MARIADB
SQLITE
```

---

# 8. Driver ≠ PostgreSQL platform

Una conexión puede utilizar:

```text
PDO_PGSQL
```

pero:

```text
PDO_PGSQL
```

es una decisión de driver.

Mientras:

```text
POSTGRESQL
```

es una plataforma SQL.

Por tanto:

```text
Driver
≠
Platform
≠
Dialect
≠
Compiler
```

---

# 9. PostgreSQL version model

```php
final readonly class PostgreSqlVersion
{
    public function __construct(
        public int $major,
        public int $minor,
    ) {}
}
```

La representación podrá conservar metadata adicional si es necesaria.

---

# 10. Version ≠ capability

Incorrecto:

```php
if ($version->major >= 15) {
    // assume everything
}
```

Preferido:

```php
if ($capabilities->supports($feature)) {
}
```

---

# 11. Capability discovery

```text
Server Discovery
+
Configured Target
+
Driver Metadata
+
Version Profile
+
Explicit Overrides
        │
        ▼
PostgreSqlCapabilityResolver
        │
        ▼
PostgreSqlCapabilitySnapshot
```

---

# 12. No discovery durante compilation

El compiler no ejecutará:

```sql
SHOW server_version;
```

ni:

```sql
SELECT version();
```

ni consultas contra:

```text
pg_catalog
information_schema
```

---

# 13. Capability snapshot

```php
final readonly class PostgreSqlCapabilitySnapshot
{
    public function __construct(
        public PostgreSqlVersion $version,
        public PostgreSqlFeatureSet $features,
        public PostgreSqlExtensionCapabilitySet $extensions,
        public CapabilityFingerprint $fingerprint,
    ) {}
}
```

---

# 14. Capability examples

El snapshot deberá responder cuestiones como:

```text
supportsReturning()
supportsOnConflict()
supportsDistinctOn()
supportsFilterClause()
supportsLateral()
supportsRecursiveCte()
supportsMaterializedCteHint()
supportsNotMaterializedCteHint()
supportsWindowFunctions()
supportsJson()
supportsJsonb()
supportsArrays()
supportsSkipLocked()
supportsNowait()
supportsUpdateFrom()
supportsDeleteUsing()
supportsMerge()
```

La lista exacta evolucionará mediante descriptors.

---

# 15. Capability granularity

Evitar:

```text
supportsPostgresAdvancedFeatures = true
```

Preferir:

```text
RETURNING
ON_CONFLICT
DISTINCT_ON
LATERAL
FILTER
JSONB
ARRAY
CTE_MATERIALIZED
LOCK_NOWAIT
LOCK_SKIP_LOCKED
MERGE
```

como capabilities independientes.

---

# 16. Capability status

```text
SUPPORTED
SUPPORTED_WITH_RESTRICTIONS
SUPPORTED_WITH_ADAPTATION
UNSUPPORTED
EXTENSION_REQUIRED
```

---

# 17. PostgreSQL dialect profile

```php
final readonly class PostgreSqlDialectProfile
{
    public function __construct(
        public IdentifierQuotePolicy $identifiers,
        public PostgreSqlPlaceholderPolicy $placeholders,
        public PostgreSqlTypeProfile $types,
        public PostgreSqlFunctionProfile $functions,
        public PostgreSqlOperatorProfile $operators,
    ) {}
}
```

---

# 18. Compilation context

```php
final readonly class PostgreSqlCompilationContext
{
    public function __construct(
        public PostgreSqlTarget $target,
        public CompilationOptions $options,
        public CompilationBudget $budget,
        public SqlRenderingProfile $rendering,
    ) {}
}
```

No contendrá recursos runtime.

---

# 19. Compiler contract

```php
interface PostgreSqlCompiler
{
    public function compile(
        CompilableDatabaseOperation $operation,
        PostgreSqlCompilationContext $context,
    ): PostgreSqlCompiledDatabaseCommand;
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
Generic SQL Lowering
        │
        ▼
PostgreSQL Dialect Adaptation
        │
        ▼
PostgreSQL Representation Validation
        │
        ▼
Alias Planning
        │
        ▼
Parameter Planning
        │
        ▼
Type/Cast Planning
        │
        ▼
PostgreSQL SQL Generation
        │
        ▼
Binding Compilation
        │
        ▼
Result Contract Compilation
        │
        ▼
Source Map
        │
        ▼
Dependencies
        │
        ▼
Fingerprint
        │
        ▼
PostgreSqlCompiledDatabaseCommand
```

---

# 21. Dialect adaptation

```php
interface PostgreSqlDialectAdapter
{
    public function adapt(
        SqlEmissionTree $tree,
        PostgreSqlCompilationContext $context,
    ): SqlEmissionTree;
}
```

Podrá adaptar:

```text
pagination
casts
functions
operators
RETURNING
ON CONFLICT
DISTINCT ON
FILTER
LATERAL
JSON
arrays
locking
set operations
vendor extensions
```

---

# 22. Adapter ≠ renderer

```text
Dialect Adapter
=
select valid PostgreSQL representation
```

Mientras:

```text
Renderer
=
serialize that representation
```

---

# 23. Identifier quoting

PostgreSQL utiliza:

```sql
"identifier"
```

---

# 24. Qualified identifiers

```sql
"users"."id"
```

---

# 25. Alias example

```sql
"users" AS "u"
```

---

# 26. Identifier escaping

Un quote interno deberá duplicarse según las reglas PostgreSQL.

La lógica estará centralizada.

---

# 27. Case folding

PostgreSQL posee comportamiento relevante respecto a identifiers no quoted.

VoltStack deberá evitar depender accidentalmente de dicho comportamiento.

---

# 28. Canonical strategy

Para identifiers generados/controlados por VoltStack se preferirá una política consistente de quoting.

Así:

```text
UserAccount
```

no dependerá de transformación implícita del servidor.

---

# 29. Identifier ≠ runtime value

Nunca:

```php
$sql = 'SELECT * FROM "' . $userInput . '"';
```

---

# 30. PostgreSQL placeholders

El modelo interno deberá ser independiente del estilo del driver.

PostgreSQL puede representarse conceptualmente mediante:

```text
$1
$2
$3
...
```

cuando la estrategia de compilación/driver correspondiente utilice positional parameters nativos.

---

# 31. Internal parameter identity

```text
ParameterId
≠
ParameterOccurrenceId
≠
BindingSlot
≠
RenderedPlaceholder
```

---

# 32. Example

Semantic expression:

```text
email = P1
AND
status = P2
```

Puede producir:

```sql
"email" = $1
AND
"status" = $2
```

---

# 33. Repeated parameter

```text
x = P1 OR y = P1
```

Puede producir:

```sql
"x" = $1 OR "y" = $1
```

si el execution/driver contract permite reutilizar el mismo positional parameter.

O:

```sql
"x" = $1 OR "y" = $2
```

con:

```text
$1 → P1
$2 → P1
```

si la estrategia concreta lo requiere.

---

# 34. Placeholder policy

Esto será responsabilidad de:

```text
PostgreSqlPlaceholderPolicy
```

no de los expression renderers.

---

# 35. Driver-aware finalization

El compiler deberá permitir que la política final de placeholders sea compatible con:

```text
native PostgreSQL protocol
PDO pgsql
future drivers
```

sin contaminar el Query AST.

---

# 36. SQL ≠ bindings

El resultado continuará separado:

```text
RenderedSql
+
BindingLayout
```

---

# 37. Parameter casts

PostgreSQL puede necesitar type information adicional en contextos ambiguos.

Ejemplo conceptual:

```sql
$1::uuid
```

---

# 38. Cast planning

No deberá añadirse un cast arbitrariamente.

La decisión deberá basarse en:

```text
QueryType
+
ExpectedDatabaseType
+
ExpressionContext
+
ParameterType
+
PostgreSQL Type Mapping
```

---

# 39. Type system integration

```text
VoltStack QueryType
        │
        ▼
PostgreSqlTypeResolver
        │
        ▼
PostgreSqlTypeDescriptor
        │
        ▼
SQL Type / Cast Representation
```

---

# 40. PostgreSQL type descriptors

Podrán modelarse tipos como:

```text
SMALLINT
INTEGER
BIGINT
NUMERIC
REAL
DOUBLE PRECISION
BOOLEAN
TEXT
VARCHAR
BYTEA
UUID
DATE
TIME
TIMESTAMP
TIMESTAMPTZ
INTERVAL
JSON
JSONB
ARRAY
ENUM
INET
CIDR
MACADDR
RANGE
MULTIRANGE
EXTENSION TYPE
```

según alcance y capabilities.

---

# 41. Semantic type ≠ physical type

```text
QueryType::String
```

no será necesariamente idéntico a:

```text
VARCHAR
```

---

# 42. Cast syntax

El compiler podrá soportar:

```sql
CAST($1 AS uuid)
```

y/o:

```sql
$1::uuid
```

según canonical rendering policy.

---

# 43. Canonical cast form

VoltStack deberá elegir una representación canónica para fingerprints/golden tests.

---

# 44. SELECT

Ejemplo:

```sql
SELECT
    "u"."id",
    "u"."email"
FROM "users" AS "u"
WHERE "u"."active" = $1
ORDER BY "u"."id" ASC
LIMIT 20
```

---

# 45. DISTINCT

```sql
SELECT DISTINCT
    "country"
FROM "users"
```

---

# 46. DISTINCT ON

PostgreSQL permite una construcción particularmente importante:

```sql
SELECT DISTINCT ON ("user_id")
    "user_id",
    "created_at"
FROM "sessions"
ORDER BY
    "user_id",
    "created_at" DESC
```

---

# 47. DISTINCT ON semantic model

No deberá representarse como un raw SQL fragment.

Debe existir un concepto estructurado:

```text
DistinctOnSpecification
├── expressions
└── ordering dependency
```

---

# 48. DISTINCT ON ordering relationship

El compiler deberá validar los requisitos estructurales que la representación PostgreSQL imponga.

No deberá inventar ordering silenciosamente.

---

# 49. DISTINCT ≠ DISTINCT ON

```text
DISTINCT
≠
DISTINCT ON
```

---

# 50. PostgreSQL-specific feature boundary

Si `DISTINCT ON` se expone como extensión PostgreSQL:

```text
PostgreSqlDistinctOnExtension
```

no deberá contaminar el Query Builder portable.

Si VoltStack crea una abstracción semántica general equivalente en el futuro, podrá elevarse a una capa superior.

---

# 51. FROM

Podrá contener:

```text
relation
subquery
CTE
VALUES
function source
LATERAL source
extension source
```

---

# 52. LATERAL

PostgreSQL soporta:

```sql
LEFT JOIN LATERAL (
    SELECT ...
) AS "x" ON TRUE
```

---

# 53. Lateral semantics

`LATERAL` deberá provenir de una dependencia de correlación explícita.

No será una keyword añadida por conveniencia.

---

# 54. Correlation model

```text
Inner Subplan
    │
    └── requires outer symbols
            │
            ▼
    Correlation Dependency
```

El compiler transforma esa dependencia en una representación PostgreSQL válida.

---

# 55. JOIN types

El compiler podrá representar, cuando sean válidos:

```text
INNER
LEFT
RIGHT
FULL
CROSS
```

---

# 56. FULL OUTER JOIN

Ejemplo:

```sql
SELECT ...
FROM "a"
FULL OUTER JOIN "b"
    ON "a"."id" = "b"."a_id"
```

---

# 57. SEMI/ANTI

Los logical operators internos:

```text
SEMI_JOIN
ANTI_JOIN
```

podrán bajar a:

```text
EXISTS
NOT EXISTS
```

cuando corresponda.

---

# 58. No NOT IN reinterpretation

El compiler nunca deberá asumir:

```text
NOT IN
=
NOT EXISTS
```

sin prueba semántica previa.

---

# 59. WHERE

```sql
WHERE "status" = $1
```

---

# 60. NULL semantics

```sql
IS NULL
IS NOT NULL
```

serán representaciones explícitas.

---

# 61. Three-valued logic

Se preservará:

```text
TRUE
FALSE
UNKNOWN
```

---

# 62. PostgreSQL boolean

PostgreSQL posee tipo boolean nativo.

La adaptación deberá respetar el `QueryType::Boolean`.

---

# 63. Boolean literals

La canonical representation podrá usar:

```sql
TRUE
FALSE
```

para literals estructurales.

Runtime boolean values seguirán el binding policy.

---

# 64. IS DISTINCT FROM

PostgreSQL proporciona:

```sql
IS DISTINCT FROM
IS NOT DISTINCT FROM
```

con semántica NULL-aware.

---

# 65. Semantic operations

VoltStack podrá modelar:

```text
DistinctFrom
NotDistinctFrom
```

como operaciones semánticas cuando corresponda.

---

# 66. Ordinary equality remains distinct

```text
=
≠
IS NOT DISTINCT FROM
```

---

# 67. GROUP BY

```sql
GROUP BY
    "department_id",
    "status"
```

---

# 68. GROUPING SETS

Cuando el capability model lo permita:

```sql
GROUP BY GROUPING SETS (
    ("region", "product"),
    ("region"),
    ()
)
```

---

# 69. ROLLUP

```sql
GROUP BY ROLLUP (
    "region",
    "product"
)
```

---

# 70. CUBE

```sql
GROUP BY CUBE (
    "region",
    "product"
)
```

---

# 71. Logical grouping model

Estas formas deberán provenir del `LogicalAggregate`, no ser inventadas por el PostgreSQL renderer.

---

# 72. HAVING

```sql
HAVING COUNT(*) > $1
```

---

# 73. Aggregate FILTER

PostgreSQL soporta:

```sql
COUNT(*) FILTER (
    WHERE "status" = 'active'
)
```

---

# 74. Structured filter

VoltStack deberá representar:

```text
AggregateCall
├── function
├── arguments
├── distinct
├── filter
└── ordering
```

cuando el aggregate model lo requiera.

---

# 75. FILTER capability

La representación:

```text
FILTER (WHERE ...)
```

será capability-gated.

---

# 76. Aggregate ORDER BY

Funciones agregadas que admitan ordering interno deberán conservarlo independientemente del ordering final.

---

# 77. Aggregate order ≠ query order

```text
Aggregate Internal Ordering
≠
Window Ordering
≠
Final Query Ordering
```

---

# 78. Window functions

Ejemplo:

```sql
ROW_NUMBER() OVER (
    PARTITION BY "department_id"
    ORDER BY "created_at" DESC
)
```

---

# 79. Window contract

Deberá preservar:

```text
partition
ordering
frame
frame boundaries
exclusion
function
```

según capabilities.

---

# 80. Named windows

PostgreSQL permite:

```sql
WINDOW "w" AS (
    PARTITION BY "department_id"
    ORDER BY "created_at"
)
```

Podrá modelarse mediante descriptors estructurados.

---

# 81. Window reuse

La reutilización sintáctica de una named window no deberá cambiar la semántica del plan.

---

# 82. ORDER BY

```sql
ORDER BY
    "created_at" DESC,
    "id" ASC
```

---

# 83. NULL ordering

PostgreSQL permite control explícito:

```sql
NULLS FIRST
NULLS LAST
```

---

# 84. Ordering descriptor

```text
OrderingTerm
├── expression
├── direction
├── null ordering
└── collation
```

---

# 85. LIMIT

```sql
LIMIT 20
```

---

# 86. OFFSET

```sql
OFFSET 40
```

---

# 87. LIMIT + OFFSET

```sql
LIMIT 20 OFFSET 40
```

---

# 88. LIMIT without ORDER BY

No inventará ordering.

---

# 89. FETCH syntax

Si se soporta una representación alternativa:

```sql
FETCH FIRST 20 ROWS ONLY
```

deberá seleccionarse mediante canonical dialect policy, no arbitrariamente.

---

# 90. CTE

```sql
WITH "active_users" AS (
    SELECT ...
)
SELECT ...
```

---

# 91. Recursive CTE

```sql
WITH RECURSIVE "tree" AS (
    ...
)
SELECT ...
```

---

# 92. Recursive semantics

El compiler recibe:

```text
anchor
recursive member
recursive binding
set operation
output contract
```

ya estructurados.

---

# 93. MATERIALIZED CTE

Cuando la versión/capability lo soporte:

```sql
WITH "x" AS MATERIALIZED (
    ...
)
```

---

# 94. NOT MATERIALIZED

Cuando sea soportado:

```sql
WITH "x" AS NOT MATERIALIZED (
    ...
)
```

---

# 95. CTE materialization hint

```text
DEFAULT
MATERIALIZED
NOT_MATERIALIZED
```

deberá ser typed.

---

# 96. Physical planning relationship

Una decisión de materialización puede originarse en:

```text
Logical Requirement
Physical Planning Intent
Explicit User Hint
```

pero el compiler sólo traduce el descriptor recibido.

---

# 97. Subqueries

Soporte estructurado:

```text
scalar
EXISTS
IN
ANY
ALL
derived
correlated
```

---

# 98. Scalar subquery

Debe conservar:

```text
0 rows → NULL
1 row  → value
>1 row → cardinality error
```

---

# 99. ANY/ALL

PostgreSQL permite formas potentes como:

```sql
$1 = ANY("values")
```

o subquery forms.

VoltStack deberá distinguir sus semánticas.

---

# 100. Arrays

PostgreSQL arrays serán una capability de primera clase.

---

# 101. Array type

Conceptualmente:

```text
PostgreSqlArrayType
└── element type
```

---

# 102. Array expression

Podrán existir descriptors:

```text
ArrayConstructor
ArraySubscript
ArraySlice
ArrayContains
ArrayOverlap
ArrayAnyComparison
ArrayAllComparison
```

según el API finalmente soportado.

---

# 103. ARRAY constructor

Ejemplo:

```sql
ARRAY[$1, $2, $3]
```

---

# 104. Array binding

Array values no deberán convertirse ingenuamente a:

```text
implode(',', $values)
```

---

# 105. Array serialization

La representación/binding deberá estar gobernada por:

```text
PostgreSqlArrayBindingDescriptor
```

y por las capabilities del driver.

---

# 106. JSON

PostgreSQL deberá distinguir:

```text
JSON
JSONB
```

---

# 107. JSON ≠ JSONB

```text
JSON
≠
JSONB
```

en términos de:

```text
storage
operators
indexability
equality semantics
capabilities
```

---

# 108. Semantic JSON operations

VoltStack podrá modelar:

```text
JsonExtract
JsonExtractScalar
JsonContains
JsonExists
JsonPath
JsonSet
JsonRemove
JsonMerge
```

---

# 109. PostgreSQL adaptation

```text
Semantic JSON Operation
        │
        ▼
PostgreSqlJsonAdapter
        │
        ▼
JSON/JSONB operator/function representation
```

---

# 110. JSON operators

No deberán aparecer como strings dispersos:

```text
->
->>
@>
?
#>
#>>
```

Deberán registrarse mediante descriptors typed.

---

# 111. Operator registry

```php
interface PostgreSqlOperatorRegistry
{
    public function resolve(
        SemanticOperatorId $operator,
        PostgreSqlTarget $target,
    ): PostgreSqlOperatorDescriptor;
}
```

---

# 112. Operator metadata

Podrá incluir:

```text
token
precedence
associativity
operand types
result type
required capability
null behavior
extension ownership
```

---

# 113. JSON path security

Dynamic JSON paths deberán ser:

```text
bound
```

cuando la sintaxis lo permita, o:

```text
validated structured tokens
```

cuando formen parte de estructura SQL.

Nunca concatenación libre.

---

# 114. SQL NULL vs JSON null

Debe preservarse:

```text
SQL NULL
≠
JSON null
```

---

# 115. JSONB containment

Una operación semántica de containment podrá adaptarse a una representación PostgreSQL nativa cuando el target sea compatible.

---

# 116. JSON type inference

El renderer no inferirá tipos.

Los tipos vendrán del Query Type System y dialect adaptation.

---

# 117. Functions

PostgreSQL functions se resolverán mediante:

```text
PostgreSqlFunctionRegistry
```

---

# 118. Function descriptor

```php
final readonly class PostgreSqlFunctionDescriptor
{
    public function __construct(
        public SemanticFunctionId $semanticId,
        public PostgreSqlFunctionName $name,
        public FunctionSignatureSet $signatures,
        public VolatilityClassification $volatility,
        public CapabilityRequirementSet $capabilities,
    ) {}
}
```

---

# 119. Function volatility

PostgreSQL distingue conceptos equivalentes a:

```text
IMMUTABLE
STABLE
VOLATILE
```

VoltStack deberá conservar información de volatilidad relevante desde etapas semánticas.

---

# 120. Compiler does not reclassify volatility

El compiler no decidirá que una función es segura para reorder.

---

# 121. Function schema

Funciones pueden ser schema-qualified:

```sql
"public"."my_function"($1)
```

---

# 122. Search path

El compiler no deberá depender implícitamente de un `search_path` mutable cuando la resolución semántica requiera identidad inequívoca.

---

# 123. Relation qualification

Cuando el semantic/schema resolution haya resuelto:

```text
catalog/schema/relation identity
```

el compiler podrá producir la forma calificada necesaria.

---

# 124. Schema identity ≠ display name

```text
ResolvedRelationId
≠
"users"
```

---

# 125. INSERT

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES ($1, $2)
```

---

# 126. Multi-row INSERT

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES
    ($1, $2),
    ($3, $4)
```

---

# 127. DEFAULT VALUES

Cuando corresponda:

```sql
INSERT INTO "users"
DEFAULT VALUES
```

---

# 128. INSERT SELECT

```sql
INSERT INTO "archive_users" (
    "id",
    "email"
)
SELECT
    "id",
    "email"
FROM "users"
WHERE ...
```

---

# 129. RETURNING

PostgreSQL `RETURNING` será una capability fundamental.

Ejemplo:

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES ($1, $2)
RETURNING
    "id",
    "created_at"
```

---

# 130. RETURNING semantic contract

```text
Mutation
+
ReturningProjection
        │
        ▼
PostgreSqlMutationRepresentation
        │
        ▼
ResultContract
```

---

# 131. RETURNING applies to mutations

El capability model deberá distinguir los statement kinds soportados.

Aunque PostgreSQL posea amplio soporte, VoltStack mantendrá el modelo granular:

```text
INSERT
UPDATE
DELETE
MERGE / future mutation forms
```

---

# 132. RETURNING ≠ generated ID

`RETURNING` puede devolver:

```text
multiple columns
expressions
old/new values where supported by target/version semantics
multiple rows
```

No se reducirá a:

```text
lastInsertId()
```

---

# 133. UPDATE

```sql
UPDATE "users"
SET
    "status" = $1
WHERE "id" = $2
RETURNING
    "id",
    "status"
```

---

# 134. UPDATE FROM

PostgreSQL permite formas como:

```sql
UPDATE "users" AS "u"
SET "status" = $1
FROM "accounts" AS "a"
WHERE "a"."user_id" = "u"."id"
```

---

# 135. UPDATE FROM representation

Debe ser estructurada:

```text
PostgreSqlUpdateFromClause
```

No raw fragment.

---

# 136. DELETE

```sql
DELETE FROM "users"
WHERE "id" = $1
RETURNING "id"
```

---

# 137. DELETE USING

Cuando sea necesario:

```sql
DELETE FROM "sessions" AS "s"
USING "users" AS "u"
WHERE
    "s"."user_id" = "u"."id"
    AND "u"."disabled" = TRUE
```

---

# 138. Mutation join semantics

No se deberá asumir que:

```text
MySQL multi-table UPDATE/DELETE
=
PostgreSQL UPDATE FROM / DELETE USING
```

---

# 139. ON CONFLICT

PostgreSQL upsert se modelará mediante una adaptación explícita.

---

# 140. Semantic upsert

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

# 141. PostgreSQL adaptation

```text
UpsertIntent
        │
        ▼
PostgreSqlUpsertAdapter
        │
        ▼
ON CONFLICT representation
```

---

# 142. DO NOTHING

```sql
INSERT INTO "users" (
    "email"
)
VALUES ($1)
ON CONFLICT ("email")
DO NOTHING
```

---

# 143. DO UPDATE

```sql
INSERT INTO "users" (
    "email",
    "name"
)
VALUES ($1, $2)
ON CONFLICT ("email")
DO UPDATE SET
    "name" = EXCLUDED."name"
```

---

# 144. EXCLUDED

`EXCLUDED` deberá ser una semantic/special relation descriptor dentro del upsert scope.

No un string mágico.

---

# 145. Conflict target

Podrá modelar:

```text
columns
constraint
index inference
predicate
```

según capabilities.

---

# 146. Conflict target validation

La validación de identidad/schema ocurre antes.

El compiler valida representabilidad PostgreSQL.

---

# 147. ON CONFLICT ≠ ON DUPLICATE KEY

```text
PostgreSQL ON CONFLICT
≠
MySQL/MariaDB ON DUPLICATE KEY UPDATE
```

---

# 148. MERGE

Si el target soporta `MERGE`, deberá modelarse como capability separada.

---

# 149. MERGE is not automatically upsert

```text
MERGE
≠
UpsertIntent
```

aunque ciertos casos puedan solaparse.

---

# 150. MERGE descriptor

Una futura representación podrá incluir:

```text
MergeStatement
├── target
├── source
├── match predicate
├── matched actions
├── not matched actions
└── output requirements
```

---

# 151. Locking

PostgreSQL locking deberá ser estructurado.

---

# 152. Lock strengths

El modelo podrá distinguir:

```text
UPDATE
NO_KEY_UPDATE
SHARE
KEY_SHARE
```

cuando las capabilities lo permitan.

---

# 153. Lock targets

PostgreSQL puede permitir:

```sql
FOR UPDATE OF "u"
```

El target list será estructurado.

---

# 154. NOWAIT

```sql
FOR UPDATE NOWAIT
```

---

# 155. SKIP LOCKED

```sql
FOR UPDATE SKIP LOCKED
```

---

# 156. Lock descriptor

```text
PostgreSqlLockClause
├── strength
├── targets
└── wait policy
```

---

# 157. Transaction requirement

El compiler puede producir:

```text
TransactionRequirement
```

pero no iniciará la transacción.

---

# 158. Set operations

PostgreSQL compiler soportará, según capability:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
```

---

# 159. Multiplicity preservation

```text
ALL
≠
DISTINCT
```

---

# 160. Parentheses

El renderer deberá conservar grouping explícito cuando sea necesario.

---

# 161. Operator precedence

Nunca dependerá de concatenación naïve.

El emission tree deberá conocer precedence.

---

# 162. Parenthesis rule

Conceptualmente:

```text
if childPrecedence < requiredPrecedence
    render parentheses
```

considerando associativity y operator semantics.

---

# 163. String concatenation

PostgreSQL puede representar concatenación mediante:

```sql
"a" || "b"
```

cuando corresponda.

Pero VoltStack modelará primero:

```text
Concat(a,b)
```

---

# 164. Pattern matching

Operaciones como:

```text
LIKE
ILIKE
```

deberán ser descriptors distintos cuando su semántica sea distinta.

---

# 165. ILIKE

`ILIKE` es PostgreSQL-specific.

Puede exponerse mediante:

```text
PostgreSqlCaseInsensitiveLike
```

o una abstracción semántica portable si VoltStack define una.

---

# 166. Regex

PostgreSQL regex operators deberán ser extensions/descriptors typed.

No strings arbitrarios.

---

# 167. Full-text search

Las capacidades PostgreSQL de full-text search no deberán incrustarse directamente en el compiler core.

Deberán integrarse mediante:

```text
FullText semantic/extension model
        ↓
PostgreSqlFullTextAdapter
```

---

# 168. Future document integration

El documento:

```text
272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
```

definirá la abstracción superior.

---

# 169. Collations

Collation será structured metadata.

Ejemplo:

```sql
ORDER BY "name" COLLATE "C"
```

---

# 170. No collation interpolation

Nunca:

```php
$sql .= ' COLLATE ' . $input;
```

---

# 171. Time zones

PostgreSQL posee tipos y operaciones donde timezone semantics son relevantes.

VoltStack deberá distinguir:

```text
timestamp without time zone
timestamp with time zone
```

en su mapping físico.

---

# 172. TIMESTAMP ≠ TIMESTAMPTZ

```text
TIMESTAMP
≠
TIMESTAMPTZ
```

---

# 173. Database time ≠ application time

El compiler no reemplazará expresiones database-time con valores PHP.

---

# 174. Interval

Los intervalos deberán representarse estructuradamente.

---

# 175. Interval safety

Una unidad temporal estructural no será concatenada desde input libre.

---

# 176. UUID

PostgreSQL UUID podrá mapearse como tipo físico nativo.

---

# 177. UUID parameter

Cuando el contexto lo requiera:

```sql
$1::uuid
```

podrá ser producido por type/cast planning.

---

# 178. ENUM

PostgreSQL enum types deberán usar identidad schema-aware.

---

# 179. Custom types

Los tipos definidos por usuario o extensiones deberán resolverse mediante:

```text
PostgreSqlCustomTypeRegistry
```

---

# 180. Extension types

Ejemplos potenciales:

```text
citext
vector
geometry
geography
ltree
hstore
```

no pertenecerán automáticamente al core.

---

# 181. Extension capability set

El target podrá declarar:

```text
installed/available extension capabilities
```

mediante snapshots obtenidos antes de compilación.

---

# 182. No extension discovery during compilation

El compiler no consultará:

```text
pg_extension
```

durante `compile()`.

---

# 183. Extension registry

```php
interface PostgreSqlCompilerExtension
{
    public function descriptor(): PostgreSqlExtensionDescriptor;
}
```

---

# 184. Extension descriptor

Deberá declarar:

```text
extension id
version
PostgreSQL version range
required capabilities
provided types
provided functions
provided operators
provided syntax
dependencies
conflicts
```

---

# 185. Registry lifecycle

```text
bootstrap
→ discover
→ validate
→ resolve dependencies
→ detect conflicts
→ freeze
```

---

# 186. No last-wins

Si dos extensiones registran incompatiblemente la misma operación:

```text
→ explicit conflict
```

---

# 187. No raw fallback

Unknown PostgreSQL extension node:

```text
→ compilation error
```

No:

```text
→ cast object to string
```

---

# 188. PostgreSQL representation validator

```php
interface PostgreSqlRepresentationValidator
{
    public function validate(
        SqlEmissionTree $tree,
        PostgreSqlCompilationContext $context,
    ): void;
}
```

---

# 189. Validation responsibilities

Validará:

```text
capabilities
statement shape
RETURNING
ON CONFLICT
DISTINCT ON
FILTER
LATERAL
locking
types
casts
arrays
JSON/JSONB
set operations
extensions
```

---

# 190. Semantic validation not repeated

No resolverá:

```text
unknown column
ambiguous symbol
join semantic equivalence
predicate safety
ORM relationships
```

Estas responsabilidades pertenecen a fases anteriores.

---

# 191. Unsupported feature

Debe fallar antes de ejecución.

Ejemplo:

```text
PostgreSQL compilation failed

Feature:
MERGE

Target:
PostgreSQL <target profile>

Capability:
UNSUPPORTED

Phase:
POSTGRESQL_REPRESENTATION_VALIDATION
```

---

# 192. Diagnostics

Los diagnostics deberán conservar:

```text
source node
semantic origin
logical origin
physical origin
execution unit
compiler phase
PostgreSQL feature
required capability
target version
extension ownership
```

cuando estén disponibles.

---

# 193. Source map

```text
SqlSourceMap
```

permitirá relacionar:

```text
SQL byte/token range
        ↕
Compiler node
        ↕
Execution unit
        ↕
Physical node
        ↕
Logical/Semantic origin
```

---

# 194. Error translation support

Esto permitirá que errores runtime posteriores puedan señalar:

```text
which original query expression produced this SQL
```

sin hacer que el compiler ejecute la consulta.

---

# 195. Compiled artifact

```php
final readonly class PostgreSqlCompiledDatabaseCommand
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

# 196. Result contract

Podrá incluir:

```text
column identity
display name
query type
database type where relevant
nullability
ordinal
result shape
observable ordering
```

---

# 197. RETURNING result

Un mutation con `RETURNING` produce un result contract real.

No deberá modelarse sólo como:

```text
affectedRows
```

---

# 198. Binding descriptors

Cada binding podrá contener:

```text
BindingSlot
ParameterId
PostgreSqlTypeDescriptor
Nullability
Sensitivity
EncodingRequirement
```

pero no el valor runtime.

---

# 199. Sensitive values

El source map/diagnostics no incluirá runtime secrets.

---

# 200. Compilation dependencies

Podrán incluir:

```text
relation identity
column identity
type identity
function identity
operator identity
collation identity
extension identity/version
capability fingerprint
compiler version
```

---

# 201. Fingerprint

Conceptualmente:

```text
PostgreSqlCompiledFingerprint
=
InputFingerprint
+
PostgreSqlTargetFingerprint
+
CapabilityFingerprint
+
DialectFingerprint
+
CompilerVersion
+
ExtensionFingerprint
+
PlaceholderPolicyFingerprint
+
TypeMappingFingerprint
+
SpecializationFingerprint
```

---

# 202. Runtime values excluded

Nunca:

```text
user email
password
tenant id value
current timestamp
connection id
transaction id
```

---

# 203. PostgreSQL cache isolation

Artifacts PostgreSQL no compartirán identidad con:

```text
MySQL
MariaDB
SQLite
```

aunque el SQL conceptual sea similar.

---

# 204. Target version and cache

Si una diferencia de versión cambia:

```text
syntax
capabilities
type behavior
compiled representation
```

deberá cambiar el fingerprint relevante.

---

# 205. Persistent runtime

Debe funcionar correctamente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
long-running CLI processes
```

---

# 206. Shared immutable state

Puede compartirse:

```text
frozen registries
immutable descriptors
renderer definitions
operator metadata
function metadata
type mappings
```

---

# 207. Operation-local state

Debe ser local:

```text
compilation context
alias state
placeholder state
binding builder
source map builder
diagnostics
render buffers
temporary resolution maps
```

---

# 208. No static current version

Prohibido:

```php
PostgreSqlCompiler::$currentVersion
```

---

# 209. No static parameter counter

Prohibido:

```php
static $parameterIndex = 0;
```

---

# 210. Operation-scoped allocator

Preferir:

```text
CompilationSession
└── ParameterAllocator
```

---

# 211. Concurrency

Dos compilaciones simultáneas:

```text
Query A → PostgreSQL target A
Query B → PostgreSQL target B
```

no deberán compartir mutable compilation state.

---

# 212. Determinism

Mismos:

```text
input
target
capabilities
extensions
options
compiler version
```

deben producir:

```text
same canonical SQL
same bindings
same source mapping semantics
same dependencies
same fingerprint
```

---

# 213. Serialization

Artifacts cacheables no deberán contener:

```text
PDO
Connection
PDOStatement
Cursor
Closure
Fiber
Promise
live schema objects
service container
```

---

# 214. PostgreSQL compiler resolution

```text
DatabaseTarget::POSTGRESQL
        │
        ▼
PostgreSqlCompilerFactory
        │
        ▼
PostgreSqlCompiler
```

---

# 215. Compiler resolution ≠ driver resolution

```text
PostgreSQL Compiler
```

no necesita saber si execution utilizará:

```text
PDO
native extension
future async driver
```

excepto cuando un explicit compilation/binding profile forme parte del contrato.

---

# 216. Driver binding profile

Cuando una diferencia de driver afecte placeholders o encoding, deberá proporcionarse como descriptor explícito.

Ejemplo conceptual:

```text
PostgreSqlDriverCompilationProfile
```

---

# 217. No driver service in compiler

El profile será immutable metadata.

No un live driver object.

---

# 218. PostgreSQL native optimizer

En operaciones delegadas:

```text
VoltStack Physical Planning
        │
        ▼
PostgreSQL SQL
        │
        ▼
PostgreSQL Planner/Optimizer
        │
        ▼
Native PostgreSQL Execution Plan
```

---

# 219. Physical intent ≠ native guarantee

Si VoltStack desea:

```text
index-oriented access
join preference
materialization preference
```

sólo podrá influir donde PostgreSQL exponga una capability segura.

---

# 220. No fictitious physical control

El compiler no deberá afirmar:

```text
HashJoin selected
```

si el SQL emitido deja la decisión final al PostgreSQL planner.

---

# 221. Execution control level

La metadata podrá conservar:

```text
DATABASE_DELEGATED
COMPILER_INFLUENCED
HYBRID
FRAMEWORK_CONTROLLED
```

desde el Physical/Execution Plan.

---

# 222. EXPLAIN

El compiler podrá compilar una operación explícita:

```text
ExplainOperation
```

pero no ejecutarla.

---

# 223. EXPLAIN formats

Si se modelan:

```text
TEXT
JSON
XML
YAML
```

deberán ser capability-gated y typed.

---

# 224. EXPLAIN ANALYZE

Debe tratarse con especial cuidado porque:

```text
EXPLAIN
```

y:

```text
EXPLAIN ANALYZE
```

no tienen las mismas implicaciones runtime.

---

# 225. Compiler does not auto-analyze

Nunca:

```text
compile query
→ EXPLAIN ANALYZE automatically
```

---

# 226. Security

El compiler deberá mantener:

```text
values → bindings
identifiers → validated identifiers
operators → registries
functions → registries
types → descriptors
collations → descriptors
extensions → validated extension nodes
```

---

# 227. Raw expressions

Si VoltStack permite escape hatches:

```text
RawExpression
```

deberán conservar una clasificación explícita.

---

# 228. Raw expression policy

No deberá tratarse como trusted simplemente por existir.

Podrá existir:

```text
TrustedFrameworkRaw
TrustedExtensionRaw
ExplicitUserRaw
```

con políticas diferentes.

---

# 229. Parameterization boundary

No todo puede parameterizarse.

Ejemplos estructurales:

```text
table identifiers
column identifiers
type names
operators
keywords
sort direction
collations
```

deben validarse/representarse estructuralmente.

---

# 230. Prepared statement friendliness

El compiler deberá generar SQL estable para maximizar reutilización de prepared statements.

---

# 231. Stable SQL shape

Idealmente:

```text
same query structure
+
different runtime values
=
same compiled SQL shape
```

---

# 232. Parameter-sensitive specialization exception

Si el planner especializa explícitamente por parámetros:

```text
same semantic query
+
different specialization
=
potentially different physical/compiled shape
```

---

# 233. Testing architecture

La suite deberá incluir:

```text
unit tests
golden SQL tests
capability tests
version-profile tests
type tests
binding tests
integration tests
real PostgreSQL tests
extension tests
persistent-runtime tests
concurrency tests
security tests
determinism tests
```

---

# 234. SELECT golden test

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
WHERE "users"."active" = $1
LIMIT 10
```

según placeholder policy.

---

# 235. RETURNING tests

Cubrir:

```text
INSERT RETURNING
UPDATE RETURNING
DELETE RETURNING
multi-column RETURNING
expression RETURNING
multi-row RETURNING
type mapping
binding layout
```

---

# 236. ON CONFLICT tests

Cubrir:

```text
DO NOTHING
DO UPDATE
column conflict target
constraint conflict target
predicate
EXCLUDED references
RETURNING
NULL behavior
generated columns
```

---

# 237. DISTINCT ON tests

Cubrir:

```text
single expression
multiple expressions
ordering
aliases
subqueries
window interaction
invalid representations
```

---

# 238. FILTER tests

Cubrir:

```text
COUNT FILTER
SUM FILTER
DISTINCT aggregate FILTER
window aggregate FILTER where legal
NULL predicates
```

---

# 239. LATERAL tests

Cubrir:

```text
CROSS JOIN LATERAL
LEFT JOIN LATERAL
correlated symbols
nested lateral
parameter interaction
```

---

# 240. Array tests

Cubrir:

```text
array constructors
array bindings
subscripts
NULL elements
empty arrays
typed empty arrays
ANY
ALL
containment
overlap
```

---

# 241. Typed empty arrays

Caso especialmente importante:

```text
ARRAY[]
```

puede requerir type information.

VoltStack deberá preservar el tipo de elemento.

---

# 242. JSON/JSONB tests

Cubrir:

```text
JSON
JSONB
SQL NULL
JSON null
extraction
scalar extraction
containment
existence
paths
binding
casts
operators
indexes only as physical metadata, not compiler decisions
```

---

# 243. Type tests

Cubrir:

```text
uuid
boolean
numeric
timestamp
timestamptz
interval
json
jsonb
arrays
custom types
extension types
```

---

# 244. Locking tests

Cubrir:

```text
FOR UPDATE
FOR NO KEY UPDATE
FOR SHARE
FOR KEY SHARE
OF
NOWAIT
SKIP LOCKED
unsupported combinations
```

---

# 245. CTE tests

Cubrir:

```text
normal CTE
recursive CTE
MATERIALIZED
NOT MATERIALIZED
multiple CTEs
dependency order
nested CTE
```

---

# 246. Set operation tests

Cubrir:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
ordering
limit
parentheses
```

---

# 247. Security tests

Cubrir ataques contra:

```text
values
identifiers
JSON paths
type names
collations
operators
function names
extension nodes
raw expressions
ORDER BY direction
```

---

# 248. Persistent worker tests

Secuencia:

```text
compile tenant A query
reset operation state
compile tenant B query
```

deberá demostrar ausencia de contaminación.

---

# 249. Cross-platform tests

La misma consulta semántica podrá producir:

```text
PostgreSQL SQL
MySQL SQL
MariaDB SQL
SQLite SQL
```

diferentes pero semánticamente equivalentes.

---

# 250. PostgreSQL-specific tests

También deberán demostrar que:

```text
PostgreSQL-only feature
```

no se acepta accidentalmente en targets incompatibles.

---

# 251. Benchmarking

Medir:

```text
dialect adaptation latency
type resolution latency
parameter planning
SQL rendering
source map construction
fingerprinting
memory allocations
compiled cache interaction
```

---

# 252. Compiler benchmark ≠ database benchmark

```text
compile()
```

deberá poder benchmarkearse sin servidor PostgreSQL.

---

# 253. Integration benchmark

Separadamente:

```text
compile
prepare
bind
execute
fetch
```

---

# 254. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Compiler\Platform\PostgreSql
```

---

# 255. Estructura propuesta

```text
PostgreSql/
├── PostgreSqlCompiler.php
├── PostgreSqlCompilerFactory.php
│
├── Target/
│   ├── PostgreSqlTarget.php
│   ├── PostgreSqlVersion.php
│   ├── PostgreSqlVersionRange.php
│   └── PostgreSqlTargetFingerprint.php
│
├── Capability/
│   ├── PostgreSqlCapabilitySnapshot.php
│   ├── PostgreSqlCapabilityResolver.php
│   ├── PostgreSqlFeatureSet.php
│   ├── PostgreSqlFeatureDescriptor.php
│   └── PostgreSqlExtensionCapabilitySet.php
│
├── Dialect/
│   ├── PostgreSqlDialectProfile.php
│   ├── PostgreSqlDialectAdapter.php
│   └── PostgreSqlDialectRenderingContract.php
│
├── Validation/
│   ├── PostgreSqlRepresentationValidator.php
│   ├── PostgreSqlFeatureValidator.php
│   ├── PostgreSqlStatementValidator.php
│   └── PostgreSqlCapabilityValidator.php
│
├── Identifier/
│   ├── PostgreSqlIdentifierRenderer.php
│   └── PostgreSqlQualifiedIdentifierRenderer.php
│
├── Placeholder/
│   ├── PostgreSqlPlaceholderPolicy.php
│   ├── PostgreSqlPlaceholderRenderer.php
│   └── PostgreSqlParameterAllocator.php
│
├── Type/
│   ├── PostgreSqlTypeResolver.php
│   ├── PostgreSqlTypeDescriptor.php
│   ├── PostgreSqlTypeRenderer.php
│   ├── PostgreSqlCastPlanner.php
│   ├── PostgreSqlArrayType.php
│   └── PostgreSqlCustomTypeRegistry.php
│
├── Statement/
│   ├── PostgreSqlSelectRenderer.php
│   ├── PostgreSqlInsertRenderer.php
│   ├── PostgreSqlUpdateRenderer.php
│   ├── PostgreSqlDeleteRenderer.php
│   └── PostgreSqlMergeRenderer.php
│
├── Clause/
│   ├── PostgreSqlReturningRenderer.php
│   ├── PostgreSqlOnConflictRenderer.php
│   ├── PostgreSqlDistinctOnRenderer.php
│   ├── PostgreSqlFilterRenderer.php
│   ├── PostgreSqlLateralRenderer.php
│   ├── PostgreSqlPaginationRenderer.php
│   ├── PostgreSqlLockingRenderer.php
│   ├── PostgreSqlCteRenderer.php
│   ├── PostgreSqlWindowRenderer.php
│   └── PostgreSqlSetOperationRenderer.php
│
├── Expression/
│   ├── PostgreSqlExpressionRenderer.php
│   ├── PostgreSqlCaseRenderer.php
│   ├── PostgreSqlPredicateRenderer.php
│   └── PostgreSqlSubqueryRenderer.php
│
├── Function/
│   ├── PostgreSqlFunctionRegistry.php
│   ├── PostgreSqlFunctionDescriptor.php
│   └── PostgreSqlFunctionRenderer.php
│
├── Operator/
│   ├── PostgreSqlOperatorRegistry.php
│   ├── PostgreSqlOperatorDescriptor.php
│   └── PostgreSqlOperatorRenderer.php
│
├── Json/
│   ├── PostgreSqlJsonAdapter.php
│   ├── PostgreSqlJsonRenderer.php
│   ├── PostgreSqlJsonOperatorRegistry.php
│   └── PostgreSqlJsonTypeDescriptor.php
│
├── Array/
│   ├── PostgreSqlArrayAdapter.php
│   ├── PostgreSqlArrayRenderer.php
│   ├── PostgreSqlArrayBindingDescriptor.php
│   └── PostgreSqlArrayOperatorRegistry.php
│
├── Upsert/
│   ├── PostgreSqlUpsertAdapter.php
│   ├── PostgreSqlConflictTarget.php
│   ├── PostgreSqlConflictAction.php
│   └── PostgreSqlExcludedRelation.php
│
├── Lock/
│   ├── PostgreSqlLockClause.php
│   ├── PostgreSqlLockStrength.php
│   └── PostgreSqlLockWaitPolicy.php
│
├── Extension/
│   ├── PostgreSqlCompilerExtension.php
│   ├── PostgreSqlCompilerExtensionRegistry.php
│   ├── PostgreSqlExtensionDescriptor.php
│   └── PostgreSqlExtensionConflictDetector.php
│
├── Diagnostic/
│   ├── PostgreSqlCompilationDiagnostic.php
│   └── PostgreSqlDiagnosticMetadata.php
│
└── Exception/
    ├── PostgreSqlCompilationException.php
    ├── UnsupportedPostgreSqlFeatureException.php
    ├── UnsupportedPostgreSqlVersionException.php
    ├── PostgreSqlCapabilityException.php
    ├── PostgreSqlTypeCompilationException.php
    ├── PostgreSqlDialectAdaptationException.php
    └── PostgreSqlRepresentationException.php
```

---

# 256. Invariantes arquitectónicos

## DB-PGSQL-001
PostgreSQL será un target de compilación independiente.

## DB-PGSQL-002
PostgreSQL Compiler sólo dependerá de contratos SQL/compiler inferiores apropiados.

## DB-PGSQL-003
Compiler no realizará Semantic Analysis.

## DB-PGSQL-004
Compiler no realizará Query Optimization.

## DB-PGSQL-005
Compiler no realizará Physical Planning.

## DB-PGSQL-006
Compiler no ejecutará queries.

## DB-PGSQL-007
Compiler no abrirá connections.

## DB-PGSQL-008
Compiler no iniciará transactions.

## DB-PGSQL-009
Compiler no hidratará entities.

## DB-PGSQL-010
Compiler no conocerá UnitOfWork.

## DB-PGSQL-011
Target version será explícita.

## DB-PGSQL-012
Version será distinta de capability.

## DB-PGSQL-013
Capabilities serán immutable snapshots.

## DB-PGSQL-014
Compiler no realizará live capability discovery.

## DB-PGSQL-015
Compiler no consultará `pg_catalog`.

## DB-PGSQL-016
Compiler no consultará `information_schema`.

## DB-PGSQL-017
Compiler no ejecutará `SELECT version()`.

## DB-PGSQL-018
Dialect profile será explícito.

## DB-PGSQL-019
Compilation context será immutable.

## DB-PGSQL-020
Compilation state mutable será operation-scoped.

## DB-PGSQL-021
Dialect adaptation será distinta de rendering.

## DB-PGSQL-022
Representation validation será distinta de semantic validation.

## DB-PGSQL-023
Identifiers serán estructurados.

## DB-PGSQL-024
Identifiers no se interpolarán desde runtime values.

## DB-PGSQL-025
Identifier quoting será determinista.

## DB-PGSQL-026
Case folding implícito no será base de identidad semántica.

## DB-PGSQL-027
Parameter identity será independiente del placeholder textual.

## DB-PGSQL-028
Binding layout será separado del SQL.

## DB-PGSQL-029
Repeated parameters conservarán identidad.

## DB-PGSQL-030
Placeholder strategy será configurable mediante profile.

## DB-PGSQL-031
Runtime values no participarán en canonical SQL.

## DB-PGSQL-032
Parameter casts requerirán type justification.

## DB-PGSQL-033
QueryType será distinto de PostgreSQL physical type.

## DB-PGSQL-034
Type resolution ocurrirá mediante descriptors.

## DB-PGSQL-035
Custom types serán registry-driven.

## DB-PGSQL-036
SELECT rendering será determinista.

## DB-PGSQL-037
DISTINCT será distinto de DISTINCT ON.

## DB-PGSQL-038
DISTINCT ON será structured.

## DB-PGSQL-039
DISTINCT ON no inventará ordering.

## DB-PGSQL-040
LATERAL será derivado de dependencia estructurada.

## DB-PGSQL-041
Correlation será resuelta antes del renderer.

## DB-PGSQL-042
JOIN semantics serán preservadas.

## DB-PGSQL-043
FULL OUTER JOIN será capability-aware.

## DB-PGSQL-044
SEMI/ANTI adaptation preservará semántica.

## DB-PGSQL-045
NOT IN no será reescrito ingenuamente como NOT EXISTS.

## DB-PGSQL-046
SQL three-valued logic será preservada.

## DB-PGSQL-047
Boolean será un tipo real del target.

## DB-PGSQL-048
Ordinary equality será distinta de NULL-safe comparisons.

## DB-PGSQL-049
IS DISTINCT FROM tendrá semántica propia.

## DB-PGSQL-050
Grouping semantics serán preservadas.

## DB-PGSQL-051
GROUPING SETS será structured.

## DB-PGSQL-052
ROLLUP será structured.

## DB-PGSQL-053
CUBE será structured.

## DB-PGSQL-054
Aggregate FILTER será structured.

## DB-PGSQL-055
Aggregate ordering será distinto del query ordering.

## DB-PGSQL-056
Window ordering será distinto del final ordering.

## DB-PGSQL-057
Window frames serán preservados.

## DB-PGSQL-058
Named windows no cambiarán semántica.

## DB-PGSQL-059
NULL ordering será explícito cuando sea requerido.

## DB-PGSQL-060
LIMIT sin ORDER BY no inventará ordering.

## DB-PGSQL-061
Pagination representation será dialect-driven.

## DB-PGSQL-062
CTE será structured.

## DB-PGSQL-063
Recursive CTE será structured.

## DB-PGSQL-064
CTE materialization preference será typed.

## DB-PGSQL-065
MATERIALIZED será capability-gated.

## DB-PGSQL-066
NOT MATERIALIZED será capability-gated.

## DB-PGSQL-067
Scalar subquery cardinality será preservada.

## DB-PGSQL-068
ANY y ALL serán operaciones semánticamente distintas.

## DB-PGSQL-069
Arrays serán first-class PostgreSQL capability.

## DB-PGSQL-070
Array values no se construirán con string concatenation.

## DB-PGSQL-071
Array element type será preservado.

## DB-PGSQL-072
Empty arrays conservarán type information.

## DB-PGSQL-073
JSON será distinto de JSONB.

## DB-PGSQL-074
SQL NULL será distinto de JSON null.

## DB-PGSQL-075
JSON operators serán typed descriptors.

## DB-PGSQL-076
JSON paths no serán interpolados libremente.

## DB-PGSQL-077
JSON type inference no pertenecerá al renderer.

## DB-PGSQL-078
Functions serán registry-driven.

## DB-PGSQL-079
Operators serán registry-driven.

## DB-PGSQL-080
Operator precedence será explícita.

## DB-PGSQL-081
Renderer añadirá parentheses según precedence/associativity.

## DB-PGSQL-082
Function volatility vendrá de metadata previa.

## DB-PGSQL-083
Compiler no reclasificará volatility.

## DB-PGSQL-084
Search path mutable no definirá identidad semántica.

## DB-PGSQL-085
Schema identities estarán resueltas previamente.

## DB-PGSQL-086
INSERT será structured.

## DB-PGSQL-087
Multi-row INSERT conservará binding identity.

## DB-PGSQL-088
DEFAULT VALUES será first-class representation.

## DB-PGSQL-089
INSERT SELECT será first-class.

## DB-PGSQL-090
RETURNING será first-class.

## DB-PGSQL-091
RETURNING no será reducido a generated ID.

## DB-PGSQL-092
RETURNING producirá ResultContract.

## DB-PGSQL-093
UPDATE FROM será structured.

## DB-PGSQL-094
DELETE USING será structured.

## DB-PGSQL-095
PostgreSQL mutation joins no serán equiparados automáticamente con MySQL mutation joins.

## DB-PGSQL-096
ON CONFLICT será una adaptación explícita.

## DB-PGSQL-097
ON CONFLICT será distinto de ON DUPLICATE KEY UPDATE.

## DB-PGSQL-098
EXCLUDED será una special relation typed.

## DB-PGSQL-099
Conflict target será structured.

## DB-PGSQL-100
MERGE será capability separada.

## DB-PGSQL-101
MERGE no será tratado automáticamente como upsert.

## DB-PGSQL-102
Lock strengths serán typed.

## DB-PGSQL-103
Lock targets serán structured.

## DB-PGSQL-104
NOWAIT será capability-gated.

## DB-PGSQL-105
SKIP LOCKED será capability-gated.

## DB-PGSQL-106
Compiler no iniciará transaction por una cláusula de locking.

## DB-PGSQL-107
UNION será distinto de UNION ALL.

## DB-PGSQL-108
INTERSECT multiplicity será preservada.

## DB-PGSQL-109
EXCEPT multiplicity será preservada.

## DB-PGSQL-110
Set-operation grouping será preservado.

## DB-PGSQL-111
Concat semantic operation será independiente de `||`.

## DB-PGSQL-112
ILIKE será PostgreSQL-specific salvo abstracción portable.

## DB-PGSQL-113
Regex operators no serán raw strings dispersos.

## DB-PGSQL-114
Full-text search se integrará mediante abstraction/extension.

## DB-PGSQL-115
Collations serán structured metadata.

## DB-PGSQL-116
Collation input no será interpolado.

## DB-PGSQL-117
TIMESTAMP será distinto de TIMESTAMPTZ.

## DB-PGSQL-118
Database current time no será sustituido por application time.

## DB-PGSQL-119
Intervals serán structured.

## DB-PGSQL-120
UUID podrá utilizar type mapping nativo.

## DB-PGSQL-121
ENUM identity será schema-aware.

## DB-PGSQL-122
Extension types no pertenecerán automáticamente al core.

## DB-PGSQL-123
Compiler no descubrirá extensiones durante compile.

## DB-PGSQL-124
Extension registries serán frozen.

## DB-PGSQL-125
No habrá last-wins extension registration.

## DB-PGSQL-126
Unknown extension no tendrá raw fallback.

## DB-PGSQL-127
Known unsupported feature fallará antes de execution.

## DB-PGSQL-128
Diagnostics conservarán target capability information.

## DB-PGSQL-129
Source map será preservado.

## DB-PGSQL-130
Compiled artifact será immutable.

## DB-PGSQL-131
Compiled artifact no contendrá runtime values.

## DB-PGSQL-132
Binding descriptors no contendrán secrets.

## DB-PGSQL-133
Result contract será explícito.

## DB-PGSQL-134
Mutation RETURNING será tratado como result-producing statement.

## DB-PGSQL-135
Compilation dependencies serán explícitas.

## DB-PGSQL-136
Target identity participará en fingerprint.

## DB-PGSQL-137
Capability identity participará en fingerprint.

## DB-PGSQL-138
Compiler version participará en fingerprint.

## DB-PGSQL-139
Extension identity participará en fingerprint.

## DB-PGSQL-140
Runtime values serán excluidos del fingerprint.

## DB-PGSQL-141
Compiled caches estarán aislados por target.

## DB-PGSQL-142
Persistent runtime shared state será immutable.

## DB-PGSQL-143
Compilation mutable state será local.

## DB-PGSQL-144
No habrá static current target.

## DB-PGSQL-145
No habrá static parameter counter.

## DB-PGSQL-146
Compiler será concurrency-safe.

## DB-PGSQL-147
Compiler será deterministic.

## DB-PGSQL-148
Cacheable descriptors no contendrán live resources.

## DB-PGSQL-149
Compiler resolution será distinta de driver resolution.

## DB-PGSQL-150
Driver-specific compilation behavior requerirá explicit profile.

## DB-PGSQL-151
PostgreSQL native optimizer será distinto del VoltStack Physical Planner.

## DB-PGSQL-152
Compiler no prometerá physical behavior que PostgreSQL no garantice.

## DB-PGSQL-153
EXPLAIN no será ejecutado por compiler.

## DB-PGSQL-154
EXPLAIN ANALYZE nunca será implícito.

## DB-PGSQL-155
Values usarán bindings por defecto.

## DB-PGSQL-156
Structural SQL elements usarán validated descriptors.

## DB-PGSQL-157
Raw expressions estarán clasificadas.

## DB-PGSQL-158
Prepared SQL shape deberá ser estable.

## DB-PGSQL-159
Parameter-sensitive specialization será explícita.

## DB-PGSQL-160
Toda transformación PostgreSQL deberá preservar semántica observable.

---

# 257. Anti-patrones

## 257.1 Compiler consultando PostgreSQL

Incorrecto:

```php
$version = $pdo->query('SHOW server_version')->fetchColumn();
```

dentro de `compile()`.

---

## 257.2 Parámetros concatenados

Incorrecto:

```php
$sql .= " WHERE email = '{$email}'";
```

---

## 257.3 Identifier concatenation

Incorrecto:

```php
$sql .= ' ORDER BY "' . $field . '"';
```

---

## 257.4 JSON operators como raw strings

Incorrecto:

```php
$expression = $column . ' @> ' . $json;
```

---

## 257.5 Arrays mediante implode

Incorrecto:

```php
$array = '{' . implode(',', $values) . '}';
```

sin type/binding policy.

---

## 257.6 DISTINCT ON como raw SQL

Incorrecto:

```php
$query->raw('DISTINCT ON (...)');
```

como implementación interna del compiler.

---

## 257.7 ON CONFLICT como string

Incorrecto:

```php
$sql .= ' ON CONFLICT ' . $userClause;
```

---

## 257.8 EXCLUDED como alias ordinario

Incorrecto:

```text
EXCLUDED
=
normal table alias
```

---

## 257.9 UPDATE FROM tratado como MySQL JOIN UPDATE

Incorrecto:

```text
PostgreSQL UPDATE FROM
=
MySQL UPDATE JOIN
```

---

## 257.10 JSON == JSONB

Incorrecto:

```text
JSON
=
JSONB
```

---

## 257.11 TIMESTAMP == TIMESTAMPTZ

Incorrecto:

```text
timestamp
=
timestamptz
```

---

## 257.12 Compiler ejecutando EXPLAIN

Incorrecto:

```text
compile
→ execute EXPLAIN
→ inspect result
```

---

## 257.13 Compiler seleccionando índices

Incorrecto:

```text
PostgreSqlCompiler
→ choose idx_users_email
```

La selección física pertenece al planner o al PostgreSQL optimizer según control level.

---

## 257.14 Global parameter counter

Incorrecto:

```php
static int $parameter = 0;
```

---

## 257.15 Global current schema

Incorrecto:

```php
PostgreSqlCompiler::$schema = 'tenant_123';
```

---

# 258. Ejemplo — RETURNING

Operación:

```text
Insert users
├── name = P1
├── email = P2
└── returning
    ├── id
    └── created_at
```

Emission tree:

```text
PostgreSqlInsert
├── Target(users)
├── Columns
│   ├── name
│   └── email
├── Values
│   ├── P1
│   └── P2
└── Returning
    ├── id
    └── created_at
```

SQL:

```sql
INSERT INTO "users" (
    "name",
    "email"
)
VALUES ($1, $2)
RETURNING
    "id",
    "created_at"
```

Bindings:

```text
$1 → P1
$2 → P2
```

---

# 259. Ejemplo — ON CONFLICT

Semantic intent:

```text
Upsert
├── target: users
├── insert
│   ├── email = P1
│   └── name = P2
├── conflict target
│   └── email
├── action
│   └── update name from inserted row
└── returning
    └── id
```

PostgreSQL representation:

```sql
INSERT INTO "users" (
    "email",
    "name"
)
VALUES ($1, $2)
ON CONFLICT ("email")
DO UPDATE SET
    "name" = EXCLUDED."name"
RETURNING "id"
```

---

# 260. Ejemplo — DISTINCT ON

Semantic PostgreSQL extension:

```text
DistinctOn
├── expressions
│   └── user_id
├── projection
│   ├── user_id
│   ├── id
│   └── created_at
└── ordering
    ├── user_id ASC
    └── created_at DESC
```

SQL:

```sql
SELECT DISTINCT ON ("user_id")
    "user_id",
    "id",
    "created_at"
FROM "sessions"
ORDER BY
    "user_id" ASC,
    "created_at" DESC
```

---

# 261. Ejemplo — FILTER

Semantic aggregate:

```text
Count(*)
└── Filter(status = P1)
```

SQL:

```sql
COUNT(*) FILTER (
    WHERE "status" = $1
)
```

---

# 262. Ejemplo — LATERAL

Logical dependency:

```text
Users U
    │
    └── LatestOrderSubquery
        requires U.id
```

PostgreSQL representation:

```sql
SELECT
    "u"."id",
    "o"."id" AS "order_id"
FROM "users" AS "u"
LEFT JOIN LATERAL (
    SELECT "o"."id"
    FROM "orders" AS "o"
    WHERE "o"."user_id" = "u"."id"
    ORDER BY "o"."created_at" DESC
    LIMIT 1
) AS "o" ON TRUE
```

---

# 263. Ejemplo — JSONB

Semantic operation:

```text
JsonContains(
    metadata,
    P1
)
```

Adaptation:

```text
PostgreSQL JSONB containment
```

SQL conceptual:

```sql
"metadata" @> $1::jsonb
```

El cast sólo será emitido cuando el type planner determine que es necesario/canónico.

---

# 264. Ejemplo — Array ANY

Semantic operation:

```text
EqualsAny(
    id,
    P1:Array<UUID>
)
```

Una representación PostgreSQL válida podría ser:

```sql
"id" = ANY($1::uuid[])
```

si el binding profile soporta dicha representación.

---

# 265. Ejemplo — Locking

Input:

```text
LockRequirement
├── strength: UPDATE
├── target: users
└── wait: SKIP_LOCKED
```

SQL:

```sql
FOR UPDATE OF "users" SKIP LOCKED
```

---

# 266. Integración con Execution Plan

```text
ExecutionPlan
    │
    ▼
DatabaseExecutionUnit
├── target requirement: PostgreSQL
├── delegated physical region
├── parameter requirements
├── output contract
├── transaction requirements
└── compilation request
        │
        ▼
PostgreSqlCompiler
```

---

# 267. Compiled execution relationship

El resultado será conceptualmente:

```text
ExecutionPlan
+
PostgreSqlCompiledDatabaseCommand
        │
        ▼
Execution Engine
```

El `ExecutionPlan` no será mutado para insertar SQL.

---

# 268. Compiler output association

Se podrá utilizar un artifact como:

```text
ExecutionCompilationBundle
├── ExecutionUnit E1
│   └── PostgreSqlCompiledDatabaseCommand C1
├── ExecutionUnit E2
│   └── FrameworkControlled
└── ...
```

La arquitectura exacta del bundle será formalizada por las capas de compilación/ejecución correspondientes.

---

# 269. Integración con Prepared Statements

```text
PostgreSqlCompiledDatabaseCommand
        │
        ▼
Prepared Statement Compilation
        │
        ▼
PreparedStatementDefinition
```

El documento:

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

formalizará esta frontera.

---

# 270. Integración con compiled cache

```text
Compilation Fingerprint
        │
        ▼
Compiled Query Cache
```

formalizado posteriormente en:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 271. Integración con Execution Engine

```text
Compiled Command
        │
        ▼
Query Executor
        │
        ▼
Statement Execution
        │
        ▼
Connection
        │
        ▼
PostgreSQL
```

El compiler termina antes de:

```text
prepare
bind runtime values
execute
fetch
```

---

# 272. Modelo de portabilidad

VoltStack adoptará tres niveles:

```text
Level 1
Portable Semantic SQL

Level 2
Capability-Aware SQL

Level 3
PostgreSQL Extensions
```

---

# 273. Nivel 1 — Portable

Ejemplos:

```text
SELECT
JOIN
WHERE
GROUP BY
ORDER BY
INSERT
UPDATE
DELETE
```

---

# 274. Nivel 2 — Capability-aware

Ejemplos:

```text
RETURNING
recursive CTE
window functions
advanced locking
```

---

# 275. Nivel 3 — PostgreSQL-specific

Ejemplos potenciales:

```text
DISTINCT ON
ILIKE
JSONB-specific operators
arrays
PostgreSQL extension types
vendor-specific full-text constructs
```

---

# 276. No lowest common denominator

VoltStack no sacrificará las capacidades de PostgreSQL para mantener artificialmente una API mínima común.

La arquitectura será:

```text
Portable Core
+
Capabilities
+
Extensions
```

---

# 277. No vendor leakage upward

La existencia de:

```text
PostgreSqlJsonbOperator
```

no significa que:

```text
ORM
Repository
UnitOfWork
```

deban conocer su sintaxis.

---

# 278. Query API integration

La API podrá ofrecer:

```text
portable API
```

y explícitamente:

```text
platform extension API
```

sin mezclar ambos niveles.

---

# 279. Error model

Excepciones propuestas:

```text
PostgreSqlCompilationException
UnsupportedPostgreSqlFeatureException
UnsupportedPostgreSqlVersionException
PostgreSqlCapabilityException
PostgreSqlRepresentationException
PostgreSqlTypeCompilationException
PostgreSqlPlaceholderException
PostgreSqlExtensionException
PostgreSqlOperatorException
PostgreSqlFunctionException
PostgreSqlInvariantException
```

---

# 280. Error provenance

Todo error deberá poder responder, cuando sea posible:

```text
what failed?
where?
which compiler phase?
which PostgreSQL feature?
which capability was required?
which source query node originated it?
```

---

# 281. Master semantic invariant

Para una compilación válida:

```text
Semantics(
    PostgreSqlCompiledDatabaseCommand
)
=
Semantics(
    CompilableDatabaseOperation
)
```

dentro del contrato observable delegado al servidor.

---

# 282. Compilation correctness equation

```text
Correct PostgreSQL Compilation
=
Semantic Preservation
∧
PostgreSQL Grammar Validity
∧
Capability Compatibility
∧
Type Compatibility
∧
Binding Correctness
∧
Output Contract Preservation
∧
Security Preservation
```

---

# 283. Validity before convenience

Nunca:

```text
unsupported but easy to render
→ render anyway
```

Siempre:

```text
validate
→ adapt safely
→ render
```

---

# 284. PostgreSQL specialization formula

```text
PostgreSqlCompilation
=
CommonSqlCompilation
+
PostgreSqlTargetResolution
+
CapabilityValidation
+
DialectAdaptation
+
TypeAndCastPlanning
+
PostgreSqlRepresentationValidation
+
ParameterPlanning
+
DeterministicRendering
+
BindingCompilation
+
ResultContractCompilation
+
SourceMapping
+
DependencyTracking
+
Fingerprinting
```

---

# 285. Separación final de responsabilidades

```text
Semantic Engine
    decides meaning

Optimizer
    chooses equivalent logical form

Logical Planner
    constructs relational operations

Physical Planner
    selects physical strategy

Execution Planner
    constructs execution topology

PostgreSQL Compiler
    generates PostgreSQL representation

Execution Engine
    executes it

PostgreSQL
    performs native planning and execution
```

---

# 286. Principio final

El PostgreSQL Compiler deberá explotar las capacidades del motor sin convertir a PostgreSQL en una dependencia conceptual del Query Engine.

La regla será:

```text
PostgreSQL specialization lives at the compiler boundary.
```

No:

```text
PostgreSQL syntax leaks through the entire database stack.
```

---

# 287. Resultado arquitectónico

```text
PostgreSqlSqlCompiler
=
GenericSqlCompilerPipeline
+
PostgreSqlTarget
+
PostgreSqlCapabilitySnapshot
+
PostgreSqlDialectProfile
+
PostgreSqlIdentifierModel
+
PostgreSqlParameterModel
+
PostgreSqlTypeSystemAdapter
+
PostgreSqlCastPlanner
+
PostgreSqlFunctionRegistry
+
PostgreSqlOperatorRegistry
+
PostgreSqlReturningSystem
+
PostgreSqlOnConflictSystem
+
PostgreSqlDistinctOnSystem
+
PostgreSqlFilterSystem
+
PostgreSqlLateralSystem
+
PostgreSqlCteSystem
+
PostgreSqlWindowSystem
+
PostgreSqlArraySystem
+
PostgreSqlJsonJsonbSystem
+
PostgreSqlLockingSystem
+
PostgreSqlSetOperationSystem
+
PostgreSqlExtensionRegistry
+
DeterministicSqlGeneration
+
PersistentRuntimeIsolation
```

---

# 288. Pipeline final

```text
CompilableDatabaseOperation
        │
        ▼
Generic SQL Lowering
        │
        ▼
PostgreSQL Target
├── Version
├── Capability Snapshot
├── Dialect Profile
├── Driver Compilation Profile
└── Extension Capability Set
        │
        ▼
PostgreSQL Dialect Adaptation
        │
        ├── types
        ├── casts
        ├── functions
        ├── operators
        ├── DISTINCT ON
        ├── FILTER
        ├── LATERAL
        ├── RETURNING
        ├── ON CONFLICT
        ├── arrays
        ├── JSON / JSONB
        ├── locking
        └── extensions
        │
        ▼
PostgreSQL Representation Validation
        │
        ▼
Alias Planning
        │
        ▼
Parameter Planning
        │
        ▼
Type/Cast Finalization
        │
        ▼
PostgreSQL SQL Generation
        │
        ├── RenderedSql
        ├── BindingLayout
        ├── ResultContract
        ├── SqlSourceMap
        ├── CompilationDependencies
        └── CompilationFingerprint
        │
        ▼
PostgreSqlCompiledDatabaseCommand
        │
        ▼
Prepared Statement / Compiled Cache
        │
        ▼
Execution Engine
```

---

# 289. Fórmula arquitectónica final

```text
PostgreSQL Compiler
=
Maximum PostgreSQL Capability
without
PostgreSQL Leakage
```

Y:

```text
Portability
≠
Lowest Common Denominator
```

La portabilidad correcta será:

```text
Portable Semantic Core
+
Explicit Capabilities
+
Explicit PostgreSQL Extensions
```

---

# 290. Siguiente documento

```text
72_DATABASE_SQLITE_SQL_COMPILER.md
```

El siguiente documento deberá formalizar la especialización SQLite, prestando especial atención a:

```text
SQLite dynamic typing
type affinity
STRICT tables
parameterization
RETURNING
UPSERT / ON CONFLICT
INSERT OR ...
limited ALTER behavior
JSON capabilities
date/time representation
boolean representation
rowid
WITHOUT ROWID
generated columns
CTE
recursive CTE
window functions
locking/transaction limitations
single-file/runtime characteristics
SQLite version-dependent capabilities
in-memory databases
persistent-worker connection isolation
```

manteniendo la frontera:

```text
SQLite Compiler
=
SQLite SQL specialization

not:
Schema Migration Planner
Connection Manager
Transaction Manager
Execution Engine
ORM
```