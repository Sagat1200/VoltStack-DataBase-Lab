# 23_DATABASE_QUERY_ARCHITECTURE.md

# VoltStack Quantum Database
## Query Engine Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 23 — Query Architecture  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura general del **Query Engine** de `VoltStack/Quantum/Database`.

El Query Engine será responsable de transformar una intención de consulta expresada mediante APIs de alto nivel en una operación ejecutable contra una base de datos concreta.

La arquitectura deberá permitir:

- Query Builder fluido;
- consultas programáticas;
- consultas ORM;
- consultas de Schema;
- consultas internas del framework;
- AST estructurado;
- análisis semántico;
- resolución de símbolos;
- inferencia de tipos;
- optimización;
- planificación;
- compilación SQL;
- prepared statements;
- binding tipado;
- ejecución;
- resultados;
- streaming;
- cache de consultas compiladas;
- extensiones;
- múltiples Dialects;
- múltiples Platforms;
- observabilidad;
- seguridad;
- persistent runtimes.

La arquitectura deberá evitar que:

```text
Query Builder
```

termine convirtiéndose en:

```text
SQL String Builder
```

---

# 2. Regla fundamental

La regla arquitectónica principal será:

```text
Query Builder
    │
    ▼
Query Model
    │
    ▼
AST
    │
    ▼
Semantic Analysis
    │
    ▼
Semantic Graph
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ▼
Compiler
    │
    ▼
CompiledQuery
    │
    ▼
Executor
```

Cada etapa tendrá una responsabilidad diferente.

---

# 3. Regla de separación

Formalmente:

```text
Builder
≠
AST
≠
Semantic Model
≠
Optimizer
≠
Planner
≠
Compiler
≠
Executor
```

---

# 4. Principio de intención

El Query Builder deberá describir:

```text
WHAT
```

la aplicación quiere consultar.

No:

```text
HOW
```

el motor SQL deberá ejecutarlo.

---

# 5. Ejemplo público

VoltStack podrá proporcionar una API similar a:

```php
$users = DB::table('users')
    ->select('id', 'name', 'email')
    ->where('active', true)
    ->orderBy('name')
    ->limit(100)
    ->get();
```

La API será simple.

Internamente:

```text
Builder API
    │
    ▼
Query Model
    │
    ▼
AST
```

No:

```text
Builder
    │
    ▼
"SELECT id, name..."
```

---

# 6. Motivación

Separar intención y SQL permite:

- portabilidad;
- optimización;
- análisis estático;
- validación;
- seguridad;
- query caching;
- observabilidad;
- Dialects;
- Platform capabilities;
- extensiones;
- ORM reutilizando el mismo engine;
- Schema tooling;
- futuras herramientas de análisis;
- query explain;
- query fingerprinting;
- mejores errores.

---

# 7. Arquitectura general

```text
Application
    │
    ▼
Public Query API
    │
    ▼
Query Builder
    │
    ▼
Query Model
    │
    ▼
Query AST
    │
    ▼
Query Normalizer
    │
    ▼
Query Validator
    │
    ▼
Semantic Analyzer
    │
    ▼
Semantic Graph
    │
    ▼
Query Optimizer
    │
    ▼
Query Planner
    │
    ▼
Execution Plan
    │
    ▼
SQL Compiler
    │
    ▼
CompiledQuery
    │
    ▼
Query Executor
    │
    ▼
Connection
    │
    ▼
Driver
    │
    ▼
Database Server
```

---

# 8. Grandes dominios

El Query Engine se dividirá en:

```text
Query API
Query Model
Query AST
Query Expression
Query Parameter
Query Type
Query Metadata
Query Context
Normalization
Validation
Semantic Analysis
Optimization
Planning
Compilation
Execution
Result
```

---

# 9. Mapa documental

Este bloque se desarrolla mediante:

```text
23_DATABASE_QUERY_ARCHITECTURE.md

24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
26_DATABASE_QUERY_AST_NODE_MODEL.md
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
31_DATABASE_QUERY_METADATA_SYSTEM.md
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
```

seguido por Semantic Query Engine:

```text
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

Query Builder:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md
```

Optimizer y Planner:

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

SQL Compiler:

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

Execution:

```text
76_DATABASE_EXECUTION_ENGINE_ARCHITECTURE.md
77_DATABASE_QUERY_EXECUTOR_SYSTEM.md
78_DATABASE_STATEMENT_EXECUTION_SYSTEM.md
79_DATABASE_PREPARED_STATEMENT_SYSTEM.md
80_DATABASE_PARAMETER_BINDING_SYSTEM.md
81_DATABASE_RESULT_SYSTEM.md
82_DATABASE_RESULT_CURSOR_SYSTEM.md
83_DATABASE_STREAMING_RESULT_SYSTEM.md
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
86_DATABASE_EXECUTION_RETRY_SYSTEM.md
```

---

# 10. Query API

Query API será la superficie pública de consultas.

Podrá ser utilizada por:

```text
Application
ORM
Repository
Active Record API
Framework internals
Schema tooling
```

---

# 11. Query API goals

Deberá ser:

- fluida;
- tipada;
- predecible;
- portable;
- IDE-friendly;
- extensible;
- segura por defecto.

---

# 12. Query API no conoce Driver

Incorrecto:

```php
$query->usePdo();
```

Correcto:

```php
$query->connection('analytics');
```

La infraestructura resolverá posteriormente el Driver.

---

# 13. Query Model

`Query Model` representa una consulta de forma estructurada.

Ejemplos:

```text
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
```

---

# 14. Query Model vs Builder

Builder será una API de construcción.

Query Model será representación.

```text
Builder
    │
    ▼
Query Model
```

---

# 15. Query Model vs AST

El Query Model representa la consulta desde una perspectiva semántica de alto nivel.

AST representa formalmente su estructura sintáctico-semántica interna.

Dependiendo de la implementación, ciertas estructuras podrán compartir value objects, pero conceptualmente seguirán siendo responsabilidades distintas.

---

# 16. Query Model example

```text
SelectQuery
├── Source
├── Projection
├── Predicates
├── Joins
├── Grouping
├── Ordering
├── Pagination
└── Metadata
```

---

# 17. Query AST

El AST será una representación estructurada e idealmente inmutable.

Ejemplo:

```text
SelectNode
├── ProjectionNode
│   ├── ColumnNode(id)
│   └── ColumnNode(name)
│
├── FromNode(users)
│
├── WhereNode
│   └── EqualityNode
│       ├── ColumnNode(active)
│       └── ParameterNode(:p1)
│
└── OrderByNode(name)
```

---

# 18. AST no es SQL

AST no almacenará:

```text
"WHERE active = ?"
```

como representación principal.

Almacenará:

```text
EqualityExpression(
    ColumnReference('active'),
    ParameterReference(...)
)
```

---

# 19. AST portable

El AST deberá ser independiente de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
PDO
```

cuando represente operaciones semánticamente portables.

---

# 20. Vendor-specific AST

Cuando una feature sea realmente específica, podrá utilizarse:

```text
Extension AST Node
```

pero sólo mediante extension points controlados.

---

# 21. Query Expressions

Las expresiones serán componentes estructurados.

Ejemplos:

```text
ColumnExpression
LiteralExpression
ParameterExpression
BinaryExpression
UnaryExpression
FunctionExpression
AggregateExpression
CaseExpression
SubqueryExpression
TupleExpression
CastExpression
```

---

# 22. Predicates

Los predicados representarán condiciones booleanas.

Ejemplos:

```text
ComparisonPredicate
AndPredicate
OrPredicate
NotPredicate
InPredicate
BetweenPredicate
ExistsPredicate
NullPredicate
LikePredicate
```

---

# 23. Expression ≠ Predicate

Aunque ciertos motores SQL permitan tratar ambos de forma similar, VoltStack mantendrá la distinción semántica cuando sea útil.

---

# 24. Parameters

Los valores dinámicos no deberán incrustarse en SQL.

Ejemplo:

```php
->where('email', $email)
```

produce conceptualmente:

```text
Parameter
├── id
├── value
├── inferred/declared type
└── metadata
```

---

# 25. Binding separation

Separar:

```text
Parameter Definition
Parameter Value
Parameter Type
Compiled Placeholder
Driver Binding
```

---

# 26. Parameter lifecycle

```text
Application Value
      │
      ▼
Query Parameter
      │
      ▼
Type Resolution
      │
      ▼
Compiled Parameter
      │
      ▼
Database Representation
      │
      ▼
Driver Binding
```

---

# 27. No string interpolation

Incorrecto:

```php
$sql = "WHERE email = '$email'";
```

---

# 28. Correcto

```text
AST Parameter
      │
      ▼
Compiled Placeholder
      │
      ▼
Prepared Statement
```

---

# 29. Query Type System

El Query Engine deberá manejar información de tipos.

Ejemplos:

```text
INTEGER
BIGINT
DECIMAL
BOOLEAN
STRING
BINARY
DATE
TIME
DATETIME
JSON
UUID
ENUM
CUSTOM
```

---

# 30. Query Type vs Database Type

No necesariamente serán equivalentes.

```text
Application/Query Type
       │
       ▼
Database Type Mapping
       │
       ▼
Platform Representation
```

---

# 31. Type inference

Semantic Engine podrá inferir tipos utilizando:

```text
Schema metadata
Parameter declarations
Expression rules
Function signatures
Column metadata
Platform metadata
```

---

# 32. Query Metadata

Metadata podrá contener información no estructural.

Ejemplos:

```text
query origin
source location
debug label
ORM origin
repository origin
operation category
cache hints
telemetry hints
```

---

# 33. Metadata no cambia semántica

Metadata puramente diagnóstica no deberá modificar el resultado lógico de una consulta.

---

# 34. Semantic metadata

Cuando metadata sí afecte comportamiento deberá estar explícitamente clasificada.

Ejemplo:

```text
consistency requirement
connection intent
locking requirement
```

---

# 35. Query Context

El Query Context representará información contextual necesaria durante procesamiento.

Ejemplos:

```text
Platform
Capabilities
Schema metadata
Type registry
Compilation mode
Connection intent
Execution scope
```

---

# 36. Query Context ≠ DatabaseContext

`DatabaseContext` es execution-scope infrastructure.

`QueryContext` pertenece a una operación concreta.

---

# 37. Query Context lifetime

Normalmente:

```text
OPERATION
```

---

# 38. No current Query global

Prohibido:

```php
QueryContext::current();
```

como estado global mutable.

---

# 39. Query Normalization

Antes del análisis semántico, ciertas representaciones podrán normalizarse.

Ejemplo:

```text
where('active', true)
```

podrá normalizarse a:

```text
ComparisonPredicate(
    ColumnReference(active),
    EQUAL,
    Parameter(true)
)
```

---

# 40. Normalization goal

Reducir múltiples formas equivalentes a una representación canónica.

---

# 41. Normalization benefits

Permite:

- simplificar semantic analysis;
- mejorar cache hit rate;
- facilitar optimización;
- mejorar fingerprinting;
- reducir combinaciones internas.

---

# 42. Normalization must preserve semantics

Regla:

```text
normalize(Q)
≈
Q
```

en significado observable.

---

# 43. Query Validation

Validation comprobará estructura y reglas antes de etapas posteriores.

Ejemplos:

```text
SELECT without source when not supported
invalid limit
invalid alias
duplicate projection alias
invalid join structure
invalid expression arity
```

---

# 44. Structural validation

Puede ejecutarse sin conexión.

---

# 45. Semantic validation

Puede requerir:

```text
schema
types
platform capabilities
```

y pertenecer al Semantic Engine.

---

# 46. Validation layers

```text
Builder Validation
      │
      ▼
Structural Validation
      │
      ▼
Semantic Validation
      │
      ▼
Capability Validation
      │
      ▼
Execution Validation
```

---

# 47. No validation duplication

Cada regla deberá tener un owner claro.

---

# 48. Semantic Analysis

El Semantic Engine transforma estructura en significado resuelto.

---

# 49. Example

Input:

```text
ColumnReference("name")
```

Después de resolución:

```text
ResolvedColumn
├── source: users
├── column: name
├── type: string
├── nullable: false
└── symbol identity
```

---

# 50. Semantic pipeline

```text
AST
 │
 ▼
Symbol Resolution
 │
 ▼
Schema Resolution
 │
 ▼
Type Inference
 │
 ▼
Relation Resolution
 │
 ▼
Constraint Analysis
 │
 ▼
Capability Validation
 │
 ▼
Semantic Graph
```

---

# 51. Semantic Graph

El resultado podrá representarse como:

```text
SemanticQueryGraph
```

---

# 52. Semantic Graph purpose

Permitirá que Optimizer y Planner trabajen sobre significado, no únicamente sobre sintaxis.

---

# 53. Example graph

```text
Query
 │
 ├── Source(users)
 │      ├── Column(id)
 │      ├── Column(name)
 │      └── Column(active)
 │
 ├── Predicate
 │      └── active = parameter
 │
 └── Projection
        ├── id
        └── name
```

---

# 54. Schema-aware resolution

Si existe metadata:

```text
users.id     BIGINT
users.name   VARCHAR
users.active BOOLEAN
```

Semantic Engine podrá validar:

```text
unknown columns
ambiguous columns
invalid comparisons
invalid joins
```

antes de enviar SQL.

---

# 55. Schema metadata optionality

No todas las consultas requerirán metadata completa.

El engine deberá permitir distintos niveles:

```text
NONE
PARTIAL
KNOWN
STRICT
```

o equivalente.

---

# 56. Raw SQL exception

Raw SQL podrá saltarse:

```text
Query Model
AST
Semantic Engine
Optimizer
Planner
Compiler
```

pero no necesariamente:

```text
parameter binding
connection management
transaction context
telemetry
timeouts
result handling
security policies
```

---

# 57. Raw SQL pipeline

```text
RawSqlQuery
    │
    ▼
Raw SQL Validation
    │
    ▼
Prepared Statement Compilation
    │
    ▼
Executor
    │
    ▼
Connection
```

---

# 58. Raw SQL is escape hatch

No será el camino arquitectónico principal.

---

# 59. Query Optimizer

VoltStack tendrá un optimizer propio.

Sin embargo:

> El optimizer de VoltStack no intentará sustituir al optimizer interno de MySQL, PostgreSQL, MariaDB o SQLite.

---

# 60. Framework-level optimization

El optimizer podrá realizar:

```text
AST normalization
constant simplification
predicate simplification
duplicate predicate elimination
redundant projection elimination
framework-generated join cleanup
query deduplication
ORM-generated query improvements
portable rewrites
```

---

# 61. Database-level optimization

El motor seguirá decidiendo:

```text
index selection
join algorithms
buffer strategies
parallel query
physical disk access
server execution plan
```

---

# 62. Optimizer input

Preferentemente:

```text
Semantic Graph
```

o una representación lógica derivada.

---

# 63. Optimizer output

```text
Optimized Semantic Query
```

o:

```text
Optimized Logical Representation
```

según la fase.

---

# 64. Optimization rules

Cada transformación deberá declarar:

```text
preconditions
semantic guarantees
capability requirements
cost assumptions
```

---

# 65. Deterministic optimization

Dado el mismo:

```text
Query
Capabilities
Schema Metadata
Optimizer Configuration
```

el resultado deberá ser determinista.

---

# 66. Query Planner

Planner determinará cómo convertir la consulta semántica en operaciones ejecutables por VoltStack.

---

# 67. Planner stages

```text
Semantic Query
      │
      ▼
Logical Plan
      │
      ▼
Optimized Logical Plan
      │
      ▼
Physical Plan
      │
      ▼
Execution Plan
```

---

# 68. Logical Plan

Describe operaciones lógicas.

Ejemplo:

```text
Scan(users)
   │
   ▼
Filter(active = :p1)
   │
   ▼
Project(id, name)
   │
   ▼
Sort(name)
   │
   ▼
Limit(100)
```

---

# 69. Physical Plan

Describe decisiones relevantes para la ejecución del framework.

No intenta modelar el plan físico interno del servidor.

---

# 70. Physical decisions

Ejemplos:

```text
single statement
multiple statements
native feature
emulated feature
transaction requirement
result mode
streaming requirement
connection requirement
```

---

# 71. Execution Plan

Representará las operaciones que el Execution Engine debe realizar.

---

# 72. Simple Execution Plan

```text
ExecutionPlan
└── StatementOperation
    └── CompilableQuery
```

---

# 73. Compound Execution Plan

Algunas features podrán requerir:

```text
ExecutionPlan
├── BeginTransaction
├── Statement A
├── Statement B
└── Commit
```

si la emulación es segura y explícitamente soportada.

---

# 74. Atomicity

El Planner deberá conocer requisitos de atomicidad cuando una operación se expanda.

---

# 75. No unsafe emulation

Si una feature no puede emularse preservando semántica:

```text
CapabilityNotSupportedException
```

será preferible.

---

# 76. SQL Compiler

Compiler transforma un plan compilable en SQL específico del Dialect/Platform.

---

# 77. Compiler input

Podrá recibir:

```text
Physical Query Plan
Compilation Context
Dialect
Platform Capabilities
```

---

# 78. Compiler output

```text
CompiledQuery
```

---

# 79. CompiledQuery

Conceptualmente:

```php
final readonly class CompiledQuery
{
    public function __construct(
        public string $sql,
        public CompiledBindingCollection $bindings,
        public QueryExecutionMetadata $metadata,
    ) {}
}
```

---

# 80. CompiledQuery immutable

Una consulta compilada deberá ser idealmente inmutable.

---

# 81. Compiler purity

El Compiler no deberá:

```text
open connections
execute statements
start transactions
hydrate entities
access current user
resolve current tenant globally
```

---

# 82. Compiler determinism

Idealmente:

```text
compile(
    QueryPlan,
    Dialect,
    Capabilities,
    Configuration
)
```

producirá siempre el mismo resultado.

---

# 83. Dialect integration

Dialect define:

```text
identifier quoting
placeholder syntax
operator syntax
function syntax
limit/offset syntax
CTE syntax
returning syntax
vendor grammar details
```

---

# 84. Platform integration

Platform define:

```text
feature semantics
capability availability
server behavior
version restrictions
transaction semantics
```

---

# 85. Compiler specialization

Podrán existir:

```text
MysqlSqlCompiler
MariadbSqlCompiler
PostgresqlSqlCompiler
SqliteSqlCompiler
```

compartiendo componentes comunes donde sea legítimo.

---

# 86. No giant compiler switch

Incorrecto:

```php
switch ($database) {
    case 'mysql':
    case 'postgres':
    case 'sqlite':
}
```

---

# 87. Correct compiler resolution

```text
Platform/Dialect
      │
      ▼
CompilerRegistry
      │
      ▼
SQL Compiler
```

---

# 88. Prepared Statement Compilation

La compilación deberá producir:

```text
SQL
+
Placeholder Mapping
+
Binding Metadata
```

---

# 89. Example

Input:

```text
active = Parameter(true)
```

PostgreSQL podría producir:

```text
active = $1
```

mientras otra implementación podría producir:

```text
active = ?
```

---

# 90. Placeholder syntax belongs to compilation

No al Query Builder.

---

# 91. Binding identity

El engine deberá mantener una relación determinista entre:

```text
ParameterId
→
CompiledPlaceholder
→
Binding
```

---

# 92. Repeated parameters

El sistema deberá definir explícitamente cómo manejar:

```text
x = :value OR y = :value
```

dependiendo de las restricciones del Driver/Dialect.

---

# 93. Compiled Query Cache

Consultas estructuralmente equivalentes podrán compartir compilación.

---

# 94. Query fingerprint

Podrá derivarse de:

```text
normalized query shape
semantic information
platform
dialect
capability fingerprint
compiler version
extension graph
```

---

# 95. Values excluded

Normalmente valores runtime como:

```text
email = "john@example.com"
```

no formarán parte del structural fingerprint.

---

# 96. Shape example

Estas consultas:

```text
email = "a@example.com"
email = "b@example.com"
```

podrán compartir:

```text
QueryShapeFingerprint
```

---

# 97. Cache safety

Nunca reutilizar una compilación si cambian elementos semánticamente relevantes.

---

# 98. Cache invalidation

Podrán afectar:

```text
compiler version
dialect version
platform capability fingerprint
extension graph
query structure
schema-sensitive semantic metadata
```

---

# 99. Executor

Executor consume:

```text
CompiledQuery
```

o un:

```text
ExecutionPlan
```

y coordina su ejecución.

---

# 100. Executor responsibilities

Incluyen:

```text
connection resolution
lease acquisition
statement preparation
parameter binding
statement execution
timeout handling
cancellation
result adaptation
failure normalization
telemetry
resource cleanup
```

---

# 101. Executor does not compile

Incorrecto:

```text
Executor
→ inspect AST
→ build SQL
```

---

# 102. Correcto

```text
Compiler
→ CompiledQuery
→ Executor
```

---

# 103. Executor does not hydrate entities

Hydration pertenece a otra capa.

---

# 104. Result pipeline

```text
Native Result
     │
     ▼
Database Result
     │
     ▼
Hydration Layer
     │
     ├── Scalar
     ├── Array
     ├── DTO
     └── Entity
```

---

# 105. Query Engine result

Query Engine podrá devolver:

```text
Result
ResultCursor
StreamingResult
AffectedRows
GeneratedValues
```

según operación.

---

# 106. ORM integration

ORM utilizará el mismo Query Engine.

```text
ORM
 │
 ▼
Persistence Planner / Entity Query
 │
 ▼
Query Model
 │
 ▼
Query Engine
```

---

# 107. ORM no genera SQL

Regla:

```text
ORM
≠
SQL Compiler
```

---

# 108. Active Record integration

```text
Active Record API
       │
       ▼
ORM
       │
       ▼
Query Engine
```

No existirá un segundo Query Engine para Models.

---

# 109. Repository integration

```text
Repository
    │
    ▼
Entity Query API
    │
    ▼
Query Model
    │
    ▼
Query Engine
```

---

# 110. Schema integration

Schema tendrá su propio:

```text
Schema Model
Schema AST
Schema Planner
Schema Compiler
```

No deberá utilizar indiscriminadamente Query AST para DDL.

---

# 111. Shared infrastructure

Schema y Query podrán compartir:

```text
types
identifiers
platform
dialect primitives
connection
execution
results
errors
telemetry
```

sin fusionar sus modelos.

---

# 112. Transaction integration

Una consulta podrá declarar requisitos como:

```text
transaction required
read-only
locking
isolation requirement
connection affinity
```

---

# 113. Transaction ownership

Query Engine no almacenará globalmente la transacción actual.

---

# 114. Transaction Context

Executor consultará el:

```text
TransactionContext
```

scoped apropiado.

---

# 115. Connection intent

Una consulta podrá generar:

```text
READ
WRITE
ADMIN
MIGRATION
```

u otro `ConnectionIntent`.

---

# 116. Intent resolution

```text
Query Semantics
      │
      ▼
Connection Requirement
      │
      ▼
Topology / Connection Resolution
```

---

# 117. Read/write classification

No deberá basarse únicamente en analizar el SQL compilado.

Idealmente se conoce antes desde el Query Model/Plan.

---

# 118. Query classification

Ejemplos:

```text
SELECT → READ
INSERT → WRITE
UPDATE → WRITE
DELETE → WRITE
```

con excepciones semánticas manejadas explícitamente.

---

# 119. Locking SELECT

Ejemplo:

```text
SELECT ... FOR UPDATE
```

podrá requerir:

```text
WRITE/PRIMARY connection
+
active transaction
```

aunque sintácticamente sea SELECT.

---

# 120. Capability validation

Features deberán validarse mediante capabilities.

Ejemplo:

```text
Query requests RETURNING
       │
       ▼
Capability Requirement
       │
       ▼
Platform Capability System
       │
       ├── Native
       ├── Emulated
       └── Unsupported
```

---

# 121. No vendor checks

Incorrecto:

```php
if ($driver === 'pgsql') {
}
```

en Query Builder.

---

# 122. Correcto

```php
if ($capabilities->supports($requirement)) {
}
```

en la capa apropiada.

---

# 123. Capability validation stage

Dependiendo de la feature podrá ocurrir en:

```text
Semantic Analysis
Planner
Compiler
Execution
```

pero deberá tener owner definido.

---

# 124. Early validation

Preferir detectar incompatibilidades lo antes posible cuando exista suficiente información.

---

# 125. Deferred validation

Si la capability depende de runtime server discovery, la validación podrá diferirse.

---

# 126. Query portability

Una consulta portable deberá poder ejecutarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin cambios en Application code cuando la semántica requerida exista.

---

# 127. Portability does not mean lowest common denominator

VoltStack podrá exponer features avanzadas.

---

# 128. Capability-based advanced features

Ejemplo:

```php
$query->returning('id');
```

podrá funcionar cuando:

```text
capability(query.returning.insert)
```

esté disponible.

---

# 129. Unsupported feature

Debe fallar explícitamente.

No degradar silenciosamente semántica.

---

# 130. Emulated feature

Sólo podrá utilizarse si:

```text
semantic equivalence
atomicity
concurrency
transaction behavior
```

son aceptables.

---

# 131. Query immutability

Las representaciones internas deberían favorecer inmutabilidad.

---

# 132. Builder mutability

El Builder público podrá ofrecer una API aparentemente mutable:

```php
$query->where(...)->orderBy(...);
```

pero internamente podrá usar:

```text
immutable query state
copy-on-write
persistent structures
```

según rendimiento.

---

# 133. Builder cloning

Deberá ser posible construir:

```php
$base = DB::table('users')->where('active', true);

$admins = $base->where('role', 'admin');
$clients = $base->where('role', 'client');
```

sin contaminación accidental.

---

# 134. Builder ownership

La semántica exacta de cloning/mutation se definirá en:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
```

---

# 135. AST immutability

AST deberá ser preferentemente:

```text
immutable
```

para facilitar:

```text
cache
concurrency
rewrites
debugging
fingerprinting
```

---

# 136. AST transformation

Optimizer no deberá modificar nodos compartidos in-place.

---

# 137. Correct transformation

```text
AST A
 │
 ▼
Rewrite
 │
 ▼
AST B
```

---

# 138. Structural sharing

Podrá utilizarse cuando sea beneficioso.

---

# 139. Query identity

Distinguir:

```text
QueryInstanceId
QueryShapeFingerprint
CompiledQueryFingerprint
ExecutionId
```

---

# 140. QueryInstanceId

Identifica una construcción concreta.

---

# 141. QueryShapeFingerprint

Identifica forma estructural.

---

# 142. CompiledQueryFingerprint

Identifica una compilación específica.

---

# 143. ExecutionId

Identifica una ejecución concreta.

---

# 144. Why separate identities

Una Query Shape puede:

```text
compile once
execute thousands of times
```

con valores distintos.

---

# 145. Query provenance

Podrá rastrearse:

```text
Application
→ Repository
→ ORM
→ Query
→ Compilation
→ Execution
```

sin acoplar el engine a esas capas.

---

# 146. Query origin

Ejemplos:

```text
APPLICATION
ORM
SCHEMA
MIGRATION
FRAMEWORK
INTERNAL
```

---

# 147. Query labels

Podrán utilizarse labels de baja cardinalidad para diagnóstico.

---

# 148. No user data in labels

No incluir:

```text
email
token
user id
tenant secret
```

como telemetry label indiscriminadamente.

---

# 149. Query telemetry

Integración opcional con `Quantum/Telemetry`.

Eventos conceptuales:

```text
QueryBuildCompleted
QuerySemanticAnalysisCompleted
QueryOptimizationCompleted
QueryCompilationCompleted
QueryExecutionStarted
QueryExecutionCompleted
QueryExecutionFailed
```

---

# 150. Telemetry overhead

Las etapas de alto detalle podrán habilitarse sólo en:

```text
development
profiling
sampling
```

---

# 151. Query events vs domain events

Query telemetry no deberá convertirse en Domain Event Bus.

---

# 152. Optional telemetry

Query Engine no dependerá obligatoriamente de:

```text
Quantum/Telemetry
```

---

# 153. Telemetry port

Podrá utilizar:

```text
QueryTelemetrySinkInterface
```

---

# 154. Null telemetry

Sin Telemetry:

```text
NullQueryTelemetrySink
```

---

# 155. Query security

El diseño deberá prevenir:

```text
SQL injection
identifier injection
unsafe raw fragments
binding confusion
secret leakage
```

---

# 156. Values vs identifiers

Valores deberán utilizar parameters.

Identifiers deberán validarse/representarse mediante:

```text
Identifier
QualifiedIdentifier
TableIdentifier
ColumnIdentifier
AliasIdentifier
```

---

# 157. Identifier safety

No tratar:

```php
$table = $_GET['table'];
DB::table($table);
```

como automáticamente seguro.

---

# 158. Dynamic identifiers

Deberán validarse contra:

```text
allowed identifier grammar
application allowlist
schema metadata
```

según contexto.

---

# 159. Identifier quoting

Pertenece al Dialect/Compiler.

---

# 160. RawExpression

Será un escape hatch explícito.

---

# 161. Raw expression policy

No mezclar:

```text
safe structured expression
```

con:

```text
trusted raw SQL
```

sin marcar la diferencia.

---

# 162. Trusted raw SQL

Una API podría requerir:

```php
RawExpression::trusted('...')
```

o mecanismo equivalente para hacer visible el riesgo.

---

# 163. Raw bindings

Incluso Raw SQL deberá soportar parámetros seguros.

---

# 164. SQL comments

Comentarios generados para tracing deberán ser sanitizados y capability/policy-aware.

---

# 165. Query validation errors

Deberán ser descriptivos.

Ejemplo:

```text
Unknown column "email_address" in query source "users".

Did you mean "email"?
```

cuando exista metadata suficiente.

---

# 166. Semantic errors

Ejemplo:

```text
Column "id" is ambiguous.

Candidates:
- users.id
- orders.id
```

---

# 167. Capability errors

Ejemplo:

```text
RETURNING is not available for this operation on the resolved platform.
```

---

# 168. Compilation errors

Ejemplo:

```text
The resolved dialect cannot compile the requested window frame.
```

---

# 169. Execution errors

Pertenecerán al Execution Error System.

---

# 170. Error hierarchy

Conceptualmente:

```text
DatabaseQueryException
├── QueryConstructionException
├── QueryValidationException
├── QuerySemanticException
├── QueryOptimizationException
├── QueryPlanningException
├── QueryCompilationException
└── QueryExecutionException
```

---

# 171. Error context

Podrá contener:

```text
query fingerprint
query stage
safe query summary
source metadata
capability requirement
```

sin exponer bindings sensibles.

---

# 172. Query explain

VoltStack podrá proporcionar:

```text
$query->explain()
```

pero deberán distinguirse dos conceptos.

---

# 173. Framework explain

Muestra:

```text
Query Model
AST
Semantic Graph
Optimization
Logical Plan
Physical Plan
Compiled SQL
Connection Requirement
```

---

# 174. Database explain

Ejecuta:

```text
EXPLAIN
```

o equivalente en el servidor.

---

# 175. Framework explain ≠ Database EXPLAIN

Ambos podrán complementarse.

---

# 176. No execution for framework explain

Idealmente:

```text
framework explain
```

podrá realizarse offline cuando toda la información necesaria esté disponible.

---

# 177. Database explain requires connection

Naturalmente puede requerir servidor.

---

# 178. Explain safety

`EXPLAIN ANALYZE` o equivalentes pueden ejecutar la consulta.

Deberán ser APIs explícitas diferentes.

---

# 179. Query debug representation

El engine podrá generar una representación:

```text
safe debug query
```

---

# 180. Debug SQL ≠ executable SQL

Un string mostrado con valores interpolados para debugging no deberá utilizarse para ejecución.

---

# 181. Sensitive binding masking

Ejemplo:

```text
password = [REDACTED]
token = [REDACTED]
```

---

# 182. Query compiler cache

Podrá vivir en application/worker scope si sólo almacena:

```text
immutable compiled artifacts
```

---

# 183. No request state in compiled cache

No incluir:

```text
tenant object
current connection
current transaction
request object
user object
```

---

# 184. Tenant-aware query processing

Multitenancy será integración opcional.

---

# 185. Tenant context

Si una consulta requiere aislamiento tenant:

```text
Tenant Query Policy
       │
       ▼
Query Transformation / Connection Resolution
```

según modelo de tenancy.

---

# 186. No core tenant dependency

Query Engine deberá funcionar sin instalar Multitenancy.

---

# 187. Row-level tenant filters

Cuando se utilicen, deberán integrarse mediante extension/policy points explícitos.

---

# 188. No invisible unsafe filters

Los filtros automáticos deberán ser:

```text
deterministic
inspectable
testable
disableable only through privileged API
```

---

# 189. Schema-per-tenant

Podrá afectar:

```text
connection/session context
schema resolution
identifier resolution
```

pero deberá limpiarse al terminar scope.

---

# 190. Database-per-tenant

Principalmente afectará:

```text
Connection Resolution
```

no Query AST semántico.

---

# 191. Sharding

Una consulta podrá generar:

```text
ShardRequirement
```

o routing metadata.

---

# 192. Query Engine does not implement shard topology

Topology System será responsable de resolver el target.

---

# 193. Query fan-out

Consultas distribuidas multi-shard serán una capacidad avanzada separada.

---

# 194. No implicit fan-out

Nunca ejecutar automáticamente:

```text
query against every shard
```

sin semántica explícita.

---

# 195. Read/write topology

Query Plan podrá declarar:

```text
ConnectionIntent
ConsistencyRequirement
AffinityRequirement
```

---

# 196. Replica safety

Una consulta marcada:

```text
READ
```

no significa automáticamente que una replica sea válida.

---

# 197. Consistency requirement

Podrá exigir:

```text
PRIMARY
REPLICA_ALLOWED
READ_YOUR_WRITES
STRONG
EVENTUAL
```

o modelo equivalente.

---

# 198. Sticky connection

Topology layer podrá aplicar sticky-primary después de writes.

Query Engine sólo expresa requerimientos.

---

# 199. Transaction affinity

Transaction Context prevalecerá sobre routing normal.

---

# 200. Query timeout

Una consulta podrá declarar:

```text
timeout
deadline
```

como metadata de ejecución.

---

# 201. Timeout ownership

Executor aplicará el timeout.

Compiler podrá generar syntax específica sólo cuando corresponda.

---

# 202. Cancellation

Cancellation será coordinada por Execution Engine y Driver capabilities.

---

# 203. Query cancellation state

Una cancelación puede afectar:

```text
statement
transaction
connection health
connection reuse
```

y deberá integrarse con Lifecycle/Reset.

---

# 204. Retry

Query Engine deberá proporcionar suficiente metadata para decidir si una operación puede reintentarse.

---

# 205. Retry is not automatic by query type alone

No asumir:

```text
SELECT = always safe to retry
```

sin contexto.

---

# 206. Retry metadata

Podrá incluir:

```text
idempotency
transaction state
side-effect classification
execution progress
failure category
```

---

# 207. Retry ownership

La política final pertenece a:

```text
Execution Resilience
```

no al Query Builder.

---

# 208. Query idempotency

Podrá clasificarse como:

```text
KNOWN_IDEMPOTENT
KNOWN_NON_IDEMPOTENT
CONTEXT_DEPENDENT
UNKNOWN
```

---

# 209. Conservative default

```text
UNKNOWN
```

no deberá convertirse automáticamente en retryable.

---

# 210. Batch queries

Query architecture deberá permitir:

```text
batch insert
batch update
batch delete
```

sin necesariamente convertirlos en múltiples queries desde Builder.

---

# 211. Planner responsibility

Planner decidirá si una operación batch puede usar:

```text
single statement
multiple statements
bulk protocol
```

según capabilities.

---

# 212. Compound execution

Un Query Plan podrá producir múltiples statements.

---

# 213. Compound query atomicity

Si múltiples statements representan una sola operación lógica, deberá declararse:

```text
transaction requirement
failure semantics
partial success semantics
```

---

# 214. No hidden partial success

Nunca ocultar que una operación compuesta puede completarse parcialmente.

---

# 215. Query result cardinality

Semantic/Plan metadata podrá representar:

```text
ZERO_OR_ONE
EXACTLY_ONE
MANY
AFFECTED_ROWS
NO_RESULT
```

cuando sea conocido.

---

# 216. ORM benefit

Esto puede ayudar a:

```text
first()
sole()
exists()
count()
```

y hydration planning.

---

# 217. Projection model

Projection deberá ser estructurada.

Ejemplo:

```text
Projection
├── Column(users.id)
├── Column(users.name)
└── Expression(count(orders.id)) AS order_count
```

---

# 218. Projection alias

Alias será un Identifier estructurado.

---

# 219. Source model

Fuentes podrán incluir:

```text
TableSource
SubquerySource
CteSource
DerivedTableSource
FunctionSource
ExtensionSource
```

---

# 220. Join model

Join deberá representar:

```text
type
left source
right source
condition
metadata
```

---

# 221. Join types

Core podrá contemplar:

```text
INNER
LEFT
RIGHT
FULL
CROSS
```

sujeto a capabilities.

---

# 222. Vendor support

No todos los motores soportarán todas las variantes.

Planner/Compiler validará capabilities.

---

# 223. Grouping

Representar:

```text
GROUP BY
HAVING
aggregates
```

estructuralmente.

---

# 224. Window functions

Serán expresiones/constructos estructurados.

---

# 225. CTE

Common Table Expressions formarán parte del Query Model.

---

# 226. Recursive CTE

Será capability-driven.

---

# 227. Set operations

Representar:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

según capabilities.

---

# 228. Subqueries

Subqueries serán Query Model/AST anidados, no strings.

---

# 229. Correlated subqueries

Semantic Engine resolverá scopes y referencias externas.

---

# 230. Symbol scopes

El Query Engine deberá modelar:

```text
query scope
subquery scope
CTE scope
alias scope
outer reference scope
```

---

# 231. Scope resolution

Se desarrollará en:

```text
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
```

---

# 232. Identifier resolution

Ejemplo:

```text
u.name
```

deberá resolverse a:

```text
source alias u
column name
```

---

# 233. Ambiguity detection

```text
SELECT id
FROM users
JOIN orders ...
```

podrá fallar si `id` es ambiguo.

---

# 234. Query constraints

Semantic analysis podrá detectar:

```text
invalid aggregate use
invalid grouping
invalid window context
invalid update target
invalid insert shape
```

---

# 235. Query mutation operations

INSERT/UPDATE/DELETE deberán utilizar el mismo pipeline conceptual.

---

# 236. Insert pipeline

```text
Insert Builder
      │
      ▼
Insert Query Model
      │
      ▼
Insert AST
      │
      ▼
Semantic Analysis
      │
      ▼
Planner
      │
      ▼
Compiler
      │
      ▼
Executor
```

---

# 237. Update pipeline

Mismo principio.

---

# 238. Delete pipeline

Mismo principio.

---

# 239. Upsert

Upsert deberá modelarse semánticamente.

---

# 240. No vendor syntax at API

Preferir:

```php
$query->upsert(...)
```

sobre:

```php
$query->onDuplicateKeyUpdate(...)
```

como API portable principal.

---

# 241. Vendor escape hatch

APIs vendor-specific podrán existir mediante extensiones.

---

# 242. Semantic operation

Conceptualmente:

```text
UpsertOperation
```

---

# 243. Planner resolution

```text
UpsertOperation
      │
      ▼
Capability Resolution
      │
      ├── Native strategy
      ├── Safe emulation
      └── Unsupported
```

---

# 244. Query transformations

El pipeline podrá aplicar transformaciones controladas.

Ejemplos:

```text
global scopes
soft delete filters
tenant filters
security filters
ORM relation constraints
```

---

# 245. Transformation stages

No todas las transformaciones deberán ocurrir en Builder.

---

# 246. Transformation pipeline

Podrá existir:

```text
Query Model
    │
    ▼
Policy Transformations
    │
    ▼
Normalization
    │
    ▼
Semantic Analysis
```

---

# 247. Transformation transparency

Cada transformación deberá ser:

```text
identifiable
ordered
inspectable
deterministic
```

---

# 248. Query transformation provenance

El debug system podrá mostrar:

```text
Original Query
      │
      ├── SoftDeleteFilter
      ├── TenantFilter
      └── ORM Scope
      ▼
Effective Query
```

---

# 249. No hidden mutable query

Transformers deberán devolver nuevas representaciones o cambios controlados.

---

# 250. Transformation ordering

Deberá ser explícito.

---

# 251. Extension transformations

Extensions podrán registrar transformers sólo en extension points definidos.

---

# 252. Security transformations

Filtros de seguridad no deberán ser removibles accidentalmente por un plugin de baja prioridad.

---

# 253. Query policy

Podrá existir un:

```text
QueryPolicyPipeline
```

separado de optimizer.

---

# 254. Policy ≠ Optimization

Importante:

```text
Tenant Filter
≠
Optimization Rule
```

---

# 255. Why

Optimizer debe preservar semántica.

Policy puede definir semántica efectiva.

---

# 256. Query canonicalization

Después de transformaciones podrá realizarse canonicalization para obtener una forma estable.

---

# 257. Canonical query

Útil para:

```text
fingerprinting
cache
deduplication
testing
debugging
```

---

# 258. Deterministic parameter ordering

Canonicalization deberá producir orden de parámetros determinista.

---

# 259. Query equality

Podrán existir varias nociones:

```text
object identity
structural equality
semantic equivalence
compiled equivalence
```

---

# 260. Structural equality

Dos AST pueden ser estructuralmente iguales.

---

# 261. Semantic equivalence

Dos AST diferentes pueden significar lo mismo.

---

# 262. Compiled equivalence

Dos consultas pueden compilar al mismo SQL para una Platform específica.

---

# 263. Cache keys

Cada cache deberá utilizar la equivalencia apropiada.

---

# 264. Query deduplication

Una futura capa podrá detectar consultas repetidas dentro de una operación.

---

# 265. Deduplication safety

No deduplicar:

```text
writes
volatile queries
locking queries
queries with side effects
```

sin prueba semántica.

---

# 266. Function model

Funciones SQL deberán representarse semánticamente.

---

# 267. Portable functions

Ejemplos:

```text
COUNT
SUM
AVG
MIN
MAX
LOWER
UPPER
COALESCE
```

---

# 268. Function registry

Podrá existir:

```text
QueryFunctionRegistry
```

---

# 269. Function signature

Podrá describir:

```text
name
argument types
return type
volatility
capability requirement
compiler strategy
```

---

# 270. Function volatility

Conceptualmente:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

o modelo neutral equivalente.

---

# 271. Optimization impact

No realizar constant folding de una función volátil.

---

# 272. Database function mapping

Una función semántica podrá compilar de forma distinta según Dialect.

---

# 273. Example

```text
StringLength(expression)
```

podría mapearse a syntax diferente según plataforma.

---

# 274. Function extension

Custom functions requerirán:

```text
semantic registration
type rules
compiler registration
capability declaration
```

según necesidad.

---

# 275. Operator model

Operadores también deberán ser estructurados.

---

# 276. Portable operators

Ejemplos:

```text
=
<>
<
<=
>
>=
+
-
*
/
AND
OR
NOT
```

---

# 277. Extended operators

JSON, regex, arrays u operadores vendor-specific utilizarán capabilities/extensions.

---

# 278. Null semantics

El Query Engine deberá modelar correctamente SQL NULL.

---

# 279. Incorrect rewrite

No transformar indiscriminadamente:

```text
column = NULL
```

a una semántica incorrecta.

---

# 280. Builder ergonomics

La API podrá convertir:

```php
->whereNull('deleted_at')
```

a:

```text
IsNullPredicate
```

---

# 281. Boolean semantics

No asumir que todos los motores representan boolean igual físicamente.

---

# 282. Query type abstraction

La semántica será:

```text
BOOLEAN
```

y Platform/Type System resolverá representación.

---

# 283. Pagination

Query Model representará:

```text
limit
offset
cursor pagination requirements
```

sin asumir syntax concreta.

---

# 284. Cursor pagination

La lógica de aplicación de cursor pagination podrá producir predicates estructurados.

---

# 285. Ordering requirement

Cursor pagination deberá validar ordering determinista.

---

# 286. Locking

Locking deberá representarse como intención semántica.

Ejemplos:

```text
FOR_UPDATE
FOR_SHARE
NOWAIT
SKIP_LOCKED
```

sujeto a capabilities.

---

# 287. Locking influences connection intent

Una locking query normalmente requerirá primary/transaction semantics.

---

# 288. Query hints

Hints serán metadata explícita.

---

# 289. Portable hints

Podrán incluir:

```text
timeout
result mode
consistency
cache policy
```

---

# 290. Vendor hints

Ejemplos de optimizer hints del servidor deberán ser extensions/raw trusted constructs.

---

# 291. Framework cost hints

El Planner podrá aceptar hints internos diferentes de SQL vendor hints.

---

# 292. Hints are not guarantees

A menos que el contrato específico indique lo contrario.

---

# 293. Query lifecycle

Una consulta podrá atravesar estados conceptuales:

```text
BUILDING
    │
    ▼
MODELED
    │
    ▼
NORMALIZED
    │
    ▼
VALIDATED
    │
    ▼
RESOLVED
    │
    ▼
OPTIMIZED
    │
    ▼
PLANNED
    │
    ▼
COMPILED
    │
    ▼
EXECUTING
    │
    ▼
COMPLETED
```

---

# 294. Failure states

Cualquier etapa podrá terminar en:

```text
FAILED
```

con stage metadata.

---

# 295. Query objects need not mutate state

Los estados anteriores representan lifecycle conceptual.

No requieren que un mismo objeto mutable cambie de estado.

---

# 296. Preferred architecture

```text
QueryModel
    │
    ▼
NormalizedQuery
    │
    ▼
SemanticQuery
    │
    ▼
OptimizedQuery
    │
    ▼
QueryPlan
    │
    ▼
CompiledQuery
```

como objetos diferentes cuando sea útil.

---

# 297. Benefits

Evita objetos gigantes con:

```text
$isNormalized
$isValidated
$isCompiled
$isExecuted
```

---

# 298. Query Engine façade

Podrá existir una API interna:

```php
interface QueryEngineInterface
{
    public function execute(
        QueryInterface $query,
        QueryExecutionRequest $request
    ): QueryResult;
}
```

---

# 299. Façade vs God object

`QueryEngine` podrá orquestar.

No deberá implementar internamente todas las etapas.

---

# 300. Correct orchestration

```text
QueryEngine
   │
   ├── Normalizer
   ├── Validator
   ├── SemanticAnalyzer
   ├── Optimizer
   ├── Planner
   ├── Compiler
   └── Executor
```

---

# 301. Each stage independently testable

Cada componente deberá poder probarse sin ejecutar todo el pipeline.

---

# 302. Pipeline object

Podrá existir:

```text
QueryPipeline
```

como composición explícita.

---

# 303. Pipeline configuration

El pipeline podrá variar para:

```text
normal query
raw query
schema query
debug explain
compile only
validate only
```

sin introducir condicionales dispersos.

---

# 304. Query operation modes

Conceptualmente:

```text
VALIDATE
ANALYZE
PLAN
COMPILE
EXECUTE
EXPLAIN
```

---

# 305. Compile-only

Permitirá generar SQL sin ejecutar.

---

# 306. Plan-only

Permitirá inspeccionar el plan.

---

# 307. Analyze-only

Permitirá herramientas IDE/static analysis.

---

# 308. Offline compilation

Será posible cuando:

```text
platform
dialect
capabilities
schema metadata
```

necesarios estén disponibles offline.

---

# 309. Online compilation

Algunas features podrán necesitar runtime discovery.

---

# 310. Compilation context

Nunca deberá abrir conexiones arbitrariamente.

Si requiere información online, ésta deberá suministrarse mediante un contexto resuelto.

---

# 311. No hidden I/O

Principio importante:

```text
Normalizer
Semantic Analyzer
Optimizer
Planner
Compiler
```

deberán evitar I/O oculto.

---

# 312. Metadata provider

Si necesitan schema metadata:

```text
SchemaMetadataProvider
```

deberá ser una dependencia explícita.

---

# 313. Metadata cache

El provider podrá usar cache o introspection según policy.

---

# 314. Semantic analysis and I/O

Preferencia:

```text
Semantic Analyzer
→ metadata abstraction
```

no:

```text
Semantic Analyzer
→ direct SQL query
```

---

# 315. Query compilation and persistent runtime

Compiler podrá ser:

```text
application/worker singleton
```

si es:

```text
stateless
immutable
reentrant
```

---

# 316. Query Builder lifetime

Normalmente:

```text
operation/application-owned
```

y nunca global mutable.

---

# 317. QueryContext lifetime

```text
operation
```

---

# 318. ExecutionContext lifetime

```text
execution operation
```

---

# 319. CompiledQuery lifetime

Puede sobrevivir entre requests si:

```text
immutable
context-free
cache-safe
```

---

# 320. Result lifetime

Normalmente operation-scoped.

---

# 321. StreamingResult lifetime

Puede mantener resources hasta cierre.

---

# 322. No compiled query captures request

Prohibido:

```text
CompiledQuery
→ Request
→ User
→ Tenant object
```

---

# 323. Tenant-specific compiled query

Si la estructura realmente cambia por tenant, el fingerprint deberá reflejarlo mediante metadata estable y no mediante referencia al objeto Tenant.

---

# 324. Query thread/coroutine safety

Representaciones inmutables serán naturalmente compartibles.

---

# 325. Mutable builders

No deberán compartirse concurrentemente.

---

# 326. Compiler concurrency

Compiler stateless deberá ser concurrency-safe.

---

# 327. Optimizer concurrency

Igualmente.

---

# 328. Planner concurrency

Igualmente, salvo state explícitamente operation-scoped.

---

# 329. FrankenPHP

En FrankenPHP:

```text
Worker
├── Compiler
├── Optimizer
├── Planner
├── Frozen Registries
│
├── Request A
│   └── QueryContext A
│
└── Request B
    └── QueryContext B
```

---

# 330. No request contamination

```text
QueryContext A
```

nunca deberá ser reutilizado como:

```text
QueryContext B
```

---

# 331. RoadRunner

Mismo modelo.

---

# 332. OpenSwoole

Cada coroutine deberá mantener sus contextos separados.

---

# 333. Query extension system

El Query Engine deberá ser extensible de forma controlada.

---

# 334. Possible extension points

```text
AST Nodes
Expressions
Predicates
Functions
Operators
Normalization Rules
Semantic Rules
Optimization Rules
Planner Strategies
Compiler Handlers
Query Metadata
```

---

# 335. Extension completeness

Agregar una nueva construcción puede requerir varias extensiones.

Ejemplo:

```text
Custom Query Feature
├── AST Node
├── Semantic Resolver
├── Type Rule
├── Capability Requirement
├── Planner Handler
└── Compiler Handler
```

---

# 336. No AST-only extensions

Registrar un nodo sin semantic/compiler support deberá fallar validation cuando intente utilizarse.

---

# 337. QueryExtensionDescriptor

Podrá declarar qué etapas soporta.

---

# 338. Extension registry freeze

Al igual que Driver:

```text
register during bootstrap
freeze before runtime
```

---

# 339. Deterministic extension ordering

Rules deberán tener orden explícito y reproducible.

---

# 340. Query extension security

Extensions no deberán recibir automáticamente:

```text
current user
container
request
credentials
```

---

# 341. Query architecture namespaces

Propuesta:

```text
VoltStack\Quantum\Database\Query\
├── Contract\
├── Model\
├── Ast\
├── Expression\
├── Predicate\
├── Parameter\
├── Type\
├── Metadata\
├── Context\
├── Normalization\
├── Validation\
├── Semantic\
├── Optimization\
├── Planning\
├── Compilation\
├── Execution\
├── Result\
├── Extension\
├── Diagnostics\
└── Exception\
```

---

# 342. Semantic namespaces

Podrán separarse:

```text
Query\Semantic\
├── Analysis\
├── Symbol\
├── Schema\
├── Type\
├── Relation\
├── Constraint\
└── Graph\
```

---

# 343. Planning namespaces

```text
Query\Planning\
├── Logical\
├── Physical\
├── Execution\
├── Strategy\
└── Exception\
```

---

# 344. Compilation namespaces

```text
Query\Compilation\
├── Contract\
├── Sql\
├── Prepared\
├── Cache\
├── Mysql\
├── Mariadb\
├── Postgresql\
└── Sqlite\
```

---

# 345. Execution namespaces

```text
Query\Execution\
├── Executor\
├── Statement\
├── Binding\
├── Timeout\
├── Cancellation\
├── Retry\
└── Error\
```

---

# 346. Dependency direction

La dirección principal será:

```text
Public Query API
      │
      ▼
Query Model
      │
      ▼
AST
      │
      ▼
Semantic
      │
      ▼
Optimization
      │
      ▼
Planning
      │
      ▼
Compilation
      │
      ▼
Execution
      │
      ▼
Connection
      │
      ▼
Driver
```

---

# 347. Forbidden reverse dependencies

No permitir:

```text
Driver → Query Builder
Connection → AST
Compiler → ORM
Optimizer → Executor
AST → Connection
Query Model → PDO
```

---

# 348. ORM dependency direction

Correcto:

```text
ORM
 │
 ▼
Query
```

Incorrecto:

```text
Query
 │
 ▼
ORM
```

---

# 349. Result/Hydration direction

```text
Execution
   │
   ▼
Result
   │
   ▼
Hydration
```

No:

```text
Result
→ EntityManager
```

---

# 350. Query architecture invariant DB-QRY-001

Query Builder nunca generará SQL directamente.

---

# 351. DB-QRY-002

Query Model no dependerá del Driver.

---

# 352. DB-QRY-003

AST será independiente del Dialect para construcciones portables.

---

# 353. DB-QRY-004

AST no almacenará SQL strings como representación primaria.

---

# 354. DB-QRY-005

Dynamic values utilizarán parámetros.

---

# 355. DB-QRY-006

Identifier quoting pertenecerá al Compiler/Dialect.

---

# 356. DB-QRY-007

Semantic Analysis ocurrirá antes de optimizaciones que requieran significado resuelto.

---

# 357. DB-QRY-008

Optimizer preservará semántica observable.

---

# 358. DB-QRY-009

Optimizer de VoltStack no sustituirá al optimizer del servidor.

---

# 359. DB-QRY-010

Planner no ejecutará consultas.

---

# 360. DB-QRY-011

Compiler no ejecutará consultas.

---

# 361. DB-QRY-012

Executor no compilará AST.

---

# 362. DB-QRY-013

Executor no hidratará Entities.

---

# 363. DB-QRY-014

ORM utilizará el mismo Query Engine.

---

# 364. DB-QRY-015

Active Record no tendrá un SQL generator independiente.

---

# 365. DB-QRY-016

Platform differences utilizarán capabilities.

---

# 366. DB-QRY-017

Query Builder no contendrá checks de nombres de Drivers.

---

# 367. DB-QRY-018

Unsupported features fallarán explícitamente.

---

# 368. DB-QRY-019

Emulation sólo será válida si preserva semántica requerida.

---

# 369. DB-QRY-020

Query transformations serán deterministas.

---

# 370. DB-QRY-021

Security policies no serán Optimization Rules.

---

# 371. DB-QRY-022

AST será preferentemente inmutable.

---

# 372. DB-QRY-023

Optimizer no mutará estructuras compartidas de forma insegura.

---

# 373. DB-QRY-024

CompiledQuery será inmutable.

---

# 374. DB-QRY-025

CompiledQuery no capturará request state.

---

# 375. DB-QRY-026

QueryContext será operation-scoped.

---

# 376. DB-QRY-027

No existirá current QueryContext global mutable.

---

# 377. DB-QRY-028

Query compilation será determinista bajo inputs equivalentes.

---

# 378. DB-QRY-029

Query fingerprints no incluirán secretos.

---

# 379. DB-QRY-030

Raw SQL será escape hatch explícito.

---

# 380. DB-QRY-031

Raw SQL seguirá utilizando parameter binding seguro.

---

# 381. DB-QRY-032

Debug SQL no se utilizará como executable SQL.

---

# 382. DB-QRY-033

Telemetry será opcional.

---

# 383. DB-QRY-034

Telemetry no deberá modificar semántica.

---

# 384. DB-QRY-035

Multitenancy será integración opcional.

---

# 385. DB-QRY-036

Tenant state no se almacenará globalmente en Query Engine.

---

# 386. DB-QRY-037

Sharding topology será responsabilidad externa al Query Model.

---

# 387. DB-QRY-038

Connection routing utilizará requisitos semánticos, no únicamente SQL parsing.

---

# 388. DB-QRY-039

Transaction affinity prevalecerá sobre routing normal.

---

# 389. DB-QRY-040

Locking queries declararán requirements apropiados.

---

# 390. DB-QRY-041

Retry policy no pertenecerá al Builder.

---

# 391. DB-QRY-042

Retry safety será conservadora.

---

# 392. DB-QRY-043

Compound operations declararán atomicity semantics.

---

# 393. DB-QRY-044

No se ocultará partial success.

---

# 394. DB-QRY-045

Subqueries serán Query Models/AST, no strings.

---

# 395. DB-QRY-046

CTEs serán estructuradas.

---

# 396. DB-QRY-047

Window functions serán estructuradas.

---

# 397. DB-QRY-048

Vendor-specific operators requerirán extensions/capabilities.

---

# 398. DB-QRY-049

NULL semantics serán preservadas.

---

# 399. DB-QRY-050

Query Functions tendrán semántica y tipos definidos.

---

# 400. DB-QRY-051

Volatile functions no serán optimizadas como constantes.

---

# 401. DB-QRY-052

No habrá I/O oculto en Compiler.

---

# 402. DB-QRY-053

Semantic metadata access será explícito.

---

# 403. DB-QRY-054

Compiler podrá ser singleton sólo si es stateless/reentrant.

---

# 404. DB-QRY-055

Mutable builders no serán compartidos concurrentemente.

---

# 405. DB-QRY-056

Persistent workers no compartirán QueryContext entre executions.

---

# 406. DB-QRY-057

Extensions se registrarán antes del registry freeze.

---

# 407. DB-QRY-058

Query extensions utilizarán APIs públicas de extensión.

---

# 408. DB-QRY-059

Query Engine no utilizará Service Locator.

---

# 409. DB-QRY-060

Query Engine no dependerá de HTTP.

---

# 410. DB-QRY-061

Query Engine no dependerá de Controller.

---

# 411. DB-QRY-062

Query Engine no dependerá de CurrentUser.

---

# 412. DB-QRY-063

Connection objects no estarán contenidos en AST.

---

# 413. DB-QRY-064

PDO nunca aparecerá en Query Model.

---

# 414. DB-QRY-065

Framework Explain y Database EXPLAIN serán conceptos diferentes.

---

# 415. DB-QRY-066

`EXPLAIN ANALYZE` requerirá una API explícita por su potencial de ejecución.

---

# 416. DB-QRY-067

Structural equality y semantic equivalence serán conceptos diferentes.

---

# 417. DB-QRY-068

Cada cache utilizará el fingerprint correspondiente a su semántica.

---

# 418. DB-QRY-069

Query policies serán distintas de optimization rules.

---

# 419. DB-QRY-070

La arquitectura permitirá compile-only, plan-only y validate-only.

---

# 420. Anti-pattern — Builder genera SQL

Incorrecto:

```php
public function where(string $column, mixed $value): static
{
    $this->sql .= " WHERE {$column} = ?";
}
```

---

# 421. Correcto

```php
public function where(string $column, mixed $value): static
{
    $this->predicates[] = new ComparisonPredicate(
        new ColumnReference($column),
        ComparisonOperator::Equal,
        new ParameterExpression($value),
    );

    return $this;
}
```

---

# 422. Anti-pattern — Vendor condition in Builder

Incorrecto:

```php
if ($this->driver === 'pgsql') {
    // PostgreSQL syntax
}
```

---

# 423. Correcto

```text
Builder
→ semantic operation
→ capability resolution
→ compiler specialization
```

---

# 424. Anti-pattern — ORM SQL generator

Incorrecto:

```text
EntityManager
→ ORM SQL Generator
→ PDO
```

---

# 425. Correcto

```text
EntityManager
      │
      ▼
Persistence / Entity Query
      │
      ▼
Query Model
      │
      ▼
Query Engine
      │
      ▼
Execution
```

---

# 426. Anti-pattern — Compiler opens connection

Incorrecto:

```php
$compiler->compile($query, DB::connection());
```

cuando la conexión es utilizada para ejecutar I/O implícito.

---

# 427. Correcto

```text
Compiler
+
Resolved Compilation Context
=
CompiledQuery
```

---

# 428. Anti-pattern — QueryContext singleton

Incorrecto:

```php
QueryContext::setCurrent($context);
```

---

# 429. Correcto

```text
ExecutionScope
   │
   ▼
Operation
   │
   ▼
Explicit QueryContext
```

---

# 430. Anti-pattern — SQL parser as architecture

No utilizar el SQL compilado para reconstruir toda la semántica que VoltStack ya conocía.

---

# 431. Correcto

Conservar metadata estructurada desde el inicio.

---

# 432. Anti-pattern — Optimizer owns authorization

Incorrecto:

```text
Optimizer Rule
→ add security filter
```

---

# 433. Correcto

```text
Query Policy
→ Effective Query
→ Optimizer
```

---

# 434. Anti-pattern — Generic feature emulation

Incorrecto:

```text
unsupported feature
→ always emulate somehow
```

---

# 435. Correcto

```text
Capability
      │
      ├── NATIVE
      ├── SAFE_EMULATION
      └── UNSUPPORTED
```

---

# 436. Anti-pattern — Query cache by SQL only

Dos consultas con SQL aparentemente igual pueden tener metadata semántica diferente.

La cache deberá utilizar fingerprints apropiados.

---

# 437. Anti-pattern — Hidden raw expression

Incorrecto:

```php
->where($arbitrarySql)
```

sin distinguir trusted SQL de structured predicate.

---

# 438. Correcto

APIs separadas:

```text
where(Expression)
whereRaw(TrustedRawSql, bindings)
```

o equivalente.

---

# 439. Arquitectura final del Query Engine

```text
                         APPLICATION
                              │
                              ▼
                      Public Query API
                              │
                              ▼
                        Query Builder
                              │
                              ▼
                         Query Model
                              │
                              ▼
                           Query AST
                              │
                              ▼
                    Policy Transformations
                              │
                              ▼
                         Normalization
                              │
                              ▼
                    Structural Validation
                              │
                              ▼
                      Semantic Analysis
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
             Symbols        Types        Relations
                │             │             │
                └─────────────┼─────────────┘
                              ▼
                       Semantic Graph
                              │
                              ▼
                         Optimization
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
                          Compiler
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                 Dialect             Platform
                    │               Capabilities
                    └─────────┬─────────┘
                              ▼
                        CompiledQuery
                              │
                              ▼
                           Executor
                              │
                              ▼
                     Connection Manager
                              │
                              ▼
                         Connection
                              │
                              ▼
                            Lease
                              │
                              ▼
                            Driver
                              │
                              ▼
                       Native Statement
                              │
                              ▼
                        Database Server
                              │
                              ▼
                         Native Result
                              │
                              ▼
                            Result
                              │
                              ▼
                         Hydration Layer
```

---

# 440. Arquitectura simplificada

La arquitectura completa puede reducirse conceptualmente a:

```text
INTENT
  │
  ▼
STRUCTURE
  │
  ▼
MEANING
  │
  ▼
OPTIMIZATION
  │
  ▼
PLAN
  │
  ▼
SQL
  │
  ▼
EXECUTION
  │
  ▼
RESULT
```

---

# 441. Correspondencia

```text
INTENT
=
Query Builder

STRUCTURE
=
Query Model + AST

MEANING
=
Semantic Engine

OPTIMIZATION
=
Query Optimizer

PLAN
=
Query Planner

SQL
=
SQL Compiler

EXECUTION
=
Execution Engine

RESULT
=
Result System
```

---

# 442. Fórmula arquitectónica

```text
VoltStack Query Engine
=
Structured Intent
+
Immutable AST
+
Semantic Resolution
+
Framework Optimization
+
Explicit Planning
+
Capability-Aware Compilation
+
Prepared Execution
+
Managed Results
```

---

# 443. Fórmula de portabilidad

```text
Portable Query
=
Semantic Intent
-
Vendor Syntax
```

La sintaxis vendor-specific aparece únicamente al final:

```text
Semantic Query
      │
      ▼
Platform/Dialect
      │
      ▼
SQL Compiler
```

---

# 444. Fórmula de seguridad

```text
Safe Query
=
Structured Identifiers
+
Structured Expressions
+
Parameters
+
Type Resolution
+
Prepared Statements
+
Explicit Raw Escape Hatch
```

---

# 445. Fórmula de optimización

```text
Framework Query Optimization
=
Canonicalization
+
Semantic Knowledge
+
Safe Rewrite Rules
+
Query Planning
```

mientras:

```text
Database Optimization
=
Indexes
+
Server Statistics
+
Join Algorithms
+
Storage Engine
+
Server Execution Planning
```

Ambos niveles son complementarios.

---

# 446. Fórmula de persistent runtime safety

```text
Persistent-Safe Query Engine
=
Immutable Shared Metadata
+
Stateless Compiler
+
Stateless Optimizer
+
Stateless Planner
+
Operation-Scoped QueryContext
+
Execution-Scoped Resources
+
No Global Mutable Query State
```

---

# 447. Fórmula de extensibilidad

```text
Query Extension
=
AST Support
+
Semantic Support
+
Capability Requirements
+
Planner Support
+
Compiler Support
```

según la naturaleza de la feature.

---

# 448. Decisión final

VoltStack adoptará un Query Engine basado en:

```text
Laravel-like Query Builder DX
+
Structured Query Model
+
Immutable AST
+
Doctrine-like Separation of Concerns
+
Semantic Query Analysis
+
VoltStack Query Optimizer
+
Logical/Physical Planning
+
Capability-Aware SQL Compilation
+
Prepared Statement Execution
+
Persistent Runtime Safety
```

---

# 449. Consecuencia arquitectónica

Una llamada aparentemente simple:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

conceptualmente recorrerá:

```text
Query Builder
      │
      ▼
Query Model
      │
      ▼
AST
      │
      ▼
Normalization
      │
      ▼
Semantic Analysis
      │
      ▼
Semantic Graph
      │
      ▼
Optimizer
      │
      ▼
Planner
      │
      ▼
Compiler
      │
      ▼
CompiledQuery
      │
      ▼
Executor
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

sin que la complejidad interna contamine la experiencia pública.

---

# 450. Regla maestra

> **El Query Builder expresa intención; el Query Model la estructura; el AST la representa; el Semantic Engine determina su significado; el Optimizer la transforma preservando semántica; el Planner decide cómo realizarla; el Compiler genera SQL específico de plataforma; y el Executor es el único responsable de ejecutarla.**

En forma compacta:

```text
Builder
   ↓
Model
   ↓
AST
   ↓
Semantic
   ↓
Optimize
   ↓
Plan
   ↓
Compile
   ↓
Execute
```

Ninguna capa deberá saltarse sus fronteras para obtener una implementación aparentemente más sencilla a costa de acoplamiento futuro.

---

# 451. Próximo documento

El siguiente documento será:

```text
24_DATABASE_QUERY_MODEL.md
```

y deberá definir formalmente el modelo intermedio común sobre el cual se representarán:

```text
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
QuerySource
Projection
Join
Grouping
Ordering
Pagination
Locking
CTE
SetOperation
Returning
QueryMetadata
QueryParameters
```

antes de profundizar en:

```text
25_DATABASE_QUERY_AST_SYSTEM.md
```

La relación será:

```text
23 Query Architecture
        │
        ▼
defines the complete pipeline

24 Query Model
        │
        ▼
defines the structured query domain

25 Query AST
        │
        ▼
defines the canonical internal tree representation
```