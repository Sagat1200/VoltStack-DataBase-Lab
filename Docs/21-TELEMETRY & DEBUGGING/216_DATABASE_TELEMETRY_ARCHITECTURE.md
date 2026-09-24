# 216_DATABASE_TELEMETRY_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Telemetry Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 216 — Database Telemetry Architecture  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `215_DATABASE_EVENT_EXTENSION_SYSTEM.md`  
**Siguiente documento:** `217_DATABASE_QUERY_TELEMETRY_SYSTEM.md`

---

# 1. Propósito

`Database Telemetry Architecture` define la arquitectura transversal mediante la cual `VoltStack/Quantum/Database` podrá observar, medir, correlacionar, diagnosticar y perfilar su comportamiento sin convertir la observabilidad en parte de la autoridad funcional del Database Engine.

La arquitectura deberá cubrir:

```text
Queries
Connections
Transactions
ORM
Hydration
Persistence
Relationships
Cache
Pagination
Bulk Operations
Import / Export
Large Dataset Processing
Distributed Database Operations
Runtime Workers
```

y producir señales como:

```text
Metrics
Traces
Structured Logs
Profiles
Diagnostics
Health Signals
```

La regla central será:

> **Telemetry observa y describe lo que Database hace; no decide qué debe hacer Database ni se convierte en una condición necesaria para que una operación de base de datos sea correcta.**

Formalmente:

```text
Database Correctness
⊥
Telemetry Availability
```

Es decir:

```text
TelemetryFailure
↛
DatabaseCorrectnessFailure
```

salvo que una aplicación configure deliberadamente una política externa más estricta.

---

# 2. Posición dentro de la arquitectura

La arquitectura general queda:

```text
Application / ORM API
        │
        ▼
Query / ORM / Persistence
        │
        ▼
Transaction / Execution
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

Telemetry observa transversalmente:

```text
              Telemetry Architecture
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
    Metrics          Traces            Logs
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                 Diagnostics
                       │
                       ▼
               Telemetry Provider
```

sin alterar la dirección principal de dependencias.

---

# 3. Objetivos

El sistema deberá proporcionar:

- instrumentación tipada;
- métricas de bajo costo;
- tracing distribuido;
- logs estructurados;
- profiling de consultas;
- correlación entre capas;
- identificación de consultas lentas;
- detección N+1;
- información para debugging;
- integración con herramientas de desarrollo;
- soporte para providers externos;
- compatibilidad con OpenTelemetry;
- sampling;
- control de cardinalidad;
- sanitización;
- protección de información sensible;
- aislamiento por request/operation;
- compatibilidad con runtimes persistentes;
- soporte multitenant;
- soporte distribuido;
- diagnósticos explicables.

---

# 4. No objetivos

Telemetry no deberá convertirse en:

```text
Query Engine
Query Optimizer
Query Planner
Connection Router
Transaction Manager
Persistence Engine
Authorization System
Audit Authority
Cache Consistency Engine
Retry Engine
Health Recovery Engine
```

---

# 5. Distinciones fundamentales

VoltStack deberá preservar:

```text
Telemetry
≠
Logging

Telemetry
≠
Events

Telemetry
≠
Audit

Telemetry
≠
Debugging

Telemetry
≠
Profiling

Telemetry
≠
Metrics

Telemetry
≠
Tracing

Telemetry
≠
Health Check

Telemetry
≠
Correctness
```

Telemetry es la arquitectura que puede integrar varias de estas señales.

---

# 6. Telemetry ≠ Events

Database Events describen acontecimientos semánticos:

```text
QueryExecuted
TransactionCommitted
EntityPersisted
```

Telemetry puede consumir esos eventos.

Pero:

```text
Database Event
≠
Telemetry Signal
```

Un evento puede producir:

```text
0..N telemetry signals
```

---

# 7. Telemetry ≠ Logging

Un log es una señal discreta.

Ejemplo:

```json
{
    "event": "database.query.slow",
    "duration_ms": 847.3,
    "query_fingerprint": "qf_92..."
}
```

Telemetry incluye además:

```text
metrics
traces
profiles
diagnostics
```

---

# 8. Telemetry ≠ Audit

Audit responde:

```text
Who did what?
When?
To which protected resource?
```

Telemetry responde principalmente:

```text
How is the system behaving?
```

Por tanto:

```text
Telemetry Data
≠
Audit Record
```

---

# 9. Telemetry ≠ Debugging

Debugging puede consumir telemetry, pero puede requerir información adicional de desarrollo.

```text
Telemetry
→ Debugging
```

no:

```text
Telemetry = Debugging
```

---

# 10. Telemetry ≠ Profiling

Profiling es una capacidad especializada.

```text
Telemetry Architecture
└── Query Profiler
```

---

# 11. Telemetry ≠ Health

Health puede derivarse parcialmente de telemetry:

```text
Telemetry Signals
↓
Health Evaluation
```

pero el Health System tendrá sus propios contratos.

---

# 12. Arquitectura conceptual

```text
Database Components
        │
        ▼
Instrumentation Points
        │
        ▼
Telemetry Signal Factory
        │
        ▼
Telemetry Context
        │
        ├────────────┬────────────┐
        ▼            ▼            ▼
     Metrics       Traces        Logs
        │            │            │
        └────────────┼────────────┘
                     ▼
              Telemetry Pipeline
                     │
          ┌──────────┼───────────┐
          ▼          ▼           ▼
       Sampling   Redaction   Aggregation
          │          │           │
          └──────────┼───────────┘
                     ▼
              Provider Adapter
                     │
          ┌──────────┼────────────┐
          ▼          ▼            ▼
     OpenTelemetry  Local      Custom
```

---

# 13. Telemetry Signal Model

VoltStack podrá definir:

```php
interface DatabaseTelemetrySignal
{
    public function name(): TelemetrySignalName;

    public function timestamp(): Instant;

    public function context(): DatabaseTelemetryContext;
}
```

---

# 14. Signal categories

```php
enum DatabaseTelemetrySignalType
{
    case METRIC;
    case TRACE;
    case LOG;
    case PROFILE;
    case DIAGNOSTIC;
}
```

---

# 15. Signal identity

Las señales deberán utilizar nombres estables.

Ejemplos:

```text
db.query.duration
db.query.executed
db.connection.acquire.duration
db.transaction.duration
db.orm.flush.duration
db.hydration.entities
```

---

# 16. Stable names ≠ PHP classes

No deberá utilizarse:

```text
VoltStack\Quantum\Database\Telemetry\QueryDurationMetric
```

como identidad externa.

---

# 17. Telemetry context

Cada señal podrá contener un contexto:

```php
final readonly class DatabaseTelemetryContext
{
    public function __construct(
        public ?OperationId $operationId,
        public ?RequestId $requestId,
        public ?TraceId $traceId,
        public ?SpanId $spanId,
        public ?TransactionId $transactionId,
        public ?ConnectionId $connectionId,
        public ?QueryId $queryId,
        public ?PersistenceDomainId $domainId,
        public ?TenantId $tenantId,
        public ?ShardId $shardId,
        public array $attributes = [],
    ) {}
}
```

---

# 18. Context ≠ global mutable state

Nunca:

```php
Telemetry::$currentTenant
Telemetry::$currentQuery
Telemetry::$currentTransaction
```

El contexto deberá viajar explícitamente o resolverse desde un scope controlado.

---

# 19. Correlation hierarchy

Conceptualmente:

```text
Request / Job / Operation
          │
          ▼
      Database Scope
          │
          ▼
      Transaction
          │
          ▼
        Query
          │
          ▼
      Connection
```

aunque una Query también puede existir fuera de una transacción explícita.

---

# 20. Operation ID

`OperationId` representa la operación lógica de aplicación/database.

Ejemplo:

```text
HTTP request
CLI command
queue job
scheduled task
background operation
```

---

# 21. Query ID

Cada ejecución podrá recibir:

```text
QueryId
```

para correlación interna.

---

# 22. QueryId ≠ QueryFingerprint

Distinción crítica:

```text
QueryId
=
individual execution identity
```

```text
QueryFingerprint
=
semantic/query-shape identity
```

---

# 23. Query fingerprint

Ejemplo:

```sql
SELECT * FROM users WHERE id = ?
```

podrá producir:

```text
qf_7b28c0...
```

sin incluir el valor:

```text
id = 581928
```

---

# 24. Semantic fingerprint

Siempre que sea posible deberá derivarse de:

```text
normalized Query Model / AST
```

y no únicamente de SQL textual.

---

# 25. Why

Esto permite agrupar:

```text
SELECT ... WHERE id = 1
SELECT ... WHERE id = 2
SELECT ... WHERE id = 3
```

como una misma forma lógica.

---

# 26. SQL fingerprint fallback

Para raw SQL podrá utilizarse:

```text
normalized SQL fingerprint
```

bajo reglas específicas.

---

# 27. Parameter values

Por default:

```text
Query Parameter Values = REDACTED
```

---

# 28. Parameter types

Podrán registrarse:

```text
string
integer
uuid
datetime
json
```

cuando sea seguro.

---

# 29. Parameter metadata

Ejemplo:

```json
{
    "parameter_count": 3,
    "parameter_types": [
        "uuid",
        "datetime",
        "boolean"
    ]
}
```

---

# 30. Sensitive SQL

Incluso el SQL puede contener información sensible cuando se usa raw SQL.

Por tanto:

```text
SQL Text
≠
Automatically Safe
```

---

# 31. Query telemetry representation

Se preferirá:

```text
query fingerprint
query operation type
tables/resources
duration
row counts
platform
execution status
```

sobre valores completos.

---

# 32. Instrumentation sources

Telemetry podrá obtener señales mediante:

```text
Direct Instrumentation
Database Events
Scoped Instrumentation
Provider Hooks
```

---

# 33. Direct instrumentation

Ejemplo:

```text
Query Executor
↓
start timer
↓
execute
↓
stop timer
```

es preferible cuando se necesita una medición precisa.

---

# 34. Event-derived telemetry

Ejemplo:

```text
TransactionCommitted
↓
Telemetry Event Bridge
↓
transaction counter
```

---

# 35. Event-derived limitation

La precisión temporal puede ser menor si:

```text
event emitted
```

no coincide exactamente con:

```text
operation boundary
```

---

# 36. Rule

> **Instrumentation crítica para medir una operación debe ubicarse en el componente que posee el lifecycle real de esa operación.**

---

# 37. Instrumentation ownership

Ejemplos:

```text
Query Executor
owns
query execution duration

Connection Manager
owns
connection acquisition duration

Transaction Manager
owns
transaction lifecycle duration

EntityManager
owns
flush orchestration duration
```

---

# 38. Metrics architecture

Tipos conceptuales:

```php
enum DatabaseMetricType
{
    case COUNTER;
    case UP_DOWN_COUNTER;
    case HISTOGRAM;
    case GAUGE;
}
```

---

# 39. Counter examples

```text
db.query.executions
db.query.failures
db.transaction.commits
db.transaction.rollbacks
db.connection.failures
```

---

# 40. Histogram examples

```text
db.query.duration
db.connection.acquire.duration
db.transaction.duration
db.orm.flush.duration
db.hydration.duration
```

---

# 41. Gauge examples

```text
db.connection.pool.active
db.connection.pool.idle
db.connection.pool.waiters
```

cuando el provider soporte observables apropiados.

---

# 42. Metric unit

Toda métrica deberá definir una unidad.

Ejemplo:

```text
seconds
bytes
rows
connections
operations
```

---

# 43. Duration units

Internamente se recomienda:

```text
Duration value object
```

y exportación en la unidad esperada por el provider.

---

# 44. Cardinality governance

Uno de los riesgos principales será:

```text
metric label explosion
```

---

# 45. High-cardinality values

No usar como labels por default:

```text
QueryId
RequestId
TraceId
TransactionId
ConnectionId
UserId
TenantId
Raw SQL
Full URL
Entity ID
```

---

# 46. Bounded attributes

Atributos razonables:

```text
db.system=mysql
db.operation=select
db.status=success
db.connection.role=replica
db.transaction.isolation=read_committed
db.orm.operation=flush
```

---

# 47. Tenant metric policy

`TenantId` podrá ser útil en algunos productos SaaS, pero:

```text
TenantId
```

puede generar cardinalidad extrema.

Por default deberá:

```text
not be a metric label
```

---

# 48. Tenant diagnostics

TenantId sí podrá aparecer en:

```text
traces
structured logs
diagnostic records
```

bajo policy de seguridad.

---

# 49. Shard labels

`ShardId` puede ser aceptable si el conjunto de shards es pequeño y bounded.

Deberá evaluarse mediante cardinality policy.

---

# 50. Cardinality policy

```php
interface DatabaseTelemetryCardinalityPolicy
{
    public function evaluate(
        TelemetryAttribute $attribute,
        DatabaseTelemetryContext $context,
    ): CardinalityDecision;
}
```

---

# 51. Cardinality decisions

```php
enum CardinalityDecision
{
    case ALLOW;
    case DROP;
    case HASH;
    case BUCKET;
}
```

---

# 52. Hash ≠ low cardinality

Hashear:

```text
TenantId
```

no reduce necesariamente cardinalidad.

---

# 53. Bucketing

Duraciones no deberán convertirse manualmente en labels.

Usar:

```text
Histogram
```

---

# 54. Trace architecture

Un trace podrá representar:

```text
Application Operation
       │
       ▼
Database Transaction
       │
       ├── Query
       ├── Query
       └── Query
```

---

# 55. Database spans

Spans potenciales:

```text
db.query
db.transaction
db.connection.acquire
db.orm.flush
db.persistence
db.hydration
db.relationship.load
db.bulk.insert
db.import
db.export
```

---

# 56. Span hierarchy

Ejemplo:

```text
HTTP Request
└── ORM Flush
    └── Transaction
        ├── INSERT users
        ├── INSERT addresses
        └── UPDATE accounts
```

---

# 57. Span ≠ event

Un span tiene:

```text
start
end
duration
status
attributes
parent relationship
```

---

# 58. Span context

```php
interface DatabaseSpan
{
    public function context(): TraceContext;

    public function attribute(
        string $name,
        TelemetryValue $value
    ): void;

    public function end(
        DatabaseSpanOutcome $outcome
    ): void;
}
```

---

# 59. Null span

Cuando tracing esté desactivado:

```php
NullDatabaseSpan
```

deberá minimizar overhead.

---

# 60. Trace sampling

No todas las operaciones necesitan un trace completo.

Estrategias:

```text
ALWAYS
NEVER
PROBABILISTIC
PARENT_BASED
SLOW_OPERATION
ERROR_BASED
CUSTOM
```

---

# 61. Sampling decision

```php
enum TelemetrySamplingDecision
{
    case RECORD;
    case DROP;
    case DEFER;
}
```

---

# 62. Parent-based sampling

Si una aplicación ya tiene un trace:

```text
HTTP Trace
```

Database deberá poder participar en él.

---

# 63. Database does not own global trace

Quantum Database consume:

```text
TraceContext
```

sin convertirse en el tracing system global.

---

# 64. Slow-query sampling

Una query lenta puede promoverse a señal detallada.

Ejemplo:

```text
normal query
→ metric only

slow query
→ metric + trace detail + diagnostic
```

---

# 65. Error sampling

Errores importantes podrán conservar mayor detalle.

---

# 66. Tail sampling

Si el provider soporta tail sampling, podrá integrarse externamente.

No será responsabilidad del core Database implementarlo completamente.

---

# 67. Structured logging architecture

Los logs deberán ser estructurados.

Preferido:

```json
{
    "event": "database.query.failed",
    "query_fingerprint": "qf_f921...",
    "db_system": "postgresql",
    "operation": "select",
    "duration_ms": 81.3,
    "failure_category": "timeout"
}
```

No depender exclusivamente de:

```text
"Query failed after 81ms!"
```

---

# 68. Log levels

Ejemplo conceptual:

```text
DEBUG
INFO
NOTICE
WARNING
ERROR
CRITICAL
```

pero Database no deberá generar ruido excesivo.

---

# 69. Success logs

No deberá registrarse cada query exitosa en producción por default.

---

# 70. Query debug logging

Podrá habilitarse explícitamente en:

```text
development
testing
diagnostic session
```

---

# 71. Error logs

Errores relevantes deberán poder producir structured logs.

---

# 72. Duplicate logging

Debe evitarse:

```text
Driver logs error
Executor logs same error
ORM logs same error
HTTP layer logs same error
```

sin valor adicional.

---

# 73. Error ownership

La capa que agrega contexto nuevo podrá enriquecer la señal, pero deberá evitar duplicación innecesaria.

---

# 74. Log correlation

Los logs podrán incluir:

```text
trace_id
span_id
operation_id
query_id
transaction_id
```

cuando corresponda.

---

# 75. Security policy

Toda telemetry deberá pasar por:

```text
DatabaseTelemetrySecurityPolicy
```

---

# 76. Sensitive categories

```php
enum TelemetrySensitivity
{
    case PUBLIC;
    case INTERNAL;
    case SENSITIVE;
    case SECRET;
}
```

---

# 77. SECRET

Ejemplos:

```text
password
database credential
access token
private key
connection secret
```

Nunca deberán exportarse.

---

# 78. SENSITIVE

Ejemplos:

```text
raw parameter values
tenant identifiers
user identifiers
full SQL
table-specific business values
```

requieren policy.

---

# 79. Redaction pipeline

```text
Raw Instrumentation Data
        │
        ▼
Classification
        │
        ▼
Redaction Policy
        │
        ▼
Safe Telemetry Signal
        │
        ▼
Provider
```

---

# 80. Redaction timing

La sanitización deberá ocurrir:

```text
before crossing untrusted/external boundary
```

---

# 81. Raw internal telemetry

Incluso in-process deberá minimizarse la creación de datos sensibles innecesarios.

---

# 82. Data minimization

Regla:

> **La forma más segura de proteger un dato de telemetry es no capturarlo cuando no es necesario.**

---

# 83. Connection telemetry security

No registrar:

```text
mysql://username:password@server/database
```

---

# 84. Safe connection representation

Preferir:

```json
{
    "db.system": "mysql",
    "db.namespace": "app",
    "server.address": "db-primary",
    "server.port": 3306,
    "connection.role": "writer"
}
```

bajo policy.

---

# 85. Server address sensitivity

Incluso hostnames internos pueden ser sensibles.

Deberá existir configuración.

---

# 86. ORM telemetry

Podrá observar:

```text
managed entity count
flush count
change-set count
insert/update/delete counts
hydration counts
relationship loads
identity map growth
```

---

# 87. ORM telemetry must not expose entities

No exportar:

```text
Entity object
```

---

# 88. Entity class naming

Podrá usarse un:

```text
stable EntityType
```

en vez de FQCN cuando sea posible.

---

# 89. Entity cardinality

`EntityType` normalmente es bounded.

`EntityId` no.

---

# 90. Transaction telemetry

Podrá registrar:

```text
duration
isolation level
nested depth
savepoint count
retry count
outcome
```

---

# 91. Transaction outcomes

Deben conservar estados reales:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 92. UNKNOWN

Telemetry nunca deberá transformar:

```text
UNKNOWN
```

en:

```text
FAILED
```

o:

```text
COMMITTED
```

por conveniencia estadística.

---

# 93. Outcome metrics

Ejemplo:

```text
db.transaction.outcomes{
    outcome="unknown"
}
```

---

# 94. Connection telemetry

Podrá medir:

```text
acquisition duration
connection creation
reuse
reset
failure
pool wait
pool utilization
endpoint role
failover
```

---

# 95. Connection identity

`ConnectionId` no deberá ser metric label.

---

# 96. Pool metrics

Deberán distinguir:

```text
configured capacity
active
idle
waiting
creating
unhealthy
```

cuando la implementación lo permita.

---

# 97. Replica telemetry

Podrá medir:

```text
replica eligibility
lag observation
routing decisions
endpoint health
```

---

# 98. Lag uncertainty

Si el lag es desconocido:

```text
lag = UNKNOWN
```

No:

```text
lag = 0
```

---

# 99. Distributed telemetry

En sharding deberá poder correlacionarse:

```text
logical operation
↓
shard A query
shard B query
shard C query
```

---

# 100. Fan-out metrics

Podrá medirse:

```text
shards_touched
fanout_duration
partial_failures
merge_duration
```

---

# 101. Partial results

Telemetry deberá preservar:

```text
PARTIAL
```

cuando una operación distribuida no pueda afirmarse completa.

---

# 102. Cache telemetry

Podrá medir:

```text
physical hits
usable hits
misses
invalidations
evictions
stale rejection
provider failure
```

---

# 103. Physical hit ≠ usable hit

Debe preservarse:

```text
PHYSICAL_HIT
≠
USABLE_HIT
```

---

# 104. Query cache telemetry

Deberá diferenciar:

```text
compiled query cache
result cache
metadata cache
entity cache
hydration cache
```

---

# 105. Hydration telemetry

Podrá medir:

```text
rows consumed
entities hydrated
entities reused from IdentityMap
scalar results
tuple results
hydration duration
```

---

# 106. Rows ≠ entities

No deberá inferirse:

```text
10 rows
=
10 entities
```

---

# 107. Relationship telemetry

Podrá observar:

```text
lazy loads
eager loads
batch loads
relationship queries
N+1 candidates
```

---

# 108. N+1 telemetry

Será desarrollado específicamente en:

```text
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

---

# 109. Bulk operation telemetry

Podrá registrar:

```text
requested rows
processed rows
affected rows
batch count
batch size
duration
failures
partial outcome
```

---

# 110. Import telemetry

Podrá observar:

```text
records read
records validated
records rejected
records persisted
batch duration
throughput
```

sin almacenar por default el contenido de cada registro.

---

# 111. Export telemetry

Podrá observar:

```text
rows exported
bytes emitted
duration
throughput
destination class
```

sin registrar datos exportados.

---

# 112. Large dataset telemetry

Deberá ser agregada.

No:

```text
one telemetry event per row
```

por default.

---

# 113. Telemetry amplification

Regla:

```text
Large Dataset Processing
×
Per-row telemetry
=
Telemetry Explosion
```

---

# 114. Aggregation policy

Para operaciones masivas:

```text
per operation
per chunk
sampled
periodic
```

según necesidad.

---

# 115. Pagination telemetry

Podrá medir:

```text
strategy
page size
high offset warnings
count strategy
count duration
```

---

# 116. Cursor pagination telemetry

Podrá medir:

```text
cursor validation failures
cursor strategy
lookahead
binding mismatch
```

sin registrar el cursor completo.

---

# 117. Chunk telemetry

Podrá medir:

```text
chunk count
rows processed
chunk duration
checkpoint count
retry count
```

---

# 118. Lazy Collection telemetry

Podrá medir:

```text
iteration started
source strategy
items yielded
chunks fetched
materialization
early termination
resource lifetime
```

sin emitir señal por item por default.

---

# 119. Telemetry scopes

```php
enum DatabaseTelemetryScope
{
    case PROCESS;
    case WORKER;
    case REQUEST;
    case OPERATION;
    case TRANSACTION;
    case QUERY;
}
```

---

# 120. Scope ownership

Estado mutable deberá vivir en el scope mínimo necesario.

---

# 121. Request scope

Ejemplos:

```text
request query count
request DB duration
N+1 observations
debug query timeline
```

---

# 122. Operation scope

Útil para:

```text
queue jobs
CLI commands
background tasks
```

---

# 123. Worker scope

Solo para información segura entre operaciones.

Ejemplo:

```text
worker lifetime metrics
connection pool state
```

---

# 124. Worker scope must not contain

```text
current tenant
current user
current transaction
request query list
```

---

# 125. Telemetry collector

```php
interface DatabaseTelemetryCollector
{
    public function record(
        DatabaseTelemetrySignal $signal
    ): void;
}
```

---

# 126. Composite collector

```text
DatabaseTelemetryCollector
├── MetricCollector
├── TraceCollector
├── LogCollector
└── DiagnosticCollector
```

---

# 127. Null collector

Cuando telemetry esté deshabilitada:

```php
NullDatabaseTelemetryCollector
```

---

# 128. No repeated enabled checks

Preferir:

```text
real implementation
or
null implementation
```

sobre múltiples:

```php
if ($telemetryEnabled) {
    ...
}
```

en hot paths.

---

# 129. Telemetry pipeline

```text
Instrumentation
↓
Signal Construction
↓
Context Enrichment
↓
Security Classification
↓
Sampling
↓
Cardinality Policy
↓
Redaction
↓
Aggregation
↓
Provider Adapter
```

---

# 130. Pipeline ordering

El orden exacto podrá variar por señal.

Ejemplo:

```text
sampling before expensive enrichment
```

puede ahorrar costo.

Pero nunca deberá saltarse protección de seguridad antes de exportación.

---

# 131. Fast path

Idealmente:

```text
Telemetry Disabled
↓
Null Instrumentation
↓
near-zero allocation
```

---

# 132. Telemetry overhead budget

La arquitectura deberá permitir definir:

```php
final readonly class DatabaseTelemetryBudget
{
    public function __construct(
        public ?int $maxSignals,
        public ?int $maxDiagnosticEntries,
        public ?int $maxAttributeCount,
        public ?int $maxAttributeBytes,
        public ?Duration $maxInstrumentationTime,
    ) {}
}
```

---

# 133. Budget exhaustion

Opciones:

```text
DROP
SAMPLE
AGGREGATE
TRUNCATE_SAFE
```

según tipo de señal.

---

# 134. Budget exhaustion ≠ DB failure

Por default:

```text
TelemetryBudgetExceeded
↛
QueryFailure
```

---

# 135. Provider abstraction

```php
interface DatabaseTelemetryProvider
{
    public function meter(): DatabaseMeter;

    public function tracer(): DatabaseTracer;

    public function logger(): DatabaseTelemetryLogger;
}
```

---

# 136. Provider ≠ Database dependency

El core deberá depender de:

```text
Database Telemetry Contracts
```

no de un vendor concreto.

---

# 137. OpenTelemetry integration

Arquitectura:

```text
Quantum Database
↓
Database Telemetry Contracts
↓
OpenTelemetry Adapter
↓
OpenTelemetry SDK
↓
OTLP / Collector / Backend
```

---

# 138. OpenTelemetry optional

VoltStack Database deberá funcionar sin instalar un SDK de OpenTelemetry.

---

# 139. Provider adapters

Posibles:

```text
Null
InMemory
OpenTelemetry
PSR Logger Bridge
Custom Metrics
Development Collector
```

---

# 140. PSR logging

Podrá existir integración con un logger compatible, pero:

```text
PSR Logger
≠
Telemetry Architecture
```

---

# 141. Telemetry provider failure

Por default:

```text
provider failure
↓
record diagnostic if possible
↓
continue DB operation
```

---

# 142. Provider failure recursion

Debe evitarse:

```text
Telemetry failure
↓
log telemetry failure
↓
logger telemetry failure
↓
log telemetry failure
↓
...
```

---

# 143. Emergency diagnostics

Podrá existir un mecanismo bounded independiente para reportar fallos del provider.

---

# 144. Telemetry provider circuit breaker

Opcionalmente:

```text
repeated provider failures
↓
temporarily disable provider
```

sin detener Database.

---

# 145. Telemetry failure policy

```php
enum DatabaseTelemetryFailurePolicy
{
    case IGNORE;
    case RECORD_LOCALLY;
    case DISABLE_PROVIDER;
    case PROPAGATE;
}
```

---

# 146. Default failure policy

```text
RECORD_LOCALLY
```

o `IGNORE` en hot paths extremadamente sensibles.

---

# 147. PROPAGATE

Deberá ser excepcional y explícito.

No default.

---

# 148. Diagnostics buffer

Para desarrollo podrá mantenerse:

```text
request-scoped bounded ring buffer
```

---

# 149. Ring buffer

Ejemplo:

```text
max 500 query records
```

Cuando se exceda:

```text
oldest diagnostic entries evicted
```

---

# 150. Bounded diagnostics

Nunca:

```text
array of every query forever
```

en worker persistente.

---

# 151. Query timeline

Debug mode podrá construir:

```text
0ms    request start
4ms    connection acquire
8ms    SELECT users
22ms   SELECT roles
31ms   ORM flush
35ms   BEGIN
38ms   UPDATE users
42ms   COMMIT
```

---

# 152. Query timeline ≠ production metrics

Es una herramienta de diagnóstico.

---

# 153. Profiling architecture

El Query Profiler utilizará:

```text
Query telemetry
Execution metadata
Query fingerprint
Duration
Result metadata
Query plan diagnostics
```

sin convertirse en Query Optimizer.

---

# 154. Slow query detection

Arquitectura:

```text
Query Duration
↓
Threshold Policy
↓
Slow Query Candidate
↓
Diagnostic / Telemetry Signal
```

---

# 155. Slow ≠ incorrect

Una query lenta puede ser funcionalmente correcta.

---

# 156. Threshold policies

Podrán depender de:

```text
operation type
environment
query category
route
database role
```

pero deberán evitar cardinalidad no controlada.

---

# 157. Adaptive threshold

Podrá existir como extensión futura, pero el baseline será policy explícita.

---

# 158. Debug information architecture

El documento 224 definirá:

```text
safe query display
bindings display policy
transaction state
connection state
ORM state
cache state
timelines
diagnostic snapshots
```

---

# 159. Debug toolbar integration

El documento 225 conectará:

```text
Database Diagnostics
↓
VoltStack Developer Debug Toolbar
```

sin acoplar Database al toolbar.

---

# 160. Telemetry events

El sistema podrá emitir eventos internos de telemetry:

```text
TelemetryProviderFailed
TelemetryBudgetExceeded
TelemetrySignalDropped
TelemetryCardinalityRejected
```

pero deberá prevenir recursion.

---

# 161. Telemetry self-observation

Telemetry puede observarse parcialmente a sí misma.

Sin embargo:

```text
Telemetry about Telemetry
```

deberá ser limitada.

---

# 162. Recursion guard

```php
final class TelemetryRecursionGuard
{
    private int $depth = 0;
}
```

scope-local.

---

# 163. Persistent runtime architecture

Para FrankenPHP:

```text
Immutable Telemetry Configuration
Immutable Metric Instruments
Provider
        │
        ├── persistent-safe
        │
        ▼
Request / Operation Scope
        ├── current trace context
        ├── diagnostics
        ├── query timeline
        ├── N+1 state
        └── temporary counters
```

---

# 164. Request reset

Al finalizar:

```text
Request
↓
flush/export scoped telemetry if required
↓
close spans
↓
release diagnostics
↓
clear request context
```

---

# 165. State leakage prevention

Nunca deberá sobrevivir accidentalmente:

```text
TenantId
RequestId
TraceId
TransactionId
Query diagnostics
Entity references
```

al siguiente request.

---

# 166. FrankenPHP

Servicios compartidos permitidos:

```text
immutable metric descriptors
compiled telemetry configuration
stateless provider adapters
thread-safe/coroutine-safe instruments
```

---

# 167. RoadRunner

Cada job deberá obtener:

```text
fresh operation telemetry context
```

---

# 168. OpenSwoole

Context state deberá ser:

```text
coroutine-local
```

o equivalente.

---

# 169. Concurrent queries

Telemetry deberá soportar:

```text
Query A
Query B
Query C
```

ejecutándose concurrentemente sin mezclar:

```text
span
query ID
connection ID
duration
```

---

# 170. Shared timer anti-pattern

Incorrecto:

```php
$this->queryStart = microtime(true);
```

en un singleton compartido.

---

# 171. Correct timer

Cada operación tendrá:

```text
QueryTelemetryScope
```

independiente.

---

# 172. Telemetry clock

Deberá utilizarse una abstracción:

```php
interface TelemetryClock
{
    public function now(): Instant;

    public function monotonic(): MonotonicTimestamp;
}
```

---

# 173. Wall clock ≠ duration clock

Para duración:

```text
monotonic clock
```

es preferible.

---

# 174. Why

Cambios del reloj del sistema no deberán producir:

```text
negative query duration
```

---

# 175. Duration formula

```text
duration =
monotonic_end
-
monotonic_start
```

---

# 176. Time precision

La precisión dependerá del runtime/provider.

No deberá inventarse precisión que la plataforma no tenga.

---

# 177. Telemetry configuration

Ejemplo conceptual:

```php
return [
    'database' => [
        'telemetry' => [
            'enabled' => true,

            'metrics' => true,
            'tracing' => true,
            'logging' => true,

            'query' => [
                'capture_sql' => false,
                'capture_bindings' => false,
                'slow_threshold' => '500ms',
            ],

            'diagnostics' => [
                'enabled' => false,
                'max_queries' => 500,
            ],

            'sampling' => [
                'strategy' => 'parent_based',
            ],
        ],
    ],
];
```

---

# 178. Secure defaults

Producción deberá preferir:

```text
capture_sql = false or sanitized
capture_bindings = false
capture_entity_values = false
capture_credentials = never
bounded diagnostics
bounded attributes
```

---

# 179. Development defaults

Development podrá permitir mayor detalle, pero todavía deberá proteger:

```text
passwords
tokens
credentials
private keys
```

---

# 180. Environment ≠ security boundary

Estar en:

```text
APP_ENV=local
```

no autoriza automáticamente mostrar secrets.

---

# 181. Telemetry configuration validation

Deberá detectar combinaciones peligrosas.

Ejemplo:

```text
capture_bindings=true
+
external_export=true
+
no_redaction_policy
```

→ configuración inválida o warning severo.

---

# 182. Telemetry attribute model

```php
final readonly class TelemetryAttribute
{
    public function __construct(
        public TelemetryAttributeName $name,
        public TelemetryValue $value,
        public TelemetrySensitivity $sensitivity,
        public TelemetryCardinality $cardinality,
    ) {}
}
```

---

# 183. Attribute cardinality classification

```php
enum TelemetryCardinality
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case UNBOUNDED;
    case UNKNOWN;
}
```

---

# 184. UNKNOWN cardinality

No deberá asumirse:

```text
UNKNOWN = LOW
```

---

# 185. Telemetry value types

Permitidos:

```text
string
integer
float
boolean
bounded primitive arrays
stable IDs where policy allows
```

---

# 186. Live objects prohibited

No pasar al provider:

```text
PDO
Connection
EntityManager
Entity
Query AST
Closure
Resource
Generator
Result Cursor
```

---

# 187. Query AST diagnostics

Si se necesita mostrar AST:

```text
AST
↓
safe diagnostic representation
```

No exportar el objeto runtime.

---

# 188. Exception telemetry

Las exceptions podrán producir:

```text
exception class category
failure category
message fingerprint
stack trace under policy
```

---

# 189. Exception messages

Pueden contener valores sensibles.

Por tanto:

```text
Exception Message
≠
Automatically Safe
```

---

# 190. Stack traces

Podrán contener:

```text
filesystem paths
application code
argument values
```

y requieren policy.

---

# 191. Error classification

Preferir:

```text
TIMEOUT
CONNECTION_FAILURE
CONSTRAINT_VIOLATION
DEADLOCK
CANCELLATION
SYNTAX_ERROR
UNKNOWN
```

sobre parsing de mensajes en consumidores.

---

# 192. Platform error code

Podrá conservarse como atributo bounded cuando sea seguro.

---

# 193. Telemetry correlation with retries

Ejemplo:

```text
Transaction Span
├── Attempt 1
│   └── deadlock
├── Attempt 2
│   └── deadlock
└── Attempt 3
    └── committed
```

---

# 194. Retry attempt

Puede ser atributo:

```text
retry_attempt=2
```

bounded.

---

# 195. Attempt outcome ≠ final outcome

Debe distinguirse:

```text
AttemptOutcome
≠
OperationOutcome
```

---

# 196. Unknown transaction telemetry

Caso:

```text
COMMIT sent
↓
connection lost
↓
outcome UNKNOWN
```

Telemetry deberá registrar:

```text
outcome=unknown
```

---

# 197. Telemetry cannot resolve unknown

No:

```text
telemetry saw COMMIT statement
therefore committed
```

---

# 198. Cache integration

Telemetry podrá usar caches internos de instrumentos/descriptors.

Pero:

```text
Telemetry Cache
≠
Database Query Cache
```

---

# 199. Event integration

Arquitectura:

```text
Database Event System
↓
Telemetry Event Bridge
↓
Database Telemetry Pipeline
```

---

# 200. Direct instrumentation + event bridge

Ambos podrán coexistir.

---

# 201. Duplicate signal prevention

Deberá existir una estrategia para evitar:

```text
direct query metric
+
QueryExecuted event metric
=
double count
```

---

# 202. Instrumentation authority

Cada métrica deberá tener un:

```text
canonical instrumentation owner
```

---

# 203. Telemetry descriptor

```php
final readonly class DatabaseTelemetryDescriptor
{
    public function __construct(
        public TelemetrySignalName $name,
        public DatabaseTelemetrySignalType $type,
        public string $unit,
        public TelemetrySensitivity $sensitivity,
        public array $allowedAttributes,
        public TelemetryInstrumentationOwner $owner,
    ) {}
}
```

---

# 204. Registry

```php
interface DatabaseTelemetryRegistry
{
    public function register(
        DatabaseTelemetryDescriptor $descriptor
    ): void;
}
```

---

# 205. Registry compilation

```text
Core descriptors
+
Package descriptors
+
Application descriptors
↓
Validation
↓
Compiled Telemetry Registry
↓
Freeze
```

---

# 206. Duplicate metric names

Deberán detectarse.

---

# 207. Descriptor conflict

Dos componentes no podrán definir:

```text
db.query.duration
```

con unidades incompatibles.

---

# 208. Unit stability

Cambiar:

```text
seconds
→ milliseconds
```

sin cambiar contrato puede romper consumidores.

---

# 209. Telemetry extension model

Paquetes podrán registrar:

```text
metrics
span enrichers
diagnostic providers
samplers
redactors
provider adapters
```

mediante contratos controlados.

---

# 210. Telemetry extension ≠ DB behavior extension

No deberá poder modificar:

```text
Query
Transaction
Connection
Persistence
```

mediante una extensión de telemetry.

---

# 211. Telemetry enrichment

```php
interface DatabaseTelemetryEnricher
{
    public function enrich(
        DatabaseTelemetrySignal $signal,
        DatabaseTelemetryContext $context,
    ): TelemetryAttributes;
}
```

---

# 212. Enricher constraints

Deberá evitar:

```text
DB queries
network calls
lazy loading
expensive reflection
```

en hot path.

---

# 213. Expensive enrichment

Podrá diferirse o habilitarse solo en diagnostic mode.

---

# 214. Explain telemetry

API conceptual:

```php
DB::telemetry()->explain();
```

---

# 215. Example

```text
DATABASE TELEMETRY

Enabled:
  yes

Metrics:
  enabled

Tracing:
  enabled

Structured Logging:
  enabled

Provider:
  OpenTelemetry

Query Instrumentation:
  duration: enabled
  fingerprint: enabled
  SQL text: disabled
  bindings: disabled

Sampling:
  parent_based

Diagnostics:
  request buffer: 500 entries

Security:
  credentials: forbidden
  parameter values: redacted
  entity values: disabled

Cardinality:
  TenantId: forbidden for metrics
  QueryId: forbidden for metrics
  ShardId: bounded

Runtime:
  request scope: isolated
  worker state: safe
```

---

# 216. Explain signal

Conceptualmente:

```php
DB::telemetry()->explainSignal(
    'db.query.duration'
);
```

Salida:

```text
SIGNAL
  db.query.duration

Type:
  HISTOGRAM

Unit:
  seconds

Owner:
  QueryExecutor

Attributes:
  db.system
  db.operation
  db.status
  connection.role

Forbidden:
  query_id
  tenant_id
  raw_sql
  bindings

Sampling:
  metric always recorded

Provider:
  OpenTelemetry
```

---

# 217. Telemetry diagnostics

Deberá poder detectar:

```text
high-cardinality attributes
duplicate metric definitions
provider failures
unclosed spans
scope leakage
oversized diagnostic buffers
unsafe SQL capture
unsafe binding capture
```

---

# 218. Scope leak detection

En development/testing:

```text
request ended
+
active DB span remains
```

→ diagnostic.

---

# 219. Unclosed transaction span

Igualmente deberá detectarse.

---

# 220. Telemetry testing

El framework deberá ofrecer:

```text
InMemoryTelemetryProvider
```

para tests.

---

# 221. Test assertions

Ejemplos:

```php
Telemetry::assertMetricRecorded(
    'db.query.executions'
);

Telemetry::assertSpanRecorded(
    'db.transaction'
);
```

---

# 222. Sensitive-data assertion

```php
Telemetry::assertNotExported(
    'secret-password'
);
```

podrá existir en testing.

---

# 223. Cardinality testing

Podrá verificarse que una métrica no use:

```text
tenant_id
query_id
user_id
```

como labels.

---

# 224. Deterministic testing clock

Tests deberán poder inyectar:

```text
FakeTelemetryClock
```

---

# 225. Deterministic IDs

Podrán utilizarse factories controladas:

```text
TraceIdFactory
SpanIdFactory
QueryIdFactory
```

para tests.

---

# 226. Benchmarking

Telemetry deberá medirse con:

```text
disabled baseline
metrics only
metrics + traces
full diagnostics
```

---

# 227. Performance target

No deberá establecerse una cifra universal sin benchmarks reales.

En cambio:

```text
Telemetry overhead
```

será una dimensión formal del Performance Model.

---

# 228. Zero-cost claim

VoltStack no deberá afirmar:

```text
zero-cost telemetry
```

si existe cualquier instrumentación.

Preferir:

```text
minimal disabled-path overhead
```

---

# 229. Telemetry memory model

Memoria consumida aproximadamente:

```text
M =
ActiveSpans
+
DiagnosticBuffer
+
MetricState
+
AggregationState
+
ProviderBuffers
```

---

# 230. Memory boundedness

Los componentes bajo control de Database deberán tener límites.

---

# 231. Provider buffering

Buffers internos del provider externo estarán gobernados por ese provider/adaptador.

VoltStack deberá configurar límites cuando sea posible.

---

# 232. Backpressure

Si un provider no puede consumir telemetry suficientemente rápido:

```text
Database
```

no deberá quedar bloqueada indefinidamente por default.

---

# 233. Backpressure strategies

```text
DROP
SAMPLE
BUFFER_BOUNDED
DEGRADE
```

---

# 234. Blocking export

Solo mediante configuración explícita.

---

# 235. Shutdown flush

En procesos tradicionales podrá intentarse:

```text
flush telemetry
```

al shutdown.

---

# 236. Persistent worker flush

No debe esperar al shutdown del worker para datos request-scoped.

---

# 237. Request finalization

```text
Request End
↓
Finalize DB spans
↓
Flush scoped diagnostics
↓
Export if required
↓
Reset scope
```

---

# 238. Export failure at request end

No deberá cambiar una respuesta DB exitosa a failure por default.

---

# 239. CLI

Una operación CLI podrá tener:

```text
operation trace
```

independiente de HTTP.

---

# 240. Queue integration

Un queue job podrá propagar:

```text
TraceContext
```

desde el sistema de colas.

Database solo consumirá ese contexto.

---

# 241. Cross-process propagation

Será responsabilidad de la capa que transporta la operación:

```text
HTTP
Queue
Messaging
```

no de Database.

---

# 242. Multitenancy integration

Cuando `NeuronTenant` o el paquete oficial Multitenancy esté instalado:

```text
TenantContext
↓
DatabaseTelemetryContext
```

bajo policy.

---

# 243. Tenant context safety

No se deberá inferir tenant desde:

```text
static global variable
```

---

# 244. Cross-tenant administrative operations

Deberán representarse explícitamente.

No utilizar un tenant ficticio.

---

# 245. Database naming

Telemetry deberá diferenciar:

```text
physical database
logical database
persistence domain
tenant
shard
```

---

# 246. Database name sensitivity

Nombres físicos de DB pueden ser infraestructura sensible.

Su exportación será configurable.

---

# 247. Driver telemetry

Driver podrá exponer:

```text
network execution duration
protocol failure
server error code
```

pero no deberá duplicar métricas del Executor.

---

# 248. Layer-specific timings

Podrá distinguirse:

```text
query.total.duration
query.driver.duration
query.hydration.duration
```

si existe instrumentación fiable.

---

# 249. Timing decomposition

Idealmente:

```text
Total Query Operation
=
Planning
+
Compilation
+
Connection Wait
+
Driver Execution
+
Result Consumption
+
Hydration
```

según el tipo de operación.

---

# 250. Overlapping phases

No siempre son estrictamente aditivas.

Por tanto:

```text
Total
≠
necessarily exact sum of all spans
```

cuando existen fases superpuestas/streaming.

---

# 251. Streaming telemetry

Una consulta streaming tiene dos tiempos relevantes:

```text
time_to_first_result
total_stream_lifetime
```

---

# 252. Cursor lifetime

También puede medirse:

```text
result_cursor.open_duration
```

---

# 253. Long-lived cursor warning

Un cursor abierto demasiado tiempo puede:

```text
hold connection
hold transaction
consume server resources
```

y deberá ser diagnosticable.

---

# 254. Lazy collection resource lifetime

Igualmente:

```text
Lazy Collection Definition Lifetime
≠
Active Source Lifetime
```

Solo la fuente abierta deberá contabilizar recursos activos.

---

# 255. Cancellation telemetry

Podrá distinguir:

```text
user cancellation
deadline cancellation
query timeout
runtime shutdown
```

---

# 256. Cancellation ≠ failure

Según contexto:

```text
CANCELLED
```

deberá conservarse como outcome propio.

---

# 257. Deadline telemetry

Podrá registrar:

```text
deadline exceeded
remaining budget
```

sin incluir timestamps innecesariamente sensibles.

---

# 258. Resource governance telemetry

Podrá observar:

```text
memory budget
query budget
connection budget
time budget
row budget
```

---

# 259. Budget violation

Debe indicar:

```text
which resource
which policy
which operation category
```

---

# 260. Database telemetry error hierarchy

```text
DatabaseTelemetryException
├── TelemetryConfigurationException
├── TelemetryRegistrationException
├── TelemetryProviderException
├── TelemetryExportException
├── TelemetrySecurityException
├── TelemetryCardinalityException
├── TelemetrySamplingException
├── TelemetryScopeException
├── TelemetryBudgetException
├── TelemetryInstrumentationException
└── TelemetryDiagnosticException
```

---

# 261. Telemetry exception handling

Estas exceptions deberán ser capturadas en boundaries apropiados según:

```text
DatabaseTelemetryFailurePolicy
```

---

# 262. Directory structure

```text
src/Quantum/Database/Telemetry/
│
├── Contract/
│   ├── DatabaseTelemetryProvider.php
│   ├── DatabaseTelemetryCollector.php
│   ├── DatabaseMeter.php
│   ├── DatabaseTracer.php
│   ├── DatabaseTelemetryLogger.php
│   ├── DatabaseTelemetryEnricher.php
│   ├── DatabaseTelemetrySampler.php
│   └── DatabaseTelemetrySecurityPolicy.php
│
├── Context/
│   ├── DatabaseTelemetryContext.php
│   ├── DatabaseTelemetryScope.php
│   ├── DatabaseTelemetryContextResolver.php
│   └── DatabaseTelemetryContextFactory.php
│
├── Signal/
│   ├── DatabaseTelemetrySignal.php
│   ├── DatabaseTelemetrySignalType.php
│   ├── TelemetrySignalName.php
│   ├── DatabaseTelemetryDescriptor.php
│   └── CompiledTelemetryDescriptor.php
│
├── Metric/
│   ├── DatabaseMetric.php
│   ├── DatabaseCounter.php
│   ├── DatabaseHistogram.php
│   ├── DatabaseGauge.php
│   ├── DatabaseMetricType.php
│   └── DatabaseMetricRegistry.php
│
├── Trace/
│   ├── DatabaseTracer.php
│   ├── DatabaseSpan.php
│   ├── DatabaseSpanContext.php
│   ├── DatabaseSpanOutcome.php
│   ├── NullDatabaseSpan.php
│   └── TraceContextBridge.php
│
├── Log/
│   ├── DatabaseTelemetryLogger.php
│   ├── DatabaseStructuredLog.php
│   ├── DatabaseLogLevel.php
│   └── DatabaseLogContext.php
│
├── Attribute/
│   ├── TelemetryAttribute.php
│   ├── TelemetryAttributeName.php
│   ├── TelemetryValue.php
│   ├── TelemetrySensitivity.php
│   ├── TelemetryCardinality.php
│   └── TelemetryAttributeSet.php
│
├── Fingerprint/
│   ├── QueryFingerprint.php
│   ├── QueryFingerprintGenerator.php
│   ├── SemanticQueryFingerprintGenerator.php
│   └── RawSqlFingerprintGenerator.php
│
├── Sampling/
│   ├── DatabaseTelemetrySampler.php
│   ├── TelemetrySamplingDecision.php
│   ├── ParentBasedSampler.php
│   ├── ProbabilisticSampler.php
│   ├── SlowOperationSampler.php
│   └── ErrorBasedSampler.php
│
├── Cardinality/
│   ├── DatabaseTelemetryCardinalityPolicy.php
│   ├── CardinalityDecision.php
│   └── CardinalityGovernor.php
│
├── Security/
│   ├── DatabaseTelemetrySecurityPolicy.php
│   ├── DatabaseTelemetryRedactor.php
│   ├── DatabaseTelemetryClassifier.php
│   └── SensitiveTelemetryValue.php
│
├── Pipeline/
│   ├── DatabaseTelemetryPipeline.php
│   ├── DatabaseTelemetryProcessor.php
│   ├── DatabaseTelemetryAggregator.php
│   └── DatabaseTelemetryExporter.php
│
├── Instrumentation/
│   ├── DatabaseInstrumentation.php
│   ├── QueryInstrumentation.php
│   ├── ConnectionInstrumentation.php
│   ├── TransactionInstrumentation.php
│   ├── OrmInstrumentation.php
│   ├── PersistenceInstrumentation.php
│   └── HydrationInstrumentation.php
│
├── Provider/
│   ├── Null/
│   │   └── NullDatabaseTelemetryProvider.php
│   │
│   ├── Memory/
│   │   └── InMemoryDatabaseTelemetryProvider.php
│   │
│   └── OpenTelemetry/
│       ├── OpenTelemetryDatabaseProvider.php
│       ├── OpenTelemetryMeterAdapter.php
│       └── OpenTelemetryTracerAdapter.php
│
├── Diagnostics/
│   ├── DatabaseTelemetryDiagnostics.php
│   ├── DatabaseTelemetryInspector.php
│   ├── DatabaseTelemetryExplainer.php
│   ├── DatabaseDiagnosticBuffer.php
│   └── TelemetryRecursionGuard.php
│
├── Runtime/
│   ├── DatabaseTelemetryBudget.php
│   ├── DatabaseTelemetryRuntimeScope.php
│   ├── DatabaseTelemetryScopeResetter.php
│   └── DatabaseTelemetryFailurePolicy.php
│
├── Registry/
│   ├── DatabaseTelemetryRegistry.php
│   ├── MutableDatabaseTelemetryRegistry.php
│   ├── CompiledDatabaseTelemetryRegistry.php
│   └── DatabaseTelemetryRegistryCompiler.php
│
├── Testing/
│   ├── FakeTelemetryClock.php
│   ├── RecordingDatabaseTelemetryProvider.php
│   ├── DatabaseTelemetryAssertions.php
│   └── DatabaseTelemetrySnapshot.php
│
└── Exception/
    ├── DatabaseTelemetryException.php
    ├── TelemetryConfigurationException.php
    ├── TelemetryRegistrationException.php
    ├── TelemetryProviderException.php
    ├── TelemetryExportException.php
    ├── TelemetrySecurityException.php
    ├── TelemetryCardinalityException.php
    ├── TelemetrySamplingException.php
    ├── TelemetryScopeException.php
    ├── TelemetryBudgetException.php
    ├── TelemetryInstrumentationException.php
    └── TelemetryDiagnosticException.php
```

---

# 263. Container integration

Servicios conceptuales:

```text
DatabaseTelemetryProvider
DatabaseTelemetryRegistry
DatabaseTelemetryPipeline
DatabaseTelemetrySecurityPolicy
DatabaseTelemetryCardinalityPolicy
DatabaseTelemetrySampler
TelemetryClock
```

---

# 264. Shared services

Podrán ser singleton si son seguros:

```text
CompiledTelemetryRegistry
Telemetry Configuration
Metric Descriptors
Stateless Policies
```

---

# 265. Scoped services

Deberán ser scoped:

```text
DatabaseTelemetryContext
Diagnostic Buffer
Query Timeline
N+1 Detector State
Active Span Registry
Operation Aggregator
```

---

# 266. Query execution integration

```text
QueryExecutor
     │
     ▼
QueryTelemetryScope.start()
     │
     ▼
Execution Engine
     │
     ├── success
     ├── failure
     └── cancellation
     │
     ▼
QueryTelemetryScope.finish(outcome)
     │
     ├── metric
     ├── span
     ├── optional log
     └── diagnostic
```

---

# 267. Transaction integration

```text
TransactionManager
      │
      ▼
TransactionTelemetryScope
      │
      ├── begin
      ├── savepoint
      ├── retry
      ├── commit
      ├── rollback
      └── unknown
      │
      ▼
Telemetry Signals
```

---

# 268. ORM integration

```text
EntityManager
     │
     ▼
Flush
     │
     ▼
ORM Telemetry Scope
     │
     ├── managed entities
     ├── change sets
     ├── persistence operations
     ├── queries
     └── duration
```

---

# 269. Full instrumentation flow

```text
Application
    │
    ▼
EntityManager / Query API
    │
    ▼
Query Engine
    │
    ▼
Query Executor ────────────────┐
    │                          │
    ▼                          ▼
Transaction Manager ─────► Telemetry Context
    │                          │
    ▼                          ▼
Connection Manager ──────► Instrumentation
    │                          │
    ▼                          ▼
Driver                    Signal Pipeline
                               │
                      ┌────────┼────────┐
                      ▼        ▼        ▼
                   Metrics   Traces    Logs
                      │        │        │
                      └────────┼────────┘
                               ▼
                         Provider Adapter
```

---

# 270. Architectural invariants

## DB-TEL-001
Telemetry será opcional para Database correctness.

## DB-TEL-002
Telemetry failure no implicará Database failure por default.

## DB-TEL-003
Telemetry será distinta de Database Events.

## DB-TEL-004
Telemetry será distinta de Logging.

## DB-TEL-005
Telemetry será distinta de Audit.

## DB-TEL-006
Telemetry será distinta de Debugging.

## DB-TEL-007
Telemetry será distinta de Profiling.

## DB-TEL-008
Telemetry será distinta de Health Check.

## DB-TEL-009
Telemetry no será Query Engine.

## DB-TEL-010
Telemetry no será Query Optimizer.

## DB-TEL-011
Telemetry no será Query Planner.

## DB-TEL-012
Telemetry no será Connection Router.

## DB-TEL-013
Telemetry no será Transaction Manager.

## DB-TEL-014
Telemetry no será Persistence Engine.

## DB-TEL-015
Telemetry no será Authorization System.

## DB-TEL-016
Telemetry no será Retry Engine.

## DB-TEL-017
Database Event será distinto de Telemetry Signal.

## DB-TEL-018
Una operación podrá generar múltiples signals.

## DB-TEL-019
Instrumentation owner será explícito.

## DB-TEL-020
Query execution timing pertenecerá al Executor.

## DB-TEL-021
Connection acquisition timing pertenecerá al Connection System.

## DB-TEL-022
Transaction lifecycle timing pertenecerá al Transaction System.

## DB-TEL-023
ORM flush timing pertenecerá al ORM.

## DB-TEL-024
Telemetry Context no será global mutable state.

## DB-TEL-025
OperationId será distinto de QueryId.

## DB-TEL-026
QueryId será distinto de QueryFingerprint.

## DB-TEL-027
QueryFingerprint representará query shape.

## DB-TEL-028
QueryFingerprint no contendrá bindings por default.

## DB-TEL-029
Semantic fingerprint será preferido cuando Query Model esté disponible.

## DB-TEL-030
Raw SQL fingerprint tendrá normalización segura.

## DB-TEL-031
Bindings estarán redacted por default.

## DB-TEL-032
Binding types podrán observarse bajo policy.

## DB-TEL-033
Raw SQL será considerado potencialmente sensible.

## DB-TEL-034
Metric names serán stable.

## DB-TEL-035
Metric units serán explícitas.

## DB-TEL-036
Metric cardinality será gobernada.

## DB-TEL-037
QueryId no será metric label.

## DB-TEL-038
RequestId no será metric label.

## DB-TEL-039
TraceId no será metric label.

## DB-TEL-040
TransactionId no será metric label.

## DB-TEL-041
ConnectionId no será metric label.

## DB-TEL-042
EntityId no será metric label.

## DB-TEL-043
TenantId no será metric label por default.

## DB-TEL-044
Hashing no se considerará reducción automática de cardinalidad.

## DB-TEL-045
Histograms serán usados para distribuciones temporales.

## DB-TEL-046
Database podrá participar en parent traces.

## DB-TEL-047
Database no será propietario del global trace.

## DB-TEL-048
Tracing desactivado tendrá Null Span fast path.

## DB-TEL-049
Sampling no cambiará Database semantics.

## DB-TEL-050
Slow-query sampling podrá aumentar detalle.

## DB-TEL-051
Error sampling podrá aumentar detalle.

## DB-TEL-052
Logs serán estructurados.

## DB-TEL-053
Cada query exitosa no se logueará en producción por default.

## DB-TEL-054
Duplicate logging deberá minimizarse.

## DB-TEL-055
Telemetry tendrá security classification.

## DB-TEL-056
SECRET nunca será exportado.

## DB-TEL-057
Credentials nunca serán telemetry payload.

## DB-TEL-058
Raw bindings serán SENSITIVE.

## DB-TEL-059
Entity values no serán capturados por default.

## DB-TEL-060
Data minimization precederá redaction cuando sea posible.

## DB-TEL-061
Connection credentials nunca serán registradas.

## DB-TEL-062
Server addresses podrán estar sujetos a policy.

## DB-TEL-063
ORM telemetry no exportará Entity objects.

## DB-TEL-064
EntityType será distinto de EntityId.

## DB-TEL-065
Transaction UNKNOWN permanecerá UNKNOWN.

## DB-TEL-066
Telemetry no inferirá commit desde intento de COMMIT.

## DB-TEL-067
ConnectionId no será metric label.

## DB-TEL-068
Replica lag UNKNOWN no será tratado como cero.

## DB-TEL-069
Distributed PARTIAL permanecerá PARTIAL.

## DB-TEL-070
Physical cache hit será distinto de usable hit.

## DB-TEL-071
Rows serán distintos de hydrated entities.

## DB-TEL-072
N+1 telemetry no será auto query rewrite.

## DB-TEL-073
Large dataset telemetry será bounded.

## DB-TEL-074
Per-row telemetry estará deshabilitada por default.

## DB-TEL-075
Lazy Collection no emitirá per-item telemetry por default.

## DB-TEL-076
Telemetry scopes serán explícitos.

## DB-TEL-077
Mutable telemetry state usará mínimo scope necesario.

## DB-TEL-078
Worker state no almacenará current tenant.

## DB-TEL-079
Worker state no almacenará current transaction.

## DB-TEL-080
Null Collector estará disponible.

## DB-TEL-081
Disabled fast path minimizará allocations.

## DB-TEL-082
Telemetry budget será bounded.

## DB-TEL-083
Budget exhaustion no fallará DB por default.

## DB-TEL-084
Core dependerá de telemetry contracts, no vendor SDK.

## DB-TEL-085
OpenTelemetry será integración opcional.

## DB-TEL-086
PSR Logger será integración, no arquitectura completa.

## DB-TEL-087
Provider failure será aislado por default.

## DB-TEL-088
Telemetry failure recursion será impedida.

## DB-TEL-089
Provider circuit breaker podrá existir.

## DB-TEL-090
PROPAGATE no será failure policy default.

## DB-TEL-091
Diagnostic buffers serán bounded.

## DB-TEL-092
Persistent worker no acumulará query history indefinidamente.

## DB-TEL-093
Query profiler será distinto de Query Optimizer.

## DB-TEL-094
Slow query será distinta de incorrect query.

## DB-TEL-095
Debug Toolbar será consumidor externo.

## DB-TEL-096
Telemetry self-observation será bounded.

## DB-TEL-097
Telemetry recursion guard será scope-local.

## DB-TEL-098
Request telemetry será reset al finalizar request.

## DB-TEL-099
RequestId no sobrevivirá al siguiente request.

## DB-TEL-100
TraceId no sobrevivirá accidentalmente al siguiente request.

## DB-TEL-101
TenantId no sobrevivirá accidentalmente al siguiente request.

## DB-TEL-102
FrankenPHP shared telemetry state será persistent-safe.

## DB-TEL-103
RoadRunner tendrá fresh operation context por job.

## DB-TEL-104
OpenSwoole tendrá coroutine-safe telemetry state.

## DB-TEL-105
Concurrent query contexts no se mezclarán.

## DB-TEL-106
Singleton shared timer estará prohibido.

## DB-TEL-107
Durations utilizarán monotonic clock cuando sea posible.

## DB-TEL-108
Wall clock será distinto de duration clock.

## DB-TEL-109
Secure production defaults estarán habilitados.

## DB-TEL-110
Development environment no será autorización para secrets.

## DB-TEL-111
Dangerous telemetry configuration será validada.

## DB-TEL-112
Telemetry attributes tendrán sensitivity classification.

## DB-TEL-113
Telemetry attributes tendrán cardinality classification.

## DB-TEL-114
UNKNOWN cardinality no será LOW.

## DB-TEL-115
Providers no recibirán live DB objects.

## DB-TEL-116
Query AST se convertirá a safe diagnostic representation.

## DB-TEL-117
Exception messages serán consideradas potencialmente sensibles.

## DB-TEL-118
Stack traces estarán sujetas a policy.

## DB-TEL-119
Failure categories serán preferidas sobre message parsing.

## DB-TEL-120
Retry attempt outcome será distinto de final outcome.

## DB-TEL-121
Telemetry no resolverá unknown transaction outcome.

## DB-TEL-122
Direct instrumentation y event-derived telemetry podrán coexistir.

## DB-TEL-123
Double counting deberá prevenirse.

## DB-TEL-124
Cada metric tendrá canonical instrumentation owner.

## DB-TEL-125
Telemetry descriptors serán compilables.

## DB-TEL-126
Duplicate metric names serán detectados.

## DB-TEL-127
Metric unit conflicts serán detectados.

## DB-TEL-128
Telemetry extensions no modificarán Database behavior.

## DB-TEL-129
Enrichers no ejecutarán queries por default.

## DB-TEL-130
Enrichers no activarán lazy loading.

## DB-TEL-131
Enrichers no realizarán network I/O en hot path por default.

## DB-TEL-132
Telemetry architecture será explainable.

## DB-TEL-133
Telemetry diagnostics detectarán unsafe capture.

## DB-TEL-134
Unclosed spans serán diagnosticables.

## DB-TEL-135
Testing provider será determinista.

## DB-TEL-136
Testing podrá verificar ausencia de sensitive data.

## DB-TEL-137
Telemetry overhead será benchmarked.

## DB-TEL-138
VoltStack no prometerá zero-cost telemetry.

## DB-TEL-139
Memory bajo control de Database será bounded.

## DB-TEL-140
Backpressure no bloqueará Database indefinidamente por default.

## DB-TEL-141
Persistent worker no esperará shutdown para limpiar request telemetry.

## DB-TEL-142
Request finalization cerrará scoped spans.

## DB-TEL-143
Telemetry export failure no cambiará DB success por default.

## DB-TEL-144
CLI tendrá operation-scoped telemetry.

## DB-TEL-145
Queue trace propagation pertenecerá a queue/messaging layer.

## DB-TEL-146
Database consumirá propagated TraceContext.

## DB-TEL-147
Multitenancy telemetry respetará TenantContext.

## DB-TEL-148
Cross-tenant operations serán explícitas.

## DB-TEL-149
Physical database será distinto de logical database.

## DB-TEL-150
Physical DB names podrán ser sensitive.

## DB-TEL-151
Driver telemetry no duplicará Executor metrics sin razón.

## DB-TEL-152
Layer timings serán diferenciables.

## DB-TEL-153
Overlapping timings no serán asumidos aditivos.

## DB-TEL-154
Streaming medirá time-to-first-result cuando sea útil.

## DB-TEL-155
Streaming podrá medir total source lifetime.

## DB-TEL-156
Long-lived cursor será diagnosticable.

## DB-TEL-157
Lazy definition lifetime será distinto de source lifetime.

## DB-TEL-158
Cancellation será distinta de failure.

## DB-TEL-159
Resource governance violations serán telemetry-visible.

## DB-TEL-160
Telemetry exceptions respetarán failure policy.

## DB-TEL-161
Telemetry nunca será fuente primaria de Database truth.

## DB-TEL-162
Telemetry nunca sustituirá Transaction state.

## DB-TEL-163
Telemetry nunca sustituirá IdentityMap.

## DB-TEL-164
Telemetry nunca sustituirá Cache consistency.

## DB-TEL-165
Telemetry nunca sustituirá Security policy.

## DB-TEL-166
Telemetry nunca sustituirá Audit cuando auditabilidad sea requisito.

## DB-TEL-167
Telemetry podrá degradarse sin degradar correctness.

## DB-TEL-168
Telemetry podrá apagarse completamente.

## DB-TEL-169
Provider-specific concepts permanecerán detrás de adapters.

## DB-TEL-170
Database Telemetry permanecerá desacoplada del backend de observabilidad.

---

# 271. Modelo formal

Sea una operación de Database:

```text
O
```

y su resultado funcional:

```text
R(O)
```

Telemetry produce:

```text
T(O)
```

donde:

```text
T(O) =
{
    metrics,
    traces,
    logs,
    diagnostics
}
```

La relación fundamental será:

```text
R(O)
```

no deberá depender de:

```text
Success(T(O))
```

por default.

Es decir:

```text
¬Success(T(O))
↛
¬Success(R(O))
```

---

# 272. Modelo de medición

Para una operación:

```text
O = [t0, t1]
```

su duración será:

```text
D(O) = M(t1) - M(t0)
```

donde:

```text
M
```

es un reloj monotónico.

---

# 273. Modelo de cardinalidad

Sea:

```text
A = {a1, a2, ..., an}
```

el conjunto de atributos de una métrica.

Su cardinalidad aproximada puede crecer como:

```text
C ≈ Π cardinality(ai)
```

Por ello, incluso atributos individualmente razonables pueden producir explosión combinatoria.

Ejemplo:

```text
20 operations
×
10 platforms
×
1000 tenants
×
500 shards
=
100,000,000 combinations
```

De ahí la necesidad de `CardinalityGovernor`.

---

# 274. Modelo de sampling

Para una operación `O`:

```text
S(O) ∈ {RECORD, DROP, DEFER}
```

La decisión de sampling afecta:

```text
observability detail
```

pero no:

```text
database semantics
```

---

# 275. Modelo de seguridad

Sea:

```text
Raw(O)
```

la información disponible internamente.

La señal exportable será:

```text
Export(O)
=
Redact(
    Minimize(
        Raw(O)
    )
)
```

bajo:

```text
SecurityPolicy
```

Por tanto:

```text
Export(O)
⊆
Raw(O)
```

conceptualmente.

---

# 276. Modelo de correlación

Una query podrá relacionarse mediante:

```text
OperationId
TransactionId
QueryId
TraceId
SpanId
```

sin convertir esos identificadores en labels métricos de alta cardinalidad.

---

# 277. Modelo de degradación

Si:

```text
TelemetryProvider = unavailable
```

entonces:

```text
TelemetryState = DEGRADED
```

pero:

```text
DatabaseState
```

permanece independiente.

---

# 278. Arquitectura final

```text
                        VoltStack Application
                                 │
                                 ▼
                      Quantum Database API
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
          Query Engine          ORM          Persistence
               │                 │                 │
               └──────────┬──────┴───────┬─────────┘
                          ▼              ▼
                    Transaction      Hydration
                          │              │
                          └──────┬───────┘
                                 ▼
                         Execution Engine
                                 │
                                 ▼
                       Connection Manager
                                 │
                                 ▼
                              Driver

         ───────────────── TELEMETRY ─────────────────

      Query      Connection      Transaction       ORM
        │            │               │              │
        ▼            ▼               ▼              ▼
   Instrument   Instrument       Instrument      Instrument
        │            │               │              │
        └────────────┴───────┬───────┴──────────────┘
                             ▼
                   DatabaseTelemetryContext
                             │
                             ▼
                    Telemetry Pipeline
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
          Metrics          Traces            Logs
             │               │                │
             └───────────────┼────────────────┘
                             ▼
                         Diagnostics
                             │
                             ▼
                    Provider Abstraction
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
          Null           OpenTelemetry      Custom
```

---

# 279. Filosofía arquitectónica

VoltStack seguirá:

```text
Observability
over
blind execution

Typed signals
over
unstructured strings

Semantic fingerprints
over
raw SQL values

Low cardinality
over
label explosion

Data minimization
over
capture everything

Explicit scopes
over
global telemetry state

Monotonic timing
over
wall-clock duration

Provider abstraction
over
vendor coupling

Bounded diagnostics
over
unlimited history

Graceful degradation
over
telemetry-driven outages
```

---

# 280. Regla maestra

La arquitectura completa puede resumirse como:

> **Database produce hechos operacionales; Telemetry los observa, mide y correlaciona mediante contratos seguros, acotados y provider-agnostic, sin adquirir autoridad sobre la ejecución que está observando.**

Por tanto:

```text
Database
        │
        ├── produces operational truth
        │
        ▼
Telemetry
        │
        ├── measures
        ├── correlates
        ├── aggregates
        ├── traces
        └── diagnoses
```

pero nunca:

```text
Telemetry
↓
defines Database truth
```

---

# 281. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
○ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
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

# 282. Siguiente documento

```text
217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
```

El siguiente documento definirá específicamente la instrumentación del ciclo completo de una consulta:

```text
Query Model
↓
Semantic Analysis
↓
Optimization
↓
Planning
↓
Compilation
↓
Execution
↓
Result Consumption
↓
Hydration
```

incluyendo:

```text
QueryId
QueryFingerprint
query metrics
query spans
query timings
phase timings
rows returned
rows affected
execution outcomes
query errors
query cancellation
query retries
prepared statements
cache interaction
replica/shard context
raw SQL security
binding redaction
query sampling
query cardinality
streaming lifetime
query diagnostics
persistent-runtime isolation
```

bajo una regla central:

> **Query Telemetry podrá describir con precisión cómo se construyó, ejecutó y consumió una consulta, pero nunca deberá modificar su AST, plan, SQL, parámetros, routing, resultado ni outcome.**