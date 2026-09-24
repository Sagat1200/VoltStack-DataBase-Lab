# 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md

# VoltStack Quantum Database
## Query Telemetry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 217 — Query Telemetry System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `216_DATABASE_TELEMETRY_ARCHITECTURE.md`  
**Siguiente documento:** `218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md`

---

# 1. Propósito

`Query Telemetry System` define cómo VoltStack observará, medirá, correlacionará y diagnosticará el ciclo completo de una consulta sin alterar su comportamiento.

El sistema deberá observar operaciones provenientes de:

```text
Query Builder
ORM
Repositories
Model API
Raw SQL
Relationship Loading
Pagination
Cursor Pagination
Chunk Processing
Lazy Collections
Bulk Operations
Import / Export
Internal Framework Queries
```

La regla central será:

> **Query Telemetry describe cómo una consulta fue preparada, planificada, compilada, ejecutada y consumida; nunca modifica su AST, parámetros, plan, routing, SQL, resultado u outcome.**

Formalmente:

```text
Query(Q)
→ Result(Q)

Telemetry(Q)
→ Observation(Query(Q))
```

pero nunca:

```text
Telemetry(Q)
→ Mutate(Query(Q))
```

---

# 2. Relación con Database Telemetry Architecture

El documento anterior definió:

```text
Database Telemetry
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry
├── ORM Telemetry
├── Query Profiler
├── Slow Query Detection
├── N+1 Telemetry
└── Debug Information
```

Este documento desarrolla:

```text
Query Telemetry
```

como especialización de esa arquitectura.

---

# 3. Query lifecycle observable

El ciclo completo puede ser:

```text
Application
    │
    ▼
Query Builder / ORM
    │
    ▼
Query Model / AST
    │
    ▼
Normalization
    │
    ▼
Semantic Analysis
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
Prepared Statement
    │
    ▼
Parameter Binding
    │
    ▼
Routing
    │
    ▼
Connection Acquisition
    │
    ▼
Execution
    │
    ▼
Result Cursor
    │
    ▼
Result Consumption
    │
    ▼
Hydration
    │
    ▼
Application Result
```

No todas las consultas atravesarán todas las fases.

---

# 4. Principio de fases opcionales

Query Telemetry no deberá asumir:

```text
Every Query
=
ORM
+
AST
+
Optimizer
+
Hydration
```

Por ejemplo, raw SQL puede recorrer:

```text
Raw SQL
→ Validation
→ Execution
→ Result
```

mientras una consulta ORM puede recorrer prácticamente todo el pipeline.

---

# 5. Distinciones fundamentales

```text
Query Telemetry
≠
Query Event

Query Telemetry
≠
Query Profiler

Query Telemetry
≠
Slow Query Detection

Query Telemetry
≠
Query Log

Query Telemetry
≠
Query Cache

Query Telemetry
≠
Query Optimizer

Query Telemetry
≠
Query Planner

Query Telemetry
≠
Query Debugger

Query Telemetry
≠
Audit Log
```

---

# 6. Query execution ≠ query definition

Una misma definición:

```php
User::query()
    ->where('active', true);
```

puede ejecutarse múltiples veces.

Por tanto:

```text
QueryDefinitionId
≠
QueryExecutionId
```

---

# 7. QueryId

Cada ejecución tendrá un identificador:

```php
final readonly class QueryId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 8. QueryId semantics

`QueryId` representa:

```text
one concrete execution attempt
```

No representa:

```text
query shape
SQL statement
business operation
transaction
request
```

---

# 9. QueryFingerprint

La identidad estructural será:

```php
final readonly class QueryFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 10. QueryId ≠ QueryFingerprint

Ejemplo:

```text
QueryId:
q_001
q_002
q_003

QueryFingerprint:
qf_users_by_id
```

para tres ejecuciones:

```sql
WHERE id = 10
WHERE id = 20
WHERE id = 30
```

---

# 11. Fingerprint source

Prioridad:

```text
Semantic Query Model
↓
Normalized AST
↓
Compiled Query Structure
↓
Normalized Raw SQL
```

---

# 12. Semantic fingerprint

Para consultas generadas por VoltStack deberá preferirse:

```text
SemanticFingerprint(QueryModel)
```

sobre:

```text
Hash(SQL)
```

---

# 13. Why semantic fingerprint

Dos dialectos pueden producir:

```text
same logical query
```

con SQL diferente.

Ejemplo conceptual:

```text
LogicalQueryFingerprint
=
same

MySQL SQL
≠
PostgreSQL SQL
```

---

# 14. SQL fingerprint

También podrá existir:

```text
SqlFingerprint
```

para análisis de compilación y plataforma.

---

# 15. Multiple fingerprints

Una ejecución podrá tener:

```text
SemanticQueryFingerprint
CompiledSqlFingerprint
ExecutionFingerprint
```

según las fases disponibles.

---

# 16. Fingerprint ≠ security boundary

Un fingerprint:

```text
does not authorize
does not authenticate
does not prove safety
```

---

# 17. QueryTelemetryContext

```php
final readonly class QueryTelemetryContext
{
    public function __construct(
        public QueryId $queryId,
        public ?QueryFingerprint $queryFingerprint,
        public ?OperationId $operationId,
        public ?TransactionId $transactionId,
        public ?ConnectionId $connectionId,
        public ?PersistenceDomainId $domainId,
        public ?TenantId $tenantId,
        public ?ShardId $shardId,
        public QueryOperation $operation,
        public QueryOrigin $origin,
        public QueryTelemetryPolicy $policy,
    ) {}
}
```

---

# 18. Query origin

```php
enum QueryOrigin
{
    case QUERY_BUILDER;
    case ORM;
    case MODEL_API;
    case REPOSITORY;
    case RELATIONSHIP;
    case PAGINATION;
    case CHUNK;
    case LAZY_COLLECTION;
    case BULK_OPERATION;
    case IMPORT;
    case EXPORT;
    case RAW_SQL;
    case FRAMEWORK_INTERNAL;
    case UNKNOWN;
}
```

---

# 19. Query operation

```php
enum QueryOperation
{
    case SELECT;
    case INSERT;
    case UPDATE;
    case DELETE;
    case UPSERT;
    case DDL;
    case PROCEDURE;
    case EXPLAIN;
    case OTHER;
    case UNKNOWN;
}
```

---

# 20. Operation classification

La clasificación deberá derivarse de información estructural cuando exista.

No depender de:

```php
str_starts_with($sql, 'SELECT')
```

como estrategia principal.

---

# 21. Raw SQL classification

Para raw SQL podrá existir un classifier limitado.

Si no puede determinarse:

```text
operation = UNKNOWN
```

---

# 22. UNKNOWN ≠ SELECT

Nunca asumir:

```text
UNKNOWN
=
read-only
```

---

# 23. Query telemetry scope

Cada ejecución utilizará un scope:

```php
interface QueryTelemetryScope
{
    public function queryId(): QueryId;

    public function phase(QueryTelemetryPhase $phase): QueryPhaseScope;

    public function finish(QueryOutcome $outcome): void;
}
```

---

# 24. Lifecycle

```text
CREATED
  │
  ▼
PREPARING
  │
  ▼
EXECUTING
  │
  ▼
CONSUMING
  │
  ▼
COMPLETED
```

con ramas:

```text
FAILED
CANCELLED
UNKNOWN
```

---

# 25. Query telemetry states

```php
enum QueryTelemetryState
{
    case CREATED;
    case PREPARING;
    case ROUTING;
    case WAITING_CONNECTION;
    case EXECUTING;
    case CONSUMING;
    case HYDRATING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 26. State machine

Transiciones inválidas deberán ser diagnosticables.

Ejemplo:

```text
COMPLETED
→ EXECUTING
```

no es válido.

---

# 27. Query phases

```php
enum QueryTelemetryPhase
{
    case BUILD;
    case NORMALIZATION;
    case VALIDATION;
    case SEMANTIC_ANALYSIS;
    case OPTIMIZATION;
    case PLANNING;
    case COMPILATION;
    case PREPARATION;
    case BINDING;
    case ROUTING;
    case CONNECTION_ACQUISITION;
    case EXECUTION;
    case FIRST_RESULT;
    case RESULT_CONSUMPTION;
    case HYDRATION;
}
```

---

# 28. Not every phase exists

Una fase ausente deberá ser:

```text
NOT_APPLICABLE
```

o simplemente no registrada.

Nunca:

```text
duration = 0
```

para fingir que ocurrió.

---

# 29. Phase timing

```php
final readonly class QueryPhaseTiming
{
    public function __construct(
        public QueryTelemetryPhase $phase,
        public Duration $duration,
    ) {}
}
```

---

# 30. Monotonic timing

Toda duración deberá derivarse de:

```text
monotonic clock
```

cuando esté disponible.

---

# 31. Query total duration

Se distinguirán varias duraciones.

```text
query.pipeline.duration
query.execution.duration
query.result_consumption.duration
query.hydration.duration
```

---

# 32. Pipeline duration

Puede cubrir:

```text
Query preparation
→ final application result
```

---

# 33. Execution duration

Debe representar el boundary real del Executor.

---

# 34. Execution duration ≠ total duration

```text
Execution Duration
≠
Query Pipeline Duration
```

---

# 35. Streaming query

Para streaming:

```text
execution start
↓
first result
↓
stream consumption
↓
cursor close
```

puede durar mucho más que la ejecución inicial.

---

# 36. Time to first result

Métrica:

```text
db.query.time_to_first_result
```

podrá ser especialmente útil para streaming.

---

# 37. Result consumption duration

```text
db.query.result_consumption.duration
```

medirá el tiempo durante el cual el consumidor procesa el Result Cursor cuando sea observable.

---

# 38. Consumer time caveat

En streaming:

```text
Result Consumption Duration
```

puede incluir tiempo que la aplicación pasa procesando cada elemento.

Por tanto:

```text
ResultConsumptionDuration
≠
PureDatabaseTime
```

---

# 39. Driver execution duration

Podrá existir:

```text
db.query.driver.duration
```

cuando el driver pueda medirlo de forma confiable.

---

# 40. Database server duration

No deberá llamarse:

```text
server_execution_time
```

a menos que provenga realmente del servidor.

---

# 41. Client measurement ≠ server measurement

```text
ClientObservedDuration
≠
DatabaseServerExecutionDuration
```

---

# 42. Query metrics

Métricas base:

```text
db.query.executions
db.query.duration
db.query.failures
db.query.cancellations
db.query.rows_returned
db.query.rows_affected
```

---

# 43. Optional metrics

```text
db.query.compilation.duration
db.query.connection_wait.duration
db.query.result_consumption.duration
db.query.hydration.duration
db.query.time_to_first_result
db.query.retries
```

---

# 44. Query execution counter

Conceptualmente:

```text
db.query.executions{
    db.system,
    db.operation,
    status,
    connection.role
}
```

---

# 45. High-cardinality prohibition

No:

```text
db.query.executions{
    sql="...",
    query_id="...",
    tenant_id="..."
}
```

por default.

---

# 46. Query duration histogram

```text
db.query.duration
```

deberá ser histogram.

---

# 47. Query duration unit

Se recomienda exportar:

```text
seconds
```

de acuerdo con el provider.

Internamente:

```text
Duration
```

---

# 48. Query status

```php
enum QueryOutcome
{
    case SUCCESS;
    case FAILED;
    case CANCELLED;
    case TIMEOUT;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 49. UNKNOWN preservation

Si la capa de ejecución no puede determinar el resultado:

```text
UNKNOWN
```

debe preservarse.

---

# 50. PARTIAL

Es relevante para:

```text
distributed queries
streaming consumption
large dataset operations
```

cuando la operación produjo solo parte de los resultados.

---

# 51. Statement success ≠ consumption success

Una consulta SELECT puede ejecutarse correctamente:

```text
statement success
```

pero fallar mientras se consume el stream.

Por tanto:

```text
StatementOutcome
≠
QueryConsumptionOutcome
```

---

# 52. Query execution outcome model

Podrá distinguir:

```php
final readonly class QueryExecutionOutcome
{
    public function __construct(
        public StatementOutcome $statement,
        public ResultConsumptionOutcome $consumption,
        public QueryOutcome $overall,
    ) {}
}
```

---

# 53. Rows returned

`rows_returned` deberá representar:

```text
physical result rows observed
```

cuando se conozca.

---

# 54. Unknown row count

Si no se conoce:

```text
rows_returned = UNKNOWN
```

No:

```text
0
```

---

# 55. Rows affected

Para mutation:

```text
rows_affected
```

deberá usar la semántica reportada por la plataforma/driver.

---

# 56. Rows matched ≠ rows changed

En algunas plataformas:

```text
matched rows
≠
changed rows
```

Telemetry deberá respetar la semántica real.

---

# 57. Rows returned ≠ entities hydrated

```text
RowsReturned
≠
EntitiesHydrated
```

JOINs pueden producir múltiples rows por entidad.

---

# 58. Entities hydrated

Será principalmente responsabilidad de:

```text
ORM / Hydration Telemetry
```

aunque pueda correlacionarse con QueryId.

---

# 59. Query span

Cada query observable podrá producir:

```text
db.query
```

span.

---

# 60. Query span structure

Ejemplo:

```text
db.query
├── query.compile
├── connection.acquire
├── driver.execute
└── result.consume
```

No es obligatorio generar subspans para todas las fases.

---

# 61. Span explosion

Crear un span para cada microfase puede ser costoso.

Por default:

```text
one query span
+
phase timings as events/attributes
```

puede ser suficiente.

---

# 62. Detailed tracing mode

En profiling/debug:

```text
query
├── semantic_analysis
├── optimization
├── planning
├── compilation
├── execution
└── hydration
```

podrá habilitarse.

---

# 63. Query span attributes

Atributos bounded:

```text
db.system
db.operation
db.query.origin
db.connection.role
db.transaction.present
db.result.mode
db.query.status
```

---

# 64. Conditional attributes

Según política:

```text
db.namespace
db.shard
query.fingerprint
sql.fingerprint
```

---

# 65. Query fingerprint cardinality

Aunque un fingerprint oculta valores:

```text
QueryFingerprint
```

puede tener cardinalidad alta.

Por ello no deberá convertirse automáticamente en metric label.

---

# 66. Fingerprint in traces

Sí puede utilizarse en:

```text
trace
diagnostic record
profile
```

bajo policy.

---

# 67. Query text capture

Modos:

```php
enum QueryTextCaptureMode
{
    case NONE;
    case FINGERPRINT_ONLY;
    case SANITIZED;
    case FULL;
}
```

---

# 68. Default

Producción:

```text
FINGERPRINT_ONLY
```

o `SANITIZED`, según configuración.

---

# 69. FULL

`FULL` deberá requerir opt-in explícito.

---

# 70. SQL may contain literals

Ejemplo:

```sql
SELECT *
FROM users
WHERE email = 'private@example.com'
```

aunque no existan bindings.

Por tanto:

```text
Raw SQL
≠
Safe SQL
```

---

# 71. Query sanitizer

```php
interface QueryTelemetrySanitizer
{
    public function sanitize(
        QueryText $query,
        QueryTelemetryContext $context,
    ): SanitizedQueryText;
}
```

---

# 72. Sanitization examples

```sql
WHERE email = 'private@example.com'
```

podría representarse:

```sql
WHERE email = ?
```

---

# 73. Sanitization ≠ SQL parser guarantee

Para raw SQL complejo, si no puede sanitizarse de forma confiable:

```text
omit query text
```

es preferible.

---

# 74. Parameter capture modes

```php
enum QueryBindingCaptureMode
{
    case NONE;
    case COUNT_ONLY;
    case TYPES_ONLY;
    case REDACTED;
    case VALUES;
}
```

---

# 75. Default binding capture

```text
TYPES_ONLY
```

o:

```text
COUNT_ONLY
```

en producción.

---

# 76. VALUES mode

Requerirá:

```text
explicit security policy
+
redaction
+
diagnostic intent
```

---

# 77. Credential values

Nunca deberán capturarse.

---

# 78. Sensitive application values

Campos como:

```text
password
token
secret
credit_card
authorization
session
```

deberán clasificarse como sensibles independientemente del nombre exacto cuando metadata lo permita.

---

# 79. Parameter telemetry model

```php
final readonly class QueryParameterTelemetry
{
    public function __construct(
        public int $position,
        public TypeId $type,
        public TelemetrySensitivity $sensitivity,
        public mixed $safeRepresentation,
    ) {}
}
```

---

# 80. No arbitrary object serialization

Nunca:

```php
serialize($binding)
```

para telemetry.

---

# 81. Value Object bindings

Deberán pasar primero por:

```text
Type System
→ safe telemetry representation
```

---

# 82. Query compilation telemetry

Podrá medir:

```text
compiler duration
dialect
platform
cache hit
compiled query size
```

---

# 83. Compiled query cache

Debe distinguir:

```text
compiled_query_cache_hit
```

de:

```text
result_cache_hit
```

---

# 84. Compilation skipped

Si se reutiliza un compiled query:

```text
compilation = SKIPPED_CACHE_HIT
```

No fingir:

```text
compilation_duration = 0
```

como si hubiese compilado.

---

# 85. Query optimizer telemetry

Podrá registrar:

```text
optimization duration
rules considered
rules applied
plan transformations
```

en modo profiler/diagnostic.

---

# 86. Optimizer telemetry ≠ optimizer control

Telemetry nunca decidirá:

```text
which rule to apply
```

---

# 87. Rule names

Los nombres de reglas pueden ser bounded si el conjunto está registrado.

---

# 88. Query planner telemetry

Podrá observar:

```text
logical plan created
physical plan selected
distributed plan
shards targeted
```

---

# 89. Plan object

No deberá exportarse directamente.

Usar:

```text
safe plan representation
```

---

# 90. Plan fingerprint

Podrá existir:

```text
QueryPlanFingerprint
```

para profiling.

---

# 91. Routing telemetry

Podrá registrar:

```text
routing decision
writer/replica
shard count
routing reason
consistency requirement
```

---

# 92. Routing reason

Ejemplos bounded:

```text
TRANSACTION_PIN
LOCKING_READ
READ_YOUR_WRITES
STICKY_WRITE
REPLICA_ELIGIBLE
WRITER_REQUIRED
SHARD_KEY
GLOBAL_FANOUT
```

---

# 93. Endpoint identity

Endpoint concreto podrá aparecer en trace/diagnostic bajo policy.

No deberá ser label métrico de cardinalidad no controlada.

---

# 94. Connection wait telemetry

Query Telemetry podrá correlacionar:

```text
connection acquisition duration
```

pero el canonical owner será:

```text
Connection Telemetry System
```

---

# 95. No double recording

Query Telemetry deberá referenciar la medición de Connection Telemetry cuando corresponda.

---

# 96. Prepared statement telemetry

Podrá observar:

```text
prepared
reused
prepare duration
statement cache hit
```

si el driver lo soporta.

---

# 97. Prepared statement ≠ query cache

```text
Prepared Statement Reuse
≠
Compiled Query Cache
≠
Result Cache
```

---

# 98. Binding telemetry

Podrá medir:

```text
binding count
binding duration
binding conversion failures
```

sin registrar valores por default.

---

# 99. Type conversion

Errores de conversión deberán poder clasificarse:

```text
TYPE_CONVERSION_FAILURE
```

---

# 100. Query retry telemetry

Una operación lógica puede tener:

```text
Query Operation
├── Attempt 1
└── Attempt 2
```

---

# 101. Attempt ID

Podrá utilizarse:

```php
final readonly class QueryAttemptId
{
    public function __construct(
        public QueryId $queryId,
        public int $attempt,
    ) {}
}
```

---

# 102. QueryId retry semantics

Dos opciones arquitectónicas son posibles:

```text
same QueryId + attempt number
```

o:

```text
logical QueryOperationId
+
unique QueryId per attempt
```

VoltStack utilizará la segunda para mayor precisión.

---

# 103. Logical query operation

```php
final readonly class QueryOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 104. Retry identity

```text
QueryOperationId = qo_100

Attempt 1
QueryId = q_101

Attempt 2
QueryId = q_102
```

---

# 105. Why

Cada intento puede usar:

```text
different connection
different replica
different duration
different failure
```

---

# 106. Retry safety

Telemetry observa retries.

No decide si son seguros.

Eso pertenece a:

```text
Retry Policy
Transaction Retry System
Execution Retry System
```

---

# 107. UNKNOWN retry outcome

Nunca deberá ocultarse mediante:

```text
attempt 1 = unknown
attempt 2 = success
```

si el segundo intento no era semánticamente seguro.

La decisión de retry pertenece al sistema correspondiente.

---

# 108. Query failure categories

```php
enum QueryFailureCategory
{
    case CONNECTION;
    case TIMEOUT;
    case DEADLOCK;
    case LOCK_TIMEOUT;
    case CONSTRAINT;
    case SYNTAX;
    case TYPE_CONVERSION;
    case BINDING;
    case PERMISSION;
    case RESOURCE_EXHAUSTION;
    case CANCELLATION;
    case DRIVER;
    case DISTRIBUTED_PARTIAL;
    case UNKNOWN;
}
```

---

# 109. Vendor codes

Podrán añadirse:

```text
vendor_error_code
sql_state
```

cuando sea seguro.

---

# 110. SQLSTATE

Puede ser útil como atributo bounded.

---

# 111. Error message

No será un metric label.

---

# 112. Stack trace

No será metric attribute.

Podrá existir únicamente en diagnostic/logging bajo policy.

---

# 113. Query cancellation

Deberá distinguir:

```text
CLIENT_CANCELLED
DEADLINE_EXCEEDED
QUERY_TIMEOUT
WORKER_SHUTDOWN
CONNECTION_ABORTED
```

---

# 114. Cancellation telemetry

Ejemplo:

```text
db.query.cancellations{
    reason="deadline"
}
```

---

# 115. Timeout ≠ cancellation

Un timeout puede causar cancelación, pero deberán conservarse ambas semánticas cuando sea útil.

---

# 116. Query deadline

Podrá registrarse:

```text
deadline_configured=true
```

sin necesidad de registrar timestamp absoluto.

---

# 117. Result telemetry

Podrá observar:

```text
result type
cursor/stream mode
rows consumed
completion state
```

---

# 118. Result modes

```php
enum QueryResultMode
{
    case BUFFERED;
    case CURSOR;
    case STREAMING;
    case SCALAR;
    case NONE;
    case UNKNOWN;
}
```

---

# 119. Streaming completion

Una consulta streaming no se considerará completamente consumida hasta:

```text
cursor exhausted
or
cursor explicitly closed
```

---

# 120. Early termination

Ejemplo:

```php
foreach ($query->cursor() as $row) {
    break;
}
```

Telemetry deberá poder registrar:

```text
result_consumption = EARLY_TERMINATED
```

---

# 121. Early termination ≠ failure

No deberá registrarse automáticamente como error.

---

# 122. Result cursor leak

Si un cursor no se cierra correctamente:

```text
RESOURCE_LEAK_CANDIDATE
```

podrá producir un diagnostic.

---

# 123. Hydration correlation

Query Telemetry podrá enlazar:

```text
QueryId
↓
HydrationTelemetry
```

---

# 124. Hydration duration

El canonical owner de:

```text
hydration duration
```

será Hydration/ORM telemetry.

Query telemetry podrá incluirla en la vista agregada.

---

# 125. Query pipeline total

Ejemplo:

```text
Total = 24 ms

Semantic Analysis   1 ms
Optimization        2 ms
Planning            1 ms
Compilation         1 ms
Connection Wait     3 ms
Execution            8 ms
Result Consumption  2 ms
Hydration            6 ms
```

---

# 126. Phase overlap

No deberá asumirse que:

```text
Total = exact sum(phases)
```

para streaming o ejecución solapada.

---

# 127. Query cache telemetry

Podrá correlacionar:

```text
result cache lookup
physical hit
usable hit
miss
stale rejection
```

---

# 128. Cache hit query semantics

Si un Result Cache satisface completamente la operación:

```text
Database Statement Execution
=
SKIPPED
```

---

# 129. Query execution metric on cache hit

No deberá incrementarse:

```text
db.query.executions
```

si ninguna query llegó al DB.

En cambio podrá existir:

```text
db.query.operations
```

para la operación lógica.

---

# 130. Logical operation ≠ physical execution

Distinción:

```text
Query Operation
≠
Database Statement Execution
```

---

# 131. Metrics distinction

Podrán existir:

```text
db.query.operations
db.query.executions
```

donde:

```text
operations >= executions
```

en presencia de caches.

---

# 132. Query operation telemetry

Puede abarcar:

```text
cache lookup
compilation
routing
execution
hydration
```

---

# 133. Result cache security

Cache key o payload nunca deberán aparecer completos en telemetry si contienen información sensible.

---

# 134. Pagination correlation

Una paginación puede producir:

```text
Pagination Operation
├── Data Query
└── Count Query
```

---

# 135. Query role

```php
enum QueryRole
{
    case PRIMARY;
    case COUNT;
    case RELATIONSHIP_LOAD;
    case EAGER_LOAD;
    case LAZY_LOAD;
    case BATCH_LOAD;
    case CHECKPOINT;
    case INTERNAL;
    case OTHER;
}
```

---

# 136. Count query

Deberá poder identificarse:

```text
query.role=count
```

sin depender del SQL textual.

---

# 137. N+1 correlation

Queries de relaciones deberán transportar metadata suficiente para que:

```text
NPlusOneTelemetrySystem
```

pueda correlacionarlas semánticamente.

---

# 138. Relationship identity

Podrá incluir:

```text
RelationshipId
```

en diagnostics/traces.

No necesariamente como metric label.

---

# 139. Chunk processing

Un traversal:

```text
ChunkOperation
├── Query 1
├── Query 2
├── Query 3
└── Query N
```

deberá correlacionar todas las queries con:

```text
ChunkOperationId
```

---

# 140. Chunk index

`chunk_index` puede ser diagnostic attribute.

No debe generar series métricas por índice.

---

# 141. Lazy Collection

Una Lazy Collection puede ejecutar consultas en momentos distintos.

El QueryTelemetryContext deberá asociarlas con:

```text
LazyTraversalId
```

si existe.

---

# 142. Bulk operations

Un Bulk Insert puede producir:

```text
1 statement
```

o:

```text
N statements
```

según plataforma/límites.

Query Telemetry observará statements.

Bulk Telemetry observará la operación lógica.

---

# 143. Import operations

```text
ImportOperationId
↓
BulkBatchId
↓
QueryOperationId
↓
QueryId
```

permite correlación jerárquica.

---

# 144. Export operations

Podrá correlacionarse:

```text
ExportOperationId
↓
QueryId
↓
Result Cursor
```

---

# 145. Internal framework queries

VoltStack deberá identificar consultas internas.

Ejemplos:

```text
migration repository
schema introspection
health checks
metadata lookup
```

---

# 146. Internal query visibility

No deberán ocultarse necesariamente.

Pero podrán:

```text
exclude from user query count
```

según el consumidor.

---

# 147. Internal ≠ irrelevant

Una query interna lenta puede afectar seriamente rendimiento.

---

# 148. Query category

Podrá existir:

```php
enum QueryCategory
{
    case APPLICATION;
    case FRAMEWORK;
    case SCHEMA;
    case MIGRATION;
    case HEALTH;
    case MAINTENANCE;
}
```

---

# 149. Schema queries

Schema Introspection puede producir consultas especiales que deben diferenciarse de application queries.

---

# 150. Distributed query telemetry

Una consulta lógica puede producir:

```text
Distributed Query
├── Shard A / Query A
├── Shard B / Query B
└── Shard C / Query C
```

---

# 151. DistributedQueryId

```php
final readonly class DistributedQueryId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 152. Parent-child relationship

```text
DistributedQueryId
      │
      ├── QueryId A
      ├── QueryId B
      └── QueryId C
```

---

# 153. Fan-out telemetry

Podrá medir:

```text
shards targeted
shards completed
shards failed
merge duration
overall outcome
```

---

# 154. ShardId cardinality

ShardId solo será metric label si:

```text
CardinalityPolicy
```

lo permite.

---

# 155. Distributed partial outcome

Si:

```text
Shard A = SUCCESS
Shard B = SUCCESS
Shard C = FAILED
```

el resultado global podrá ser:

```text
PARTIAL
```

según la operación.

---

# 156. Partial ≠ success

Nunca colapsarlo a:

```text
SUCCESS
```

para simplificar dashboards.

---

# 157. Replica query telemetry

Podrá registrar:

```text
connection.role=replica
```

y:

```text
routing_reason
```

---

# 158. Replica lag

La observación de lag pertenece principalmente al Replica System.

Query Telemetry podrá correlacionar la evidencia usada para routing.

---

# 159. Replica lag UNKNOWN

Debe preservarse como:

```text
UNKNOWN
```

---

# 160. Sticky connection

Podrá indicarse:

```text
routing_reason=sticky_write
```

---

# 161. Transaction correlation

Una Query dentro de una transacción deberá enlazar:

```text
TransactionId
```

en trace/diagnostic.

---

# 162. TransactionId cardinality

Nunca como metric label por default.

---

# 163. Transaction depth

Podrá registrarse como bounded attribute:

```text
transaction.depth
```

si tiene límites razonables.

---

# 164. Savepoint correlation

Queries de control transaccional podrán clasificarse aparte.

---

# 165. Control statements

Ejemplos:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

no deberán necesariamente contarse como application queries.

---

# 166. Transaction telemetry authority

Será `Transaction Telemetry System` quien posea su semántica.

---

# 167. Query sampling

```php
interface QueryTelemetrySampler
{
    public function decide(
        QueryTelemetryContext $context,
        QueryTelemetryStage $stage,
    ): TelemetrySamplingDecision;
}
```

---

# 168. Pre-execution sampling

Puede decidirse antes de ejecutar basándose en:

```text
origin
operation
parent trace
environment
```

---

# 169. Post-execution promotion

Una query inicialmente poco instrumentada podrá promover diagnóstico cuando:

```text
duration > threshold
```

o:

```text
outcome = failure
```

---

# 170. Tail information limitation

Si no se capturó información al inicio:

```text
cannot magically reconstruct it later
```

---

# 171. Minimal retained context

Para slow/error promotion podrá conservarse un contexto mínimo y seguro:

```text
start time
fingerprint
operation
origin
```

---

# 172. Slow Query Detector integration

Flujo:

```text
QueryTelemetry
↓
QueryDuration
↓
SlowQueryDetection
↓
SlowQueryDiagnostic
```

---

# 173. Query Telemetry ≠ slow query threshold

El threshold pertenece al documento:

```text
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
```

---

# 174. Query Profiler integration

```text
QueryTelemetry Records
+
Phase Timings
+
Plan Metadata
+
Execution Metadata
↓
Query Profiler
```

---

# 175. Query Telemetry ≠ profiler

Telemetry produce evidencia.

Profiler la analiza.

---

# 176. Debug Information integration

En development:

```text
QueryTelemetryRecord
↓
Request Diagnostic Buffer
↓
Debug Information System
```

---

# 177. QueryTelemetryRecord

```php
final readonly class QueryTelemetryRecord
{
    public function __construct(
        public QueryId $queryId,
        public ?QueryOperationId $operationId,
        public ?QueryFingerprint $fingerprint,
        public QueryOperation $operation,
        public QueryOrigin $origin,
        public QueryOutcome $outcome,
        public Duration $duration,
        public array $phaseTimings,
        public QueryResultTelemetry $result,
        public QueryRoutingTelemetry $routing,
        public QuerySecurityView $securityView,
    ) {}
}
```

---

# 178. Immutable record

Una vez finalizada la ejecución:

```text
QueryTelemetryRecord
```

deberá ser immutable.

---

# 179. Active scope ≠ record

Durante ejecución existe:

```text
Mutable/Scoped Instrumentation State
```

Después:

```text
Immutable Telemetry Record
```

---

# 180. Query telemetry memory

Request diagnostics podrán almacenar records.

Production metrics no necesitan almacenar todos los records.

---

# 181. No global query history

Nunca:

```php
static array $queries = [];
```

en un worker persistente.

---

# 182. Diagnostic buffer

Debe ser:

```text
request/operation scoped
+
bounded
```

---

# 183. Diagnostic overflow

Si se alcanza el límite:

```text
drop oldest
or
drop additional
```

según policy.

---

# 184. Overflow metadata

Deberá poder indicar:

```text
records_dropped = N
```

---

# 185. Query telemetry resource budget

```php
final readonly class QueryTelemetryBudget
{
    public function __construct(
        public int $maxRecords,
        public int $maxAttributesPerRecord,
        public int $maxQueryTextBytes,
        public int $maxBindingEntries,
        public int $maxPhaseTimings,
    ) {}
}
```

---

# 186. Query text truncation

Si se permite SQL sanitizado y excede el límite:

```text
truncate safely
```

pero deberá indicarse:

```text
query_text_truncated=true
```

---

# 187. Truncated SQL ≠ executable SQL

Nunca deberá reutilizarse una representación telemetry para ejecución.

---

# 188. Telemetry representations are one-way

```text
Query
↓
Telemetry Representation
```

No:

```text
Telemetry Representation
↓
Executable Query
```

---

# 189. Persistent runtime

FrankenPHP requiere:

```text
QueryTelemetryContext
QueryTelemetryScope
ActiveQueryRegistry
DiagnosticBuffer
```

por request/operation.

---

# 190. Shared persistent-safe components

Podrán compartirse:

```text
QueryTelemetryPolicy
QueryFingerprintGenerator
Metric instruments
Compiled descriptors
Stateless sanitizer
```

si son realmente inmutables/stateless.

---

# 191. Request cleanup

Al finalizar:

```text
close active query scopes
↓
diagnose leaks
↓
release records
↓
clear context
```

---

# 192. Active query leak

Si queda una query scope activa:

```text
ACTIVE_QUERY_SCOPE_LEAK
```

podrá registrarse.

---

# 193. RoadRunner

Cada job deberá iniciar con:

```text
fresh QueryTelemetry operation scope
```

---

# 194. OpenSwoole

Active Query State deberá ser:

```text
coroutine-local
```

o equivalente.

---

# 195. Concurrent query support

Ejemplo:

```text
Operation
├── Query A
├── Query B
└── Query C
```

Cada una tendrá:

```text
independent QueryId
independent timer
independent span
independent connection correlation
```

---

# 196. No shared current query

Prohibido:

```php
$this->currentQuery = $query;
```

en servicios singleton.

---

# 197. Query nesting

Una query puede provocar indirectamente otra query mediante código de aplicación.

Ejemplo:

```text
Query A hydration
↓
lifecycle callback
↓
Query B
```

Telemetry deberá representar ambas sin sobrescribir contexto.

---

# 198. Query context stack

Cuando sea necesario podrá utilizarse un stack scope-local:

```text
Query A
└── Query B
```

pero la relación parent-child deberá representar semántica real, no solo orden temporal.

---

# 199. N+1 lazy load nesting

Un lazy load puede correlacionarse con la operación que lo provocó mediante:

```text
RelationshipLoadContext
```

---

# 200. Query source location

En debug mode podrá capturarse:

```text
file
line
call site
```

bajo sampling.

---

# 201. Source location cost

Stack inspection es costoso.

No deberá hacerse para cada query en producción por default.

---

# 202. Source location security

Paths del filesystem pueden ser sensibles.

Requieren policy.

---

# 203. Query explain diagnostics

API conceptual:

```php
DB::telemetry()->explainQuery($queryId);
```

---

# 204. Example explain output

```text
QUERY TELEMETRY
────────────────────────────────

Query ID:
  q_0192

Operation ID:
  qo_0064

Fingerprint:
  qf_2a9817

Origin:
  ORM

Operation:
  SELECT

Role:
  PRIMARY

Platform:
  PostgreSQL

Routing:
  replica

Routing Reason:
  REPLICA_ELIGIBLE

Transaction:
  none

Outcome:
  SUCCESS

Total Duration:
  18.4 ms

Phases:
  semantic analysis    0.7 ms
  optimization         0.9 ms
  planning             0.4 ms
  compilation          cache hit
  connection wait      1.3 ms
  execution            8.1 ms
  result consumption   1.6 ms
  hydration            5.4 ms

Rows Returned:
  25

Entities Hydrated:
  25

Query Text:
  hidden by policy

Bindings:
  2
  types: [boolean, datetime]

Result Cache:
  MISS

Compiled Query Cache:
  HIT

Trace:
  sampled

Warnings:
  none
```

---

# 205. Slow query explain example

```text
QUERY TELEMETRY
────────────────────────────────

Outcome:
  SUCCESS

Duration:
  1.82 s

Slow Query:
  YES

Primary Cost:
  execution

Execution:
  1.69 s

Connection Wait:
  4 ms

Hydration:
  38 ms

Rows:
  14

Possible Diagnostic:
  database execution dominates total duration
```

---

# 206. Diagnostics ≠ optimizer advice certainty

Telemetry podrá decir:

```text
execution dominates duration
```

pero no afirmar automáticamente:

```text
add index X
```

sin análisis suficiente.

---

# 207. Query telemetry configuration

```php
return [
    'database' => [
        'telemetry' => [
            'query' => [
                'enabled' => true,

                'metrics' => true,
                'tracing' => true,

                'text' => 'fingerprint_only',
                'bindings' => 'types_only',

                'phases' => [
                    'compilation' => true,
                    'execution' => true,
                    'hydration' => true,
                ],

                'source_location' => false,

                'diagnostics' => [
                    'enabled' => false,
                    'max_records' => 500,
                ],
            ],
        ],
    ],
];
```

---

# 208. QueryTelemetryPolicy

```php
final readonly class QueryTelemetryPolicy
{
    public function __construct(
        public bool $metrics,
        public bool $tracing,
        public QueryTextCaptureMode $queryText,
        public QueryBindingCaptureMode $bindings,
        public bool $phaseTimings,
        public bool $sourceLocation,
        public QueryTelemetryBudget $budget,
    ) {}
}
```

---

# 209. Policy compilation

Configuration deberá compilarse antes del hot path cuando sea posible.

```text
Configuration
↓
Validation
↓
Compiled QueryTelemetryPolicy
↓
Runtime
```

---

# 210. Security validation

Configuraciones peligrosas deberán detectarse:

```text
bindings=VALUES
+
external provider
+
no redaction
```

---

# 211. QueryTelemetryInstrumentation

```php
interface QueryTelemetryInstrumentation
{
    public function start(
        QueryExecutionDescriptor $query,
        DatabaseTelemetryContext $context,
    ): QueryTelemetryScope;
}
```

---

# 212. Null instrumentation

```php
final class NullQueryTelemetryInstrumentation
    implements QueryTelemetryInstrumentation
{
    public function start(
        QueryExecutionDescriptor $query,
        DatabaseTelemetryContext $context,
    ): QueryTelemetryScope {
        return NullQueryTelemetryScope::instance();
    }
}
```

---

# 213. Instrumentation boundary

El Executor podría hacer:

```php
$scope = $telemetry->start($descriptor, $context);

try {
    $result = $executor->execute($query);

    $scope->finish(
        QueryOutcome::SUCCESS
    );

    return $result;
} catch (QueryCancelledException $e) {
    $scope->finish(
        QueryOutcome::CANCELLED
    );

    throw $e;
} catch (\Throwable $e) {
    $scope->finish(
        QueryOutcome::FAILED
    );

    throw $e;
}
```

---

# 214. Streaming caveat

El ejemplo anterior no es suficiente para streaming.

El scope deberá transferirse al:

```text
Result Cursor lifecycle
```

hasta cierre/agotamiento.

---

# 215. Query scope ownership transfer

```text
Executor
↓
Streaming Result
↓
Result Cursor
↓
close()
↓
finish QueryTelemetryScope
```

---

# 216. Ownership must be explicit

Nunca depender únicamente de:

```php
__destruct()
```

para finalizar telemetry.

---

# 217. Destructor fallback

Destructor podrá ser:

```text
best-effort leak safety
```

no correctness mechanism.

---

# 218. Query telemetry events

Signals internos posibles:

```text
QueryTelemetryStarted
QueryPhaseCompleted
QueryExecutionCompleted
QueryResultConsumptionCompleted
QueryTelemetryCompleted
QueryTelemetryDropped
```

---

# 219. Per-phase event noise

`QueryPhaseCompleted` no deberá exportarse necesariamente como evento público.

Puede ser señal interna.

---

# 220. Per-row signals prohibited by default

Nunca emitir:

```text
QueryRowReturned
```

por cada row en producción normal.

---

# 221. Telemetry aggregation

Para streaming:

```text
rows consumed
bytes consumed
```

deberán agregarse localmente.

---

# 222. Byte count

Solo se registrará si puede medirse razonablemente.

No estimar bytes con serialización arbitraria.

---

# 223. Query security view

```php
final readonly class QuerySecurityView
{
    public function __construct(
        public ?QueryFingerprint $fingerprint,
        public ?SanitizedQueryText $queryText,
        public QueryParameterTelemetrySummary $parameters,
    ) {}
}
```

---

# 224. Security view immutable

La señal exportada deberá usar:

```text
safe representation
```

y no tener referencia al Query original.

---

# 225. Raw SQL escape hatch

Consultas raw deberán tener políticas más conservadoras.

---

# 226. Raw SQL diagnostic

Si no puede sanitizarse:

```text
query_text = omitted
query_fingerprint = normalized_hash_if_safe
```

---

# 227. Raw SQL with interpolation

VoltStack deberá diagnosticar, cuando sea detectable:

```text
raw SQL with literal values
```

pero Query Telemetry no sustituye:

```text
SQL Injection Prevention System
```

---

# 228. Security telemetry boundary

Query Telemetry podrá reportar:

```text
unsafe query construction candidate
```

pero la prevención pertenece al bloque de Security.

---

# 229. Query plan diagnostics

Cuando el provider/planner permita:

```text
logical plan fingerprint
physical plan fingerprint
estimated cost
```

podrán adjuntarse bajo profiling.

---

# 230. Estimated cost ≠ execution duration

```text
PlannerEstimatedCost
≠
WallClockDuration
```

---

# 231. Database EXPLAIN

Query Telemetry no deberá ejecutar automáticamente:

```sql
EXPLAIN
```

sobre cada query.

---

# 232. Why

Puede:

```text
add load
change timing
require permissions
have platform-specific behavior
```

---

# 233. Explicit profiling

EXPLAIN deberá ser parte de una operación de profiling explícita.

---

# 234. Query telemetry and read/write routing

La señal deberá registrar el routing ya decidido.

No influir en él.

```text
Routing System
↓
Routing Decision
↓
Query Telemetry
```

---

# 235. Query telemetry and cache

Igualmente:

```text
Cache Consistency System
↓
Cache Decision
↓
Query Telemetry
```

No:

```text
Telemetry
↓
Cache Decision
```

---

# 236. Query telemetry and retry

```text
Retry System
↓
Retry Decision
↓
Query Telemetry
```

---

# 237. Query telemetry and authorization

```text
Authorization
↓
Scoped Query
↓
Query Telemetry
```

Telemetry no deberá reconstruir datos filtrados por autorización.

---

# 238. Authorization leakage

Query text/metadata no deberá revelar:

```text
hidden tenant
hidden resource
forbidden identifier
```

más allá de policy.

---

# 239. Multitenancy

TenantContext podrá correlacionarse.

Pero:

```text
TenantId
```

se considerará high-cardinality/sensitive.

---

# 240. Tenant metric aggregation

Preferir:

```text
all tenant queries
```

o bounded tenant classes/tiers si existe necesidad legítima.

No una serie por tenant por default.

---

# 241. Query telemetry testing

El sistema deberá ofrecer:

```php
QueryTelemetry::assertExecuted(
    operation: QueryOperation::SELECT
);
```

---

# 242. Fingerprint assertion

```php
QueryTelemetry::assertFingerprintExecuted(
    $fingerprint
);
```

---

# 243. Query count assertion

```php
QueryTelemetry::assertQueryCount(3);
```

podrá apoyar pruebas de rendimiento/N+1.

---

# 244. Internal query filtering

Assertions deberán permitir:

```php
includeInternal: false
```

---

# 245. No sensitive values assertion

```php
QueryTelemetry::assertDoesNotContain(
    'secret-value'
);
```

---

# 246. Fake clock

Permite probar:

```text
duration
timeout classification
slow query promotion
```

de forma determinista.

---

# 247. Benchmark modes

Deberán medirse al menos:

```text
telemetry disabled
metrics only
metrics + trace
full phase timings
full diagnostics
```

---

# 248. Hot path allocation

La implementación deberá minimizar:

```text
temporary arrays
string formatting
stack traces
SQL copies
binding copies
```

cuando no sean necesarios.

---

# 249. Lazy attribute evaluation

Atributos costosos podrán representarse como:

```text
lazy telemetry attribute
```

solo si el pipeline los solicita.

---

# 250. Lazy telemetry callback safety

Estas callbacks no deberán:

```text
execute queries
perform network I/O
mutate DB state
trigger lazy relations
```

---

# 251. Directory structure

```text
src/Quantum/Database/Telemetry/Query/
│
├── Contract/
│   ├── QueryTelemetryInstrumentation.php
│   ├── QueryTelemetryScope.php
│   ├── QueryTelemetrySampler.php
│   ├── QueryTelemetrySanitizer.php
│   └── QueryFingerprintGenerator.php
│
├── Context/
│   ├── QueryTelemetryContext.php
│   ├── QueryTelemetryContextFactory.php
│   └── QueryTelemetryContextResolver.php
│
├── Identity/
│   ├── QueryId.php
│   ├── QueryOperationId.php
│   ├── QueryAttemptId.php
│   ├── QueryFingerprint.php
│   ├── SqlFingerprint.php
│   └── DistributedQueryId.php
│
├── Model/
│   ├── QueryOperation.php
│   ├── QueryOrigin.php
│   ├── QueryRole.php
│   ├── QueryCategory.php
│   ├── QueryOutcome.php
│   ├── QueryResultMode.php
│   ├── QueryFailureCategory.php
│   └── QueryTelemetryState.php
│
├── Phase/
│   ├── QueryTelemetryPhase.php
│   ├── QueryPhaseScope.php
│   ├── QueryPhaseTiming.php
│   └── QueryPhaseTimeline.php
│
├── Record/
│   ├── QueryTelemetryRecord.php
│   ├── QueryExecutionOutcome.php
│   ├── QueryResultTelemetry.php
│   ├── QueryRoutingTelemetry.php
│   ├── QueryCacheTelemetry.php
│   └── QuerySecurityView.php
│
├── Parameter/
│   ├── QueryParameterTelemetry.php
│   ├── QueryParameterTelemetrySummary.php
│   ├── QueryBindingCaptureMode.php
│   └── QueryParameterRedactor.php
│
├── Text/
│   ├── QueryTextCaptureMode.php
│   ├── SanitizedQueryText.php
│   ├── QueryTextSanitizer.php
│   └── RawSqlTelemetrySanitizer.php
│
├── Fingerprint/
│   ├── SemanticQueryFingerprintGenerator.php
│   ├── SqlFingerprintGenerator.php
│   └── QueryPlanFingerprintGenerator.php
│
├── Metric/
│   ├── QueryMetricRecorder.php
│   ├── QueryMetricDescriptor.php
│   └── QueryMetricRegistry.php
│
├── Trace/
│   ├── QuerySpanFactory.php
│   ├── QuerySpanEnricher.php
│   └── QueryTracePolicy.php
│
├── Sampling/
│   ├── QueryTelemetrySampler.php
│   ├── QueryTelemetrySamplingPolicy.php
│   └── QueryTelemetryPromotionPolicy.php
│
├── Instrumentation/
│   ├── DefaultQueryTelemetryInstrumentation.php
│   ├── NullQueryTelemetryInstrumentation.php
│   ├── DefaultQueryTelemetryScope.php
│   ├── StreamingQueryTelemetryScope.php
│   └── QueryTelemetryScopeFactory.php
│
├── Diagnostics/
│   ├── QueryTelemetryInspector.php
│   ├── QueryTelemetryExplainer.php
│   ├── QueryDiagnosticBuffer.php
│   ├── QueryTimeline.php
│   └── QueryTelemetryLeakDetector.php
│
├── Policy/
│   ├── QueryTelemetryPolicy.php
│   ├── CompiledQueryTelemetryPolicy.php
│   └── QueryTelemetryBudget.php
│
├── Testing/
│   ├── RecordingQueryTelemetry.php
│   ├── QueryTelemetryAssertions.php
│   └── QueryTelemetryTestRecord.php
│
└── Exception/
    ├── QueryTelemetryException.php
    ├── QueryTelemetryStateException.php
    ├── QueryTelemetrySecurityException.php
    ├── QueryTelemetryBudgetException.php
    └── QueryTelemetryInstrumentationException.php
```

---

# 252. Architectural invariants

## DB-QTEL-001
Query Telemetry observará consultas sin modificarlas.

## DB-QTEL-002
Query Telemetry no modificará Query AST.

## DB-QTEL-003
Query Telemetry no modificará Query Model.

## DB-QTEL-004
Query Telemetry no modificará bindings.

## DB-QTEL-005
Query Telemetry no modificará SQL.

## DB-QTEL-006
Query Telemetry no modificará Query Plan.

## DB-QTEL-007
Query Telemetry no modificará routing.

## DB-QTEL-008
Query Telemetry no modificará results.

## DB-QTEL-009
Query Telemetry no modificará outcome.

## DB-QTEL-010
Query Telemetry será distinta de Query Events.

## DB-QTEL-011
Query Telemetry será distinta de Query Profiler.

## DB-QTEL-012
Query Telemetry será distinta de Slow Query Detection.

## DB-QTEL-013
Query Telemetry será distinta de Query Logging.

## DB-QTEL-014
Query Telemetry será distinta de Query Cache.

## DB-QTEL-015
Query Telemetry será distinta de Query Optimizer.

## DB-QTEL-016
Query Telemetry será distinta de Query Planner.

## DB-QTEL-017
Query definition será distinta de query execution.

## DB-QTEL-018
Cada ejecución tendrá QueryId.

## DB-QTEL-019
QueryId será distinto de QueryFingerprint.

## DB-QTEL-020
QueryFingerprint no incluirá valores por default.

## DB-QTEL-021
Semantic fingerprint será preferido sobre SQL hash cuando sea posible.

## DB-QTEL-022
Raw SQL podrá usar normalized SQL fingerprint.

## DB-QTEL-023
Fingerprint no será security boundary.

## DB-QTEL-024
QueryOrigin será explícito.

## DB-QTEL-025
UNKNOWN operation no será asumida SELECT.

## DB-QTEL-026
Query telemetry lifecycle será explícito.

## DB-QTEL-027
Invalid telemetry state transitions serán diagnosticables.

## DB-QTEL-028
Fases no aplicables no se representarán falsamente como duración cero.

## DB-QTEL-029
Duraciones usarán monotonic clock cuando sea posible.

## DB-QTEL-030
Execution duration será distinta de pipeline duration.

## DB-QTEL-031
Result consumption duration será distinta de database execution duration.

## DB-QTEL-032
Client duration será distinta de server duration.

## DB-QTEL-033
Query duration será histogram.

## DB-QTEL-034
Query metric labels serán bounded.

## DB-QTEL-035
QueryId no será metric label.

## DB-QTEL-036
Raw SQL no será metric label.

## DB-QTEL-037
TenantId no será metric label por default.

## DB-QTEL-038
Query outcome UNKNOWN permanecerá UNKNOWN.

## DB-QTEL-039
PARTIAL permanecerá PARTIAL.

## DB-QTEL-040
Statement success será distinto de result consumption success.

## DB-QTEL-041
Unknown row count no será cero.

## DB-QTEL-042
Rows returned serán distintos de entities hydrated.

## DB-QTEL-043
Rows matched podrán ser distintos de rows changed.

## DB-QTEL-044
Query spans no producirán micro-span explosion por default.

## DB-QTEL-045
Detailed phase spans serán opt-in.

## DB-QTEL-046
QueryFingerprint será tratado como potencialmente high-cardinality.

## DB-QTEL-047
Raw query text estará desactivado o sanitizado por default.

## DB-QTEL-048
FULL query capture requerirá opt-in.

## DB-QTEL-049
Raw SQL será considerado potencialmente sensitive.

## DB-QTEL-050
Un SQL no sanitizable podrá omitirse.

## DB-QTEL-051
Binding values no se capturarán por default.

## DB-QTEL-052
Binding types podrán capturarse de forma segura.

## DB-QTEL-053
Credentials nunca serán bindings telemetry.

## DB-QTEL-054
No habrá arbitrary object serialization para bindings.

## DB-QTEL-055
Value Objects usarán safe telemetry representation.

## DB-QTEL-056
Compiled query cache será distinto de Result Cache.

## DB-QTEL-057
Compilation cache hit no fingirá compilation duration cero.

## DB-QTEL-058
Optimizer telemetry no controlará optimizer rules.

## DB-QTEL-059
Query Plan no será exportado como live object.

## DB-QTEL-060
Routing telemetry observará decisiones ya tomadas.

## DB-QTEL-061
Connection acquisition metric tendrá canonical owner en Connection Telemetry.

## DB-QTEL-062
Query Telemetry evitará double recording de connection metrics.

## DB-QTEL-063
Prepared Statement será distinto de query cache.

## DB-QTEL-064
Binding telemetry no registrará values por default.

## DB-QTEL-065
Retries tendrán identidad explícita.

## DB-QTEL-066
Cada retry attempt podrá tener QueryId independiente.

## DB-QTEL-067
Logical QueryOperationId agrupará attempts.

## DB-QTEL-068
Telemetry no decidirá retry safety.

## DB-QTEL-069
Failure categories serán estructuradas.

## DB-QTEL-070
Error message no será metric label.

## DB-QTEL-071
Stack trace no será metric attribute.

## DB-QTEL-072
Cancellation reason será estructurado.

## DB-QTEL-073
Cancellation será distinta de failure.

## DB-QTEL-074
Timeout será distinto de generic cancellation.

## DB-QTEL-075
Result mode será explícito.

## DB-QTEL-076
Streaming query scope podrá sobrevivir al Executor call.

## DB-QTEL-077
Streaming scope finalizará al agotar/cerrar cursor.

## DB-QTEL-078
Early termination será observable.

## DB-QTEL-079
Early termination no será failure por default.

## DB-QTEL-080
Result cursor leaks serán diagnosticables.

## DB-QTEL-081
Hydration telemetry tendrá su propio instrumentation owner.

## DB-QTEL-082
Phase timings no serán asumidos siempre aditivos.

## DB-QTEL-083
Physical cache hit será distinto de usable hit.

## DB-QTEL-084
Result Cache hit podrá evitar DB execution.

## DB-QTEL-085
Logical query operation será distinta de physical execution.

## DB-QTEL-086
db.query.operations podrá diferir de db.query.executions.

## DB-QTEL-087
Cache keys sensibles no serán exportados completos.

## DB-QTEL-088
Pagination count query será identificable semánticamente.

## DB-QTEL-089
Query roles serán explícitos.

## DB-QTEL-090
Relationship queries podrán transportar RelationshipId.

## DB-QTEL-091
RelationshipId no será automáticamente metric label.

## DB-QTEL-092
Chunk queries serán correlacionables con traversal.

## DB-QTEL-093
Chunk index no creará metric series.

## DB-QTEL-094
Lazy Collection queries serán correlacionables.

## DB-QTEL-095
Bulk operation será distinta de sus statements.

## DB-QTEL-096
Import operation será distinta de sus queries.

## DB-QTEL-097
Export operation será distinta de sus queries.

## DB-QTEL-098
Internal framework queries serán clasificables.

## DB-QTEL-099
Internal query no será asumida irrelevante.

## DB-QTEL-100
Schema queries serán distinguibles.

## DB-QTEL-101
Distributed query tendrá identidad lógica.

## DB-QTEL-102
Shard query tendrá QueryId propio.

## DB-QTEL-103
Distributed partial failure permanecerá PARTIAL.

## DB-QTEL-104
ShardId estará sujeto a cardinality policy.

## DB-QTEL-105
Replica routing será observable.

## DB-QTEL-106
Replica lag UNKNOWN permanecerá UNKNOWN.

## DB-QTEL-107
Sticky routing será diagnosticable.

## DB-QTEL-108
TransactionId podrá correlacionarse en traces.

## DB-QTEL-109
TransactionId no será metric label.

## DB-QTEL-110
Transaction telemetry tendrá autoridad sobre transaction semantics.

## DB-QTEL-111
Query sampling no modificará query semantics.

## DB-QTEL-112
Slow/error queries podrán promoverse a mayor detalle.

## DB-QTEL-113
Post-execution promotion no reconstruirá información nunca capturada.

## DB-QTEL-114
Minimal retained context será seguro.

## DB-QTEL-115
Slow Query Detector consumirá Query Telemetry.

## DB-QTEL-116
Query Profiler consumirá Query Telemetry.

## DB-QTEL-117
Debug Information consumirá Query Telemetry.

## DB-QTEL-118
Completed telemetry records serán immutable.

## DB-QTEL-119
Active instrumentation state será distinto del immutable record.

## DB-QTEL-120
No existirá global unbounded query history.

## DB-QTEL-121
Diagnostic buffers serán scoped.

## DB-QTEL-122
Diagnostic buffers serán bounded.

## DB-QTEL-123
Dropped diagnostic records serán contabilizables.

## DB-QTEL-124
Query telemetry tendrá resource budget.

## DB-QTEL-125
Query text tendrá size limit.

## DB-QTEL-126
Truncation será explícita.

## DB-QTEL-127
Telemetry query representation nunca será ejecutable.

## DB-QTEL-128
Persistent runtime query state será scoped.

## DB-QTEL-129
Request end limpiará query telemetry state.

## DB-QTEL-130
Active scope leaks serán diagnosticables.

## DB-QTEL-131
RoadRunner tendrá fresh scope por job.

## DB-QTEL-132
OpenSwoole tendrá coroutine-safe query telemetry.

## DB-QTEL-133
Concurrent queries tendrán timers independientes.

## DB-QTEL-134
Concurrent queries tendrán spans independientes.

## DB-QTEL-135
No existirá singleton currentQuery mutable.

## DB-QTEL-136
Nested queries no sobrescribirán parent telemetry.

## DB-QTEL-137
Source location será opt-in/sampled.

## DB-QTEL-138
Source location no se capturará en producción por default.

## DB-QTEL-139
Filesystem paths serán potencialmente sensitive.

## DB-QTEL-140
Query telemetry será explainable.

## DB-QTEL-141
Diagnostics no presentarán optimizer advice como certeza sin evidencia.

## DB-QTEL-142
Query telemetry policy será compilable.

## DB-QTEL-143
Dangerous telemetry configuration será validada.

## DB-QTEL-144
Null instrumentation estará disponible.

## DB-QTEL-145
Disabled instrumentation minimizará overhead.

## DB-QTEL-146
Streaming transferirá explícitamente scope ownership.

## DB-QTEL-147
Destructor no será correctness mechanism.

## DB-QTEL-148
Per-row telemetry estará deshabilitada por default.

## DB-QTEL-149
Streaming row metrics se agregarán.

## DB-QTEL-150
Bytes no se estimarán mediante arbitrary serialization.

## DB-QTEL-151
Safe security view no contendrá referencias al Query original.

## DB-QTEL-152
Raw SQL tendrá política conservadora.

## DB-QTEL-153
Query Telemetry no sustituirá SQL Injection Prevention.

## DB-QTEL-154
EXPLAIN no se ejecutará automáticamente por query.

## DB-QTEL-155
Planner estimated cost será distinto de measured duration.

## DB-QTEL-156
Telemetry observará routing después de la decisión.

## DB-QTEL-157
Telemetry observará cache decision después de la decisión.

## DB-QTEL-158
Telemetry observará retry decision después de la decisión.

## DB-QTEL-159
Telemetry no reconstruirá información eliminada por authorization.

## DB-QTEL-160
Tenant context será tratado como sensitive/high-cardinality.

## DB-QTEL-161
Query telemetry assertions podrán excluir internal queries.

## DB-QTEL-162
Sensitive-data assertions estarán disponibles.

## DB-QTEL-163
Telemetry overhead tendrá benchmarks propios.

## DB-QTEL-164
Hot path evitará SQL formatting innecesario.

## DB-QTEL-165
Hot path evitará stack inspection innecesaria.

## DB-QTEL-166
Lazy telemetry attributes no ejecutarán queries.

## DB-QTEL-167
Lazy telemetry attributes no activarán lazy loading.

## DB-QTEL-168
Telemetry provider failure no cambiará query outcome por default.

## DB-QTEL-169
Query Telemetry nunca será fuente de Database truth.

## DB-QTEL-170
Query Telemetry permanecerá provider-agnostic.

---

# 253. Modelo formal

Sea una consulta lógica:

```text
Q
```

y una ejecución concreta:

```text
E_i(Q)
```

Cada ejecución tendrá:

```text
QueryId(E_i)
```

mientras:

```text
Fingerprint(E_i(Q))
=
Fingerprint(Q)
```

para ejecuciones estructuralmente equivalentes.

---

# 254. Modelo de operación e intento

Sea:

```text
O(Q)
```

una operación lógica que puede requerir `n` intentos:

```text
O(Q)
=
{
    E1(Q),
    E2(Q),
    ...
    En(Q)
}
```

Entonces:

```text
QueryOperationId(E1)
=
QueryOperationId(E2)
=
...
=
QueryOperationId(En)
```

pero:

```text
QueryId(E1)
≠
QueryId(E2)
≠
...
≠
QueryId(En)
```

---

# 255. Modelo temporal

Para ejecución `E`:

```text
Tpipeline(E)
=
M(end_pipeline)
-
M(start_pipeline)
```

y:

```text
Texecution(E)
=
M(end_execution)
-
M(start_execution)
```

Normalmente:

```text
Texecution(E)
≤
Tpipeline(E)
```

aunque las definiciones exactas dependerán del modo de consumo.

---

# 256. Modelo de streaming

Para un stream:

```text
Tfirst
=
M(first_result)
-
M(execution_start)
```

y:

```text
Tstream
=
M(cursor_close)
-
M(execution_start)
```

donde:

```text
Tstream
```

puede incluir procesamiento del consumidor.

---

# 257. Modelo de seguridad

La representación observable será:

```text
TelemetryView(Q)
=
Policy(
    Sanitize(
        Minimize(Q)
    )
)
```

y nunca deberá utilizarse para reconstruir:

```text
ExecutableQuery(Q)
```

---

# 258. Modelo de cache

Sea una operación lógica `O`.

Si:

```text
ResultCache(O) = usable_hit
```

entonces:

```text
QueryOperations += 1
QueryExecutions += 0
```

Si:

```text
ResultCache(O) = miss
```

y se ejecuta un statement:

```text
QueryOperations += 1
QueryExecutions += 1
```

---

# 259. Modelo distribuido

Para consulta distribuida `D`:

```text
D
=
{
    Q1@ShardA,
    Q2@ShardB,
    ...
    Qn@ShardN
}
```

Telemetry deberá preservar tanto:

```text
DistributedQueryId(D)
```

como:

```text
QueryId(Qi)
```

---

# 260. Arquitectura final

```text
                    Query API / ORM / Repository
                              │
                              ▼
                         Query Model
                              │
                              ▼
                       QueryOperationId
                              │
                              ▼
                    Query Telemetry Scope
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   Semantic/Plan          Compilation            Routing
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                     Connection Acquisition
                              │
                              ▼
                          QueryId
                              │
                              ▼
                         Execution
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
             Failure                     Result
                                             │
                                             ▼
                                       Consumption
                                             │
                                             ▼
                                         Hydration
                                             │
                                             ▼
                                     Query Outcome
                                             │
                                             ▼
                                   Immutable Record
                                             │
                         ┌───────────────────┼──────────────────┐
                         ▼                   ▼                  ▼
                      Metrics              Trace            Diagnostics
                         │                   │                  │
                         └───────────────────┼──────────────────┘
                                             ▼
                                  Database Telemetry Pipeline
```

---

# 261. Filosofía arquitectónica

VoltStack seguirá:

```text
Semantic identity
over
SQL string identity

Execution attempts
over
ambiguous query counters

Structured outcomes
over
success/failure booleans

Monotonic timing
over
wall-clock timing

Safe fingerprints
over
raw values

Bounded metrics
over
high-cardinality labels

Explicit streaming lifecycle
over
executor-only timing

Operation correlation
over
isolated query records

Provider abstraction
over
vendor-specific instrumentation

Observation
over
behavior modification
```

---

# 262. Regla maestra

> **Toda consulta deberá poder ser observada desde su identidad lógica hasta su ejecución física y consumo final, preservando fases, intentos, routing, resultados, errores y contexto suficientes para diagnosticar el sistema, pero sin permitir que la instrumentación adquiera autoridad sobre la consulta observada.**

En forma compacta:

```text
Query
↓
Execute
↓
Observe
↓
Measure
↓
Correlate
↓
Diagnose
```

Nunca:

```text
Observe
↓
Mutate Query
```

---

# 263. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
○ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
○ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
○ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
○ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
○ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 264. Siguiente documento

```text
218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
```

El siguiente documento deberá definir la observabilidad del ciclo completo de conexiones:

```text
Connection Request
↓
Connection Resolution
↓
Routing
↓
Pool Acquisition
↓
Wait
↓
Creation / Reuse
↓
Authentication
↓
Active Use
↓
Reset
↓
Return to Pool / Close
```

incluyendo:

```text
ConnectionId
logical vs physical connections
connection acquisition latency
pool wait time
connection creation time
connection reuse
connection lifetime
connection role
writer/replica telemetry
endpoint telemetry
health state
connection failures
connection resets
pool saturation
resource exhaustion
failover correlation
transaction pinning
tenant/shard correlation
connection leaks
persistent runtime reuse
FrankenPHP
RoadRunner
OpenSwoole
cardinality governance
credential protection
```

bajo la regla central:

> **Connection Telemetry observará la adquisición, uso, reutilización, estado y liberación de conexiones sin participar en su selección, routing, pooling, failover, reset o lifecycle funcional.**