# 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md

# VoltStack Quantum Database
## Debug Information System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 224 — Debug Information System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md`  
**Siguiente documento:** `225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md`

---

# 1. Propósito

`Debug Information System` define la representación diagnóstica central mediante la cual VoltStack Database transformará evidencia técnica proveniente de múltiples subsistemas en información:

```text
estructurada
inmutable
redactada
serializable
acotada
segura
correlacionable
provider-agnostic
```

para ser consumida posteriormente por:

```text
Developer Debug Toolbar
CLI diagnostics
Testing
Structured Logs
Telemetry exporters
Error pages
Administrative diagnostics
Development tools
```

La regla central será:

> **Debug Information representa una vista diagnóstica derivada del estado observado; nunca será el propio estado vivo del Database Runtime.**

Por tanto:

```text
DebugInformation
≠
Connection

DebugInformation
≠
EntityManager

DebugInformation
≠
UnitOfWork

DebugInformation
≠
IdentityMap

DebugInformation
≠
ResultCursor

DebugInformation
≠
TransactionContext

DebugInformation
≠
Query AST mutable
```

---

# 2. Problema arquitectónico

Un sistema de debugging ingenuo podría almacenar:

```php
[
    'connection' => $pdo,
    'entity_manager' => $entityManager,
    'query' => $query,
    'result' => $cursor,
]
```

Esto sería problemático porque introduce:

```text
retención de objetos runtime
memory leaks
state leakage
serialization failures
cross-request contamination
secret exposure
lazy-loading accidental
driver coupling
provider coupling
concurrency hazards
```

VoltStack deberá realizar siempre una transformación:

```text
Runtime State
     ↓
Safe Snapshot
     ↓
Debug Model
     ↓
Consumer
```

---

# 3. Posición arquitectónica

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry
├── ORM Telemetry
├── Query Profiler
├── Slow Query Detection
├── N+1 Telemetry
│
├── Debug Information System      ← este documento
│
└── Developer Debug Toolbar
```

---

# 4. Fuentes del sistema

Debug Information podrá recibir información de:

```text
Query Telemetry
Connection Telemetry
Transaction Telemetry
ORM Telemetry
Query Profiler
Slow Query Detection
N+1 Detection
Cache System
Schema System
Migration System
Read/Write Routing
Replica System
Sharding
Resource Governance
Runtime
Security
```

pero no deberá depender obligatoriamente de todos ellos.

---

# 5. Principio de snapshot

Toda información deberá convertirse en:

```text
value objects
primitive-safe values
bounded collections
stable identifiers
safe enums
redacted text
```

antes de entrar al Debug Information System.

---

# 6. Runtime object boundary

No se permitirá almacenar directamente:

```text
PDO
mysqli connection
driver handles
prepared statements
ResultCursor
EntityManager
UnitOfWork
IdentityMap
Entity instances
LazyCollection iterator
Transaction object
ConnectionLease object
Coroutine-local runtime objects
```

---

# 7. Distinciones fundamentales

```text
Debug Information
≠
Telemetry

Debug Information
≠
Logs

Debug Information
≠
Metrics

Debug Information
≠
Tracing

Debug Information
≠
Profiler

Debug Information
≠
Debug Toolbar

Debug Information
≠
Exception

Debug Information
≠
Runtime State

Debug Snapshot
≠
Live Object

Diagnostic
≠
Recommendation

Observed
≠
Inferred
```

---

# 8. Responsabilidad

El sistema deberá:

```text
collect safe diagnostic fragments
normalize them
correlate them
bound them
redact them
freeze them
serialize them
expose them safely
```

---

# 9. No responsabilidad

No deberá:

```text
execute queries
reroute connections
rollback transactions
hydrate entities
clear ORM state
run migrations
modify cache
restart workers
retry operations
change configuration
```

---

# 10. DebugInformationContext

```php
final readonly class DebugInformationContext
{
    public function __construct(
        public DebugSessionId $sessionId,
        public DebugScopeId $scopeId,
        public DebugEnvironment $environment,
        public DebugInformationPolicy $policy,
    ) {}
}
```

---

# 11. DebugSessionId

Representa una sesión diagnóstica.

```php
final readonly class DebugSessionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 12. DebugScopeId

Podrá representar:

```text
HTTP request
CLI command
Queue job
Test
Explicit profiling scope
Worker operation
```

---

# 13. DebugSession ≠ request

Una sesión de debugging puede envolver:

```text
one request
multiple internal operations
explicit diagnostic scope
test case
```

---

# 14. Debug environment

```php
enum DebugEnvironment
{
    case DEVELOPMENT;
    case TESTING;
    case STAGING;
    case PRODUCTION;
}
```

---

# 15. Environment ≠ authorization

Estar en `DEVELOPMENT` no autoriza automáticamente a ver secretos.

---

# 16. DebugInformation

Modelo raíz:

```php
final readonly class DatabaseDebugInformation
{
    public function __construct(
        public DebugSessionId $sessionId,
        public DebugSummary $summary,
        public DebugTimeline $timeline,
        public DebugSectionCollection $sections,
        public DebugDiagnosticCollection $diagnostics,
        public DebugSecurityMetadata $security,
        public DebugCoverage $coverage,
    ) {}
}
```

---

# 17. Inmutabilidad

Una vez finalizada:

```text
DatabaseDebugInformation
```

deberá ser immutable.

---

# 18. Debug information lifecycle

```text
OPEN
↓
COLLECTING
↓
FINALIZING
↓
REDACTING
↓
FROZEN
```

Estados alternos:

```text
PARTIAL
FAILED
DROPPED
```

---

# 19. Active collector ≠ finalized information

Durante ejecución podrá existir:

```text
mutable scoped collector
```

Después:

```text
immutable debug snapshot
```

---

# 20. Debug coverage

```php
enum DebugCoverage
{
    case COMPLETE;
    case PARTIAL;
    case MINIMAL;
    case UNKNOWN;
}
```

---

# 21. Coverage semantics

`PARTIAL` puede significar:

```text
telemetry sampling
budget exhaustion
unsupported driver detail
missing profiler data
disabled ORM telemetry
```

---

# 22. Missing ≠ zero

Si no sabemos:

```text
transaction count
```

no se representará necesariamente como:

```text
0
```

sino como:

```text
UNKNOWN
```

cuando la diferencia importe.

---

# 23. Debug section model

El documento utilizará secciones tipadas:

```text
Summary
Queries
Connections
Transactions
ORM
Cache
Slow Queries
N+1
Distribution
Schema
Runtime
Security
Diagnostics
```

---

# 24. DebugSection

```php
interface DebugSection
{
    public function id(): DebugSectionId;

    public function title(): string;

    public function priority(): int;
}
```

---

# 25. Section IDs

```php
enum DebugSectionId
{
    case SUMMARY;
    case QUERIES;
    case CONNECTIONS;
    case TRANSACTIONS;
    case ORM;
    case CACHE;
    case SLOW_QUERIES;
    case N_PLUS_ONE;
    case DISTRIBUTION;
    case SCHEMA;
    case RUNTIME;
    case SECURITY;
    case DIAGNOSTICS;
}
```

---

# 26. DebugSummary

Debe proporcionar una vista compacta:

```text
query count
query time
connection count
transactions
ORM operations
slow queries
N+1 detections
errors
warnings
coverage
```

---

# 27. Ejemplo

```text
DATABASE DEBUG SUMMARY
────────────────────────────

Queries:
  18

Database time:
  42 ms

Connection wait:
  6 ms

Transactions:
  1

ORM flushes:
  1

Slow queries:
  2

N+1 patterns:
  1

Warnings:
  3

Coverage:
  COMPLETE
```

---

# 28. Summary count ownership

Los valores deberán derivarse de los subsistemas canónicos.

No recalcular arbitrariamente si ya existe una medición oficial.

---

# 29. Query debug section

Podrá mostrar:

```text
QueryId
fingerprint
operation
source
duration
rows
role
transaction
status
cache
profile severity
```

---

# 30. Query SQL

La representación SQL dependerá de policy:

```text
NONE
FINGERPRINT
NORMALIZED_REDACTED
FULL
```

---

# 31. Default

Producción:

```text
FINGERPRINT
```

o:

```text
NORMALIZED_REDACTED
```

si está autorizado.

---

# 32. Query parameters

Por default:

```text
count
types
```

No valores.

---

# 33. QueryDebugRecord

```php
final readonly class QueryDebugRecord
{
    public function __construct(
        public QueryId $queryId,
        public ?QueryFingerprint $fingerprint,
        public QueryOperation $operation,
        public QueryOrigin $origin,
        public QueryOutcome $outcome,
        public Duration $duration,
        public DebugQueryText $queryText,
        public DebugParameterSummary $parameters,
        public QueryDebugContext $context,
    ) {}
}
```

---

# 34. Safe query text

```php
final readonly class DebugQueryText
{
    public function __construct(
        public DebugQueryTextMode $mode,
        public ?string $value,
        public bool $truncated,
    ) {}
}
```

---

# 35. Debug text ≠ executable SQL

Nunca reutilizar:

```text
debug SQL representation
```

como:

```text
CompiledQuery
```

---

# 36. Query timeline

Podrá mostrar:

```text
compile
connection wait
execution
consumption
hydration
ORM
```

cuando esté disponible.

---

# 37. Slow query marker

Cada query podrá incluir:

```text
slow=true
```

más referencia a su detection.

---

# 38. N+1 marker

Una query dependiente podrá incluir:

```text
n_plus_one_group=n1_x
```

solo en debug representation.

---

# 39. Connection debug section

Podrá mostrar:

```text
logical connection
role
endpoint alias
pool
acquisition duration
created/reused
lease duration
state
```

---

# 40. ConnectionDebugRecord

```php
final readonly class ConnectionDebugRecord
{
    public function __construct(
        public PhysicalConnectionId $connectionId,
        public ?PoolId $poolId,
        public ConnectionRole $role,
        public ConnectionAcquisitionOutcome $acquisition,
        public Duration $acquisitionDuration,
        public DebugEndpointView $endpoint,
    ) {}
}
```

---

# 41. Endpoint privacy

No mostrar por default:

```text
raw IP
full host
DSN
username
password
```

---

# 42. Endpoint alias

Preferir:

```text
writer-primary
replica-eu-02
analytics
```

o stable safe ID.

---

# 43. Transaction debug section

Podrá mostrar:

```text
TransactionOperationId
attempts
outcome
duration
isolation requested/effective
query count
retry
deadlock
savepoints
```

---

# 44. TransactionDebugRecord

```php
final readonly class TransactionDebugRecord
{
    public function __construct(
        public TransactionOperationId $operationId,
        public int $attempts,
        public TransactionOutcome $outcome,
        public Duration $duration,
        public TransactionIsolationDebugView $isolation,
        public int $queries,
    ) {}
}
```

---

# 45. UNKNOWN transaction outcome

Debe mostrarse explícitamente:

```text
UNKNOWN
```

y nunca maquillarse como:

```text
FAILED
```

---

# 46. ORM debug section

Podrá incluir:

```text
EntityManager scope
IdentityMap size
IdentityMap peak
UnitOfWork counts
flushes
hydration
lazy loads
eager loads
EntityManager state
```

---

# 47. No entity dump

Nunca mostrar automáticamente:

```php
var_dump($entity);
```

---

# 48. Entity type

Podrá mostrarse mediante:

```text
EntityTypeId
```

o safe FQCN en development si policy lo permite.

---

# 49. Entity identifiers

Por default:

```text
hidden
```

---

# 50. ORMDebugSnapshot

```php
final readonly class ORMDebugSnapshot
{
    public function __construct(
        public EntityManagerScopeId $scopeId,
        public ORMEntityManagerState $state,
        public int $identityMapSize,
        public int $identityMapPeak,
        public UnitOfWorkDebugSummary $unitOfWork,
        public int $flushCount,
        public int $lazyLoadCount,
    ) {}
}
```

---

# 51. Cache debug section

Podrá diferenciar:

```text
Query Cache
Result Cache
Metadata Cache
Entity Cache
Hydration Cache
Compiled Query Cache
```

---

# 52. Cache hit semantics

Debe preservar:

```text
PHYSICAL_HIT
USABLE_HIT
MISS
STALE_REJECTED
```

cuando corresponda.

---

# 53. Cache key protection

No mostrar cache key completa si puede incluir:

```text
tenant
parameters
business IDs
```

---

# 54. Cache debug representation

Preferir:

```text
cache kind
fingerprint
hit state
layer
duration
```

---

# 55. Slow query section

Deberá consumir detecciones del documento 222.

Podrá mostrar:

```text
classification
severity
duration
threshold
baseline
dominant phase
recommendation
```

---

# 56. N+1 section

Deberá consumir detecciones del documento 223.

Podrá mostrar:

```text
relationship
root operation
parents
dependent queries
aggregate cost
confidence
recommendation
source location
```

---

# 57. Distribution section

Podrá mostrar:

```text
single/multi-shard
shards involved
writer/replica roles
critical shard
fan-out count
merge duration
partial failures
```

---

# 58. Distribution safety

Shard IDs podrán mostrarse como:

```text
shard-01
```

si son safe internal IDs.

No necesariamente endpoints físicos.

---

# 59. Schema section

Podrá incluir:

```text
schema generation
metadata generation
migration state
schema introspection status
```

pero no deberá ejecutar introspection adicional solo para llenar el toolbar por default.

---

# 60. Debugging must not create hidden DB I/O

Regla fundamental:

> **Generar Debug Information no deberá ejecutar queries adicionales inesperadas.**

---

# 61. Consecuencia

No:

```text
debug panel opened
↓
run schema introspection
↓
run EXPLAIN
↓
run health checks
```

automáticamente.

---

# 62. Explicit diagnostic enrichment

Operaciones adicionales deberán ser:

```text
explicit
authorized
budgeted
safe
```

---

# 63. Runtime section

Podrá mostrar:

```text
runtime
worker mode
request scope
worker ID token
persistent runtime
connection reuse
reset state
```

---

# 64. Worker identifier

No deberá convertirse en información de infraestructura sensible sin policy.

---

# 65. FrankenPHP

Podrá mostrar:

```text
Runtime:
  FrankenPHP

Worker mode:
  persistent

Physical DB connections reused:
  yes
```

---

# 66. RoadRunner

Misma abstracción.

---

# 67. OpenSwoole

Podrá mostrar:

```text
coroutine-safe scope
```

sin exponer internals innecesarios.

---

# 68. Security section

No será una lista de secretos.

Deberá mostrar:

```text
redaction active
sensitive fields removed
unsafe diagnostics suppressed
query text policy
parameter policy
tenant isolation status
```

---

# 69. DebugSecurityMetadata

```php
final readonly class DebugSecurityMetadata
{
    public function __construct(
        public bool $redacted,
        public DebugQueryTextMode $queryTextMode,
        public DebugParameterMode $parameterMode,
        public int $suppressedFields,
        public bool $tenantIsolationApplied,
    ) {}
}
```

---

# 70. Debug Information Security Model

Pipeline:

```text
Telemetry Record
↓
Debug Projection
↓
Data Minimization
↓
Sensitive Classification
↓
Redaction
↓
Authorization Filter
↓
Serialization
↓
Consumer
```

---

# 71. Redaction before consumer

Nunca permitir:

```text
consumer
↓
decides what to hide
```

como única barrera.

El core deberá producir una vista segura.

---

# 72. Defense in depth

El consumidor también podrá aplicar filtros adicionales.

---

# 73. DebugRedactor

```php
interface DebugInformationRedactor
{
    public function redact(
        DatabaseDebugInformation $information,
        DebugRedactionContext $context,
    ): DatabaseDebugInformation;
}
```

---

# 74. Pre-projection minimization

Idealmente los secretos no deberían entrar nunca al modelo de debug.

Redaction posterior será segunda barrera.

---

# 75. Sensitive classification

```php
enum DebugSensitivity
{
    case PUBLIC;
    case INTERNAL;
    case SENSITIVE;
    case SECRET;
}
```

---

# 76. SECRET

Nunca deberá exportarse.

---

# 77. SENSITIVE

Solo bajo policy/authorization explícita.

---

# 78. INTERNAL

Puede mostrarse en development autorizado.

---

# 79. PUBLIC

Información operacional segura.

---

# 80. Passwords

Siempre:

```text
SECRET
```

---

# 81. Tokens

Siempre:

```text
SECRET
```

---

# 82. Query values

Normalmente:

```text
SENSITIVE
```

o `SECRET` según metadata.

---

# 83. SQL structure

Generalmente:

```text
INTERNAL
```

---

# 84. Tenant ID

Normalmente:

```text
SENSITIVE/INTERNAL
```

según contexto.

---

# 85. File path

Puede ser:

```text
INTERNAL
```

porque revela estructura del servidor.

---

# 86. Source location

Policy-controlled.

---

# 87. Serialization

Debug Information deberá ser serializable a formatos como:

```text
array
JSON
structured log record
debug toolbar payload
```

---

# 88. Serialization contract

```php
interface DebugInformationSerializer
{
    public function serialize(
        DatabaseDebugInformation $information
    ): DebugPayload;
}
```

---

# 89. Serialization ≠ PHP serialize()

Nunca depender de:

```php
serialize($runtimeObject);
```

---

# 90. Stable schema

El payload de debug deberá tener versión:

```text
debug_schema_version
```

---

# 91. DebugInformationVersion

```php
final readonly class DebugInformationVersion
{
    public function __construct(
        public int $major,
        public int $minor,
    ) {}
}
```

---

# 92. Versioning

Permitirá evolucionar:

```text
debug toolbar
IDE integrations
external tools
tests
```

sin depender de objetos internos.

---

# 93. Public debug schema ≠ internal class graph

Una buena frontera deberá permitir modificar internals sin romper consumers.

---

# 94. Debug payload example

```json
{
  "version": "1.0",
  "summary": {
    "queries": 18,
    "slow_queries": 2,
    "n_plus_one": 1
  },
  "queries": [],
  "transactions": [],
  "orm": {},
  "diagnostics": []
}
```

---

# 95. Bounded payload

El payload deberá tener límites.

---

# 96. DebugInformationBudget

```php
final readonly class DebugInformationBudget
{
    public function __construct(
        public int $maxQueries,
        public int $maxConnections,
        public int $maxTransactions,
        public int $maxDiagnostics,
        public int $maxNPlusOneDetections,
        public int $maxSlowQueries,
        public int $maxPayloadBytes,
    ) {}
}
```

---

# 97. Payload overflow

Cuando se excede el budget:

```text
truncate detail
retain summary
record dropped counts
```

---

# 98. Never unbounded

Prohibido:

```php
$queries[] = $query;
```

sin límite en persistent workers.

---

# 99. Query overflow

Si existen:

```text
50,000 queries
```

el panel puede conservar:

```text
first N
slowest N
representative samples
aggregate counts
```

---

# 100. Truncation metadata

Debe mostrar:

```text
queries_recorded: 500
queries_total: 50,000
queries_dropped: 49,500
```

---

# 101. Selection strategy

Podrá combinar:

```text
FIRST
LAST
SLOWEST
FAILED
SAMPLED
```

---

# 102. Diagnostic priority

Con budget limitado, prioridad:

```text
UNKNOWN outcomes
failures
critical slow queries
N+1 high severity
security diagnostics
warnings
ordinary query detail
```

---

# 103. DebugPriority

```php
enum DebugPriority
{
    case CRITICAL;
    case HIGH;
    case NORMAL;
    case LOW;
}
```

---

# 104. Priority ≠ severity

Priority decide retención/presentación.

Severity describe problema.

---

# 105. Timeline system

Una de las capacidades centrales será reconstruir:

```text
request
├── connection acquire
├── query
│   ├── execution
│   └── hydration
├── transaction
│   ├── query
│   └── commit
└── ORM flush
```

---

# 106. DebugTimeline

```php
final readonly class DebugTimeline
{
    public function __construct(
        public array $entries,
        public Duration $duration,
        public bool $truncated,
    ) {}
}
```

---

# 107. DebugTimelineEntry

```php
final readonly class DebugTimelineEntry
{
    public function __construct(
        public DebugTimelineEntryId $id,
        public DebugTimelineType $type,
        public Duration $offset,
        public Duration $duration,
        public DebugTimelineStatus $status,
        public ?DebugCorrelationId $parent,
    ) {}
}
```

---

# 108. Timeline clocks

Duraciones deben basarse en monotonic clock.

Offsets relativos pueden derivarse desde el inicio del scope.

---

# 109. No wall-clock ordering assumptions

Concurrent operations pueden solaparse.

---

# 110. Timeline overlap

Debe permitirse:

```text
Query A ─────────
   Query B ─────────
```

en runtimes concurrentes.

---

# 111. Parent-child ≠ temporal containment

Un child lógico puede no quedar perfectamente contenido si existe asynchronous continuation.

---

# 112. Correlation graph

Además de timeline, puede existir:

```text
Correlation Graph
```

para modelar relaciones lógicas.

---

# 113. DebugCorrelationId

```php
final readonly class DebugCorrelationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 114. Correlation types

```text
query → transaction
query → ORM operation
query → connection lease
query → N+1 detection
slow query → query profile
flush → transaction
relationship load → query
```

---

# 115. Graph ≠ runtime reference

Todos los enlaces serán mediante IDs.

---

# 116. Query detail retrieval

Toolbar podrá solicitar:

```text
query detail by DebugRecordId
```

sin tener acceso al Query object original.

---

# 117. DebugRecordId

Cada registro de debug podrá tener:

```php
final readonly class DebugRecordId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 118. DebugRecordId lifetime

Scoped a la sesión diagnóstica.

---

# 119. No persistent business identity

No reutilizar IDs de negocio como DebugRecordId.

---

# 120. Diagnostics model

Todos los diagnósticos deberán seguir una estructura común.

---

# 121. DebugDiagnostic

```php
final readonly class DebugDiagnostic
{
    public function __construct(
        public DebugDiagnosticCode $code,
        public DebugSeverity $severity,
        public DebugConfidence $confidence,
        public string $summary,
        public array $evidence,
        public array $recommendations,
        public ?DebugRecordId $relatedRecord,
    ) {}
}
```

---

# 122. Severity

```php
enum DebugSeverity
{
    case INFO;
    case NOTICE;
    case WARNING;
    case ERROR;
    case CRITICAL;
}
```

---

# 123. Confidence

```php
enum DebugConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 124. Evidence

Debe ser estructurada.

No solo texto libre.

---

# 125. Recommendation

También estructurada cuando sea posible.

---

# 126. Diagnostic example

```text
Code:
  DB_SLOW_HYDRATION

Severity:
  WARNING

Confidence:
  HIGH

Observed:
  hydration = 620 ms
  DB execution = 31 ms
  rows = 42,000

Recommendation:
  evaluate narrower projection
```

---

# 127. Diagnostic code registry

Los códigos deberán ser estables.

Ejemplo:

```text
DB_SLOW_QUERY
DB_SLOW_CONNECTION_WAIT
DB_N_PLUS_ONE
DB_TRANSACTION_UNKNOWN
DB_ENTITY_MANAGER_TAINTED
DB_POOL_SATURATION
DB_STATE_LEAK_CANDIDATE
```

---

# 128. Stable diagnostic codes

Permiten:

```text
IDE integration
testing
filtering
documentation links
support workflows
```

---

# 129. Diagnostic text may evolve

El código estable no deberá depender de wording exacto.

---

# 130. Documentation link

Opcionalmente:

```text
docs_reference
```

podrá acompañar un diagnostic.

---

# 131. No external network lookup required

Mostrar debug information no dependerá de internet.

---

# 132. Diagnostics aggregation

Diagnósticos repetidos podrán agruparse.

---

# 133. Example

En lugar de:

```text
N+1 warning × 200
```

mostrar:

```text
1 N+1 pattern
200 dependent queries
```

---

# 134. Error integration

Cuando una operación falla:

```text
DatabaseException
```

podrá enlazarse a:

```text
DebugDiagnostic
```

sin reemplazar la excepción.

---

# 135. Exception ≠ diagnostic

La excepción:

```text
control flow/error semantics
```

El diagnostic:

```text
developer explanation
```

---

# 136. Production behavior

En producción podrá existir Debug Information mínimo para:

```text
support IDs
telemetry correlation
safe errors
```

sin exposición al usuario final.

---

# 137. Production debug policy

Normalmente:

```text
query values = disabled
SQL = fingerprint/redacted
source paths = disabled
stack traces = disabled or internal-only
entity names = limited
```

---

# 138. Development policy

Podrá permitir más detalle:

```text
normalized SQL
source location
timeline
ORM stats
profile breakdown
recommendations
```

---

# 139. Testing policy

Podrá maximizar determinismo:

```text
stable IDs
fake clock
no nondeterministic timestamps
explicit ordering
```

---

# 140. DebugInformationPolicy

```php
final readonly class DebugInformationPolicy
{
    public function __construct(
        public bool $enabled,
        public DebugQueryTextMode $queryText,
        public DebugParameterMode $parameters,
        public bool $sourceLocations,
        public bool $timeline,
        public bool $diagnostics,
        public bool $runtimeDetails,
        public DebugInformationBudget $budget,
    ) {}
}
```

---

# 141. Query text modes

```php
enum DebugQueryTextMode
{
    case NONE;
    case FINGERPRINT;
    case NORMALIZED_REDACTED;
    case FULL;
}
```

---

# 142. Parameter modes

```php
enum DebugParameterMode
{
    case NONE;
    case COUNT;
    case TYPES;
    case REDACTED_VALUES;
}
```

`FULL_VALUES` no será un default recomendado.

---

# 143. Policy compilation

```text
config
↓
validation
↓
compiled policy
↓
runtime
```

---

# 144. Hot path

Debug Information no deberá construir grandes strings durante cada query si:

```text
toolbar disabled
```

---

# 145. Lazy debug projection

Podrá utilizarse una estrategia:

```text
Telemetry record
↓
lightweight reference/snapshot
↓
final projection at scope end
```

si no retiene objetos runtime.

---

# 146. Lazy projection limitation

No podrá depender de objetos que pueden haber sido reseteados.

Por tanto deberá conservar datos suficientes como value snapshots.

---

# 147. Finalization

Al finalizar el scope:

```text
collect pending fragments
↓
aggregate
↓
truncate
↓
redact
↓
freeze
↓
release active state
```

---

# 148. DebugInformationCollector

```php
interface DebugInformationCollector
{
    public function add(
        DebugFragment $fragment
    ): void;

    public function finalize(): DatabaseDebugInformation;
}
```

---

# 149. DebugFragment

```php
interface DebugFragment
{
    public function section(): DebugSectionId;

    public function priority(): DebugPriority;
}
```

---

# 150. Typed fragments

Ejemplos:

```text
QueryDebugFragment
ConnectionDebugFragment
TransactionDebugFragment
ORMDebugFragment
SlowQueryDebugFragment
NPlusOneDebugFragment
```

---

# 151. Fragment immutability

Preferiblemente cada fragmento deberá ser immutable.

---

# 152. Collector mutability

El collector scoped puede ser mutable internamente.

---

# 153. Provider independence

El collector no deberá depender de:

```text
Symfony Profiler
Laravel Debugbar
Clockwork
OpenTelemetry vendor UI
```

---

# 154. Toolbar adapter

Será documento 225.

---

# 155. Null collector

```php
final class NullDebugInformationCollector
    implements DebugInformationCollector
{
    public function add(DebugFragment $fragment): void
    {
    }

    public function finalize(): DatabaseDebugInformation
    {
        return DatabaseDebugInformation::empty();
    }
}
```

---

# 156. Near-zero disabled overhead

Cuando debug esté deshabilitado, los adapters deberán evitar construir fragments costosos.

---

# 157. Collector registration

Podrá integrarse mediante:

```text
Database Telemetry Pipeline
```

o bridges específicos.

---

# 158. No circular dependency

Evitar:

```text
QueryTelemetry
→ DebugSystem
→ QueryTelemetry
```

---

# 159. Preferred dependency

```text
Telemetry
↓
Debug Projection Bridge
↓
Debug Collector
```

---

# 160. QueryDebugProjectionFactory

```php
interface QueryDebugProjectionFactory
{
    public function fromTelemetry(
        QueryTelemetryRecord $record
    ): QueryDebugFragment;
}
```

---

# 161. Connection projection

Mismo patrón.

---

# 162. Transaction projection

Mismo patrón.

---

# 163. ORM projection

Mismo patrón.

---

# 164. Detection projection

Slow Query y N+1 podrán convertirse a fragments.

---

# 165. Correlation registry

Para asociar records sin objetos:

```php
interface DebugCorrelationRegistry
{
    public function correlate(
        DebugRecordId $left,
        DebugRecordId $right,
        DebugCorrelationType $type,
    ): void;
}
```

---

# 166. Correlation registry bounded

También deberá estar sujeto a budget.

---

# 167. Orphaned correlations

Si un record fue descartado por budget:

```text
correlation
```

deberá:

```text
drop safely
```

o señalar target unavailable.

---

# 168. Debug ordering

No dependerá únicamente de insertion order.

Cada record podrá tener:

```text
monotonic offset
sequence number
```

---

# 169. Sequence number

Útil cuando timestamps empatan.

---

# 170. Concurrency

Con múltiples fibers/coroutines:

```text
sequence
```

deberá ser thread/coroutine-safe según runtime.

---

# 171. Serialization ordering

Podrá definirse orden estable para tests.

---

# 172. DebugInformationSerializer JSON

Debe convertir:

```text
enums
IDs
durations
timestamps
```

a una representación estable.

---

# 173. Duration format

Payload interno podrá usar:

```text
nanoseconds integer
```

o unidad canónica.

UI podrá convertir a:

```text
ms
µs
s
```

---

# 174. No floating precision ambiguity

Internamente preferir integer duration units.

---

# 175. Timestamp security

Wall-clock timestamps pueden revelar información operacional.

Solo incluir cuando realmente sea necesario.

---

# 176. Request timeline

Usar offsets relativos permite:

```text
0.0 ms
5.2 ms
17.8 ms
```

sin requerir timestamps absolutos.

---

# 177. Debug storage

El core puede mantener un store temporal:

```text
RequestScopedDebugStore
```

---

# 178. Historical storage

No será responsabilidad obligatoria del core.

---

# 179. DebugInformationStore

```php
interface DebugInformationStore
{
    public function put(
        DatabaseDebugInformation $information
    ): void;

    public function get(
        DebugSessionId $sessionId
    ): ?DatabaseDebugInformation;
}
```

---

# 180. In-memory store

Apropiado para request-local toolbar.

---

# 181. Shared store

Podrá existir adapter:

```text
Redis
filesystem
external service
```

si se requiere acceso posterior.

---

# 182. Shared store security

Debe considerar:

```text
encryption
TTL
authorization
tenant isolation
```

---

# 183. Debug TTL

Información de debugging deberá tener retención corta por default.

---

# 184. Debug information ≠ audit log

No es un registro legal/permanente.

---

# 185. Audit data

Pertenece al futuro:

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

---

# 186. Security distinction

```text
Debug Log
≠
Audit Log
```

---

# 187. Request response integration

Toolbar puede recibir:

```text
DebugSessionId
```

mediante header interno o mecanismo de desarrollo.

---

# 188. No sensitive payload in headers

No introducir debug payload completo en headers.

---

# 189. Session token

Si se expone:

```text
DebugSessionToken
```

deberá ser:

```text
opaque
unpredictable
scoped
expiring
```

si sirve para recuperar información.

---

# 190. Session ID ≠ authorization

Conocer el ID no deberá bastar para acceder a diagnostics en producción.

---

# 191. DebugInformationAccessPolicy

```php
interface DebugInformationAccessPolicy
{
    public function canAccess(
        DebugAccessContext $context,
        DatabaseDebugInformation $information,
    ): bool;
}
```

---

# 192. Toolbar access

En development local podría permitirse ampliamente.

En staging/production deberá ser restrictivo.

---

# 193. Debug export

Exporters posibles:

```text
Toolbar
JSON
CLI
Structured Log
Test Recorder
IDE integration
```

---

# 194. Exporter contract

```php
interface DebugInformationExporter
{
    public function export(
        DatabaseDebugInformation $information
    ): void;
}
```

---

# 195. Exporter failure

No deberá romper la aplicación por default.

---

# 196. Security fail-closed

Sin embargo, si redaction falla:

```text
do not export sensitive payload
```

---

# 197. Important distinction

```text
Telemetry exporter failure
→ fail-open operationally

Security/redaction failure
→ fail-closed for data exposure
```

---

# 198. Error handling

Jerarquía:

```text
DebugInformationException
├── DebugCollectionException
├── DebugProjectionException
├── DebugSerializationException
├── DebugRedactionException
├── DebugBudgetException
├── DebugAccessException
└── DebugExporterException
```

---

# 199. Collection error

Un fragmento corrupto no deberá invalidar necesariamente todo el debug snapshot.

Podrá producir:

```text
PARTIAL
```

---

# 200. Critical debug errors

Si afectan seguridad:

```text
drop affected section
```

---

# 201. Diagnostics about debugging

El sistema podrá indicar:

```text
query details truncated
telemetry coverage partial
23 records dropped
source locations disabled
```

---

# 202. Meta diagnostics

Estas señales ayudan al desarrollador a interpretar correctamente el panel.

---

# 203. Debug collection metrics

Opcionalmente:

```text
database.debug.payload.bytes
database.debug.records
database.debug.records_dropped
database.debug.finalization.duration
```

---

# 204. Avoid telemetry recursion

Estas métricas no deberán generar nuevos DebugFragments que causen recursión infinita.

---

# 205. Debug instrumentation exclusion

El propio proceso de:

```text
serialize/store/export debug data
```

deberá poder marcarse como:

```text
telemetry-internal
```

---

# 206. No self-observation loop

Evitar:

```text
Debug exporter uses database
↓
query telemetry
↓
debug collector
↓
debug exporter uses database
...
```

---

# 207. Internal telemetry suppression scope

Podrá existir:

```php
interface TelemetrySuppressionScope
{
    public function close(): void;
}
```

para operaciones internas estrictamente controladas.

---

# 208. Suppression must be narrow

Nunca deshabilitar telemetry globalmente por worker.

---

# 209. Debug toolbar database storage

Por default no deberá persistir su propio payload usando la misma DB observada durante la misma request si eso genera loops.

---

# 210. Alternative

Usar:

```text
in-memory
separate backend
post-request exporter
```

---

# 211. Persistent runtime safety

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el Debug Collector deberá ser estrictamente scoped.

---

# 212. State that must reset

```text
fragments
record IDs
correlations
timeline
diagnostics
budgets
truncation state
session token
source locations
```

---

# 213. Immutable shared state

Puede compartirse:

```text
compiled policies
serializers
redactors
diagnostic registry
formatters
```

si son stateless.

---

# 214. No static request collector

Prohibido:

```php
static DebugCollector $current;
```

---

# 215. FrankenPHP

Flujo:

```text
Worker
├── Request A
│   └── DebugSession A
│
├── reset
│
└── Request B
    └── DebugSession B
```

---

# 216. RoadRunner

Fresh session por job.

---

# 217. OpenSwoole

Coroutine-local o context-local collector.

---

# 218. Concurrent debug sessions

Un worker podrá sostener:

```text
Session A
Session B
Session C
```

concurrentemente.

---

# 219. No shared mutable timeline

Cada session tendrá su timeline.

---

# 220. Memory safety

Finalizar una sesión deberá liberar:

```text
runtime fragments
temporary correlation maps
large text buffers
```

después de crear snapshot/store según policy.

---

# 221. Snapshot lifetime

Puede sobrevivir más tiempo porque ya es:

```text
safe immutable data
```

---

# 222. Large query text

Deberá truncarse.

---

# 223. Large plans

Execution plans podrán ser particularmente grandes.

---

# 224. Plan budget

Podrá existir:

```text
max_plan_bytes
```

---

# 225. Plan detail levels

```text
NONE
SUMMARY
NORMALIZED
FULL
```

---

# 226. Default plan mode

```text
SUMMARY
```

en debug avanzado.

---

# 227. Raw DB plan

Puede contener:

```text
table names
index names
paths
server metadata
```

y deberá pasar por security policy.

---

# 228. ORM metadata dump

Tampoco deberá mostrarse completo.

---

# 229. Metadata summary

Preferir:

```text
entity count
relationship count
metadata generation
cache state
```

---

# 230. Source locations

Podrán agregarse a diagnostics:

```text
file
line
operation
```

---

# 231. Path redaction

Podrá convertir:

```text
/var/www/customer/app/...
```

a:

```text
app/...
```

---

# 232. Windows paths

También:

```text
C:\Projects\App\...
```

a una ruta relativa segura.

---

# 233. DebugSourceLocation

```php
final readonly class DebugSourceLocation
{
    public function __construct(
        public string $file,
        public int $line,
    ) {}
}
```

---

# 234. No source contents

No almacenar código fuente completo por default.

---

# 235. Diagnostic recommendations

Podrán incluir:

```text
use eager loading
inspect pool pressure
reduce projection
inspect transaction scope
review query plan
```

---

# 236. Recommendations need codes

```php
enum DebugRecommendationCode
{
    case USE_BATCH_LOADING;
    case REVIEW_QUERY_PLAN;
    case REDUCE_PROJECTION;
    case REVIEW_CONNECTION_POOL;
    case REDUCE_TRANSACTION_SCOPE;
    case USE_CHUNKING;
}
```

---

# 237. Structured recommendations

Permiten al IDE/Toolbar enlazar documentación.

---

# 238. Recommendation ≠ auto-fix

Siempre.

---

# 239. Debug toolbar integration contract

El documento 225 deberá consumir:

```text
DatabaseDebugInformation
```

y no acceder directamente a internals del Database Engine.

---

# 240. Critical boundary

```text
Database Internals
↓
Debug Information System
↓
Debug Toolbar
```

No:

```text
Toolbar
↓
reach into EntityManager / Connection / UoW
```

---

# 241. Beneficio

Esto mantiene:

```text
separation
testability
security
runtime safety
UI independence
```

---

# 242. API conceptual

```php
$debug = DB::debug()->current();
```

podría devolver:

```text
DatabaseDebugInformation
```

solo en contexts autorizados.

---

# 243. No live DB access

El objeto devuelto no tendrá métodos como:

```php
$debug->connection()->query(...)
```

---

# 244. Read-only API

Solo:

```text
inspect
filter
serialize
render
```

---

# 245. Filter API

Podrá existir:

```php
$debug->queries()
    ->slow()
    ->all();
```

sobre records inmutables.

---

# 246. Filter ≠ database query

Es operación sobre el snapshot local.

---

# 247. Testing API

Ejemplos:

```php
$debug->assertNoCriticalDiagnostics();
```

---

# 248. Query count

```php
$debug->assertQueryCount(5);
```

---

# 249. No N+1

```php
$debug->assertNoDiagnostic(
    DebugDiagnosticCode::DB_N_PLUS_ONE
);
```

---

# 250. No unknown transaction

```php
$debug->assertNoDiagnostic(
    DebugDiagnosticCode::DB_TRANSACTION_UNKNOWN
);
```

---

# 251. Testing determinism

Debug IDs en testing podrán utilizar un generador determinista/fake.

---

# 252. Fake clock

Timeline tests deberán usar:

```text
FakeMonotonicClock
```

---

# 253. Snapshot tests

Podrán comprobar JSON estable.

Pero evitar depender de campos volátiles.

---

# 254. DebugInformationBuilder

```php
interface DebugInformationBuilder
{
    public function build(
        DebugInformationContext $context,
        DebugFragmentCollection $fragments,
    ): DatabaseDebugInformation;
}
```

---

# 255. Build pipeline

```text
fragments
↓
normalize
↓
deduplicate
↓
aggregate
↓
prioritize
↓
apply budget
↓
redact
↓
freeze
```

---

# 256. Redaction order

Redaction deberá ocurrir antes de:

```text
external store
export
toolbar serialization
```

---

# 257. Internal store

Incluso el store interno debería preferir datos minimizados.

---

# 258. Debug fragments security

Cada fragment factory deberá:

```text
extract only required data
```

---

# 259. Avoid full-object conversion

No:

```php
get_object_vars($entityManager);
```

---

# 260. DTO projection

Sí:

```php
new ORMDebugFragment(
    identityMapSize: 40,
    dirtyCount: 2,
);
```

---

# 261. Debug schema versioning

Payload:

```text
database-debug/1.0
```

podrá evolucionar a:

```text
database-debug/1.1
database-debug/2.0
```

---

# 262. Backward compatibility

Minor:

```text
additive optional fields
```

Major:

```text
breaking schema changes
```

---

# 263. Toolbar compatibility

Toolbar podrá negociar/versionar schema.

---

# 264. Debug plugins

Extensiones futuras podrán añadir:

```text
custom sections
custom diagnostics
custom render metadata
```

sin acceso a objetos runtime.

---

# 265. CustomDebugSectionProvider

```php
interface CustomDebugSectionProvider
{
    public function build(
        DebugInformationContext $context
    ): ?DebugSection;
}
```

---

# 266. Extension constraints

Custom provider deberá:

```text
respect budget
respect security
avoid hidden DB I/O
avoid mutable runtime retention
```

---

# 267. Provider timeout

Un custom debug provider no deberá bloquear indefinidamente finalization.

---

# 268. Provider failure

Deberá degradar a:

```text
PARTIAL
```

cuando sea seguro.

---

# 269. Diagnostics registry

```php
interface DebugDiagnosticRegistry
{
    public function register(
        DebugDiagnosticDefinition $definition
    ): void;
}
```

---

# 270. Stable codes

Registry deberá impedir colisiones de códigos.

---

# 271. Naming convention

Ejemplo:

```text
DB_QUERY_*
DB_CONN_*
DB_TX_*
DB_ORM_*
DB_CACHE_*
DB_N1_*
DB_RUNTIME_*
```

---

# 272. Query diagnostic examples

```text
DB_QUERY_SLOW_DATABASE
DB_QUERY_HIGH_HYDRATION
DB_QUERY_TIMEOUT
DB_QUERY_UNKNOWN_OUTCOME
```

---

# 273. Connection examples

```text
DB_CONN_POOL_WAIT
DB_CONN_RESET_FAILED
DB_CONN_LEAK_CANDIDATE
```

---

# 274. Transaction examples

```text
DB_TX_LONG_RUNNING
DB_TX_UNKNOWN_OUTCOME
DB_TX_CONNECTION_AFFINITY
```

---

# 275. ORM examples

```text
DB_ORM_ENTITY_MANAGER_TAINTED
DB_ORM_IDENTITY_MAP_GROWTH
DB_ORM_PENDING_WORK
```

---

# 276. N+1 example

```text
DB_N1_RELATIONSHIP
```

---

# 277. Runtime example

```text
DB_RUNTIME_STATE_LEAK_CANDIDATE
```

---

# 278. Severity mapping

Cada code podrá tener:

```text
default severity
```

pero detection específica puede ajustarla.

---

# 279. Diagnostic grouping

Toolbar podrá agrupar:

```text
Performance
Correctness
Security
Resource
ORM
Distribution
```

---

# 280. Debug category

```php
enum DebugDiagnosticCategory
{
    case PERFORMANCE;
    case CORRECTNESS;
    case RESOURCE;
    case SECURITY;
    case ORM;
    case DISTRIBUTION;
    case RUNTIME;
}
```

---

# 281. Debug summary score

VoltStack no deberá inventar un:

```text
database health score = 87/100
```

en V1 sin un modelo sólido.

---

# 282. Prefer concrete diagnostics

Más útil:

```text
2 slow queries
1 N+1
0 failed transactions
```

---

# 283. Security diagnostics

Podrá indicar:

```text
raw SQL exposure disabled
parameter values redacted
```

pero no listar secretos detectados.

---

# 284. Sensitive field detection count

Puede mostrar:

```text
12 values redacted
```

sin revelar contenido.

---

# 285. DebugInformationFactory

Podrá ser un servicio stateless/shared:

```php
final class DefaultDebugInformationFactory
{
    // builds immutable snapshots
}
```

---

# 286. Collector scope

Collector:

```text
request-scoped
job-scoped
operation-scoped
```

---

# 287. Scope selection

Será Runtime System quien resuelva scope.

---

# 288. Debug information and errors

Si ocurre fatal DB exception antes de finalization:

```text
best-effort partial snapshot
```

podrá generarse.

---

# 289. Crash-safe limitations

No prometer debug completeness ante:

```text
process crash
OOM
segmentation fault
hard kill
```

---

# 290. Partial failure snapshot

Puede marcar:

```text
coverage=PARTIAL
finalization_incomplete=true
```

---

# 291. OOM consideration

Cuando existe memory pressure, no intentar construir un payload enorme.

---

# 292. Emergency minimal snapshot

Podrá contener solo:

```text
error code
query fingerprint
transaction outcome
connection role
diagnostic IDs
```

---

# 293. Resource-aware finalization

Si budget/memory pressure es crítico:

```text
drop detail
retain critical summary
```

---

# 294. Debug information and encryption

Si se persiste fuera del proceso:

```text
encryption-at-rest
```

puede ser requisito del adapter.

Core deberá soportar esta integración.

---

# 295. Debug information and retention

Por defecto:

```text
short-lived
```

---

# 296. Debug information and compliance

El sistema deberá facilitar:

```text
data minimization
retention control
access control
redaction
```

---

# 297. Debug information and multitenancy

Cuando Multitenancy esté instalado:

```text
DebugInformation
```

deberá quedar asociado a un:

```text
TenantDebugContext
```

sin mezclar tenants.

---

# 298. Cross-tenant admin

Solo roles administrativos explícitos podrán ver agregados cross-tenant.

---

# 299. Tenant-specific toolbar

Nunca deberá incluir:

```text
query from tenant B
```

durante request de tenant A.

---

# 300. Tenant identity display

Preferir:

```text
tenant context active
```

o safe alias según policy.

---

# 301. Sharding

Debug section podrá mostrar:

```text
logical shard
physical query count
fan-out
```

sin revelar topología completa.

---

# 302. DebugInformation and failover

Podrá indicar:

```text
connection acquired after failover
```

si Telemetry lo reporta.

---

# 303. Failover details

No deberá inferir causas no observadas.

---

# 304. Query retry

Podrá mostrar:

```text
attempt 1 failed
attempt 2 succeeded
```

---

# 305. Transaction retry

Misma filosofía.

---

# 306. Query cancellation

Podrá mostrar reason bounded:

```text
timeout
deadline
client cancellation
```

---

# 307. Correlation with request lifecycle

DebugInformation podrá integrarse con:

```text
RequestId / OperationId
```

de VoltStack general.

---

# 308. Framework independence

Database Debug Information no deberá depender de HTTP Request concrete classes.

---

# 309. Generic operation metadata

```php
final readonly class DebugOperationMetadata
{
    public function __construct(
        public string $kind,
        public ?string $name,
    ) {}
}
```

---

# 310. Operation names

Deberán ser bounded/safe.

No incluir user input arbitrario.

---

# 311. DebugInfo diagnostics for query volume

Podrá existir:

```text
DB_QUERY_HIGH_COUNT
```

pero:

```text
high count
≠
N+1
```

---

# 312. High count diagnostic

Ejemplo:

```text
Queries:
  800

Unique fingerprints:
  5

N+1:
  none confirmed
```

---

# 313. High aggregate DB time

También:

```text
DB_QUERY_HIGH_AGGREGATE_TIME
```

---

# 314. Slowest queries

Toolbar podrá recibir:

```text
top 10 slowest
```

precalculadas en Debug Information.

---

# 315. Avoid UI recomputing semantics

El toolbar no deberá rehacer:

```text
N+1 detection
slow query classification
transaction state analysis
```

Solo renderizar.

---

# 316. Key boundary

> **La inteligencia diagnóstica pertenece al backend Debug Information System; la toolbar es una vista.**

---

# 317. Toolbar must be replaceable

Podremos crear:

```text
VoltStack official toolbar
CLI inspector
IDE plugin
external dashboard
```

sobre el mismo schema.

---

# 318. Portable debug schema

Es uno de los objetivos principales del sistema.

---

# 319. DebugInfo comparison

En testing/desarrollo futuro podrá compararse:

```text
Debug Snapshot A
vs
Debug Snapshot B
```

para observar:

```text
query count delta
slow query delta
N+1 delta
ORM memory delta
```

---

# 320. Not benchmark truth

Debug snapshots no reemplazan benchmarks controlados.

---

# 321. Debug information as artifact

Podrá exportarse a JSON para bug reports.

---

# 322. Bug report export

Debe pasar por una política aún más estricta de redacción.

---

# 323. Share-safe mode

Podrá existir:

```text
DebugExportMode::SHARE_SAFE
```

que elimine:

```text
paths
hosts
tenant IDs
SQL values
connection aliases sensibles
source metadata
```

---

# 324. Developer mode

```text
DebugExportMode::DEVELOPER
```

puede conservar más información autorizada.

---

# 325. Internal support mode

```text
DebugExportMode::SUPPORT
```

podrá configurarse separadamente.

---

# 326. Export modes ≠ access control

Ambos se aplican.

---

# 327. Debug export signature

Opcionalmente un payload podría incluir:

```text
framework version
debug schema version
platform capability fingerprint
```

para soporte.

---

# 328. No credentials ever

Incluso en support mode:

```text
credentials = never
```

---

# 329. Directory structure

```text
src/Quantum/Database/Telemetry/Debug/
│
├── Contract/
│   ├── DebugInformationCollector.php
│   ├── DebugInformationBuilder.php
│   ├── DebugInformationSerializer.php
│   ├── DebugInformationStore.php
│   ├── DebugInformationExporter.php
│   ├── DebugInformationRedactor.php
│   ├── DebugInformationAccessPolicy.php
│   └── DebugCorrelationRegistry.php
│
├── Identity/
│   ├── DebugSessionId.php
│   ├── DebugScopeId.php
│   ├── DebugRecordId.php
│   ├── DebugTimelineEntryId.php
│   └── DebugCorrelationId.php
│
├── Context/
│   ├── DebugInformationContext.php
│   ├── DebugAccessContext.php
│   ├── DebugRedactionContext.php
│   └── DebugOperationMetadata.php
│
├── Model/
│   ├── DatabaseDebugInformation.php
│   ├── DebugSummary.php
│   ├── DebugCoverage.php
│   ├── DebugInformationVersion.php
│   ├── DebugPriority.php
│   └── DebugEnvironment.php
│
├── Section/
│   ├── DebugSection.php
│   ├── DebugSectionId.php
│   ├── DebugSectionCollection.php
│   ├── QueryDebugSection.php
│   ├── ConnectionDebugSection.php
│   ├── TransactionDebugSection.php
│   ├── ORMDebugSection.php
│   ├── CacheDebugSection.php
│   ├── SlowQueryDebugSection.php
│   ├── NPlusOneDebugSection.php
│   ├── DistributionDebugSection.php
│   ├── SchemaDebugSection.php
│   ├── RuntimeDebugSection.php
│   └── SecurityDebugSection.php
│
├── Fragment/
│   ├── DebugFragment.php
│   ├── DebugFragmentCollection.php
│   ├── QueryDebugFragment.php
│   ├── ConnectionDebugFragment.php
│   ├── TransactionDebugFragment.php
│   ├── ORMDebugFragment.php
│   ├── SlowQueryDebugFragment.php
│   └── NPlusOneDebugFragment.php
│
├── Query/
│   ├── QueryDebugRecord.php
│   ├── QueryDebugContext.php
│   ├── DebugQueryText.php
│   ├── DebugQueryTextMode.php
│   ├── DebugParameterSummary.php
│   └── DebugParameterMode.php
│
├── Connection/
│   ├── ConnectionDebugRecord.php
│   └── DebugEndpointView.php
│
├── Transaction/
│   ├── TransactionDebugRecord.php
│   └── TransactionIsolationDebugView.php
│
├── ORM/
│   ├── ORMDebugSnapshot.php
│   └── UnitOfWorkDebugSummary.php
│
├── Timeline/
│   ├── DebugTimeline.php
│   ├── DebugTimelineEntry.php
│   ├── DebugTimelineType.php
│   └── DebugTimelineStatus.php
│
├── Correlation/
│   ├── DefaultDebugCorrelationRegistry.php
│   ├── DebugCorrelationType.php
│   └── DebugCorrelationEdge.php
│
├── Diagnostic/
│   ├── DebugDiagnostic.php
│   ├── DebugDiagnosticCode.php
│   ├── DebugDiagnosticCategory.php
│   ├── DebugDiagnosticCollection.php
│   ├── DebugSeverity.php
│   ├── DebugConfidence.php
│   ├── DebugRecommendation.php
│   ├── DebugRecommendationCode.php
│   ├── DebugDiagnosticDefinition.php
│   └── DebugDiagnosticRegistry.php
│
├── Security/
│   ├── DebugSensitivity.php
│   ├── DebugSecurityMetadata.php
│   ├── DefaultDebugInformationRedactor.php
│   └── DefaultDebugInformationAccessPolicy.php
│
├── Projection/
│   ├── QueryDebugProjectionFactory.php
│   ├── ConnectionDebugProjectionFactory.php
│   ├── TransactionDebugProjectionFactory.php
│   ├── ORMDebugProjectionFactory.php
│   ├── SlowQueryDebugProjectionFactory.php
│   └── NPlusOneDebugProjectionFactory.php
│
├── Policy/
│   ├── DebugInformationPolicy.php
│   ├── CompiledDebugInformationPolicy.php
│   ├── DebugInformationBudget.php
│   └── DebugExportMode.php
│
├── Builder/
│   ├── DefaultDebugInformationBuilder.php
│   ├── DebugSectionBuilder.php
│   └── DebugSummaryBuilder.php
│
├── Collector/
│   ├── DefaultDebugInformationCollector.php
│   └── NullDebugInformationCollector.php
│
├── Serialization/
│   ├── JsonDebugInformationSerializer.php
│   ├── ArrayDebugInformationSerializer.php
│   └── DebugPayload.php
│
├── Store/
│   ├── RequestScopedDebugStore.php
│   ├── InMemoryDebugInformationStore.php
│   └── NullDebugInformationStore.php
│
├── Export/
│   ├── StructuredLogDebugExporter.php
│   ├── JsonDebugExporter.php
│   └── NullDebugInformationExporter.php
│
├── Runtime/
│   ├── DebugRuntimeContext.php
│   ├── DebugRuntimeResetter.php
│   └── TelemetrySuppressionScope.php
│
├── Extension/
│   ├── CustomDebugSectionProvider.php
│   └── DebugExtensionRegistry.php
│
├── Testing/
│   ├── RecordingDebugInformationCollector.php
│   ├── DebugInformationAssertions.php
│   ├── FakeDebugClock.php
│   └── FakeDebugIdGenerator.php
│
└── Exception/
    ├── DebugInformationException.php
    ├── DebugCollectionException.php
    ├── DebugProjectionException.php
    ├── DebugSerializationException.php
    ├── DebugRedactionException.php
    ├── DebugBudgetException.php
    ├── DebugAccessException.php
    └── DebugExporterException.php
```

---

# 330. Architectural invariants

## DB-DBG-001
Debug Information será una representación derivada, no runtime state.

## DB-DBG-002
Debug Information no almacenará PDO.

## DB-DBG-003
Debug Information no almacenará Connection mutable.

## DB-DBG-004
Debug Information no almacenará EntityManager.

## DB-DBG-005
Debug Information no almacenará UnitOfWork.

## DB-DBG-006
Debug Information no almacenará IdentityMap.

## DB-DBG-007
Debug Information no almacenará Entity objects.

## DB-DBG-008
Debug Information no almacenará ResultCursor.

## DB-DBG-009
Debug Information no almacenará TransactionContext mutable.

## DB-DBG-010
Debug Information será distinta de Telemetry.

## DB-DBG-011
Debug Information será distinta de Logs.

## DB-DBG-012
Debug Information será distinta de Metrics.

## DB-DBG-013
Debug Information será distinta de Tracing.

## DB-DBG-014
Debug Information será distinta de Profiler.

## DB-DBG-015
Debug Information será distinta de Debug Toolbar.

## DB-DBG-016
Debug Information será distinta de Exception.

## DB-DBG-017
Debug Snapshot será distinto de live object.

## DB-DBG-018
Completed Debug Information será immutable.

## DB-DBG-019
Active collector podrá ser mutable solo dentro de su scope.

## DB-DBG-020
Debug coverage será explícita.

## DB-DBG-021
Missing information no se convertirá automáticamente en cero.

## DB-DBG-022
Sections serán tipadas.

## DB-DBG-023
Summary utilizará mediciones canonical.

## DB-DBG-024
Debug SQL estará sujeto a policy.

## DB-DBG-025
Raw SQL estará oculto por default en producción.

## DB-DBG-026
Parameter values estarán ocultos por default.

## DB-DBG-027
Debug SQL no será reutilizable para ejecución.

## DB-DBG-028
Slow Query detections serán consumidas, no recalculadas por UI.

## DB-DBG-029
N+1 detections serán consumidas, no recalculadas por UI.

## DB-DBG-030
Endpoint credentials nunca se mostrarán.

## DB-DBG-031
DSN completo nunca se mostrará.

## DB-DBG-032
Transaction UNKNOWN permanecerá UNKNOWN.

## DB-DBG-033
ORM entities no serán dumped.

## DB-DBG-034
Cache keys sensibles no se mostrarán completas.

## DB-DBG-035
Debugging no ejecutará hidden DB I/O por default.

## DB-DBG-036
Schema introspection adicional será explícita.

## DB-DBG-037
EXPLAIN adicional será explícito.

## DB-DBG-038
Health checks adicionales serán explícitos.

## DB-DBG-039
Runtime details estarán sujetos a security policy.

## DB-DBG-040
Security section no contendrá secretos.

## DB-DBG-041
Debug payload será redactado antes del consumer.

## DB-DBG-042
Data minimization ocurrirá antes de export.

## DB-DBG-043
SECRET nunca será exportable.

## DB-DBG-044
SENSITIVE requerirá policy explícita.

## DB-DBG-045
Debug payload será serializable.

## DB-DBG-046
Serialization no dependerá de PHP serialize() de runtime objects.

## DB-DBG-047
Debug schema tendrá versión.

## DB-DBG-048
Debug schema será distinto del internal class graph.

## DB-DBG-049
Debug payload será bounded.

## DB-DBG-050
Query records serán bounded.

## DB-DBG-051
Transaction records serán bounded.

## DB-DBG-052
Diagnostics serán bounded.

## DB-DBG-053
Overflow conservará summary cuando sea posible.

## DB-DBG-054
Dropped record count será observable.

## DB-DBG-055
Critical diagnostics tendrán prioridad de retención.

## DB-DBG-056
Timeline soportará concurrencia.

## DB-DBG-057
Timeline usará monotonic durations.

## DB-DBG-058
Parent-child correlation será distinta de temporal ordering.

## DB-DBG-059
Correlations utilizarán IDs, no object references.

## DB-DBG-060
DebugRecordIds serán scoped.

## DB-DBG-061
Diagnostics tendrán stable codes.

## DB-DBG-062
Diagnostic code será distinto de wording.

## DB-DBG-063
Observed será distinto de inferred.

## DB-DBG-064
Inferred será distinto de recommended.

## DB-DBG-065
Diagnostic confidence será explícita.

## DB-DBG-066
Exception será distinta de diagnostic.

## DB-DBG-067
Production Debug Information será minimal/safe por default.

## DB-DBG-068
Development mode no autorizará secret exposure.

## DB-DBG-069
Testing podrá usar deterministic IDs.

## DB-DBG-070
Testing podrá usar fake monotonic clock.

## DB-DBG-071
Debug collector no dependerá de concrete toolbar.

## DB-DBG-072
Null collector estará disponible.

## DB-DBG-073
Disabled debug tendrá overhead mínimo.

## DB-DBG-074
No existirán circular dependencies entre telemetry y debug.

## DB-DBG-075
Debug projection bridges consumirán telemetry records.

## DB-DBG-076
Debug correlation registry será bounded.

## DB-DBG-077
Orphan correlations se manejarán de forma segura.

## DB-DBG-078
Serialization tendrá ordering estable cuando sea necesario.

## DB-DBG-079
Durations usarán representación precisa.

## DB-DBG-080
Wall timestamps absolutos serán opcionales.

## DB-DBG-081
Historical debug storage no será responsabilidad obligatoria del core.

## DB-DBG-082
Debug Information será distinta de Audit Log.

## DB-DBG-083
Shared debug store deberá aplicar seguridad y TTL.

## DB-DBG-084
Debug Session ID no será authorization token.

## DB-DBG-085
Debug access estará sujeto a access policy.

## DB-DBG-086
Exporter failure no romperá aplicación por default.

## DB-DBG-087
Redaction failure no deberá exponer payload inseguro.

## DB-DBG-088
Telemetry exporter failure podrá fail-open operacionalmente.

## DB-DBG-089
Security/redaction failure deberá fail-closed para exposición.

## DB-DBG-090
Partial collection errors podrán producir PARTIAL debug info.

## DB-DBG-091
Meta diagnostics indicarán truncation/coverage.

## DB-DBG-092
Debug metrics no producirán recursion.

## DB-DBG-093
Internal suppression scopes serán narrow.

## DB-DBG-094
Suppression no será mutable global por worker.

## DB-DBG-095
Persistent runtime collector será scoped.

## DB-DBG-096
Fragments no sobrevivirán accidentalmente entre requests.

## DB-DBG-097
Correlations no sobrevivirán accidentalmente entre requests.

## DB-DBG-098
Budgets no sobrevivirán accidentalmente entre requests.

## DB-DBG-099
OpenSwoole debug context será coroutine-safe.

## DB-DBG-100
RoadRunner tendrá fresh debug session por job.

## DB-DBG-101
FrankenPHP tendrá fresh debug session por request.

## DB-DBG-102
Immutable policies podrán compartirse.

## DB-DBG-103
Static mutable current collector estará prohibido.

## DB-DBG-104
Concurrent sessions serán aisladas.

## DB-DBG-105
Finalization liberará temporary runtime state.

## DB-DBG-106
Large SQL será truncado según budget.

## DB-DBG-107
Large execution plans serán bounded.

## DB-DBG-108
Metadata dumps completos no serán default.

## DB-DBG-109
Source paths serán redactables.

## DB-DBG-110
Source code completo no será almacenado por default.

## DB-DBG-111
Recommendations tendrán structured codes cuando sea posible.

## DB-DBG-112
Recommendations nunca serán auto-fixes.

## DB-DBG-113
Debug Toolbar consumirá Debug Information, no internals.

## DB-DBG-114
Toolbar no accederá directamente a EntityManager.

## DB-DBG-115
Toolbar no accederá directamente a Connection.

## DB-DBG-116
Toolbar no accederá directamente a UnitOfWork.

## DB-DBG-117
Debug Information API será read-only.

## DB-DBG-118
Filtering local no ejecutará queries.

## DB-DBG-119
Testing podrá hacer assertions sobre debug snapshot.

## DB-DBG-120
Build pipeline aplicará budget antes del payload final.

## DB-DBG-121
Redaction ocurrirá antes de external export.

## DB-DBG-122
Fragment factories extraerán solo información necesaria.

## DB-DBG-123
Full-object introspection estará prohibida.

## DB-DBG-124
Debug schema seguirá versioning explícito.

## DB-DBG-125
Custom debug sections deberán respetar security.

## DB-DBG-126
Custom debug sections deberán respetar budget.

## DB-DBG-127
Custom debug sections no realizarán hidden DB I/O por default.

## DB-DBG-128
Custom provider failure podrá degradar a PARTIAL.

## DB-DBG-129
Diagnostic registry evitará code collisions.

## DB-DBG-130
Diagnostic categories serán bounded.

## DB-DBG-131
V1 no inventará opaque health scores.

## DB-DBG-132
Concrete diagnostics serán preferidos.

## DB-DBG-133
Sensitive-field counts podrán mostrarse sin contenido.

## DB-DBG-134
Collector scope será decidido por Runtime System.

## DB-DBG-135
Crash/OOM podrá limitar completeness.

## DB-DBG-136
Emergency minimal snapshot será permitido.

## DB-DBG-137
Resource pressure reducirá detail antes de aumentar presión.

## DB-DBG-138
Persistent debug storage podrá requerir encryption.

## DB-DBG-139
Debug retention será corta por default.

## DB-DBG-140
Debug architecture facilitará data minimization.

## DB-DBG-141
Multitenancy isolation será preservado.

## DB-DBG-142
Tenant A debug data no contendrá tenant B.

## DB-DBG-143
Cross-tenant debug requerirá authorization explícita.

## DB-DBG-144
Shard topology no será expuesta innecesariamente.

## DB-DBG-145
Failover info será evidence-based.

## DB-DBG-146
Retry attempts serán representables.

## DB-DBG-147
Cancellation reason será bounded.

## DB-DBG-148
Database Debug Information será independiente de HTTP.

## DB-DBG-149
Database Debug Information será independiente de CLI.

## DB-DBG-150
Operation metadata será bounded/safe.

## DB-DBG-151
High query count será distinto de N+1.

## DB-DBG-152
Slowest queries podrán precomputarse.

## DB-DBG-153
UI no recalculará semantic diagnostics.

## DB-DBG-154
Diagnostic intelligence permanecerá en backend.

## DB-DBG-155
Debug Toolbar será reemplazable.

## DB-DBG-156
Debug schema será portable.

## DB-DBG-157
Debug snapshot comparison no sustituirá benchmarks.

## DB-DBG-158
Bug report export tendrá policy de redaction reforzada.

## DB-DBG-159
Share-safe export mode será soportable.

## DB-DBG-160
Credentials nunca serán exportadas en ningún mode.

## DB-DBG-161
Debug snapshot podrá incluir framework version.

## DB-DBG-162
Debug snapshot podrá incluir debug schema version.

## DB-DBG-163
Debug snapshot podrá incluir capability fingerprint seguro.

## DB-DBG-164
Debug Information nunca será Database Truth.

## DB-DBG-165
Debug Information preservará UNKNOWN cuando corresponda.

## DB-DBG-166
Debug Information no inventará métricas faltantes.

## DB-DBG-167
Debug Information permanecerá provider-agnostic.

## DB-DBG-168
Debug Information permanecerá runtime-safe.

## DB-DBG-169
Debug Information será segura para consumidores no privilegiados solo después de policy/redaction apropiada.

## DB-DBG-170
Debug Information será la frontera oficial entre Database internals y herramientas de desarrollo.

---

# 331. Modelo formal

Sea el estado observable del Database Runtime:

```text
S
```

y el conjunto de registros de telemetría:

```text
T(S)
```

Debug Information deberá construirse como:

```text
D
=
Freeze(
    Redact(
        Bound(
            Normalize(
                Project(T(S))
            )
        )
    )
)
```

---

# 332. Propiedades

Debe cumplirse:

```text
D
```

es:

```text
immutable
bounded
safe
serializable
```

y:

```text
D
```

no contiene referencias mutables a:

```text
S
```

---

# 333. Regla de no-retención

Formalmente:

```text
References(D, RuntimeObjects) = ∅
```

para objetos como:

```text
Connection
EntityManager
Entity
Cursor
TransactionContext
```

---

# 334. Debug coverage

Sea:

```text
O
```

el conjunto de observaciones disponibles y:

```text
R
```

las requeridas por la policy.

El sistema podrá derivar:

```text
Coverage(D)
=
f(O,R)
```

como:

```text
COMPLETE
PARTIAL
MINIMAL
UNKNOWN
```

sin fingir precisión numérica innecesaria.

---

# 335. Budget model

Sea:

```text
B
```

el budget de debug y:

```text
C(D)
```

el costo de representación.

Debe cumplirse aproximadamente:

```text
C(D) ≤ B
```

mediante:

```text
aggregation
sampling
truncation
prioritization
```

---

# 336. Security model

La vista exportable será:

```text
D_safe
=
Authorize(
    Redact(
        Minimize(D_internal)
    )
)
```

Nunca:

```text
D_internal
→ external consumer
```

sin procesamiento de seguridad.

---

# 337. Timeline model

Para cada evento `e`:

```text
e =
(type, offset, duration, status, correlation)
```

con:

```text
offset ≥ 0
duration ≥ 0
```

cuando sean conocidos.

Operaciones concurrentes podrán solaparse.

---

# 338. Correlation model

Una relación diagnóstica:

```text
A → B
```

deberá representarse como:

```text
DebugRecordId(A)
+
DebugCorrelationType
+
DebugRecordId(B)
```

y nunca mediante referencias directas a runtime objects.

---

# 339. Arquitectura final

```text
                 Database Runtime
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Query Telemetry Connection      Transaction
        │          Telemetry        Telemetry
        │              │              │
        └──────────────┼──────────────┘
                       │
                 ORM Telemetry
                       │
                       ▼
                 Query Profiler
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Slow Query           N+1
         Detection         Detection
              │                 │
              └────────┬────────┘
                       ▼
               Debug Projections
                       │
                       ▼
                Debug Fragments
                       │
                       ▼
              Debug Information
                   Builder
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          Budget   Redaction  Correlation
             │         │         │
             └─────────┼─────────┘
                       ▼
          Immutable DatabaseDebugInformation
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
 Debug Toolbar       CLI/IDE          Tests
```

---

# 340. Filosofía arquitectónica

VoltStack seguirá:

```text
Snapshots
over
live runtime objects

Stable IDs
over
object references

Structured diagnostics
over
free-form dumps

Redaction
over
raw sensitive state

Bounded payloads
over
unlimited query history

Backend diagnostic intelligence
over
UI-specific analysis

Portable schema
over
toolbar coupling

Explicit UNKNOWN
over
invented certainty

Runtime isolation
over
worker-global state

Safe debugging
over
maximum information exposure
```

---

# 341. Regla maestra

> **VoltStack Database deberá transformar toda evidencia de observabilidad destinada a herramientas de desarrollo en un modelo diagnóstico seguro, inmutable, acotado y serializable antes de entregarla a cualquier consumidor externo al runtime de base de datos.**

En forma compacta:

```text
Runtime
↓
Telemetry
↓
Projection
↓
Normalize
↓
Correlate
↓
Prioritize
↓
Bound
↓
Redact
↓
Freeze
↓
Debug Information
↓
Consumer
```

Nunca:

```text
Consumer
↓
Live Database Runtime Objects
```

---

# 342. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
✓ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
✓ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
✓ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
✓ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 343. Siguiente documento

```text
225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

El siguiente documento deberá definir la integración visual y de desarrollo sobre la frontera estable creada en este documento.

Deberá diseñar:

```text
Database Debug Toolbar Panel
Query List
Query Timeline
Query Detail
Connection Panel
Transaction Panel
ORM Panel
Cache Panel
Slow Query Panel
N+1 Panel
Distribution Panel
Diagnostics Panel
Search
Filtering
Sorting
Expandable Records
Source Links
EXPLAIN on-demand
Copy-safe SQL
Redacted Parameters
Request Summary
Debug Session Retrieval
Persistent Runtime Support
Toolbar Security
Production Restrictions
IDE Links
JSON API
Plugin/extensible panels
Performance budgets
```

preservando la regla:

> **La Developer Debug Toolbar será una capa de presentación sobre `DatabaseDebugInformation`; no deberá acceder directamente a Connection, EntityManager, UnitOfWork, Query objects, TransactionContext ni ningún otro estado vivo del Database Runtime.**