# 66_DATABASE_SQL_COMPILER_ARCHITECTURE.md

# VoltStack Quantum Database
## SQL Compiler Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 66 — SQL Compiler Architecture  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`SQL Compiler Architecture` define la arquitectura responsable de transformar representaciones estructuradas, semánticamente resueltas y físicamente planificadas de VoltStack en comandos ejecutables por una plataforma SQL concreta.

Formalmente:

```text
Compile(
    CompilableDatabaseOperation,
    SqlCompilationContext
)
→
CompiledDatabaseCommand
```

La responsabilidad fundamental del compilador será:

```text
Structured Database Operation
        │
        ▼
Platform-Specific SQL Representation
```

sin volver a realizar:

```text
semantic analysis
query optimization
logical planning
physical planning
execution planning
query execution
```

El principio central será:

```text
Compiler
=
Representation Transformation

Compiler
≠
Semantic Engine
≠
Optimizer
≠
Planner
≠
Executor
```

---

# 2. Posición arquitectónica

El SQL Compiler se ubica después de la planificación física y de ejecución, pero antes de la ejecución efectiva contra el driver.

```text
Query Builder
     │
     ▼
Query Model / AST
     │
     ▼
Normalization
     │
     ▼
Validation
     │
     ▼
Semantic Analysis
     │
     ▼
SemanticQueryArtifact
     │
     ▼
Optimizer
     │
     ▼
OptimizedQueryArtifact
     │
     ▼
Logical Planner
     │
     ▼
LogicalQueryPlan
     │
     ▼
Physical Planner
     │
     ▼
PhysicalQueryPlan
     │
     ▼
Execution Planner
     │
     ▼
ExecutionPlan
     │
     ▼
┌─────────────────────────────────────────┐
│           SQL Compiler System           │
│                                         │
│  Compilation Coordinator                │
│  SQL AST / Emission Model               │
│  Dialect Compiler                       │
│  Expression Compiler                    │
│  Predicate Compiler                     │
│  Relation Compiler                      │
│  Parameter Placeholder Compiler         │
│  Identifier Quoting                     │
│  Capability Validation                  │
│  Prepared Statement Compilation         │
│  Compilation Diagnostics                │
└─────────────────────────────────────────┘
     │
     ▼
CompiledDatabaseCommand
     │
     ▼
Execution Engine
     │
     ▼
Connection / Driver
     │
     ▼
Database
```

---

# 3. Objetivo arquitectónico

VoltStack necesita soportar inicialmente:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin contaminar las capas superiores con condiciones como:

```php
if ($database === 'mysql') {
    ...
}

if ($database === 'pgsql') {
    ...
}
```

El Query Engine debe trabajar principalmente con:

```text
semantic constructs
logical operators
physical operations
capabilities
```

mientras que el SQL Compiler conoce:

```text
SQL syntax
dialect representation
identifier quoting
placeholder syntax
vendor-specific constructs
platform-specific statement forms
```

---

# 4. Distinción fundamental

Debe mantenerse:

```text
Driver
≠
Dialect
≠
Platform
≠
Compiler
≠
Connection
```

Cada concepto representa una responsabilidad diferente.

---

# 5. Driver

El Driver maneja comunicación runtime.

Ejemplos:

```text
PDO MySQL
PDO PostgreSQL
PDO SQLite
```

Sus responsabilidades incluyen posteriormente:

```text
prepare
bind
execute
fetch
close
driver errors
```

No genera SQL semánticamente.

---

# 6. Dialect

El Dialect describe convenciones sintácticas de SQL.

Por ejemplo:

```text
identifier quoting
placeholder rules
LIMIT syntax
RETURNING syntax
locking syntax
JSON operators
```

---

# 7. Platform

`DatabasePlatform` describe capacidades y comportamiento de la plataforma.

Ejemplos:

```text
supportsReturning()
supportsWindowFunctions()
supportsRecursiveCte()
supportsLateralJoin()
supportsJsonOperators()
supportsUpdateLimit()
supportsDeleteReturning()
```

---

# 8. Compiler

El Compiler transforma estructuras conocidas a representación SQL compatible con una plataforma.

```text
Structured Operation
      │
      ▼
Compiler
      │
      ▼
SQL + Binding Layout
```

---

# 9. Connection

La conexión representa una sesión/runtime channel hacia la base.

El Compiler no deberá necesitar una conexión viva para compilar una query ordinaria.

---

# 10. Regla fundamental

```text
SQL Compilation must be possible
without opening a database connection.
```

---

# 11. No hidden I/O

Durante compilación estará prohibido realizar implícitamente:

```text
connect()
query("SHOW ...")
query("EXPLAIN ...")
query("DESCRIBE ...")
query("SELECT version()")
schema introspection
runtime statistics lookup
```

La información requerida deberá llegar mediante snapshots/contextos explícitos.

---

# 12. Entrada del compilador

El compilador no debería recibir simplemente:

```php
string $sql
```

Su entrada será una representación estructurada.

Conceptualmente:

```php
interface CompilableDatabaseOperation
{
    public function operationId(): DatabaseOperationId;

    public function operationKind(): DatabaseOperationKind;
}
```

---

# 13. Origen de operaciones compilables

Principalmente:

```text
ExecutionPlan
      │
      ▼
DatabaseExecutionUnit
      │
      ▼
CompilableDatabaseOperation
```

---

# 14. DatabaseExecutionUnit ≠ CompilableDatabaseOperation

La unidad de ejecución describe trabajo.

La operación compilable describe la parte estructurada que debe representarse en SQL.

```text
DatabaseExecutionUnit
=
execution orchestration

CompilableDatabaseOperation
=
database command representation input
```

---

# 15. Compilable operation kinds

V1 podrá soportar:

```text
SELECT
INSERT
UPDATE
DELETE
```

y posteriormente:

```text
DDL
UTILITY
EXPLAIN
ADMINISTRATION
EXTENSION
```

---

# 16. Query AST y Physical Plan

El Compiler no deberá reconstruir la query desde cero.

La operación compilable podrá referenciar:

```text
Query AST fragments
Physical Plan properties
Execution requirements
Semantic metadata
```

de manera explícita y congelada.

---

# 17. Compilation contract

Contrato principal:

```php
interface SqlCompiler
{
    public function compile(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): CompiledDatabaseCommand;
}
```

---

# 18. SqlCompilationContext

Modelo conceptual:

```php
final readonly class SqlCompilationContext
{
    public function __construct(
        public DatabasePlatform $platform,
        public SqlDialect $dialect,
        public PlatformCapabilitySnapshot $capabilities,
        public SqlCompilationConfiguration $configuration,
        public SqlCompilationBudget $budget,
        public SqlCompilerExtensionSet $extensions,
        public CompilationMetadata $metadata,
    ) {}
}
```

---

# 19. Contexto explícito

No contendrá:

```text
current PDO
current transaction
current ResultCursor
current ORM EntityManager
global tenant singleton
HTTP request
```

---

# 20. Platform identity

El contexto podrá incluir:

```text
PlatformId
PlatformVersion
CapabilitySnapshot
DialectVersion
```

cuando sean necesarios para una compilación correcta.

---

# 21. Version ≠ capability

No deberá asumirse:

```text
PostgreSQL 17
=
fixed capability set hardcoded everywhere
```

Preferentemente:

```text
PlatformVersion
      │
      ▼
Capability Resolution
      │
      ▼
CapabilitySnapshot
```

y el Compiler consulta capabilities.

---

# 22. Vendor checks

Evitar:

```php
if ($platform->name() === 'postgresql') {
    ...
}
```

cuando realmente la pregunta sea:

```php
if ($capabilities->supportsReturning()) {
    ...
}
```

---

# 23. Cuándo sí importa el dialecto

Algunas diferencias son puramente sintácticas.

Por ejemplo:

```text
identifier quoting
operator spelling
placeholder syntax
statement grammar
```

En esos casos el Dialect Compiler es la abstracción correcta.

---

# 24. Arquitectura principal

```text
CompilableDatabaseOperation
            │
            ▼
   SqlCompilationCoordinator
            │
            ▼
    Compilation Validation
            │
            ▼
     Compiler Selection
            │
            ▼
    Structured SQL Lowering
            │
            ▼
     SQL Emission Model
            │
            ▼
      Dialect Rendering
            │
            ▼
 Parameter Layout Compilation
            │
            ▼
   CompiledDatabaseCommand
```

---

# 25. SqlCompilationCoordinator

Será el punto de entrada de alto nivel.

```php
interface SqlCompilationCoordinator
{
    public function compile(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): CompiledDatabaseCommand;
}
```

---

# 26. Coordinator ≠ God Compiler

El coordinator sólo orquesta.

No deberá contener toda la lógica de:

```text
SELECT
INSERT
UPDATE
DELETE
expressions
predicates
joins
CTEs
windows
JSON
dialect syntax
```

---

# 27. Compilers especializados

Arquitectura propuesta:

```text
SqlCompilationCoordinator
        │
        ├── SelectSqlCompiler
        ├── InsertSqlCompiler
        ├── UpdateSqlCompiler
        ├── DeleteSqlCompiler
        │
        ├── ExpressionSqlCompiler
        ├── PredicateSqlCompiler
        ├── RelationSqlCompiler
        ├── JoinSqlCompiler
        ├── AggregateSqlCompiler
        ├── WindowSqlCompiler
        ├── SetOperationSqlCompiler
        ├── CteSqlCompiler
        └── ExtensionSqlCompiler
```

---

# 28. Composition over inheritance

No se recomienda:

```text
AbstractSqlCompiler
     ↓
AbstractMysqlCompiler
     ↓
MysqlSelectCompiler
     ↓
MysqlSpecialSelectCompiler
```

con jerarquías profundas.

Preferir:

```text
small compiler components
+
explicit composition
+
dialect services
+
capability services
```

---

# 29. Query compiler family

Se podrá definir:

```php
interface QueryStatementCompiler
{
    public function supports(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): bool;

    public function compile(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): SqlStatementCompilation;
}
```

---

# 30. Statement compiler selection

La selección debe ser:

```text
typed
deterministic
validated
```

No depender de orden accidental de registro.

---

# 31. SQL AST interno

VoltStack podrá utilizar una representación intermedia de emisión:

```text
SqlEmissionTree
```

entre el Query/Physical representation y el texto final.

---

# 32. Query AST ≠ SQL AST

Distinción importante:

```text
Query AST
=
database-independent semantic query structure

SQL AST / Emission Model
=
dialect-oriented representation ready for SQL rendering
```

---

# 33. Razón

Ejemplo:

```text
Pagination(limit=10, offset=20)
```

es una intención estructurada.

Su representación SQL puede depender del dialecto.

---

# 34. SQL emission model

Modelo conceptual:

```php
interface SqlEmissionNode
{
    public function kind(): SqlEmissionNodeKind;
}
```

---

# 35. Posibles nodos

```text
SqlSelectStatement
SqlInsertStatement
SqlUpdateStatement
SqlDeleteStatement
SqlProjection
SqlFrom
SqlJoin
SqlPredicate
SqlExpression
SqlOrderBy
SqlGroupBy
SqlWindow
SqlCte
SqlSetOperation
SqlLimit
SqlOffset
SqlReturning
SqlLockingClause
SqlIdentifier
SqlPlaceholder
SqlLiteral
SqlExtensionNode
```

---

# 36. SQL AST no será segunda semántica

El `SqlEmissionTree` no deberá convertirse en otro Query Engine.

No realizará:

```text
symbol resolution
type inference
join resolution
query optimization
cardinality estimation
```

---

# 37. Lowering

La transformación:

```text
CompilableDatabaseOperation
→
SqlEmissionTree
```

será denominada:

```text
SQL Lowering
```

---

# 38. SQL Lowering ≠ Execution Lowering

Recordando el documento anterior:

```text
Execution Lowering
=
Physical Plan → Execution topology

SQL Lowering
=
Compilable DB operation → SQL-oriented representation
```

---

# 39. SQL Lowering ≠ Rendering

También:

```text
SQL Lowering
≠
SQL Rendering
```

Lowering construye estructura.

Rendering produce texto.

---

# 40. Pipeline recomendado

```text
Compilable Operation
       │
       ▼
Compilation Validation
       │
       ▼
SQL Lowering
       │
       ▼
SqlEmissionTree
       │
       ▼
Dialect Adaptation
       │
       ▼
SqlEmissionTree
       │
       ▼
Placeholder Allocation
       │
       ▼
SQL Rendering
       │
       ▼
Compiled SQL Text
       │
       ▼
Binding Layout
       │
       ▼
CompiledDatabaseCommand
```

El pipeline exacto será detallado en:

```text
67_DATABASE_SQL_COMPILER_PIPELINE.md
```

---

# 41. Compilation Validation

Antes de generar SQL deberán verificarse requisitos como:

```text
operation kind supported
required capabilities available
required extension compiler available
parameter definitions valid
dialect construct representable
physical requirements representable
security requirements preserved
```

---

# 42. Validation ≠ semantic validation

No se volverá a comprobar:

```text
does column exist?
is alias ambiguous?
is aggregate legal?
is window scope valid?
```

Eso ya pertenece a capas anteriores.

---

# 43. Compiler validation

La pregunta será:

> ¿Puede esta operación ya válida representarse correctamente en este dialecto/plataforma?

---

# 44. Capability failure

Ejemplo:

```text
Operation requires DELETE RETURNING
```

pero:

```text
Platform capability:
supportsDeleteReturning = false
```

Si no existe una emulación previamente autorizada y semánticamente equivalente:

```text
CompilationCapabilityException
```

---

# 45. No semantic invention

El Compiler no debe responder:

```text
"la plataforma no soporta esto,
así que cambiaré la query por otra parecida"
```

---

# 46. Emulation ownership

Una emulación compleja que cambie estrategia deberá haber sido seleccionada por:

```text
Physical Planner
```

o representada explícitamente en:

```text
ExecutionPlan
```

antes de llegar al Compiler.

---

# 47. Ejemplo RETURNING

No:

```text
Compiler sees unsupported RETURNING
→ emits UPDATE
→ emits SELECT
```

por iniciativa propia.

Sí:

```text
Physical/Execution Plan
→ explicit emulation strategy
→ multiple compilable operations
→ Compiler renders each operation
```

---

# 48. Compiler cannot change observable semantics

Debe mantenerse:

```text
Semantics(CompiledDatabaseCommand)
=
Semantics(CompilableDatabaseOperation)
```

dentro del contrato de la plataforma.

---

# 49. SQL Renderer

Contrato:

```php
interface SqlRenderer
{
    public function render(
        SqlEmissionTree $tree,
        SqlRenderingContext $context,
    ): RenderedSql;
}
```

---

# 50. RenderedSql

```php
final readonly class RenderedSql
{
    public function __construct(
        public string $sql,
        public SqlSourceMap $sourceMap,
    ) {}
}
```

---

# 51. SQL text

El texto SQL será una salida del proceso.

No la representación interna principal durante todo el Compiler.

---

# 52. String concatenation

Evitar una arquitectura basada exclusivamente en:

```php
$sql = 'SELECT ';
$sql .= implode(',', $columns);
$sql .= ' FROM ';
$sql .= $table;
```

distribuida por todo el sistema.

---

# 53. Structured rendering

Preferir:

```text
semantic/physical structures
       ↓
SQL emission nodes
       ↓
centralized renderer
       ↓
SQL text
```

---

# 54. Beneficios

Esto facilita:

```text
dialect portability
correct quoting
placeholder tracking
source maps
debugging
extension nodes
testing
deterministic output
```

---

# 55. Identifier compiler

Los identifiers serán estructurados.

```php
final readonly class SqlIdentifier
{
    public function __construct(
        public IdentifierPartList $parts,
    ) {}
}
```

---

# 56. Identifier ≠ raw string

No:

```php
$table = 'users; DROP TABLE users';
```

inyectado como identifier SQL.

---

# 57. Identifier quoting

Será responsabilidad del Dialect.

Ejemplo conceptual:

```text
users.name
```

podrá producir:

```sql
"users"."name"
```

o:

```sql
`users`.`name`
```

según dialecto.

---

# 58. Identifier quote ≠ value escaping

Nunca confundir:

```text
identifier quoting
```

con:

```text
runtime value binding
```

---

# 59. Runtime values

Los valores runtime deberán permanecer como parámetros.

Ejemplo:

```text
Predicate:
email = Parameter(P1)
```

se compila conceptualmente como:

```sql
email = ?
```

o equivalente.

---

# 60. No value interpolation

Nunca:

```php
$sql = "WHERE email = '$email'";
```

---

# 61. Placeholder Compiler

El Compiler asignará placeholders concretos.

---

# 62. ParameterId ≠ placeholder

```text
ParameterId
≠
ExecutionParameterSlotId
≠
SqlPlaceholderId
≠
RenderedPlaceholder
≠
DriverBindingPosition
```

---

# 63. Placeholder styles

El sistema deberá poder representar:

```text
?
$1
$2
:name
```

sin que las capas superiores dependan de ello.

---

# 64. PlaceholderStyle

```php
enum PlaceholderStyle
{
    case POSITIONAL_QUESTION_MARK;
    case POSITIONAL_NUMBERED;
    case NAMED;
}
```

---

# 65. Platform-specific selection

Ejemplo conceptual:

```text
MySQL PDO
→ ?

PostgreSQL native-like representation
→ $1 / driver-appropriate form

SQLite
→ ?
```

La representación concreta dependerá del driver/compiler integration.

---

# 66. Placeholder allocator

```php
interface SqlPlaceholderAllocator
{
    public function allocate(
        ParameterId $parameter,
        PlaceholderAllocationContext $context,
    ): SqlPlaceholder;
}
```

---

# 67. Operation-scoped allocator

Nunca:

```php
static int $placeholderCounter;
```

---

# 68. Repeated parameter

Una misma `ParameterId` utilizada múltiples veces puede requerir:

```text
same logical parameter
+
multiple rendered placeholders
```

dependiendo del driver.

---

# 69. Parameter occurrence

Por ello se distinguirán:

```text
ParameterDefinition
ParameterOccurrence
SqlPlaceholder
DriverBinding
```

---

# 70. ParameterOccurrenceId

Modelo:

```php
final readonly class ParameterOccurrence
{
    public function __construct(
        public ParameterOccurrenceId $id,
        public ParameterId $parameter,
        public SqlPlaceholder $placeholder,
    ) {}
}
```

---

# 71. Binding layout

El resultado compilado deberá contener un layout.

```php
final readonly class CompiledBindingLayout
{
    public function __construct(
        public CompiledBindingSlotList $slots,
        public ParameterOccurrenceMap $occurrences,
    ) {}
}
```

---

# 72. Binding layout ≠ values

El layout describe:

```text
what
where
how
```

pero no necesariamente:

```text
current runtime value
```

---

# 73. CompiledBindingSlot

Puede contener:

```text
ParameterId
placeholder
position/name
binding type
conversion requirement
sensitivity
```

---

# 74. Type conversion

La semántica de tipos ya fue resuelta.

El Compiler puede seleccionar representación concreta.

Ejemplo:

```text
Domain<UserId>
      ↓
QueryType Integer
      ↓
Platform BIGINT
      ↓
Driver integer/string binding strategy
```

---

# 75. Compiler does not infer domain meaning

No deberá deducir:

```text
"esta columna parece terminar en _id,
por tanto es UserId"
```

---

# 76. Literal compilation

No todo valor SQL es runtime parameter.

Algunos literales estructurales pueden ser compilables directamente.

Ejemplos:

```text
NULL
TRUE
FALSE
DEFAULT
```

---

# 77. Literal policy

Debe existir una política explícita para diferenciar:

```text
SQL structural literal
runtime value
compile-time constant
raw SQL
```

---

# 78. Runtime value default

Por defecto:

```text
application value
→ parameter
```

---

# 79. LIMIT parameters

La capacidad de parameterizar:

```text
LIMIT ?
```

puede variar según plataforma/driver.

La decisión deberá ser capability-driven.

---

# 80. Compile-time specialization

Cuando una plataforma exija literalizar cierto valor estructural, deberá existir un mecanismo explícito:

```text
CompileTimeSpecialization
```

y no interpolación ad hoc.

---

# 81. Specialization fingerprint

Si un valor afecta SQL generado:

```text
CompiledQueryFingerprint
```

deberá reflejarlo apropiadamente.

---

# 82. Sensitive specialization

Valores sensibles no deberán convertirse en SQL literal sólo para facilitar compilación.

---

# 83. Expression compilation

El `ExpressionSqlCompiler` manejará estructuras como:

```text
ColumnReference
ParameterExpression
LiteralExpression
BinaryExpression
UnaryExpression
FunctionExpression
CastExpression
CaseExpression
TupleExpression
SubqueryExpression
AggregateExpression
WindowExpression
ExtensionExpression
RawExpression
```

---

# 84. Expression compiler ≠ semantic resolver

Recibe expresiones ya resueltas.

---

# 85. Function compilation

Una función semántica:

```text
FunctionId::STRING_LENGTH
```

puede tener representación diferente según dialecto.

---

# 86. Semantic function ≠ SQL function spelling

Ejemplo conceptual:

```text
Semantic Function
    STRING_LENGTH
```

puede mapearse a diferentes construcciones SQL.

---

# 87. Function compiler registry

```php
interface SqlFunctionCompiler
{
    public function supports(
        FunctionSemanticId $function,
        SqlCompilationContext $context,
    ): bool;

    public function compile(
        FunctionExpression $expression,
        SqlCompilationContext $context,
    ): SqlEmissionNode;
}
```

---

# 88. Operator compilation

Similarmente:

```text
semantic operator
≠
textual SQL operator
```

---

# 89. Predicate compilation

Debe soportar:

```text
comparison
AND
OR
NOT
IS NULL
IS NOT NULL
BETWEEN
IN
EXISTS
LIKE
regex
distinctness
JSON predicates
tuple predicates
raw predicates
extension predicates
```

---

# 90. SQL 3VL preservation

El Compiler deberá preservar las decisiones semánticas relacionadas con:

```text
TRUE
FALSE
UNKNOWN
NULL
```

---

# 91. No PHP boolean translation

Nunca traducir semántica SQL a:

```text
PHP truthiness
```

durante compilación.

---

# 92. Predicate parentheses

El renderer deberá preservar precedencia correctamente.

---

# 93. Precedence model

El Dialect podrá exponer:

```php
interface SqlOperatorPrecedenceTable
{
    public function precedence(
        SqlOperatorId $operator,
    ): SqlPrecedence;
}
```

---

# 94. Parenthesis strategy

Preferir:

```text
correctness
+
determinism
```

sobre minimizar obsesivamente cada paréntesis.

---

# 95. Relation compilation

`RelationSqlCompiler` manejará:

```text
table relation
view relation
derived table
subquery
CTE reference
VALUES relation
table function
extension relation
```

---

# 96. Relation identity ≠ table name

La relación semántica puede ser una instancia específica con alias propio.

---

# 97. Alias compilation

Aliases se asignarán desde estructuras ya resueltas.

No deberán reinventar symbol resolution.

---

# 98. Alias allocator

Cuando sea necesario generar aliases internos:

```text
SqlAliasAllocator
```

será operation-scoped y determinista.

---

# 99. Generated aliases

Ejemplo:

```text
__vs_q1
__vs_q2
```

pero la convención exacta será parte del compiler configuration.

---

# 100. Collision avoidance

Generated aliases deberán evitar colisión con aliases del usuario.

---

# 101. Alias generation ≠ semantic identity

Siempre:

```text
SemanticRelationId
≠
SqlAlias
```

---

# 102. Join compilation

El Join Compiler recibe un join físico/lógico ya seleccionado.

---

# 103. Join Compiler no reordena

Nunca:

```text
Compiler:
A JOIN B
→
B JOIN A
```

porque cree que será más rápido.

---

# 104. Join Compiler no cambia tipo

Nunca:

```text
LEFT JOIN
→
INNER JOIN
```

durante rendering.

---

# 105. Join syntax

Sí puede decidir representación sintáctica:

```text
JOIN
INNER JOIN
LEFT JOIN
LEFT OUTER JOIN
CROSS JOIN
```

según dialecto/configuración, manteniendo semántica.

---

# 106. USING

Una representación estructurada `USING` podrá conservarse si la plataforma la soporta y el plan lo permite.

---

# 107. USING fallback

Una transformación de:

```text
USING(a)
```

a una condición equivalente podrá realizarse sólo si el Compiler posee una regla de representación exacta y toda la semántica necesaria ya está explícita.

No debe re-resolver columnas.

---

# 108. CTE compilation

El CTE Compiler manejará:

```text
WITH
WITH RECURSIVE
CTE definitions
CTE references
column lists
materialization syntax where supported
```

---

# 109. CTE dependency order

El orden válido de CTEs deberá venir resuelto o ser derivable de dependency metadata explícita.

No mediante parsing de SQL.

---

# 110. Recursive CTE

El Compiler representa la recursión.

No determina su semántica.

---

# 111. Materialization intent

Puede compilar:

```text
MATERIALIZED
NOT MATERIALIZED
```

cuando la plataforma lo soporte y el plan lo haya seleccionado.

---

# 112. Unsupported materialization hint

No deberá emitir sintaxis inválida.

---

# 113. Set operations

El Compiler deberá representar:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
```

según capabilities.

---

# 114. Set operation semantics

No deberá sustituir:

```text
UNION ALL
```

por:

```text
UNION
```

---

# 115. Unsupported set operation

Si una estrategia equivalente requiere emulación compleja, debe haber sido planificada antes.

---

# 116. Aggregation

El Compiler manejará:

```text
GROUP BY
GROUPING SETS
ROLLUP
CUBE
aggregate calls
aggregate DISTINCT
aggregate FILTER
aggregate ordering
```

según capabilities.

---

# 117. Aggregate FILTER

Si una plataforma no soporta:

```sql
COUNT(*) FILTER (WHERE ...)
```

el Compiler no inventará automáticamente una expresión `CASE` salvo que exista una regla de representación formalmente declarada como exacta y autorizada.

---

# 118. Compiler-local representational rewrite

Se permitirá una categoría muy limitada:

```text
Representational Rewrite
```

---

# 119. Representational Rewrite

Es una transformación:

```text
same already-decided semantics
+
same physical strategy
+
different SQL spelling
```

---

# 120. Ejemplo

```text
Boolean literal TRUE
```

puede representarse como:

```sql
TRUE
```

o una representación equivalente requerida por la plataforma.

---

# 121. No logical rewrite

No pertenece al Compiler:

```text
predicate pushdown
join elimination
subquery decorrelation
join reordering
projection pruning
```

---

# 122. Window compilation

Debe representar:

```text
OVER
PARTITION BY
ORDER BY
ROWS
RANGE
GROUPS
frame boundaries
frame exclusion
named windows
```

según capabilities.

---

# 123. Effective frame

El Compiler deberá recibir el frame semánticamente efectivo cuando su representación requiera conocerlo.

No volverá a inferir defaults semánticos.

---

# 124. Window ordering ≠ query ordering

La compilación conservará ambos scopes por separado.

---

# 125. DML compilation

Compilers específicos:

```text
InsertSqlCompiler
UpdateSqlCompiler
DeleteSqlCompiler
```

---

# 126. INSERT

Deberá soportar estructuras como:

```text
VALUES
INSERT SELECT
DEFAULT VALUES
conflict strategy
RETURNING
```

según capabilities.

---

# 127. UPDATE

Deberá representar:

```text
target
assignments
sources
joins/from
predicate
ordering
limit
returning
```

sólo donde el physical plan y plataforma lo permitan.

---

# 128. DELETE

Igualmente:

```text
target
using/from
joins
predicate
ordering
limit
returning
```

según dialecto/capabilities.

---

# 129. DML safety

El Compiler no decide si un UPDATE/DELETE sin WHERE es permitido.

Esa política debe haber sido validada antes.

---

# 130. DML mutation semantics

No deberá cambiar:

```text
mutation target
mutation set
assignment semantics
RETURNING semantics
locking requirements
```

---

# 131. Upsert

Upsert será un concepto estructurado.

No una cadena:

```php
'on duplicate key update ...'
```

en Query Builder.

---

# 132. Platform lowering de upsert

La representación puede variar:

```text
ON CONFLICT
ON DUPLICATE KEY UPDATE
```

u otras formas.

El Compiler traduce una estrategia previamente validada.

---

# 133. Upsert ≠ generic UPDATE

No se reducirá ad hoc a:

```text
SELECT
then INSERT or UPDATE
```

porque eso cambia concurrencia/atomicidad.

---

# 134. Locking compilation

El Compiler podrá representar:

```text
FOR UPDATE
FOR SHARE
NOWAIT
SKIP LOCKED
```

cuando estén explícitamente requeridos y soportados.

---

# 135. Locking strategy

No decidirá si la query debería usar locks.

---

# 136. Pagination compilation

Debe representar:

```text
LIMIT
OFFSET
FETCH FIRST
```

o sintaxis equivalente.

---

# 137. Pagination semantics

El Compiler no añadirá `ORDER BY` automáticamente.

---

# 138. LIMIT ≠ deterministic order

Se conserva:

```text
LIMIT
does not imply
ORDER BY
```

---

# 139. NULL ordering

Las diferencias:

```text
NULLS FIRST
NULLS LAST
```

deberán modelarse explícitamente.

---

# 140. Emulación de NULL ordering

Sólo podrá usarse una representación alternativa cuando sea exacta dentro de las capacidades y tipos conocidos.

---

# 141. Collation

Collation será semántica observable.

El Compiler deberá preservar:

```text
collation identity
case sensitivity requirements
locale requirements
```

cuando formen parte del query.

---

# 142. No default collation guessing

No deberá asumir que:

```text
database default collation
```

es equivalente a una collation explícitamente requerida.

---

# 143. JSON compilation

JSON será capability-driven.

---

# 144. Semantic JSON operations

Ejemplo:

```text
JsonExtract
JsonContains
JsonExists
JsonSet
```

podrán mapearse a diferentes construcciones.

---

# 145. No vendor JSON in Query Builder

Idealmente el usuario no tendrá que escribir:

```text
JSON_EXTRACT(...)
```

para una operación portable conocida.

---

# 146. Vendor escape hatch

Cuando necesite funcionalidad específica:

```text
RawExpression
```

o:

```text
Semantic Extension
```

según el caso.

---

# 147. Raw SQL

Raw permanece un escape hatch explícito.

---

# 148. RawExpression compiler

No:

```text
parse raw SQL
optimize raw SQL
resolve raw identifiers
```

---

# 149. Raw rendering

El Compiler sólo podrá insertar raw SQL en un contexto previamente autorizado.

---

# 150. Raw parameters

Incluso raw deberá favorecer:

```text
raw structure
+
explicit parameters
```

y no interpolación.

---

# 151. Raw trust

El Compiler conservará:

```text
RawTrustMetadata
```

cuando sea necesario para diagnostics/security.

---

# 152. Security

El SQL Compiler constituye una frontera crítica de seguridad.

---

# 153. SQL injection prevention

La regla fundamental:

```text
Runtime Values
→
Bindings

Identifiers
→
Structured Identifier Compiler

SQL Syntax
→
Compiler/Renderer

Raw SQL
→
Explicit Escape Hatch
```

---

# 154. No generic escape()

VoltStack no basará su seguridad en:

```php
escape($userInput)
```

para construir SQL arbitrario.

---

# 155. Structural security

La protección principal proviene de separar:

```text
syntax
identifiers
parameters
raw fragments
```

en tipos distintos.

---

# 156. Security provenance

Los nodos generados por:

```text
tenant policy
authorization policy
framework security policy
```

deberán conservarse durante compilación.

---

# 157. Compiler cannot drop security predicates

Aunque un dialecto complique la representación.

---

# 158. Security failure

Si una política no puede representarse:

```text
CompilationSecurityException
```

o capability failure equivalente.

Nunca:

```text
silently omit policy
```

---

# 159. Sensitive binding metadata

El binding layout conservará:

```text
sensitive
secret
token
credential
PII classification where available
```

sin contener necesariamente el valor.

---

# 160. SQL diagnostics

Los diagnostics deberán poder mostrar SQL con placeholders.

---

# 161. Safe diagnostic SQL

Ejemplo:

```sql
SELECT *
FROM users
WHERE email = ?
  AND password_hash = ?
```

No:

```sql
SELECT *
FROM users
WHERE email = 'real@example.com'
  AND password_hash = '...'
```

---

# 162. Debug bindings

Si una herramienta muestra bindings, deberá pasar por políticas de redacción del sistema de Telemetry/Debug.

No será responsabilidad del SQL Compiler exponerlos.

---

# 163. SQL source map

Una característica importante será:

```text
SqlSourceMap
```

---

# 164. Propósito

Permite relacionar posiciones del SQL generado con:

```text
Query AST node
Semantic node
Physical node
Execution unit
Extension node
```

cuando sea posible.

---

# 165. Ejemplo

```text
SQL chars 0..6
→ SELECT statement

SQL chars 32..47
→ PredicateNode P14

SQL chars 48..49
→ ParameterOccurrence PO3
```

---

# 166. Beneficio

Facilita:

```text
database error diagnostics
debug toolbar
query explain
compiler tests
developer experience
```

---

# 167. Database error mapping

Posteriormente:

```text
Database syntax error at position 47
       │
       ▼
SqlSourceMap
       │
       ▼
PredicateNode P14
       │
       ▼
developer diagnostic
```

---

# 168. Source map ≠ source code map

Puede mapear a estructuras internas aunque la query haya sido construida mediante API.

---

# 169. Deterministic compilation

Mismos:

```text
compilable operation
platform
capabilities
dialect
configuration
extensions
```

deberán producir el mismo resultado compilado.

---

# 170. Determinism formula

```text
Compile(O, C) = Compile(O, C)
```

para el mismo snapshot/contexto.

---

# 171. No wall-clock dependency

La compilación no dependerá de:

```text
current time
random numbers
request order
global counters
memory addresses
```

salvo datos explícitos de especialización.

---

# 172. Generated names

Aliases/placeholders internos deberán generarse determinísticamente.

---

# 173. Compiler state

Todo estado mutable será:

```text
compilation-operation scoped
```

---

# 174. SqlCompilationSession

Podrá existir:

```php
final class SqlCompilationSession
{
    public function __construct(
        public readonly SqlCompilationContext $context,
        public readonly SqlAliasAllocator $aliases,
        public readonly SqlPlaceholderAllocator $placeholders,
        public readonly SqlSourceMapBuilder $sourceMap,
        public readonly SqlCompilationBudgetTracker $budget,
    ) {}
}
```

---

# 175. Session ≠ shared service

Una `SqlCompilationSession` no se comparte entre queries concurrentes.

---

# 176. Shared immutable state

Sí podrán compartirse:

```text
compiler registry
dialect descriptors
capability descriptors
function compiler registry
operator compiler registry
extension descriptors
```

una vez congelados.

---

# 177. FrankenPHP safety

En persistent workers:

```text
shared:
    frozen compiler registry
    immutable dialect descriptors
    immutable compiler components

operation-scoped:
    compilation session
    placeholder allocator
    alias allocator
    source map builder
    diagnostics
    budget
```

---

# 178. No leakage

Después de compilar:

```text
placeholder state
alias state
diagnostics
temporary nodes
```

no deben afectar la siguiente query.

---

# 179. Compiler registry

Modelo:

```php
interface SqlCompilerRegistry
{
    public function statementCompiler(
        DatabaseOperationKind $kind,
        SqlCompilationContext $context,
    ): QueryStatementCompiler;
}
```

---

# 180. Registry lifecycle

```text
bootstrap
   │
   ▼
discover
   │
   ▼
validate
   │
   ▼
resolve conflicts
   │
   ▼
freeze
```

---

# 181. No runtime mutation

Una vez activo el worker:

```text
CompilerRegistry
```

no deberá cambiar durante queries activas.

---

# 182. Platform compiler architecture

Se evitarán dos extremos.

### Extremo A

Un único mega compiler:

```text
if mysql...
elseif postgres...
elseif sqlite...
```

### Extremo B

Copiar completamente todo el Compiler por plataforma.

---

# 183. Modelo recomendado

```text
Shared Compiler Architecture
        │
        ├── common statement lowering
        ├── common expression lowering
        ├── common predicate lowering
        ├── common parameter model
        └── common diagnostics
                │
                ▼
       Dialect / Platform Services
        ├── MySQL
        ├── MariaDB
        ├── PostgreSQL
        └── SQLite
```

---

# 184. Platform specialization

Cuando una plataforma necesite comportamiento suficientemente diferente, podrá registrar:

```text
specialized compiler component
```

para un constructo concreto.

---

# 185. Ejemplo

```text
UpsertSqlCompiler
```

puede tener implementaciones:

```text
MySqlUpsertCompiler
PostgreSqlUpsertCompiler
SQLiteUpsertCompiler
```

sin duplicar todo `SelectSqlCompiler`.

---

# 186. MariaDB

MariaDB tendrá identidad de plataforma propia.

No será tratada permanentemente como:

```text
MySQL with another name
```

---

# 187. Shared capabilities

Podrá reutilizar componentes con MySQL cuando sus capacidades/sintaxis sean compatibles.

---

# 188. Capability-driven reuse

```text
shared behavior
because capability/syntax contract matches
```

no porque:

```text
vendor names look related
```

---

# 189. PostgreSQL

Podrá especializar:

```text
numbered placeholders
RETURNING
DISTINCT ON
JSON operators
array constructs
locking
CTE/materialization syntax
```

según contratos futuros.

---

# 190. SQLite

Deberá tratarse como plataforma completa.

No como:

```text
"test-only fake database"
```

---

# 191. SQLite capabilities

Las capacidades dependerán también de versión/build cuando corresponda.

---

# 192. Compiler extension system

El Compiler deberá ser extensible sin modificar el core.

---

# 193. Extension examples

```text
custom function
custom operator
custom predicate
custom relation
custom statement
custom platform construct
custom SQL emission node
```

---

# 194. SqlCompilerExtension

Contrato conceptual:

```php
interface SqlCompilerExtension
{
    public function id(): ExtensionId;

    public function version(): ExtensionVersion;

    public function register(
        SqlCompilerExtensionRegistry $registry,
    ): void;
}
```

---

# 195. Extension semantic prerequisite

Una extensión que introduce nueva semántica no podrá aparecer sólo en Compiler.

Debe existir desde el Query/Semantic Extension System.

---

# 196. Compiler-only extension

Sólo será apropiada para:

```text
representation of already-known semantics
```

---

# 197. Unknown extension

Nunca:

```text
unknown node
→ cast to string
```

---

# 198. Correct behavior

```text
UnknownSqlCompilationExtensionException
```

---

# 199. Extension capabilities

Toda extensión deberá declarar capabilities requeridas.

---

# 200. Extension fingerprint

La versión de la extensión deberá participar en:

```text
CompiledQueryFingerprint
```

cuando afecte la salida.

---

# 201. Compilation budget

La compilación tendrá límites.

---

# 202. SqlCompilationBudget

Puede limitar:

```text
AST nodes visited
SQL emission nodes
nesting depth
generated aliases
generated placeholders
rendered SQL length
CTE count
set operation branches
extension expansions
source map entries
```

---

# 203. Purpose

Protege contra:

```text
pathological generated queries
extension expansion
recursive structures
resource exhaustion
```

---

# 204. Budget exhaustion

Debe producir:

```text
SqlCompilationBudgetExceededException
```

No SQL truncado.

---

# 205. Maximum SQL length

No deberá existir un límite arbitrario universal escondido.

Debe ser:

```text
configurable
platform-aware where needed
explicit
```

---

# 206. Depth protection

Especialmente para:

```text
deep nested predicates
nested subqueries
nested CASE
nested set operations
```

---

# 207. Compilation complexity

Idealmente el rendering final será aproximadamente:

```text
O(number of emitted SQL nodes)
```

salvo constructos especiales.

---

# 208. Avoid quadratic concatenation

El renderer no debería construir grandes SQL mediante concatenaciones ineficientes repetitivas.

---

# 209. SqlTextBuilder

Puede existir una abstracción especializada:

```php
final class SqlTextBuilder
{
    public function append(...): void;

    public function appendIdentifier(...): void;

    public function appendPlaceholder(...): void;

    public function build(): string;
}
```

---

# 210. SqlTextBuilder ≠ raw builder API

Será infraestructura interna del Compiler.

No una API pública para queries.

---

# 211. Whitespace policy

El renderer tendrá una política determinista.

Por ejemplo:

```text
COMPACT
PRETTY
DEBUG
```

---

# 212. Semantic equality of formatting

Cambiar whitespace no cambia semántica, pero puede cambiar:

```text
SQL text fingerprint
debug output
prepared statement cache keys
```

Por eso la política debe ser explícita.

---

# 213. Canonical rendering

Para cache se recomienda una forma:

```text
CANONICAL
```

estable.

---

# 214. Pretty SQL

Debe ser principalmente diagnóstico.

---

# 215. CompiledDatabaseCommand

Artefacto principal de salida:

```php
final readonly class CompiledDatabaseCommand
{
    public function __construct(
        public CompiledDatabaseCommandId $id,
        public DatabaseOperationKind $kind,
        public RenderedSql $sql,
        public CompiledBindingLayout $bindings,
        public CompiledResultContract $result,
        public CompiledCommandRequirementSet $requirements,
        public CompilationDependencySet $dependencies,
        public CompiledQueryFingerprint $fingerprint,
        public CompilationMetadata $metadata,
    ) {}
}
```

---

# 216. Compiled command inmutable

El artifact será inmutable.

---

# 217. Compiled command ≠ prepared statement

Siempre:

```text
CompiledDatabaseCommand
≠
PreparedStatement
```

---

# 218. Compiled command ≠ execution instance

También:

```text
CompiledDatabaseCommand
≠
StatementExecutionInstance
```

---

# 219. Runtime flow

```text
CompiledDatabaseCommand
        │
        ▼
Execution Engine
        │
        ▼
Connection
        │
        ▼
prepare(SQL)
        │
        ▼
Prepared Statement
        │
        ▼
bind(runtime values)
        │
        ▼
execute()
```

---

# 220. Prepared statement compilation

La preparación arquitectónica para prepared statements se profundizará en:

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

---

# 221. Binding conversion

El compiled command podrá indicar conversiones requeridas:

```text
UUID object → string
DateTime → platform representation
enum → scalar representation
JSON value → JSON string/binary representation
```

---

# 222. Conversion timing

La conversión de valores reales ocurre normalmente durante binding/runtime.

El Compiler produce el:

```text
BindingConversionPlan
```

---

# 223. BindingConversionPlan

No contiene el valor.

Contiene instrucciones tipadas.

---

# 224. Result contract compilation

El Compiler también deberá describir cómo interpretar la forma de salida a nivel de database result.

---

# 225. CompiledResultContract

Puede contener:

```text
column positions
column aliases
database representation types
semantic result mapping IDs
affected-row expectation
RETURNING shape
```

---

# 226. Result contract ≠ hydration

No crea entidades.

---

# 227. Column alias stability

Los aliases generados para resultados deberán ser deterministas cuando sean utilizados para mapping.

---

# 228. Duplicate output names

El Compiler no deberá asumir que nombres duplicados son imposibles.

Puede necesitar aliases internos para evitar ambigüedad del driver.

---

# 229. User-visible name vs internal alias

Distinguir:

```text
OutputSemanticName
InternalSqlAlias
DriverColumnLabel
```

---

# 230. Result mapping

El mapping deberá preservar la identidad semántica de cada output.

---

# 231. Compilation dependencies

`CompilationDependencySet` podrá contener:

```text
platform identity/version
dialect version
capability fingerprint
compiler version
compiler configuration
extension versions
schema dependencies where representation-sensitive
physical/execution strategy fingerprint
```

---

# 232. Schema dependency

No toda compilación necesita fingerprint del schema completo.

Preferir dependencias específicas.

---

# 233. Example

Si la query sólo depende de:

```text
users.id
users.email
```

el compiled artifact no debería invalidarse necesariamente porque se añadió una tabla no relacionada.

---

# 234. Dependency precision

Esto será importante para:

```text
Compiled Query Cache
```

del documento 75.

---

# 235. CompiledQueryFingerprint

Conceptualmente:

```text
CompiledQueryFingerprint
=
OperationFingerprint
+
PlatformFingerprint
+
DialectFingerprint
+
CapabilityFingerprint
+
CompilerVersion
+
CompilerConfigurationFingerprint
+
ExtensionFingerprint
+
RepresentationSpecializationFingerprint
```

---

# 236. Runtime bindings excluded

Por defecto:

```text
CompiledQueryFingerprint
```

no incluirá valores runtime.

---

# 237. Specialization exception

Si una representación requiere compile-time specialization:

```text
SpecializationFingerprint
```

sí deberá incluir la información estructural relevante.

---

# 238. Semantic fingerprint ≠ compiled fingerprint

```text
SemanticFingerprint
≠
LogicalPlanFingerprint
≠
PhysicalPlanFingerprint
≠
ExecutionPlanFingerprint
≠
CompiledQueryFingerprint
```

---

# 239. Cada fingerprint responde otra pregunta

```text
Semantic:
same meaning?

Logical:
same logical operations?

Physical:
same physical strategy?

Execution:
same executable topology?

Compiled:
same target representation?
```

---

# 240. Compilation cacheability

Un compiled command podrá ser cacheable si:

```text
deterministic
dependency-complete
runtime-value-independent
extension-stable
platform-compatible
```

---

# 241. Non-cacheable compilation

Podrá marcarse explícitamente cuando exista:

```text
volatile compile-time extension
runtime-specialized SQL
non-deterministic external compiler
unsupported dependency tracking
```

aunque estos casos deberían ser excepcionales.

---

# 242. Compiler purity

Objetivo ideal:

```text
CompiledCommand
=
f(Operation, CompilationContext)
```

sin efectos externos.

---

# 243. Functional core

El Compiler debería acercarse a un:

```text
functional core
```

con:

```text
immutable inputs
operation-scoped builders
immutable output
no I/O
```

---

# 244. Diagnostics

`SqlCompilationDiagnostic` podrá incluir:

```text
severity
code
message
source node
platform
capability
extension
suggestion
```

---

# 245. Diagnostic codes

Ejemplos:

```text
DB-COMP-UNSUPPORTED-CAPABILITY
DB-COMP-UNREPRESENTABLE-OPERATION
DB-COMP-UNKNOWN-EXTENSION
DB-COMP-INVALID-IDENTIFIER
DB-COMP-PLACEHOLDER-LIMIT
DB-COMP-SQL-SIZE-LIMIT
DB-COMP-UNSUPPORTED-LOCKING
DB-COMP-UNSUPPORTED-RETURNING
```

---

# 246. Diagnostics ≠ silent fallback

No:

```text
warning + silently change semantics
```

---

# 247. Compilation trace

Para DEBUG:

```text
SqlCompilationTrace
```

podrá registrar:

```text
statement compiler selected
lowering rules used
dialect adaptations
aliases allocated
placeholder allocations
representational rewrites
capabilities checked
extensions invoked
rendering completed
```

---

# 248. Trace redaction

Nunca deberá registrar valores sensibles.

---

# 249. Explain compilation

Una herramienta futura podrá mostrar:

```text
Semantic Query
      ↓
Physical Plan
      ↓
Execution Unit
      ↓
SQL Compilation
      ↓
Generated SQL
```

---

# 250. Example diagnostic output

```text
Compiler:
PostgreSQLSqlCompiler

Operation:
SELECT

Capabilities:
CTE               yes
WINDOW             yes
RETURNING          yes
LATERAL            yes

Parameters:
P1 → $1
P2 → $2

SQL:
SELECT "u"."id", "u"."email"
FROM "users" AS "u"
WHERE "u"."active" = $1
  AND "u"."created_at" >= $2
ORDER BY "u"."created_at" DESC
LIMIT 20
```

---

# 251. No values in explain

El output no debe mostrar automáticamente:

```text
P1 = true
P2 = 2026-09-01...
```

si esos valores son runtime/sensitive.

---

# 252. Compiler error model

Excepciones propuestas:

```text
SqlCompilationException
UnsupportedSqlCompilationException
CompilationCapabilityException
CompilationSecurityException
SqlCompilationBudgetExceededException
SqlRenderingException
SqlIdentifierCompilationException
SqlExpressionCompilationException
SqlPredicateCompilationException
SqlRelationCompilationException
SqlJoinCompilationException
SqlCteCompilationException
SqlSetOperationCompilationException
SqlAggregateCompilationException
SqlWindowCompilationException
SqlDmlCompilationException
SqlPlaceholderCompilationException
SqlBindingLayoutException
SqlCompilerExtensionException
UnknownSqlCompilationExtensionException
CompiledResultContractException
```

---

# 253. Exception context

Errores deberán poder incluir:

```text
operation id
semantic node id
physical node id
execution unit id
compiler component
platform
capability
source map location
```

cuando estén disponibles.

---

# 254. Error ≠ vendor exception

Una excepción del Compiler no será una:

```text
PDOException
```

porque todavía no hay ejecución.

---

# 255. Database syntax errors

Si el SQL generado resulta rechazado por la base, eso ocurre posteriormente.

El sistema podrá mapearlo de vuelta mediante:

```text
CompiledDatabaseCommand
+
SqlSourceMap
```

---

# 256. Compiler correctness

Un SQL Compiler correcto debe satisfacer al menos:

```text
semantic preservation
syntactic validity for target dialect
parameter safety
identifier correctness
capability compliance
deterministic rendering
result-shape preservation
security preservation
```

---

# 257. Formalización

Para una operación `O`, plataforma `P` y contexto `C`:

```text
Compile(O, P, C) = S
```

deberá cumplir:

```text
Meaning_P(S)
=
Meaning(O)
```

dentro del dominio soportado.

---

# 258. No stronger guarantee than platform

Si una plataforma no puede representar una semántica:

```text
Compilation Failure
```

es preferible a:

```text
Approximate Semantics
```

---

# 259. Portability principle

```text
Portability
≠
Lowest Common Denominator
```

VoltStack podrá utilizar capacidades avanzadas cuando estén disponibles.

---

# 260. Capability-aware portability

El mismo Query Model podrá producir diferentes representaciones correctas por plataforma.

---

# 261. Example

Una operación semántica portable:

```text
CurrentTimestampExpression
```

puede tener distintas representaciones sintácticas.

El Query Model sigue siendo común.

---

# 262. Platform-specific feature

Una extensión PostgreSQL específica podrá requerir:

```text
Capability(PostgreSqlFeatureX)
```

y fallará limpiamente en otra plataforma.

---

# 263. Compiler profiles

Podrán existir configuraciones como:

```text
PRODUCTION
DEBUG
CANONICAL
```

pero no deberán cambiar semántica.

---

# 264. Profile effects

Pueden cambiar:

```text
whitespace
source map detail
trace detail
assertion level
diagnostic detail
```

No:

```text
join order
filter semantics
transaction semantics
security predicates
```

---

# 265. Testing architecture

El Compiler requerirá una suite especialmente rigurosa.

---

# 266. Test categories

```text
unit compilation tests
golden SQL tests
AST lowering tests
dialect tests
capability tests
parameter tests
identifier tests
expression tests
predicate tests
join tests
CTE tests
set operation tests
aggregate tests
window tests
DML tests
raw SQL tests
extension tests
security tests
determinism tests
persistent runtime tests
cross-platform tests
property tests
fuzz tests
integration tests
```

---

# 267. Golden tests

Ejemplo:

Input estructurado:

```text
Select(users)
Projection(id)
Predicate(active = P1)
```

Expected PostgreSQL:

```sql
SELECT "users"."id"
FROM "users"
WHERE "users"."active" = $1
```

---

# 268. Golden tests limitation

No deben ser la única estrategia.

Un string correcto puede ocultar errores semánticos difíciles.

---

# 269. Structural tests

También verificar:

```text
binding layout
source map
result contract
dependencies
fingerprints
capability checks
```

---

# 270. Cross-platform tests

La misma operación deberá probarse en:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando sea portable.

---

# 271. Negative capability tests

Ejemplo:

```text
Operation requires capability X
Platform does not support X
```

debe producir error esperado.

---

# 272. Identifier injection tests

Inputs maliciosos como:

```text
users; DROP TABLE users
```

no deberán convertirse en sintaxis arbitraria cuando sean tratados como identifiers.

---

# 273. Value injection tests

Valores como:

```text
' OR 1=1 --
```

deberán permanecer bindings.

---

# 274. Raw tests

Raw SQL debe verificar que:

```text
escape hatch is explicit
parameter handling remains structured
security metadata preserved
```

---

# 275. Placeholder tests

Casos:

```text
same parameter repeated
nested subquery parameters
CTE parameters
set operation parameters
generated predicates
correlated queries
large parameter counts
```

---

# 276. Parameter ordering

Debe ser determinista.

---

# 277. Source map tests

Cada región relevante deberá apuntar al nodo correcto.

---

# 278. Determinism tests

Compilar múltiples veces:

```text
Compile(O, C)
```

deberá producir:

```text
same SQL
same binding layout
same source map structure
same fingerprint
```

---

# 279. Persistent runtime tests

Compilar:

```text
Query A
Query B
Query A
```

en el mismo worker deberá producir resultados independientes.

---

# 280. No allocator leakage

Los placeholders de Query B no deberán continuar accidentalmente los contadores de Query A.

---

# 281. Concurrency tests

Dos compilaciones concurrentes usando el mismo registry deberán ser seguras.

---

# 282. Property tests

Propiedades útiles:

```text
all runtime values become parameters unless explicitly specialized

all emitted placeholders have binding slots

all binding slots correspond to valid parameters

all structured identifiers pass through dialect quoting

all emitted extension nodes have registered compiler

all required capabilities are validated
```

---

# 283. Fuzzing

Especialmente útil para:

```text
nested expressions
identifier edge cases
Unicode identifiers
deep predicates
parameter counts
raw boundaries
parentheses
string literals
JSON paths
```

---

# 284. Integration tests

El SQL generado deberá ejecutarse contra bases reales en CI cuando sea posible.

---

# 285. Platform matrix

Ejemplo:

| Feature | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---:|---:|---:|---:|
| Basic SELECT | ✓ | ✓ | ✓ | ✓ |
| INSERT | ✓ | ✓ | ✓ | ✓ |
| UPDATE | ✓ | ✓ | ✓ | ✓ |
| DELETE | ✓ | ✓ | ✓ | ✓ |
| CTE | capability | capability | capability | capability |
| Recursive CTE | capability | capability | capability | capability |
| Window | capability | capability | capability | capability |
| RETURNING | capability | capability | capability | capability |
| JSON | capability | capability | capability | capability |
| LATERAL | capability | capability | capability | capability |

La tabla real deberá derivarse de capabilities/versiones, no convertirse en hardcode arquitectónico.

---

# 286. Directory structure propuesta

```text
VoltStack/Quantum/Database/Query/Compiler
├── Contract
│   ├── SqlCompiler.php
│   ├── SqlCompilationCoordinator.php
│   ├── QueryStatementCompiler.php
│   ├── SqlRenderer.php
│   ├── SqlFunctionCompiler.php
│   ├── SqlPlaceholderAllocator.php
│   └── SqlCompilerExtension.php
│
├── Compilation
│   ├── SqlCompilationContext.php
│   ├── SqlCompilationSession.php
│   ├── SqlCompilationConfiguration.php
│   ├── SqlCompilationBudget.php
│   ├── SqlCompilationBudgetTracker.php
│   └── CompilationMetadata.php
│
├── Operation
│   ├── CompilableDatabaseOperation.php
│   ├── SelectCompilableOperation.php
│   ├── InsertCompilableOperation.php
│   ├── UpdateCompilableOperation.php
│   └── DeleteCompilableOperation.php
│
├── Statement
│   ├── SelectSqlCompiler.php
│   ├── InsertSqlCompiler.php
│   ├── UpdateSqlCompiler.php
│   └── DeleteSqlCompiler.php
│
├── Expression
│   ├── ExpressionSqlCompiler.php
│   ├── FunctionSqlCompilerRegistry.php
│   └── OperatorSqlCompilerRegistry.php
│
├── Predicate
│   └── PredicateSqlCompiler.php
│
├── Relation
│   ├── RelationSqlCompiler.php
│   ├── JoinSqlCompiler.php
│   └── SqlAliasAllocator.php
│
├── Cte
│   └── CteSqlCompiler.php
│
├── SetOperation
│   └── SetOperationSqlCompiler.php
│
├── Aggregation
│   └── AggregateSqlCompiler.php
│
├── Window
│   └── WindowSqlCompiler.php
│
├── Parameter
│   ├── ParameterOccurrence.php
│   ├── ParameterOccurrenceId.php
│   ├── SqlPlaceholder.php
│   ├── SqlPlaceholderId.php
│   ├── CompiledBindingLayout.php
│   ├── CompiledBindingSlot.php
│   └── BindingConversionPlan.php
│
├── Identifier
│   ├── SqlIdentifier.php
│   └── SqlIdentifierCompiler.php
│
├── Emission
│   ├── SqlEmissionNode.php
│   ├── SqlEmissionTree.php
│   ├── SqlStatementNode.php
│   ├── SqlExpressionNode.php
│   ├── SqlPredicateNode.php
│   └── SqlExtensionNode.php
│
├── Rendering
│   ├── SqlTextBuilder.php
│   ├── SqlRenderingContext.php
│   ├── RenderedSql.php
│   ├── SqlRenderingMode.php
│   └── SqlOperatorPrecedenceTable.php
│
├── Result
│   ├── CompiledResultContract.php
│   └── CompiledResultColumn.php
│
├── Command
│   ├── CompiledDatabaseCommand.php
│   ├── CompiledDatabaseCommandId.php
│   ├── CompiledCommandRequirementSet.php
│   └── CompiledQueryFingerprint.php
│
├── SourceMap
│   ├── SqlSourceMap.php
│   └── SqlSourceMapBuilder.php
│
├── Dependency
│   └── CompilationDependencySet.php
│
├── Registry
│   ├── SqlCompilerRegistry.php
│   └── SqlCompilerCatalog.php
│
├── Extension
│   ├── SqlCompilerExtensionRegistry.php
│   └── SqlCompilerExtensionDescriptor.php
│
├── Diagnostic
│   ├── SqlCompilationDiagnostic.php
│   ├── SqlCompilationTrace.php
│   └── SqlCompilationExplainer.php
│
├── Platform
│   ├── Common
│   ├── MySQL
│   ├── MariaDB
│   ├── PostgreSQL
│   └── SQLite
│
└── Exception
    ├── SqlCompilationException.php
    ├── CompilationCapabilityException.php
    ├── CompilationSecurityException.php
    ├── SqlCompilationBudgetExceededException.php
    ├── SqlRenderingException.php
    └── SqlCompilerExtensionException.php
```

---

# 287. Dependencias permitidas

```text
SQL Compiler
    │
    ├── Query Model / AST contracts
    ├── Semantic IDs and metadata
    ├── Physical Plan contracts
    ├── Execution Plan database-operation contracts
    ├── Database Platform
    ├── Dialect
    ├── Capability System
    ├── Query Type System
    ├── Parameter contracts
    └── Compiler Extension contracts
```

---

# 288. Dependencias prohibidas

```text
SQL Compiler
    ✗ live Connection
    ✗ PDO
    ✗ PDOStatement
    ✗ ResultCursor
    ✗ EntityManager
    ✗ UnitOfWork
    ✗ IdentityMap
    ✗ HTTP Request
    ✗ Controller
    ✗ current tenant singleton
    ✗ mutable global runtime state
```

---

# 289. Invariantes arquitectónicos

## DB-SQLCOMP-001

SQL Compiler será distinto de Query Builder.

## DB-SQLCOMP-002

SQL Compiler será distinto de Semantic Engine.

## DB-SQLCOMP-003

SQL Compiler será distinto de Query Optimizer.

## DB-SQLCOMP-004

SQL Compiler será distinto de Logical Planner.

## DB-SQLCOMP-005

SQL Compiler será distinto de Physical Planner.

## DB-SQLCOMP-006

SQL Compiler será distinto de Execution Planner.

## DB-SQLCOMP-007

SQL Compiler será distinto de Execution Engine.

## DB-SQLCOMP-008

Compiler transformará representación, no semántica.

## DB-SQLCOMP-009

Compiler no ejecutará queries.

## DB-SQLCOMP-010

Compiler no abrirá conexiones.

## DB-SQLCOMP-011

Compiler no realizará schema introspection oculta.

## DB-SQLCOMP-012

Compiler no consultará estadísticas runtime ocultamente.

## DB-SQLCOMP-013

Compiler no ejecutará EXPLAIN.

## DB-SQLCOMP-014

Compiler no utilizará live transaction state.

## DB-SQLCOMP-015

CompilableDatabaseOperation será estructurada.

## DB-SQLCOMP-016

CompiledDatabaseCommand será inmutable.

## DB-SQLCOMP-017

CompiledDatabaseCommand será distinto de PreparedStatement.

## DB-SQLCOMP-018

CompiledDatabaseCommand será distinto de ExecutionInstance.

## DB-SQLCOMP-019

Query AST será distinto de SQL Emission Tree.

## DB-SQLCOMP-020

SQL Emission Tree no será un segundo Semantic Engine.

## DB-SQLCOMP-021

SQL Lowering será distinto de Execution Lowering.

## DB-SQLCOMP-022

SQL Lowering será distinto de SQL Rendering.

## DB-SQLCOMP-023

Compilation Validation será distinta de Semantic Validation.

## DB-SQLCOMP-024

Compiler validará representabilidad.

## DB-SQLCOMP-025

Compiler no inventará semántica ante capability failure.

## DB-SQLCOMP-026

Complex emulation deberá planificarse antes de compilation.

## DB-SQLCOMP-027

Compiler-local rewrites serán sólo representacionales.

## DB-SQLCOMP-028

Compiler no hará predicate pushdown.

## DB-SQLCOMP-029

Compiler no hará join reordering.

## DB-SQLCOMP-030

Compiler no hará join elimination.

## DB-SQLCOMP-031

Compiler no hará subquery decorrelation.

## DB-SQLCOMP-032

Compiler no hará cardinality estimation.

## DB-SQLCOMP-033

Compiler no seleccionará physical join algorithms.

## DB-SQLCOMP-034

Compiler no seleccionará indexes.

## DB-SQLCOMP-035

Identifier compilation será estructurada.

## DB-SQLCOMP-036

Identifier quoting será distinto de value escaping.

## DB-SQLCOMP-037

Runtime values serán parameters por defecto.

## DB-SQLCOMP-038

Runtime values no serán interpolados en SQL.

## DB-SQLCOMP-039

ParameterId será distinto de SQL placeholder.

## DB-SQLCOMP-040

SQL placeholder será distinto de driver binding position.

## DB-SQLCOMP-041

Parameter occurrence será first-class.

## DB-SQLCOMP-042

Placeholder allocation será operation-scoped.

## DB-SQLCOMP-043

Placeholder allocation no usará global counters.

## DB-SQLCOMP-044

Binding layout será distinto de runtime binding values.

## DB-SQLCOMP-045

Binding conversion plan no contendrá runtime value.

## DB-SQLCOMP-046

Semantic types no serán reinventados por Compiler.

## DB-SQLCOMP-047

Compiler podrá seleccionar platform representation types.

## DB-SQLCOMP-048

Structural SQL literals serán distintos de application values.

## DB-SQLCOMP-049

Application values serán parameterized por defecto.

## DB-SQLCOMP-050

Compile-time specialization será explícita.

## DB-SQLCOMP-051

Specialization que afecte SQL afectará fingerprint.

## DB-SQLCOMP-052

Sensitive values no serán literalizados arbitrariamente.

## DB-SQLCOMP-053

Expression Compiler no resolverá symbols.

## DB-SQLCOMP-054

Semantic function identity será distinta de SQL spelling.

## DB-SQLCOMP-055

Semantic operator identity será distinta de SQL spelling.

## DB-SQLCOMP-056

Predicate compilation preservará SQL 3VL.

## DB-SQLCOMP-057

Compiler no usará PHP truthiness para predicates.

## DB-SQLCOMP-058

Predicate precedence será preservada.

## DB-SQLCOMP-059

Relation identity será distinta de SQL alias.

## DB-SQLCOMP-060

Generated aliases serán deterministas.

## DB-SQLCOMP-061

Generated aliases evitarán collisions.

## DB-SQLCOMP-062

Alias allocator será operation-scoped.

## DB-SQLCOMP-063

Join Compiler no reordenará joins.

## DB-SQLCOMP-064

Join Compiler no cambiará join type.

## DB-SQLCOMP-065

JOIN USING no será re-resuelto semánticamente.

## DB-SQLCOMP-066

CTE recursion será representada, no reinterpretada.

## DB-SQLCOMP-067

CTE materialization semantics serán preservadas.

## DB-SQLCOMP-068

Set operation multiplicity será preservada.

## DB-SQLCOMP-069

UNION ALL no será sustituido por UNION.

## DB-SQLCOMP-070

Aggregate semantics serán preservadas.

## DB-SQLCOMP-071

Window semantics serán preservadas.

## DB-SQLCOMP-072

Window frame defaults semánticos no serán reinventados.

## DB-SQLCOMP-073

Window ordering será distinto de final query ordering.

## DB-SQLCOMP-074

INSERT semantics serán preservadas.

## DB-SQLCOMP-075

UPDATE mutation set será preservado.

## DB-SQLCOMP-076

DELETE mutation set será preservado.

## DB-SQLCOMP-077

DML safety policy no será decidida por Compiler.

## DB-SQLCOMP-078

Upsert será estructurado.

## DB-SQLCOMP-079

Compiler no implementará upsert mediante unsafe select-then-write.

## DB-SQLCOMP-080

Lock requirements serán preservados.

## DB-SQLCOMP-081

Compiler no inventará locking.

## DB-SQLCOMP-082

Pagination semantics serán preservadas.

## DB-SQLCOMP-083

LIMIT no implicará ordering.

## DB-SQLCOMP-084

NULL ordering será explícito cuando observable.

## DB-SQLCOMP-085

Collation semantics serán preservadas.

## DB-SQLCOMP-086

Compiler no adivinará collation equivalence.

## DB-SQLCOMP-087

JSON compilation será capability-driven.

## DB-SQLCOMP-088

Raw SQL seguirá siendo escape hatch explícito.

## DB-SQLCOMP-089

Compiler no parseará raw SQL para optimizarlo.

## DB-SQLCOMP-090

Raw runtime values deberán permanecer parameterizable cuando sea posible.

## DB-SQLCOMP-091

SQL injection prevention será estructural.

## DB-SQLCOMP-092

Security predicates no podrán omitirse.

## DB-SQLCOMP-093

Tenant predicates no podrán omitirse.

## DB-SQLCOMP-094

Authorization predicates no podrán omitirse.

## DB-SQLCOMP-095

Unrepresentable security semantics producirán failure.

## DB-SQLCOMP-096

Diagnostics usarán placeholders.

## DB-SQLCOMP-097

Compiler diagnostics no expondrán sensitive values.

## DB-SQLCOMP-098

SqlSourceMap será soportable.

## DB-SQLCOMP-099

Source maps podrán mapear SQL a query structures.

## DB-SQLCOMP-100

Compilation será determinista.

## DB-SQLCOMP-101

Compilation no dependerá de wall clock.

## DB-SQLCOMP-102

Compilation no dependerá de random values.

## DB-SQLCOMP-103

Compilation no dependerá de memory addresses.

## DB-SQLCOMP-104

Compilation mutable state será operation-scoped.

## DB-SQLCOMP-105

SqlCompilationSession no será compartida entre queries concurrentes.

## DB-SQLCOMP-106

Compiler registry podrá ser compartido si es immutable.

## DB-SQLCOMP-107

Compiler registry será frozen después de bootstrap.

## DB-SQLCOMP-108

Registry conflict resolution será explícita.

## DB-SQLCOMP-109

Compiler selection no dependerá de registration order.

## DB-SQLCOMP-110

Platform-specific behavior favorecerá capabilities.

## DB-SQLCOMP-111

Vendor conditionals no sustituirán capability contracts.

## DB-SQLCOMP-112

MariaDB tendrá platform identity propia.

## DB-SQLCOMP-113

SQLite será plataforma de primera clase.

## DB-SQLCOMP-114

Compiler extensions no podrán introducir semántica desconocida tardíamente.

## DB-SQLCOMP-115

Unknown compiler extensions producirán error.

## DB-SQLCOMP-116

Compiler extension versions participarán en fingerprints.

## DB-SQLCOMP-117

Compiler tendrá complexity budget.

## DB-SQLCOMP-118

Budget exhaustion no producirá SQL parcial válido.

## DB-SQLCOMP-119

Rendered SQL size será bounded/configurable.

## DB-SQLCOMP-120

Renderer evitará quadratic string construction.

## DB-SQLCOMP-121

Rendering policy será determinista.

## DB-SQLCOMP-122

Canonical rendering será estable.

## DB-SQLCOMP-123

Pretty rendering no cambiará semántica.

## DB-SQLCOMP-124

Compiled result contract será explícito.

## DB-SQLCOMP-125

Compiled result contract será distinto de hydration.

## DB-SQLCOMP-126

Output semantic identity será preservada.

## DB-SQLCOMP-127

Internal SQL alias será distinto de user-visible output name.

## DB-SQLCOMP-128

Compilation dependencies serán explícitas.

## DB-SQLCOMP-129

Dependency tracking evitará invalidación innecesariamente global.

## DB-SQLCOMP-130

CompiledQueryFingerprint será distinto de SemanticFingerprint.

## DB-SQLCOMP-131

CompiledQueryFingerprint será distinto de PhysicalPlanFingerprint.

## DB-SQLCOMP-132

Runtime bindings serán excluidos del compiled fingerprint por defecto.

## DB-SQLCOMP-133

Compiler será cache-friendly.

## DB-SQLCOMP-134

Cacheability requerirá dependency completeness.

## DB-SQLCOMP-135

Compiler buscará comportamiento funcional/puro.

## DB-SQLCOMP-136

Compilation exceptions serán distintas de driver exceptions.

## DB-SQLCOMP-137

Compiler no lanzará PDOException por actividad propia.

## DB-SQLCOMP-138

Unsupported semantics fallarán antes de emitir comando inválido.

## DB-SQLCOMP-139

Portability no significará lowest common denominator.

## DB-SQLCOMP-140

Platform-specific capabilities podrán aprovecharse explícitamente.

## DB-SQLCOMP-141

Compiler profiles no cambiarán query semantics.

## DB-SQLCOMP-142

Golden tests no serán la única estrategia de testing.

## DB-SQLCOMP-143

Cross-platform conformance será obligatorio para features portables.

## DB-SQLCOMP-144

Injection tests serán obligatorios.

## DB-SQLCOMP-145

Determinism tests serán obligatorios.

## DB-SQLCOMP-146

Persistent worker isolation será obligatorio.

## DB-SQLCOMP-147

Concurrent compilation deberá ser segura.

## DB-SQLCOMP-148

Every emitted placeholder tendrá binding metadata.

## DB-SQLCOMP-149

Every binding slot corresponderá a un parámetro válido.

## DB-SQLCOMP-150

Every emitted extension node tendrá compiler registrado.

## DB-SQLCOMP-151

Every required capability será validada.

## DB-SQLCOMP-152

Compiler no dependerá del ORM.

## DB-SQLCOMP-153

Active Record no tendrá compiler SQL separado.

## DB-SQLCOMP-154

Repository API no tendrá compiler SQL separado.

## DB-SQLCOMP-155

Query Builder y ORM convergerán en el mismo Compiler.

## DB-SQLCOMP-156

Compiler no contendrá current tenant global.

## DB-SQLCOMP-157

Compiler no contendrá current request global.

## DB-SQLCOMP-158

Compiler no contendrá current connection global.

## DB-SQLCOMP-159

Compilation result deberá ser completamente describible sin ejecutar la query.

## DB-SQLCOMP-160

Semantics(CompiledCommand) deberá preservar Semantics(CompilableOperation).

---

# 290. Anti-patrones

## 290.1 SQL desde Query Builder

Incorrecto:

```php
$query->where('email', $email);

return "SELECT ... WHERE email = '$email'";
```

El Builder crea estructura, no SQL.

---

## 290.2 Mega Compiler

Incorrecto:

```php
switch ($database) {
    case 'mysql':
        // thousands of lines
    case 'pgsql':
        // thousands of lines
}
```

---

## 290.3 Compiler que optimiza

Incorrecto:

```text
Compiler sees LEFT JOIN
→ detects null rejecting predicate
→ converts to INNER JOIN
```

Eso pertenece al Optimizer.

---

## 290.4 Compiler que selecciona index

Incorrecto:

```text
Compiler chooses users_email_idx
```

Eso pertenece al Physical Planner.

---

## 290.5 Compiler que abre conexión

Incorrecto:

```php
$version = $pdo->query('SELECT version()');
```

La capability/version debe venir en contexto explícito.

---

## 290.6 Interpolación

Incorrecto:

```php
$sql = "WHERE id = {$id}";
```

---

## 290.7 Identifiers como valores

Incorrecto:

```php
bindValue(':table', 'users');
```

Los identifiers no son runtime values.

---

## 290.8 Values como identifiers

Igualmente incorrecto:

```php
quoteIdentifier($email);
```

---

## 290.9 Raw fallback automático

Incorrecto:

```text
unknown query node
→ convert node to string
→ append SQL
```

---

## 290.10 Capability failure silencioso

Incorrecto:

```text
RETURNING unsupported
→ silently remove RETURNING
```

---

## 290.11 Estado global de placeholders

Incorrecto:

```php
static $parameterIndex = 0;
```

---

## 290.12 Dialect por todas partes

Incorrecto:

```php
if ($db === 'mysql') ...
```

repetido en cientos de componentes.

---

## 290.13 ORM SQL compiler

Incorrecto:

```text
ORM SQL Compiler
Query Builder SQL Compiler
Repository SQL Compiler
```

VoltStack tendrá un solo pipeline de compilación.

---

# 291. Ejemplo end-to-end

Query:

```php
$query = $db
    ->table('users')
    ->select('id', 'email')
    ->where('active', true)
    ->where('created_at', '>=', $date)
    ->orderBy('created_at', 'desc')
    ->limit(20);
```

El Builder produce estructura:

```text
SelectQuery
├── Projection
│   ├── users.id
│   └── users.email
├── From
│   └── users
├── Predicate
│   ├── users.active = P1
│   └── users.created_at >= P2
├── Order
│   └── users.created_at DESC
└── Limit
    └── 20
```

---

# 292. Después del pipeline

```text
Query AST
   ↓
Semantic Query
   ↓
Optimized Query
   ↓
Logical Plan
   ↓
Physical Plan
   ↓
Execution Plan
```

puede existir:

```text
DatabaseExecutionUnit U1
```

con una operación compilable.

---

# 293. PostgreSQL compilation

La operación puede bajar a:

```text
SqlSelectStatement
├── Projection
├── From
├── Where
├── OrderBy
└── Limit
```

y producir:

```sql
SELECT "u"."id", "u"."email"
FROM "users" AS "u"
WHERE "u"."active" = $1
  AND "u"."created_at" >= $2
ORDER BY "u"."created_at" DESC
LIMIT 20
```

Binding layout:

```text
$1 → Parameter P1 → Boolean
$2 → Parameter P2 → DateTime
```

---

# 294. MySQL compilation

La misma semántica podría producir:

```sql
SELECT `u`.`id`, `u`.`email`
FROM `users` AS `u`
WHERE `u`.`active` = ?
  AND `u`.`created_at` >= ?
ORDER BY `u`.`created_at` DESC
LIMIT 20
```

Binding layout:

```text
? occurrence #1 → P1
? occurrence #2 → P2
```

---

# 295. Lo que no cambia

Entre ambos:

```text
query meaning
predicate semantics
projection
ordering requirement
limit
parameter identity
```

---

# 296. Lo que puede cambiar

```text
identifier quoting
placeholder syntax
function spelling
platform-specific syntax
representational details
```

---

# 297. Arquitectura de separación por plataforma

```text
                    Query / Plan Structures
                             │
                             ▼
                    Common SQL Lowering
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Expressions     Predicates      Statements
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    SqlEmissionTree
                             │
                             ▼
                    Platform Adapter
                             │
          ┌──────────┬───────┼────────┬──────────┐
          ▼          ▼       ▼        ▼
        MySQL     MariaDB PostgreSQL SQLite
          │          │       │        │
          └──────────┴───────┼────────┘
                             ▼
                         Renderer
                             │
                             ▼
                 CompiledDatabaseCommand
```

---

# 298. Relación con DatabasePlatform

```text
DatabasePlatform
=
what the database can do

SqlDialect
=
how that capability is expressed

SqlCompiler
=
how VoltStack structures are transformed

Driver
=
how the command is sent

Connection
=
where the command is sent
```

---

# 299. Relación con Execution Plan

```text
ExecutionPlan
=
what executable work exists

SQL Compiler
=
how database-executed units are represented

Execution Engine
=
how those units are run
```

---

# 300. Relación con Prepared Statements

```text
SQL Compiler
        │
        ▼
CompiledDatabaseCommand
        │
        ▼
Prepared Statement System
        │
        ▼
Driver Prepared Statement
```

---

# 301. Relación con Query Cache

El SQL Compiler no será el Query Cache.

---

# 302. Relación con Compiled Query Cache

Posteriormente:

```text
CompiledDatabaseCommand
        │
        ▼
Compiled Query Cache
```

podrá reutilizar artifacts compilados.

---

# 303. Relación con Telemetry

Compiler puede producir:

```text
compilation duration
node count
SQL length
placeholder count
extension count
cacheability
diagnostic count
```

pero no enviará directamente a un backend concreto.

---

# 304. Relación con Security

```text
Security Model
      │
      ▼
Semantic/Query Policy
      │
      ▼
Explicit Query Structure
      │
      ▼
Compiler
      │
      ▼
SQL preserving policy
```

El Compiler es la última frontera antes de convertir esas políticas a sintaxis SQL.

---

# 305. Relación con Multitenancy

Core Compiler no dependerá de:

```text
VoltStack/Quantum/Multitenancy
```

Las restricciones tenant deberán llegar como:

```text
query structure
metadata
parameters
connection resolution requirements
```

---

# 306. Relación con persistent runtimes

La arquitectura deberá funcionar igualmente en:

```text
PHP-FPM
FrankenPHP
RoadRunner
OpenSwoole
CLI
workers
tests
```

sin mantener estado de compilación entre operaciones.

---

# 307. Modelo completo

```text
                    SemanticQueryArtifact
                              │
                              ▼
                         Optimizer
                              │
                              ▼
                    OptimizedQueryArtifact
                              │
                              ▼
                       Logical Planner
                              │
                              ▼
                     LogicalQueryPlan
                              │
                              ▼
                       Physical Planner
                              │
                              ▼
                     PhysicalQueryPlan
                              │
                              ▼
                      Execution Planner
                              │
                              ▼
                        ExecutionPlan
                              │
                              ▼
                  DatabaseExecutionUnit
                              │
                              ▼
               CompilableDatabaseOperation
                              │
                              ▼
              ┌──────────────────────────┐
              │       SQL Compiler       │
              │                          │
              │ Validation               │
              │ Lowering                 │
              │ Dialect Adaptation       │
              │ Placeholder Allocation   │
              │ Rendering                │
              │ Binding Layout           │
              │ Result Contract          │
              │ Source Map               │
              └─────────────┬────────────┘
                            │
                            ▼
                CompiledDatabaseCommand
                            │
                            ▼
                     Execution Engine
                            │
                            ▼
                        Connection
                            │
                            ▼
                          Driver
                            │
                            ▼
                         Database
```

---

# 308. Fórmula arquitectónica

```text
SQLCompiler
=
CompilationValidation
+
StructuredLowering
+
DialectAdaptation
+
IdentifierCompilation
+
ExpressionCompilation
+
PredicateCompilation
+
StatementCompilation
+
PlaceholderAllocation
+
SQLRendering
+
BindingLayoutCompilation
+
ResultContractCompilation
+
DependencyTracking
+
SourceMapping
```

---

# 309. Fórmula de seguridad

```text
SafeSqlCompilation
=
StructuredSyntax
+
StructuredIdentifiers
+
BoundRuntimeValues
+
ExplicitRawBoundaries
+
CapabilityValidation
+
SecurityPreservation
```

---

# 310. Fórmula de portabilidad

```text
PortableCompilation
=
CommonSemanticModel
+
CapabilityDrivenPlanning
+
DialectSpecificRepresentation
```

No:

```text
PortableCompilation
=
VendorConditionalsEverywhere
```

---

# 311. Fórmula de pureza

```text
CompiledDatabaseCommand
=
Compile(
    ImmutableCompilableOperation,
    ImmutableCompilationContext
)
```

sin:

```text
HiddenIO
+
GlobalState
+
RuntimeBindings
+
ExecutionSideEffects
```

---

# 312. Fórmula de equivalencia

```text
Meaning_TargetDatabase(
    CompiledDatabaseCommand
)
=
Meaning(
    CompilableDatabaseOperation
)
```

---

# 313. Invariante maestro

```text
Compiler may change representation.

Compiler must never change meaning.
```

---

# 314. Block 6

Este documento abre:

```text
Block 6 — SQL Compiler
```

compuesto por:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
67_DATABASE_SQL_COMPILER_PIPELINE.md
68_DATABASE_SQL_GENERATION_SYSTEM.md
69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 315. Distribución de responsabilidades del bloque

## Documento 66

```text
Overall Compiler Architecture
```

## Documento 67

```text
Compilation pipeline
phases
passes
ordering
validation
lowering
rendering lifecycle
```

## Documento 68

```text
SQL text/emission generation
identifiers
expressions
predicates
formatting
source maps
```

## Documentos 69–72

```text
platform-specific compiler architecture
```

para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

## Documento 73

```text
Compiler Extension System
```

## Documento 74

```text
Prepared Statement Compilation
parameter layout
binding compilation
```

## Documento 75

```text
Compiled Query Cache
fingerprints
dependencies
invalidation
persistent reuse
```

---

# 316. Próximo documento

```text
67_DATABASE_SQL_COMPILER_PIPELINE.md
```

deberá definir en detalle:

```text
CompilableDatabaseOperation
        │
        ▼
Pre-Compilation Validation
        │
        ▼
Compiler Resolution
        │
        ▼
Structural SQL Lowering
        │
        ▼
Dialect Adaptation
        │
        ▼
Parameter/Placeholder Planning
        │
        ▼
SQL Emission
        │
        ▼
Binding Layout Compilation
        │
        ▼
Result Contract Compilation
        │
        ▼
Dependency/Fingerprint Construction
        │
        ▼
Post-Compilation Validation
        │
        ▼
CompiledDatabaseCommand
```

incluyendo:

```text
phase ordering
compiler passes
phase contracts
artifacts
validation checkpoints
failure behavior
budgets
determinism
extension participation
diagnostics
persistent-runtime isolation
```

---

# 317. Principio final

> El SQL Compiler de VoltStack será una frontera de representación: recibirá una operación cuya semántica y estrategia ya han sido decididas y producirá una representación SQL segura, determinista y compatible con la plataforma, sin volver a actuar como optimizador, planner o executor.

En forma compacta:

```text
Query Meaning
    │
    │ already resolved
    ▼
Physical Strategy
    │
    │ already selected
    ▼
Execution Topology
    │
    │ already planned
    ▼
SQL Compiler
    │
    │ representation only
    ▼
CompiledDatabaseCommand
    │
    │ no execution yet
    ▼
Execution Engine
```

La regla que debe permanecer durante todo el Block 6 será:

```text
SQL Compiler
=
"How do I represent this operation for this SQL platform?"

not

"What should this query mean?"
not
"How should this query be optimized?"
not
"Which physical strategy should be used?"
not
"Should I execute it now?"
```