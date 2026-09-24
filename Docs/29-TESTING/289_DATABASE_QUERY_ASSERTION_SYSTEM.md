# 289_DATABASE_QUERY_ASSERTION_SYSTEM.md

# VoltStack Quantum Database
## Database Query Assertion System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 289 — Database Query Assertion System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md`  
**Siguiente documento:** `290_DATABASE_SCHEMA_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Query Assertion System** de VoltStack.

Su responsabilidad es proporcionar una infraestructura oficial, estructurada y semántica para comprobar el comportamiento de las consultas generadas, planificadas y ejecutadas por:

- Query Builder;
- Query AST;
- Semantic Query Engine;
- Query Optimizer;
- Query Planner;
- SQL Compiler;
- Query Executor;
- ORM;
- Relationship Loader;
- Pagination;
- Cache;
- Multitenancy;
- Sharding;
- Read/Write Routing;
- Security;
- Telemetry.

El sistema permitirá assertions como:

```php
$this->assertDatabaseQueryCount(3);

$this->assertDatabaseQueryExecuted(
    QueryExpectation::select('users')
);

$this->assertNoDatabaseQueries();

$this->assertQueryUsesWriter();

$this->assertNoNPlusOneQueries();
```

sin obligar a todos los tests a comparar SQL textual.

Regla central:

> **Una assertion de consulta deberá observar la representación más semántica capaz de demostrar la propiedad evaluada; el SQL textual sólo será la autoridad principal cuando el SQL generado sea precisamente el comportamiento bajo prueba.**

Por tanto:

```text
Business Query Behavior
≠
SQL String Equality
```

y:

```text
Query Assertion
≠
SQL String Assertion
```

---

# 2. Objetivos

El sistema deberá permitir comprobar:

```text
qué consulta ocurrió
cuántas veces ocurrió
por qué ocurrió
qué Query Model la originó
qué AST fue utilizado
qué plan fue seleccionado
qué SQL fue compilado
qué bindings fueron utilizados
qué conexión fue seleccionada
qué transacción estaba activa
qué tenant estaba activo
qué shard fue utilizado
qué resultado produjo
cuánto tardó
si fue cache hit/miss
si formó parte de un patrón N+1
```

sin mezclar todas estas propiedades en una única assertion frágil.

---

# 3. Query Assertion ≠ Query Execution

El sistema de assertions:

```text
observa
registra
normaliza
filtra
compara
reporta
```

pero no:

```text
ejecuta consultas
modifica AST
optimiza queries
elige conexiones
genera SQL
```

---

# 4. Query Assertion ≠ Query Profiler

El profiler recopila información de rendimiento.

El Query Assertion System utiliza evidencia para comprobar expectativas.

```text
Profiler
≠
Assertion Engine
```

aunque ambos puedan consumir una infraestructura común de observación.

---

# 5. Query Assertion ≠ Telemetry

Telemetry está orientada a observabilidad operacional.

Testing assertions están orientadas a:

```text
deterministic verification
```

---

# 6. Query Assertion ≠ N+1 Detector

El sistema podrá consumir:

```text
NPlusOneDetectionResult
```

pero no reimplementará el detector.

---

# 7. Arquitectura general

```text
Query Definition
      │
      ▼
Query Builder
      │
      ▼
Query Model / AST
      │
      ▼
Semantic Engine
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
Executor
      │
      ▼
Database
```

El Testing Observation Layer podrá observar varios puntos:

```text
Builder
   │
   ├──── Query Model Observation
   │
AST
   │
   ├──── AST Observation
   │
Planner
   │
   ├──── Plan Observation
   │
Compiler
   │
   ├──── Compilation Observation
   │
Executor
   │
   ├──── Execution Observation
   │
Database
   │
   └──── Result / Outcome Observation
```

---

# 8. Query Observation Pipeline

```text
Database Operation
       │
       ▼
Query Observation Hooks
       │
       ▼
Observation Normalizer
       │
       ▼
Query Test Record
       │
       ▼
Query Recorder
       │
       ▼
Assertion Engine
       │
       ▼
Assertion Result
```

---

# 9. QueryTestRecord

La unidad principal de evidencia será un registro estructurado.

Conceptualmente:

```php
final readonly class QueryTestRecord
{
    public function __construct(
        public QueryExecutionId $executionId,
        public QueryFingerprint $fingerprint,
        public QueryOperationType $operation,
        public ?QueryModelSnapshot $queryModel,
        public ?QueryAstSnapshot $ast,
        public ?QueryPlanSnapshot $plan,
        public ?CompiledQuerySnapshot $compiledQuery,
        public BindingSnapshot $bindings,
        public QueryExecutionContextSnapshot $context,
        public QueryOutcomeSnapshot $outcome,
    ) {}
}
```

---

# 10. Query record ≠ production event

Un `QueryTestRecord` es infraestructura de testing.

No deberá convertirse en dependencia obligatoria del runtime productivo.

---

# 11. Observation levels

VoltStack reconocerá varios niveles:

```text
QUERY_MODEL
AST
SEMANTIC
LOGICAL_PLAN
PHYSICAL_PLAN
COMPILED
EXECUTION
OUTCOME
```

---

# 12. Assertion level

Toda assertion deberá declarar o inferir qué nivel necesita.

Ejemplo:

```text
assertWherePredicate(...)
```

puede trabajar sobre:

```text
AST
```

mientras:

```text
assertSql(...)
```

requiere:

```text
COMPILED
```

---

# 13. Highest Semantic Layer Principle

Si la propiedad puede demostrarse en:

```text
Query Model
```

no deberá bajar innecesariamente a:

```text
SQL text
```

---

# 14. Ejemplo

Queremos comprobar:

```text
query filters users by active=true
```

Preferible:

```php
$this->assertQueryExecuted(
    QueryExpectation::select('users')
        ->where('active', '=', true)
);
```

en lugar de:

```php
$this->assertSqlContains(
    'WHERE `active` = ?'
);
```

---

# 15. Why

La primera assertion puede ser portable entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

La segunda está acoplada al dialecto.

---

# 16. Query Recorder

El recorder deberá ser scope-local.

```php
interface QueryRecorder
{
    public function record(QueryTestRecord $record): void;

    public function records(): QueryTestRecordCollection;

    public function reset(): void;
}
```

---

# 17. Recorder Scope

```text
Test
Request
Operation
Worker
```

deberán distinguirse.

---

# 18. No global mutable recorder

Prohibido:

```php
static array $queries = [];
```

como estado global compartido.

---

# 19. Parallel tests

Cada test deberá tener:

```text
QueryRecorderScope
```

independiente.

---

# 20. Persistent runtimes

FrankenPHP, RoadRunner y OpenSwoole deberán limpiar el estado de testing entre scopes.

---

# 21. Query Recording Modes

Podrán existir:

```text
OFF
MINIMAL
SEMANTIC
COMPILED
FULL
```

---

# 22. MINIMAL

Registra:

```text
operation
fingerprint
outcome
```

---

# 23. SEMANTIC

Añade:

```text
Query Model
AST metadata
semantic metadata
```

---

# 24. COMPILED

Añade:

```text
compiled SQL
bindings metadata
dialect
```

---

# 25. FULL

Puede incluir:

```text
plan
connection context
transaction context
timing
cache information
```

con redacción.

---

# 26. Recording overhead

Los tests deberán poder activar sólo la evidencia necesaria.

---

# 27. Query Fingerprint

El fingerprint representa identidad semántica/estructural.

Ejemplo conceptual:

```text
SELECT users
WHERE email = ?
```

podrá producir el mismo fingerprint para:

```text
email = alice@example.com
email = bob@example.com
```

---

# 28. Fingerprint ≠ SQL hash

Un fingerprint semántico no deberá depender necesariamente del SQL textual.

---

# 29. Fingerprint use

Útil para:

```text
deduplication
query counting
N+1 correlation
query matching
performance assertions
```

---

# 30. Query Operation Type

Valores conceptuales:

```text
SELECT
INSERT
UPDATE
DELETE
DDL
TRANSACTION_CONTROL
ADMINISTRATIVE
CUSTOM
```

---

# 31. Query Type Assertions

Ejemplo:

```php
$this->assertQueryTypeExecuted(
    QueryOperationType::SELECT
);
```

---

# 32. Query Count Assertions

API conceptual:

```php
$this->assertDatabaseQueryCount(5);
```

---

# 33. Count scope

La assertion deberá poder especificar:

```text
all queries
SELECT only
writer only
specific fingerprint
specific entity/table
specific transaction
specific tenant
specific shard
```

---

# 34. Example

```php
$this->assertDatabaseQueryCount(
    2,
    QueryFilter::select()
);
```

---

# 35. Count ≠ N+1

```text
10 queries
```

no demuestra:

```text
N+1
```

---

# 36. Query Absence Assertions

Ejemplo:

```php
$this->assertNoDatabaseQueries();
```

---

# 37. Useful cases

Ideal para comprobar:

```text
cache hit
IdentityMap hit
pure computation
authorization rejection before DB
```

---

# 38. No-query assertion scope

Podrá limitarse:

```text
during callback
```

---

# 39. Callback assertion

Ejemplo:

```php
$this->assertNoDatabaseQueries(function () use ($user) {
    $user->getAlreadyLoadedProperty();
});
```

---

# 40. Query Executed Assertion

Ejemplo:

```php
$this->assertQueryExecuted(
    QueryExpectation::select('users')
);
```

---

# 41. Query Not Executed

```php
$this->assertQueryNotExecuted(
    QueryExpectation::delete('users')
);
```

---

# 42. QueryExpectation

Objeto declarativo:

```php
final class QueryExpectation
{
    public static function select(string $source): self;

    public function where(
        string $field,
        string $operator,
        mixed $value = null
    ): self;
}
```

---

# 43. Expectation ≠ Query Builder

`QueryExpectation` no será otra API para construir consultas.

Sólo describe propiedades que deben observarse.

---

# 44. Partial matching

Por defecto una expectation podrá comprobar sólo los aspectos declarados.

Ejemplo:

```text
SELECT users
```

no requiere conocer todos los predicates.

---

# 45. Exact matching

Podrá solicitarse:

```text
EXACT
```

cuando se necesite.

---

# 46. Match modes

```text
PARTIAL
EXACT
STRUCTURAL
SEMANTIC
```

---

# 47. AST Assertions

El sistema deberá permitir comprobar estructuras del AST.

Ejemplo:

```text
SelectNode
├── Source(users)
└── Predicate
    └── Equals(active, parameter)
```

---

# 48. AST assertion example

Conceptualmente:

```php
$this->assertQueryAst(
    AstExpectation::select()
        ->hasPredicate(
            PredicateExpectation::equals('active')
        )
);
```

---

# 49. AST assertions should be typed

No depender de:

```text
JSON string
```

cuando existan nodos tipados.

---

# 50. AST structural equality

Podrá utilizarse cuando la estructura exacta sea la propiedad evaluada.

---

# 51. AST semantic equivalence

Dos AST diferentes podrían representar lógica equivalente.

Por tanto:

```text
StructuralEquality
≠
SemanticEquivalence
```

---

# 52. Semantic Query Assertions

Podrán verificar:

```text
resolved table
resolved column
resolved relation
inferred type
resolved join
constraint
```

---

# 53. Example

```text
User.profile
```

puede resolver a:

```text
users
JOIN profiles
```

La assertion semántica deberá poder verificar la relación resuelta sin depender del SQL final.

---

# 54. Symbol Resolution Assertions

Ejemplos:

```text
field resolved
relationship resolved
ambiguous symbol rejected
unknown symbol rejected
```

---

# 55. Type Inference Assertions

Ejemplo:

```text
parameter :id
→ INTEGER
```

---

# 56. Query Constraint Assertions

Podrán verificar:

```text
tenant predicate exists
soft delete predicate exists
authorization scope exists
```

---

# 57. Critical security use

Ejemplo:

```php
$this->assertQueryHasTenantConstraint($tenantId);
```

---

# 58. Tenant assertion ≠ string search

No buscar simplemente:

```text
tenant_id
```

en SQL.

Deberá comprobar la semántica correspondiente cuando sea posible.

---

# 59. Soft Delete Assertion

Ejemplo:

```php
$this->assertSoftDeleteScopeApplied(User::class);
```

---

# 60. Authorization Query Scope

Podrá comprobarse que el scope autorizado fue incorporado antes de ejecución.

---

# 61. Logical Plan Assertions

Podrán verificar:

```text
scan
join
filter
projection
aggregation
sort
limit
```

del Logical Query Plan.

---

# 62. Physical Plan Assertions

VoltStack podrá verificar decisiones propias como:

```text
writer route
replica route
batch relation load
select-in strategy
join eager load
```

---

# 63. Physical plan ≠ DB EXPLAIN plan

El plan físico de VoltStack no será necesariamente el plan interno del DBMS.

---

# 64. DB EXPLAIN assertions

Podrán existir en suites especializadas, pero serán integración específica.

---

# 65. Plan Assertion Example

```php
$this->assertQueryPlan(
    QueryPlanExpectation::select()
        ->usesReplica()
);
```

---

# 66. Optimizer Assertions

Podrán verificar:

```text
predicate normalized
redundant predicate removed
join simplified
query deduplicated
```

---

# 67. Optimizer correctness

La assertion deberá comprobar que la transformación preserva semántica.

---

# 68. SQL Assertions

Cuando se prueba el Compiler:

```php
$this->assertCompiledSql(
    'SELECT "id", "name" FROM "users" WHERE "id" = ?'
);
```

será válido.

---

# 69. SQL normalization

La comparación podrá soportar:

```text
EXACT
NORMALIZED
TOKENIZED
```

---

# 70. EXACT

Compara exactamente.

Útil cuando formatting también forma parte del contrato.

---

# 71. NORMALIZED

Puede normalizar:

```text
whitespace
line breaks
insignificant formatting
```

sin alterar tokens semánticos.

---

# 72. TOKENIZED

Compara secuencia de tokens SQL.

---

# 73. No unsafe SQL normalization

No deberá modificar:

```text
quoted strings
identifiers
operators
literal semantics
```

---

# 74. Dialect-aware assertions

El SQL esperado deberá estar asociado a:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando corresponda.

---

# 75. Cross-platform compiler tests

Ejemplo:

```text
same Query AST
        │
 ┌──────┼─────────┐
 ▼      ▼         ▼
MySQL PostgreSQL SQLite
```

cada uno puede producir SQL distinto pero semánticamente equivalente.

---

# 76. SQL Snapshot Assertions

Podrán utilizarse para suites amplias de Compiler.

---

# 77. Snapshot stability

Deberá existir serialización determinista.

---

# 78. Snapshot review

Un cambio de snapshot deberá tratarse como cambio de comportamiento, no actualizarse automáticamente sin revisión.

---

# 79. Binding Assertions

Podrán comprobar:

```text
number of bindings
binding names
binding order
logical types
driver types
redaction status
```

---

# 80. Binding values

Los valores podrán comprobarse explícitamente en tests controlados.

---

# 81. Sensitive bindings

Deberán respetar clasificación.

Ejemplo:

```text
password
token
secret
```

no deberá aparecer en diagnostics por defecto.

---

# 82. Binding assertion example

```php
$this->assertQueryBinding(
    name: 'id',
    value: 42,
    type: DatabaseType::INTEGER
);
```

---

# 83. Positional bindings

El sistema deberá poder verificar:

```text
position 1
position 2
...
```

---

# 84. Named bindings

También:

```text
:id
:email
```

cuando el driver/compiler los utilice.

---

# 85. Binding order

Importante para prepared statements posicionales.

---

# 86. Parameter Type Assertions

Ejemplo:

```php
$this->assertParameterType(
    'created_at',
    DatabaseType::INSTANT
);
```

---

# 87. Type assertion levels

Podrá distinguir:

```text
logical type
converted persistent type
driver binding type
```

---

# 88. Value Conversion Assertion

Podrá comprobar:

```text
PHP Value
→ canonical DB value
```

sin requerir SQL.

---

# 89. Enum assertions

Podrá comprobar que:

```text
Status::ACTIVE
```

se convierte al valor persistente correcto.

---

# 90. Query Context Assertions

El registro podrá contener:

```text
database
tenant
shard
connection role
transaction
request
operation
read/write intent
deadline
```

---

# 91. Connection Role Assertion

Ejemplo:

```php
$this->assertQueryUsesWriter();
```

---

# 92. Replica Assertion

```php
$this->assertQueryUsesReplica();
```

---

# 93. Route eligibility

La assertion deberá distinguir:

```text
query eligible for replica
```

de:

```text
query actually executed on replica
```

---

# 94. Transaction Assertion

Ejemplo:

```php
$this->assertQueryExecutedInsideTransaction();
```

---

# 95. Transaction identity

Podrá verificarse que varias queries pertenezcan a:

```text
same TransactionId
```

---

# 96. Transaction attempt

También:

```text
attempt 1
attempt 2
```

para retries.

---

# 97. Tenant Assertions

Ejemplo:

```php
$this->assertQueryTenant($tenantId);
```

---

# 98. Tenant Context ≠ Tenant Predicate

Ambas son propiedades distintas.

Puede existir:

```text
tenant context = A
```

pero faltar accidentalmente:

```text
tenant predicate
```

en shared-schema mode.

Ambas deberán poder probarse.

---

# 99. Database-per-tenant

En este caso la seguridad puede depender de:

```text
resolved logical database
```

más que de predicate.

---

# 100. Schema-per-tenant

Podrá comprobarse:

```text
resolved schema
```

---

# 101. Shard Assertions

Ejemplo:

```php
$this->assertQueryShard('shard-03');
```

---

# 102. Shard route

Podrá comprobarse:

```text
resolved shard
route reason
routing key
topology generation
```

---

# 103. Cross-shard assertions

Podrán verificar que una consulta:

```text
fans out
```

sólo cuando sea explícitamente permitida.

---

# 104. No accidental fan-out

Ejemplo:

```php
$this->assertSingleShardQuery();
```

---

# 105. Query Ordering Assertions

El recorder conservará orden lógico de ejecución.

---

# 106. Example

```php
$this->assertQueriesExecutedInOrder([
    QueryExpectation::insert('orders'),
    QueryExpectation::insert('outbox'),
]);
```

---

# 107. Ordering caution

Sólo deberá verificarse cuando el orden sea parte del contrato.

---

# 108. Partial order

Para concurrencia podrá ser necesario:

```text
A before B
```

sin exigir orden total.

---

# 109. Query Sequence Assertions

Podrá existir:

```text
FIRST
BEFORE
AFTER
IMMEDIATELY_BEFORE
IMMEDIATELY_AFTER
```

---

# 110. Query Group

Varias queries podrán asociarse a:

```text
flush
relationship load
pagination request
transaction attempt
```

---

# 111. Group Assertions

Ejemplo:

```text
this flush executed exactly:
2 INSERT
1 UPDATE
0 DELETE
```

---

# 112. ORM Query Assertions

Podrán comprobar consultas generadas por:

```text
find()
repository
entity query
flush()
relationship loading
```

---

# 113. IdentityMap Assertion

Ejemplo:

```php
$this->assertNoDatabaseQueries(function () use ($em) {
    $em->find(User::class, 42);
});
```

cuando la entidad ya esté managed.

---

# 114. Lazy Loading Assertion

Ejemplo:

```php
$this->assertDatabaseQueryCount(1, function () use ($user) {
    $user->posts;
});
```

---

# 115. Lazy policy assertion

Cuando:

```text
LazyLoadingPolicy = FORBID
```

podrá verificarse:

```text
no query executed
+
exception generated
```

---

# 116. Eager Loading Assertions

Podrá comprobarse:

```text
root query
relationship query count
strategy
```

sin asumir que eager loading siempre implica JOIN.

---

# 117. Eager ≠ JOIN

Una assertion como:

```text
assertEagerLoaded(posts)
```

no deberá exigir:

```text
JOIN posts
```

si el planner seleccionó `SELECT_IN`.

---

# 118. Batch Relationship Assertions

Podrá comprobar:

```text
N owners
→ 1 batched relation query
```

---

# 119. N+1 Assertions

API conceptual:

```php
$this->assertNoNPlusOneQueries();
```

---

# 120. Semantic detector

La assertion consumirá:

```text
NPlusOneDetectionResult
```

del sistema definido previamente.

---

# 121. No threshold-only N+1 detection

Incorrecto:

```text
queries > 10
→ N+1
```

---

# 122. N+1 Evidence

Podrá incluir:

```text
repeated semantic fingerprint
parent iteration correlation
relationship metadata
confidence
```

---

# 123. N+1 confidence

Assertions podrán definir:

```text
minimum confidence
```

---

# 124. Example

```php
$this->assertNoNPlusOneQueries(
    minimumConfidence: DetectionConfidence::HIGH
);
```

---

# 125. Slow Query Assertions

Podrá integrarse con:

```text
SlowQueryDetectionSystem
```

---

# 126. Example

```php
$this->assertNoSlowDatabaseQueries();
```

---

# 127. Slow threshold

Deberá provenir de policy explícita.

---

# 128. Test timing caution

Wall-clock CI variability puede hacer frágiles estas assertions.

---

# 129. Prefer deterministic timing

Cuando se pruebe lógica del detector:

```text
FakeMonotonicClock
```

---

# 130. Real performance tests

Los thresholds reales pertenecerán principalmente a:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 131. Query Cache Assertions

Podrán comprobar:

```text
cache hit
cache miss
cache bypass
cache invalidation
```

---

# 132. Cache Hit Assertion

Ejemplo:

```text
second operation
→ zero DB executions
```

cuando esa sea la semántica esperada.

---

# 133. Cache layer distinction

La assertion deberá distinguir:

```text
Query Cache
Result Cache
Entity Cache
Metadata Cache
Compiled Query Cache
```

---

# 134. Compiled Query Cache

Un hit aquí:

```text
does not imply zero DB queries
```

Sólo evita recompilación.

---

# 135. Result Cache

Puede evitar ejecución DB.

---

# 136. Query compilation assertions

Podrá comprobarse:

```text
compiled once
executed N times
```

---

# 137. Prepared Statement Assertions

Podrá verificar:

```text
prepared once
executed multiple times
```

cuando corresponda.

---

# 138. Prepared statement reuse

No deberá asumirse universalmente si driver/runtime no lo garantiza.

---

# 139. Query Retry Assertions

Ejemplo:

```text
attempt 1 → deadlock
attempt 2 → success
```

Podrá comprobarse:

```text
same semantic operation
different attempt
```

---

# 140. Retry count

```php
$this->assertQueryAttempts(2);
```

---

# 141. Statement retry vs transaction retry

La assertion deberá distinguirlos.

---

# 142. Security Assertions

Podrán verificar:

```text
parameterization
identifier validation
tenant scope
authorization scope
redaction
raw expression policy
```

---

# 143. Parameterization Assertion

Ejemplo:

```php
$this->assertQueryIsParameterized();
```

---

# 144. Parameterization meaning

No significa que absolutamente ningún literal pueda aparecer.

SQL keywords, safe compiler-generated constants y determinados literals estructurales pueden existir.

---

# 145. User input assertion

Podrá comprobarse que un valor proveniente de input se convirtió en binding y no en estructura SQL.

---

# 146. Raw Expression Assertion

Podrá verificar:

```text
raw expression absent
```

o:

```text
explicitly trusted raw expression used
```

---

# 147. Identifier Assertion

Podrá comprobar que identifiers provienen de:

```text
validated identifier model
```

---

# 148. Redaction Assertions

Ejemplo:

```php
$this->assertQueryDiagnosticsDoNotContain(
    $password
);
```

---

# 149. Better structured assertion

Preferible:

```php
$this->assertBindingRedacted('password');
```

---

# 150. Query Audit Assertions

Podrá comprobarse que una operación sensible produjo el audit record correspondiente.

---

# 151. Audit ≠ telemetry

Las assertions deberán conservar esta separación.

---

# 152. Query Event Assertions

Podrán comprobar eventos:

```text
QueryPlanningStarted
QueryCompiled
QueryExecuting
QueryExecuted
QueryFailed
```

según el Event System.

---

# 153. Event order

Sólo si es contrato.

---

# 154. Query Failure Assertions

Ejemplo:

```php
$this->assertQueryFailedWith(
    UniqueConstraintViolation::class
);
```

---

# 155. Native error assertions

Los tests del driver podrán comprobar mapping de:

```text
SQLSTATE
vendor code
```

---

# 156. Higher layers

Deberán preferir la excepción normalizada.

---

# 157. Query Outcome Assertions

Estados conceptuales:

```text
SUCCESS
FAILED
CANCELLED
TIMED_OUT
UNKNOWN
```

---

# 158. UNKNOWN

No deberá convertirse en:

```text
FAILED
```

para simplificar assertions.

---

# 159. Cancellation Assertion

Podrá comprobar:

```text
cancellation requested
statement cancellation attempted
connection disposition
outcome
```

---

# 160. Query Timeout Assertion

Deberá distinguir:

```text
DB timeout
driver timeout
VoltStack deadline
test timeout
```

---

# 161. Query Result Assertions

El sistema de query assertions no deberá convertirse en sistema general de assertions de entidades.

---

# 162. Result metadata

Sí podrá comprobar:

```text
affected rows
generated identifier
result shape
cursor usage
```

cuando sean parte de ejecución.

---

# 163. Affected Rows

Ejemplo:

```php
$this->assertAffectedRows(1);
```

---

# 164. Platform caveat

Affected row semantics pueden variar.

La assertion deberá utilizar el significado normalizado por VoltStack.

---

# 165. Generated ID Assertion

Podrá verificar que Execution Engine recibió/proporcionó identificador.

---

# 166. Streaming Assertions

Podrán comprobar:

```text
cursor used
not fully materialized
cursor closed
```

---

# 167. Large Dataset Assertions

Podrá comprobar:

```text
chunk query count
cursor strategy
keyset predicate
bounded batch size
```

---

# 168. Pagination Assertions

Podrán verificar:

```text
limit
offset
cursor boundary
stable ordering
lookahead N+1
count query
```

---

# 169. Cursor Pagination

Preferir assertions semánticas sobre:

```text
CursorBoundary
OrderingSpecification
```

---

# 170. Cursor ≠ encoded string

No será necesario decodificar manualmente el cursor en tests de alto nivel.

---

# 171. Count Query Assertions

Podrá verificarse por separado:

```text
Data Query
Count Query
```

---

# 172. Pagination count

No deberá asumirse que ambos queries son estructuralmente iguales.

---

# 173. Full-Text Search Assertions

Podrán comprobar:

```text
FullTextSearchNode
ranking
language
search mode
```

antes del Compiler.

---

# 174. JSON Query Assertions

Podrán verificar:

```text
JsonPath
contains
extract
missing/null semantics
```

---

# 175. Spatial Query Assertions

Podrán verificar:

```text
geometry predicate
SRID
distance semantics
CRS transformation request
```

---

# 176. Capability Assertions

Podrá verificarse qué capabilities influyeron en la planificación.

---

# 177. Example

```text
RETURNING required
→ capability supported
→ returning plan selected
```

---

# 178. Capability dependency record

El plan podrá exponer:

```text
required capabilities
preferred capabilities
fallbacks considered
```

para testing.

---

# 179. Capability Assertion

Ejemplo:

```php
$this->assertQueryRequiresCapability(
    'query.returning'
);
```

---

# 180. Capability ≠ Vendor

No:

```php
$this->assertQueryIsPostgres();
```

para comprobar una feature portable.

---

# 181. Cross-platform Assertions

El sistema deberá facilitar ejecutar el mismo test semántico contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 182. Same semantics, different SQL

Ejemplo:

```text
Semantic Assertion
       │
       ├── MySQL SQL
       ├── MariaDB SQL
       ├── PostgreSQL SQL
       └── SQLite SQL
```

---

# 183. Platform-specific assertions

Podrán añadirse cuando la propiedad sea realmente específica.

---

# 184. Unsupported feature

Un test deberá distinguir:

```text
feature unsupported
```

de:

```text
query incorrectly compiled
```

---

# 185. UNKNOWN capability

Tampoco deberá tratarse automáticamente como unsupported.

---

# 186. Assertion Context

Toda assertion deberá conocer:

```text
Test Scope
Recorder
Filter
Expected Observation Level
Redaction Policy
```

---

# 187. QueryFilter

Ejemplo conceptual:

```php
QueryFilter::create()
    ->operation(QueryOperationType::SELECT)
    ->tenant($tenantId)
    ->transaction($transactionId);
```

---

# 188. Filters composables

Podrán combinarse:

```text
AND
OR
NOT
```

de forma limitada y determinista.

---

# 189. Filter ≠ Query Builder

No ejecuta consultas.

Sólo selecciona registros observados.

---

# 190. Assertion Cardinality

Podrá expresarse:

```text
NONE
AT_LEAST_ONE
EXACTLY_ONE
EXACTLY_N
AT_MOST_N
```

---

# 191. Example

```php
$this->assertQuery(
    expectation: $expectation,
    cardinality: QueryCardinality::exactly(1)
);
```

---

# 192. Query Match Result

Podrá contener:

```text
matched records
unmatched expectation fields
near matches
diagnostics
```

---

# 193. Near Match Diagnostics

Ejemplo:

```text
Expected:
SELECT users
tenant=42
writer

Found:
SELECT users
tenant=42
replica
```

---

# 194. Better failure messages

En lugar de:

```text
Failed asserting that false is true.
```

deberá mostrar evidencia relevante.

---

# 195. Failure diagnostic example

```text
Expected:
1 SELECT on users
using WRITER

Observed:
2 queries

#1 SELECT users
role=REPLICA

#2 SELECT profiles
role=REPLICA
```

---

# 196. SQL diff

Cuando la assertion sea SQL:

```text
Expected SQL
Actual SQL
Token Diff
```

---

# 197. AST diff

Cuando sea AST:

```text
Expected Predicate:
active = ?

Actual:
status = ?
```

---

# 198. Binding diff

Ejemplo:

```text
Expected:
id INTEGER = 42

Actual:
id STRING = "42"
```

---

# 199. Security diagnostics

Los diffs deberán aplicar redacción.

---

# 200. Assertion DSL

Podrá existir una API fluida:

```php
$this->databaseQueries()
    ->select()
    ->from('users')
    ->where('active', true)
    ->usingWriter()
    ->exactly(1)
    ->assert();
```

---

# 201. DSL design

Deberá mantenerse:

```text
typed
discoverable
IDE-friendly
composable
```

---

# 202. Laravel-like DX

Podrán ofrecerse helpers simples:

```php
assertDatabaseQueryCount(2);
assertNoDatabaseQueries();
```

sin sacrificar el modelo interno tipado.

---

# 203. Advanced API

Para casos complejos:

```php
QueryAssertion::for($recorder)
    ->matching($filter)
    ->expect($expectation)
    ->assert();
```

---

# 204. PHPUnit Integration

Podrán proporcionarse traits:

```php
use InteractsWithDatabaseQueries;
```

---

# 205. Framework-independent core

El motor de assertions no deberá depender conceptualmente de PHPUnit.

---

# 206. Adapter

```text
Query Assertion Core
        │
        ├── PHPUnit Adapter
        └── future testing adapters
```

---

# 207. Query Capture Block

API posible:

```php
$queries = $this->captureDatabaseQueries(function () {
    // operation
});
```

---

# 208. Capture result

```text
QueryCapture
├── records
├── duration
├── scope
└── failures
```

---

# 209. Nested capture

Deberá definirse cuidadosamente.

Posibles políticas:

```text
INHERIT
ISOLATE
REJECT
```

---

# 210. Default nested capture

Podrá heredar y crear sub-scope lógico sin duplicar records físicamente.

---

# 211. Query Capture ≠ Transaction

Capturar queries no cambia semántica transaccional.

---

# 212. Async/Concurrent capture

Cada actor/coroutine deberá conservar contexto propio.

---

# 213. Correlation

Los registros podrán incluir:

```text
operation id
actor id
coroutine id
```

cuando aplique.

---

# 214. No dependence on thread-local only

PHP persistent runtimes pueden requerir context propagation explícita.

---

# 215. FrankenPHP

Cada request/operation de test deberá tener recorder aislado.

---

# 216. RoadRunner

Cada worker deberá limpiar recorder entre jobs/requests.

---

# 217. OpenSwoole

El recorder deberá ser coroutine-safe.

---

# 218. Background Jobs

Una query ejecutada en otro job no pertenecerá automáticamente al capture del request original.

---

# 219. Cross-process assertions

Requerirán un collector explícito.

---

# 220. Local recorder default

El sistema base será in-process.

---

# 221. Test Query Events

El Query Assertion System podrá suscribirse a eventos internos.

---

# 222. Preferred instrumentation

La instrumentación deberá ubicarse en fronteras estables.

Ejemplo:

```text
Query Executor
```

es mejor punto para execution recording que parchear PDO directamente en todos los tests.

---

# 223. Compiler recording

Se realizará en Compiler pipeline cuando sea necesario.

---

# 224. Planner recording

Igualmente en Planner.

---

# 225. No production semantic changes

Activar query assertions no deberá cambiar:

```text
routing
optimization
transaction behavior
cache semantics
```

---

# 226. Observation must be passive

Principio:

```text
Observed Execution
≈
Unobserved Execution
```

salvo overhead razonable.

---

# 227. Recorder failure

Un error interno del recorder en testing deberá producir un error de infraestructura claro.

---

# 228. Production recorder

Normalmente no estará habilitado como Query Assertion Recorder.

---

# 229. Query Assertion Exceptions

```text
DatabaseQueryAssertionException
├── QueryNotObservedException
├── UnexpectedQueryException
├── QueryCountAssertionException
├── QueryStructureAssertionException
├── QueryAstAssertionException
├── QueryPlanAssertionException
├── CompiledSqlAssertionException
├── QueryBindingAssertionException
├── QueryContextAssertionException
├── QueryOrderingAssertionException
├── QuerySecurityAssertionException
├── QueryPerformanceAssertionException
└── QueryRecorderException
```

---

# 230. Proposed namespace

```text
src/Quantum/Database/Testing/Query/
├── Contract/
│   ├── QueryRecorder.php
│   ├── QueryObserver.php
│   └── QueryAssertionEngine.php
│
├── Model/
│   ├── QueryTestRecord.php
│   ├── QueryCapture.php
│   ├── QueryObservationLevel.php
│   ├── QueryRecordingMode.php
│   ├── QueryCardinality.php
│   └── QueryMatchResult.php
│
├── Recorder/
│   ├── DefaultQueryRecorder.php
│   ├── ScopedQueryRecorder.php
│   └── QueryRecorderScope.php
│
├── Observation/
│   ├── QueryModelObserver.php
│   ├── QueryAstObserver.php
│   ├── QueryPlanObserver.php
│   ├── QueryCompilerObserver.php
│   └── QueryExecutionObserver.php
│
├── Expectation/
│   ├── QueryExpectation.php
│   ├── AstExpectation.php
│   ├── QueryPlanExpectation.php
│   ├── BindingExpectation.php
│   └── QueryContextExpectation.php
│
├── Filter/
│   ├── QueryFilter.php
│   └── QueryFilterEvaluator.php
│
├── Assertion/
│   ├── QueryAssertion.php
│   ├── QueryCountAssertion.php
│   ├── QueryAbsenceAssertion.php
│   ├── QueryAstAssertion.php
│   ├── QueryPlanAssertion.php
│   ├── CompiledSqlAssertion.php
│   ├── BindingAssertion.php
│   ├── RoutingAssertion.php
│   ├── TenantAssertion.php
│   ├── ShardAssertion.php
│   ├── SecurityAssertion.php
│   └── PerformanceAssertion.php
│
├── Match/
│   ├── QueryMatcher.php
│   ├── AstMatcher.php
│   ├── PlanMatcher.php
│   └── BindingMatcher.php
│
├── Diagnostic/
│   ├── QueryAssertionDiagnostic.php
│   ├── QueryDiff.php
│   └── QueryNearMatchFinder.php
│
├── Snapshot/
│   ├── QuerySnapshotSerializer.php
│   └── SqlSnapshotNormalizer.php
│
├── Integration/
│   └── PHPUnit/
│       └── InteractsWithDatabaseQueries.php
│
└── Exception/
    └── ...
```

---

# 231. Test Query Snapshot

Los snapshots deberán contener sólo información estable.

Evitar:

```text
random execution ID
wall-clock timestamp
memory address
connection object hash
```

salvo que sean normalizados.

---

# 232. Stable Snapshot

Podrá contener:

```text
query type
semantic source
AST
logical plan
compiled SQL
binding types
```

según nivel.

---

# 233. Query IDs

IDs efímeros deberán reemplazarse por aliases estables en snapshots.

---

# 234. Golden Tests

El Compiler podrá utilizar archivos golden para múltiples dialectos.

Ejemplo:

```text
select_basic.mysql.sql
select_basic.mariadb.sql
select_basic.postgresql.sql
select_basic.sqlite.sql
```

---

# 235. Golden file caution

No deberá usarse golden testing para toda lógica de alto nivel.

---

# 236. Cross-version SQL

Si una capability cambia con versión:

```text
PostgreSQL X
PostgreSQL Y
```

el snapshot deberá declarar la capability profile relevante, no depender sólo del número de versión.

---

# 237. Query Assertion and Capability System

```text
CapabilitySnapshot
      ↓
Planner
      ↓
Compiler
      ↓
QueryTestRecord
      ↓
Assertion
```

---

# 238. Assert fallback

Podrá comprobarse que:

```text
feature unsupported
→ fallback strategy selected
```

---

# 239. Assert no semantic downgrade

Si una capability requerida para correctness falta:

```text
planner rejects
```

en lugar de producir SQL degradado silenciosamente.

---

# 240. Query Assertion and Resilience

Podrá comprobar:

```text
retry occurred
failover occurred
connection discarded
circuit breaker prevented execution
```

---

# 241. Circuit breaker

Si el breaker rechaza antes de DB:

```text
assertNoDatabaseQueries()
```

puede ser apropiado.

---

# 242. Query Assertion and Health

Health checks pueden ejecutar queries especiales.

El recorder deberá permitir excluir:

```text
system queries
health queries
diagnostic queries
```

de assertions de aplicación.

---

# 243. Query Origin

Cada record podrá tener:

```text
APPLICATION
ORM
MIGRATION
HEALTH
DIAGNOSTIC
ADMINISTRATION
INTERNAL
```

---

# 244. Origin filtering

Ejemplo:

```php
$this->assertDatabaseQueryCount(
    3,
    QueryFilter::application()
);
```

---

# 245. Internal Query

No deberá ocultarse completamente.

Debe poder observarse cuando sea necesario.

---

# 246. Query Purpose

Opcionalmente:

```text
ENTITY_LOAD
RELATION_LOAD
PAGINATION_DATA
PAGINATION_COUNT
LOCK
HEALTH_CHECK
MIGRATION
```

---

# 247. Purpose ≠ SQL comment

Deberá viajar como metadata interna, no depender de comentarios SQL.

---

# 248. Query Source Location

En modo debug/testing podrá capturarse:

```text
test
application callsite
repository
relationship
```

de forma limitada.

---

# 249. Source capture cost

Deberá ser opcional por su overhead.

---

# 250. N+1 diagnostics

Source location puede mejorar:

```text
N+1 detected at UserController.php
```

pero no forma parte de correctness.

---

# 251. Assertion Safety

Una assertion nunca deberá ejecutar una consulta adicional inadvertidamente para comprobar otra consulta, salvo que sea explícitamente una DB state assertion.

---

# 252. Example

Incorrecto:

```text
assert query executed
→ run another SELECT internally
```

---

# 253. Passive evidence first

Query assertions utilizarán principalmente evidencia ya registrada.

---

# 254. Database State Assertions

Serán un sistema distinto/complementario.

---

# 255. Query assertion ≠ database row assertion

```text
assertDatabaseHas()
```

y:

```text
assertQueryExecuted()
```

responden preguntas distintas.

---

# 256. Invariantes

## DB-QASSERT-001

Query Assertion no será Query Execution.

## DB-QASSERT-002

Query Assertion no será Query Profiler.

## DB-QASSERT-003

Query Assertion no será Telemetry.

## DB-QASSERT-004

Query Assertion no reimplementará N+1 detection.

## DB-QASSERT-005

Las assertions usarán la capa semántica más alta suficiente.

## DB-QASSERT-006

SQL textual no será el default para lógica de alto nivel.

## DB-QASSERT-007

SQL exacto será válido para Compiler tests.

## DB-QASSERT-008

Query Recorder será scope-local.

## DB-QASSERT-009

No existirá recorder global mutable compartido.

## DB-QASSERT-010

Parallel tests tendrán recorders aislados.

## DB-QASSERT-011

Persistent workers resetearán recorder state.

## DB-QASSERT-012

OpenSwoole mantendrá aislamiento por coroutine.

## DB-QASSERT-013

Recording mode será configurable.

## DB-QASSERT-014

Recording no capturará información innecesaria.

## DB-QASSERT-015

Query fingerprint no será necesariamente SQL hash.

## DB-QASSERT-016

Query count no demostrará N+1.

## DB-QASSERT-017

No-query assertions podrán limitarse a callbacks.

## DB-QASSERT-018

QueryExpectation no será Query Builder.

## DB-QASSERT-019

Partial matching será soportado.

## DB-QASSERT-020

Exact matching será explícito.

## DB-QASSERT-021

AST assertions serán tipadas.

## DB-QASSERT-022

Structural equality será distinta de semantic equivalence.

## DB-QASSERT-023

Semantic assertions podrán comprobar symbol resolution.

## DB-QASSERT-024

Semantic assertions podrán comprobar type inference.

## DB-QASSERT-025

Tenant constraint podrá verificarse semánticamente.

## DB-QASSERT-026

Soft-delete constraint podrá verificarse semánticamente.

## DB-QASSERT-027

Authorization scope podrá verificarse antes de ejecución.

## DB-QASSERT-028

Logical Plan será distinto de DB execution plan.

## DB-QASSERT-029

VoltStack Physical Plan será distinto de DB EXPLAIN.

## DB-QASSERT-030

Optimizer assertions no dependerán innecesariamente del SQL.

## DB-QASSERT-031

SQL normalization no cambiará semántica.

## DB-QASSERT-032

Dialect-aware SQL assertions serán soportadas.

## DB-QASSERT-033

Cross-platform semantic tests permitirán SQL distinto.

## DB-QASSERT-034

Snapshot serialization será determinista.

## DB-QASSERT-035

Snapshot changes requerirán revisión.

## DB-QASSERT-036

Binding metadata será observable.

## DB-QASSERT-037

Sensitive binding values estarán redactados por defecto.

## DB-QASSERT-038

Binding order podrá verificarse.

## DB-QASSERT-039

Logical type será distinto de driver binding type.

## DB-QASSERT-040

Query Context será observable.

## DB-QASSERT-041

Replica eligibility será distinta de actual replica execution.

## DB-QASSERT-042

Transaction identity podrá observarse.

## DB-QASSERT-043

Transaction retry attempts serán distinguibles.

## DB-QASSERT-044

Tenant Context será distinto de Tenant Predicate.

## DB-QASSERT-045

Database-per-tenant podrá comprobar logical database.

## DB-QASSERT-046

Schema-per-tenant podrá comprobar schema.

## DB-QASSERT-047

Shard route será observable.

## DB-QASSERT-048

Fan-out deberá poder detectarse.

## DB-QASSERT-049

Ordering sólo se verificará cuando sea contrato.

## DB-QASSERT-050

Partial ordering será soportado para concurrencia.

## DB-QASSERT-051

Query groups podrán asociarse a flush/load/pagination.

## DB-QASSERT-052

IdentityMap hit podrá comprobarse mediante ausencia de query.

## DB-QASSERT-053

Lazy loading podrá medirse por query capture.

## DB-QASSERT-054

Eager loading no significará JOIN.

## DB-QASSERT-055

Batch loading podrá verificarse semánticamente.

## DB-QASSERT-056

N+1 assertion consumirá detector semántico.

## DB-QASSERT-057

N+1 no se inferirá sólo por threshold.

## DB-QASSERT-058

N+1 confidence podrá formar parte de la assertion.

## DB-QASSERT-059

Slow query assertion usará policy explícita.

## DB-QASSERT-060

Unit slow-query tests podrán usar FakeClock.

## DB-QASSERT-061

Real performance thresholds pertenecerán a Performance Testing.

## DB-QASSERT-062

Cache layers serán distinguibles.

## DB-QASSERT-063

Compiled Query Cache hit no implicará cero DB executions.

## DB-QASSERT-064

Result Cache hit sí podrá evitar ejecución DB.

## DB-QASSERT-065

Prepared statement reuse no se asumirá universalmente.

## DB-QASSERT-066

Retry count será observable.

## DB-QASSERT-067

Statement retry será distinto de transaction retry.

## DB-QASSERT-068

Security assertions podrán comprobar parameterization.

## DB-QASSERT-069

User values no deberán convertirse accidentalmente en estructura SQL.

## DB-QASSERT-070

Raw expressions serán observables.

## DB-QASSERT-071

Identifier validation podrá comprobarse.

## DB-QASSERT-072

Diagnostics aplicarán redaction.

## DB-QASSERT-073

Audit será distinto de telemetry.

## DB-QASSERT-074

Event assertions sólo comprobarán ordering cuando sea contrato.

## DB-QASSERT-075

Higher layers preferirán normalized errors.

## DB-QASSERT-076

UNKNOWN query outcome permanecerá UNKNOWN.

## DB-QASSERT-077

Timeout source será distinguible.

## DB-QASSERT-078

Query assertions no sustituirán entity assertions.

## DB-QASSERT-079

Affected rows usarán semántica normalizada.

## DB-QASSERT-080

Streaming assertions podrán comprobar cursor lifecycle.

## DB-QASSERT-081

Pagination assertions podrán comprobar ordering estable.

## DB-QASSERT-082

Cursor pagination assertions usarán boundary semántico.

## DB-QASSERT-083

Count Query será distinto de Data Query.

## DB-QASSERT-084

Full-text assertions podrán operar antes del Compiler.

## DB-QASSERT-085

JSON assertions podrán operar sobre JSON AST.

## DB-QASSERT-086

Spatial assertions preservarán SRID/CRS semantics.

## DB-QASSERT-087

Capability assertions no dependerán de vendor name.

## DB-QASSERT-088

Required capability failure podrá verificarse.

## DB-QASSERT-089

Fallback strategy podrá verificarse.

## DB-QASSERT-090

Semantic downgrade silencioso deberá ser detectable.

## DB-QASSERT-091

Query filters no ejecutarán consultas.

## DB-QASSERT-092

Cardinality será explícita.

## DB-QASSERT-093

Near-match diagnostics estarán disponibles.

## DB-QASSERT-094

Failure diagnostics serán estructurados.

## DB-QASSERT-095

Failure diagnostics no expondrán secrets.

## DB-QASSERT-096

Assertion DSL será typed e IDE-friendly.

## DB-QASSERT-097

Simple helpers coexistirán con API avanzada.

## DB-QASSERT-098

Assertion Core no dependerá obligatoriamente de PHPUnit.

## DB-QASSERT-099

PHPUnit será adapter.

## DB-QASSERT-100

Query capture no será Transaction.

## DB-QASSERT-101

Nested captures tendrán policy definida.

## DB-QASSERT-102

Async captures tendrán contexto explícito.

## DB-QASSERT-103

Cross-process capture requerirá collector explícito.

## DB-QASSERT-104

Instrumentation utilizará fronteras estables.

## DB-QASSERT-105

Activar assertions no cambiará query semantics.

## DB-QASSERT-106

Observation será pasiva.

## DB-QASSERT-107

Recorder failure será infrastructure failure.

## DB-QASSERT-108

Golden SQL tests se reservarán principalmente al Compiler.

## DB-QASSERT-109

Capability profile será preferible a vendor/version assumptions.

## DB-QASSERT-110

Health queries podrán filtrarse por origin.

## DB-QASSERT-111

Diagnostic queries podrán filtrarse por origin.

## DB-QASSERT-112

Internal queries seguirán siendo observables.

## DB-QASSERT-113

Query purpose será metadata interna.

## DB-QASSERT-114

Query purpose no dependerá de SQL comments.

## DB-QASSERT-115

Source location capture será opcional.

## DB-QASSERT-116

Query assertion no ejecutará DB queries ocultas.

## DB-QASSERT-117

Query assertion será distinta de Database State Assertion.

## DB-QASSERT-118

Same semantic query podrá generar SQL diferente por plataforma.

## DB-QASSERT-119

Unsupported capability será distinta de UNKNOWN.

## DB-QASSERT-120

UNKNOWN capability no será tratado automáticamente como unsupported.

## DB-QASSERT-121

QueryTestRecord será immutable snapshot.

## DB-QASSERT-122

Query records no conservarán referencias mutables a AST/Plan activos.

## DB-QASSERT-123

Query execution IDs efímeros no contaminarán snapshots.

## DB-QASSERT-124

Binding snapshots no almacenarán secretos sin opt-in explícito.

## DB-QASSERT-125

Recorder reset eliminará estado del test anterior.

## DB-QASSERT-126

Recorder reset failure impedirá reuse.

## DB-QASSERT-127

Worker-global query records estarán prohibidos.

## DB-QASSERT-128

Coroutine-global mutable recorder estará prohibido.

## DB-QASSERT-129

Query assertion matching será determinista.

## DB-QASSERT-130

Matching no dependerá de object identity cuando exista identidad semántica.

## DB-QASSERT-131

Query fingerprint deberá ser estable para su generation contract.

## DB-QASSERT-132

Fingerprint version podrá cambiar de forma controlada.

## DB-QASSERT-133

Snapshot format tendrá versionado.

## DB-QASSERT-134

Compiler snapshots declararán dialect/capability profile.

## DB-QASSERT-135

Query origin podrá formar parte de filtros.

## DB-QASSERT-136

System queries no alterarán counts de aplicación cuando sean excluidas explícitamente.

## DB-QASSERT-137

No-query assertion deberá declarar su scope.

## DB-QASSERT-138

Query ordering tendrá sequence metadata.

## DB-QASSERT-139

Concurrent execution no impondrá orden total artificial.

## DB-QASSERT-140

Assertions de seguridad operarán antes del redaction cuando necesiten verificar clasificación, pero diagnostics permanecerán redactados.

## DB-QASSERT-141

Test instrumentation no podrá modificar bindings.

## DB-QASSERT-142

Test instrumentation no podrá modificar Query AST.

## DB-QASSERT-143

Test instrumentation no podrá modificar routing.

## DB-QASSERT-144

Test instrumentation no podrá modificar transaction state.

## DB-QASSERT-145

Test instrumentation no podrá modificar cache policy.

## DB-QASSERT-146

Query assertion helpers serán seguros para workers persistentes.

## DB-QASSERT-147

Test records tendrán bounded lifecycle.

## DB-QASSERT-148

Large test suites no acumularán records indefinidamente.

## DB-QASSERT-149

Assertion failures conservarán evidencia suficiente para reproducción.

## DB-QASSERT-150

El Query Assertion System verificará consultas sin convertirse en una segunda implementación del Query Engine.

---

# 257. Anti-patrones

## 257.1 Comparar SQL para todo

Incorrecto:

```php
assertSame(
    'SELECT * FROM users WHERE active = ?',
    $sql
);
```

si la propiedad real es simplemente:

```text
active users are filtered
```

---

## 257.2 Contar queries para detectar N+1

```text
queries > 10
→ N+1
```

es insuficiente.

---

## 257.3 Recorder global

Produce contaminación entre tests.

---

## 257.4 Guardar bindings sensibles

Especialmente:

```text
password
token
API key
credentials
```

---

## 257.5 Confundir eager loading con JOIN

VoltStack puede elegir:

```text
SELECT_IN
BATCH
JOIN
```

---

## 257.6 Query assertion que ejecuta otra query

Cambia el comportamiento observado.

---

## 257.7 Assertions basadas en vendor

```text
if PostgreSQL then...
```

cuando la propiedad es capability-based.

---

## 257.8 Exigir orden total en concurrencia

Puede producir tests frágiles.

---

## 257.9 Snapshot de IDs efímeros

Genera cambios irrelevantes.

---

## 257.10 SQL normalization agresiva

Puede esconder diferencias semánticas reales.

---

## 257.11 Usar QueryRecorder como profiler productivo

Son responsabilidades diferentes.

---

## 257.12 QueryExpectation como segundo Query Builder

Duplicaría el Query Engine.

---

## 257.13 Confundir cache compile hit con cero queries

El DB todavía puede ejecutarse.

---

## 257.14 Tratar UNKNOWN como failure

Pierde información crítica.

---

## 257.15 Ocultar queries internas completamente

Dificulta diagnosticar comportamiento.

---

# 258. Estrategia de testing del propio sistema

El Query Assertion System deberá probarse en varias capas.

### Unit Tests

```text
QueryMatcher
AstMatcher
BindingMatcher
QueryFilter
Cardinality
SQL normalization
diagnostics
```

### Integration Tests

```text
Query Builder
→ Compiler
→ Executor
→ Recorder
→ Assertion
```

### Platform Tests

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

### Persistent Runtime Tests

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 259. Ejemplo — Query semántica

```php
$users = User::query()
    ->where('active', true)
    ->get();

$this->assertQueryExecuted(
    QueryExpectation::select('users')
        ->where('active', '=', true)
);
```

La assertion no necesita conocer:

```text
identifier quoting
placeholder syntax
dialect formatting
```

---

# 260. Ejemplo — Compiler

```php
$query = Query::select('users')
    ->where('id', '=', Parameter::named('id'));

$compiled = $compiler->compile($query);

$this->assertCompiledSql(
    'SELECT "users".* FROM "users" WHERE "id" = ?',
    mode: SqlComparisonMode::NORMALIZED
);
```

Aquí el SQL sí es la propiedad bajo prueba.

---

# 261. Ejemplo — IdentityMap

```php
$user = $em->find(User::class, 42);

$this->assertNoDatabaseQueries(function () use ($em) {
    $em->find(User::class, 42);
});
```

Esto comprueba que la segunda resolución puede satisfacerse desde IdentityMap.

---

# 262. Ejemplo — Eager Loading

```php
$users = User::query()
    ->with('posts')
    ->get();

$this->assertRelationshipLoaded('posts');

$this->assertNoNPlusOneQueries();
```

No exige que exista un JOIN.

---

# 263. Ejemplo — Tenant

```php
$this->assertQuery(
    expectation: QueryExpectation::select('orders')
        ->tenant($tenant->id),
    cardinality: QueryCardinality::atLeastOne()
);
```

En shared-schema mode podrá comprobar además:

```text
tenant predicate
```

---

# 264. Ejemplo — Shard

```php
$this->assertQueryShard('shard-eu-03');
$this->assertSingleShardQuery();
```

---

# 265. Ejemplo — Writer

```php
$order->status = OrderStatus::PAID;
$em->flush();

$this->assertQueryExecuted(
    QueryExpectation::update('orders')
        ->usingConnectionRole(ConnectionRole::WRITER)
);
```

---

# 266. Ejemplo — Retry

```text
Attempt 1
UPDATE orders
→ deadlock

Attempt 2
UPDATE orders
→ success
```

Assertions:

```php
$this->assertQueryAttempts(2);
$this->assertTransactionAttempts(2);
```

---

# 267. Ejemplo — Security

Input:

```text
"Robert'); DROP TABLE users;--"
```

La assertion deberá demostrar que el valor fue tratado como:

```text
binding value
```

y no como:

```text
query structure
```

---

# 268. Ejemplo — Cache

Primera operación:

```text
SELECT user
→ DB
→ result cache population
```

Segunda:

```text
same operation
→ result cache hit
```

Podrá comprobarse:

```php
$this->assertNoDatabaseQueries($secondOperation);
```

si la policy garantiza ese comportamiento.

---

# 269. Modelo formal

Sea:

```text
Q = query operation
O(Q) = set of observations
E = expectation
```

La assertion:

```text
Assert(Q, E)
```

es válida cuando:

```text
Match(
    Relevant(O(Q)),
    E
)
=
true
```

---

# 270. Relevant observation

La selección deberá ser:

```text
Relevant(O(Q))
=
MinimumEvidenceRequired(E)
```

No:

```text
AllPossibleInternalState
```

---

# 271. Semantic assertion rule

Si:

```text
Property(E)
```

puede demostrarse mediante:

```text
QueryModel
```

entonces:

```text
PreferredLevel(E)
=
QueryModel
```

y no necesariamente:

```text
CompiledSQL
```

---

# 272. SQL assertion rule

Si:

```text
Property(E)
=
ExactDialectCompilation
```

entonces:

```text
PreferredLevel(E)
=
CompiledSQL
```

---

# 273. N+1 assertion rule

```text
NPlusOne
=
RepeatedSemanticPattern
+
ParentIterationCorrelation
+
Relationship/Operation Evidence
```

No:

```text
QueryCount > ArbitraryThreshold
```

---

# 274. Query Count Rule

Para scope `S` y filter `F`:

```text
Count(S,F)
=
| { q ∈ Records(S) : Match(q,F) } |
```

---

# 275. Security Rule

Una query será considerada correctamente parametrizada respecto a un input `v` cuando:

```text
v ∈ BindingValues
```

y:

```text
v ∉ UntrustedQueryStructure
```

según el modelo estructurado del Query Engine.

---

# 276. Arquitectura consolidada

```text
                    DATABASE OPERATION
                            │
                            ▼
                     Query Builder
                            │
                    ┌───────┴────────┐
                    ▼                │
                Query Model          │
                    │                │
                [Observer]           │
                    │                │
                    ▼                │
                    AST              │
                    │                │
                [Observer]           │
                    │                │
                    ▼                │
               Semantic Engine       │
                    │                │
                    ▼                │
                  Planner            │
                    │                │
                [Observer]           │
                    │                │
                    ▼                │
                 Compiler            │
                    │                │
                [Observer]           │
                    │                │
                    ▼                │
                 Executor            │
                    │                │
                [Observer]           │
                    │                │
                    ▼                │
                   DBMS              │
                    │                │
                    ▼                │
                  Outcome            │
                    │                │
                    └───────┬────────┘
                            ▼
                    QueryTestRecord
                            │
                            ▼
                     Query Recorder
                            │
                 ┌──────────┼──────────┐
                 ▼          ▼          ▼
               Filter    Matcher    Diagnostics
                 │          │          │
                 └──────────┼──────────┘
                            ▼
                     Assertion Engine
                            │
                            ▼
                    PASS / FAILURE
```

---

# 277. Regla final

> **El Database Query Assertion System deberá permitir comprobar la intención, estructura, planificación, compilación, routing y ejecución de consultas sin acoplar innecesariamente los tests a detalles internos o SQL específico de una plataforma.**

La jerarquía preferida será:

```text
Domain/Semantic Intent
        ↓
Query Model
        ↓
AST
        ↓
Semantic Graph
        ↓
Query Plan
        ↓
Compiled Query
        ↓
SQL
        ↓
Driver Execution
```

La assertion deberá colocarse:

```text
as high as possible
```

pero:

```text
as low as necessary
```

para demostrar la propiedad requerida.

En forma compacta:

```text
Query Assertion
≠
SQL Assertion
```

```text
Query Count
≠
N+1 Detection
```

```text
Eager Loading
≠
JOIN
```

```text
Tenant Context
≠
Tenant Predicate
```

```text
Replica Eligible
≠
Replica Used
```

```text
Compiled Cache Hit
≠
No Database Query
```

y:

```text
Reliable Query Testing
=
Semantic Observation
+
Scoped Recording
+
Typed Expectations
+
Capability Awareness
+
Security Redaction
+
Useful Diagnostics
+
Real Integration Evidence
```

---

# 278. Siguiente documento

```text
290_DATABASE_SCHEMA_TESTING_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Database Schema Testing System
│
├── Schema Assertion Architecture
├── Schema Model Assertions
├── Table Assertions
├── Column Assertions
├── Index Assertions
├── Foreign Key Assertions
├── Constraint Assertions
├── Schema Introspection Assertions
├── Coverage Assertions
├── Schema Diff Assertions
├── Migration Assertions
├── Schema Compiler Assertions
├── Platform Compatibility Assertions
├── Capability-aware Schema Tests
├── Destructive Change Assertions
├── Zero-Downtime Schema Assertions
├── Schema Snapshot Testing
├── Cross-Platform Schema Conformance
├── Test Database Isolation
├── Schema Reset
├── Persistent Runtime Safety
└── Diagnostics
```

manteniendo como principio:

> **Las pruebas de Schema deberán distinguir cuidadosamente entre la definición deseada, el modelo normalizado, lo observado realmente en el DBMS y las operaciones necesarias para transformar un estado en otro; que una definición compile correctamente no demuestra que el esquema real de la base de datos tenga esa estructura.**