# 221_DATABASE_QUERY_PROFILER_SYSTEM.md

# VoltStack Quantum Database
## Query Profiler System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 221 — Query Profiler System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `220_DATABASE_ORM_TELEMETRY_SYSTEM.md`  
**Siguiente documento:** `222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md`

---

# 1. Propósito

`Query Profiler System` define la arquitectura mediante la cual VoltStack podrá reconstruir, correlacionar, medir y explicar el costo completo de una operación de consulta a base de datos.

Una consulta no deberá analizarse únicamente como:

```text
SQL
+
Execution Time
```

porque el costo observado por la aplicación puede incluir:

```text
Query Construction
↓
Semantic Analysis
↓
Optimization
↓
Planning
↓
Compilation
↓
Connection Resolution
↓
Pool Wait
↓
Network / Driver
↓
Database Execution
↓
Result Transfer
↓
Result Consumption
↓
Hydration
↓
IdentityMap Resolution
↓
Relationship Assembly
↓
ORM Processing
```

Por tanto, la regla central será:

> **El Query Profiler de VoltStack reconstruye y analiza evidencia de ejecución para explicar dónde se consumen tiempo y recursos; no modifica automáticamente Query Models, Query AST, Query Plans, SQL, índices, conexiones, transacciones ni comportamiento ORM.**

Formalmente:

```text
Telemetry Evidence
        ↓
Correlation
        ↓
Query Profile
        ↓
Analysis
        ↓
Diagnostics
        ↓
Recommendations
```

Nunca:

```text
Profiler
   ↓
Rewrite Query
   ↓
Execute Different SQL
```

---

# 2. Problema arquitectónico

Un profiler limitado a SQL podría observar:

```sql
SELECT *
FROM users
WHERE status = ?
```

con:

```text
Database execution: 12 ms
```

y concluir:

```text
Query cost = 12 ms
```

Sin embargo, la aplicación podría experimentar:

```text
Semantic analysis       1 ms
Compilation             2 ms
Pool wait              35 ms
Database execution     12 ms
Result transfer         8 ms
Hydration              74 ms
Relationship assembly  19 ms
────────────────────────────
Total                 151 ms
```

La consulta SQL no es necesariamente el componente dominante.

VoltStack necesita distinguir:

```text
Slow SQL
Slow Connection Acquisition
Slow Result Transfer
Slow Hydration
Slow ORM Assembly
Slow Transaction Wait
Slow Query Compilation
Slow Overall Operation
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
│
├── Query Profiler                 ← este documento
│
├── Slow Query Detection
├── N+1 Telemetry
├── Debug Information
└── Developer Debug Toolbar
```

El Profiler consume evidencia.

No deberá convertirse en productor canonical de eventos de ejecución.

---

# 4. Fuentes principales

```text
217 Query Telemetry
        │
218 Connection Telemetry
        │
219 Transaction Telemetry
        │
220 ORM Telemetry
        │
        ▼
 Query Profiler
```

También podrá correlacionar evidencia de:

```text
Query Planner
SQL Compiler
Executor
Result System
Result Cursor
Streaming
Hydration
Cache
Replica Routing
Sharding
Resource Governance
```

---

# 5. Principio de propiedad

Cada subsistema sigue siendo dueño de su medición.

Ejemplo:

```text
Query Compiler
→ compilation duration

Connection Manager
→ acquisition duration

Executor
→ execution duration

Hydrator
→ hydration duration
```

El Profiler:

```text
correlates
aggregates
interprets
explains
```

---

# 6. Distinciones fundamentales

VoltStack preservará:

```text
Query Profiler
≠
Query Telemetry

Query Profiler
≠
Query Optimizer

Query Profiler
≠
Query Planner

Query Profiler
≠
SQL Compiler

Query Profiler
≠
Executor

Query Profiler
≠
Slow Query Detector

Query Profiler
≠
N+1 Detector

Query Profiler
≠
Database EXPLAIN

Query Profile
≠
Execution Plan

Query Fingerprint
≠
Raw SQL

Database Execution Time
≠
Application Query Time

Slow Query
≠
Slow ORM Operation

High Frequency
≠
High Cost

High Individual Cost
≠
High Aggregate Cost
```

---

# 7. Objetivos

El sistema deberá permitir:

```text
profile individual queries
profile groups of queries
correlate ORM operations
correlate transactions
correlate connections
decompose query latency
aggregate by fingerprint
detect expensive phases
identify high-frequency queries
identify high-total-cost query families
compare profiles
support development diagnostics
support production-safe profiling
provide inputs to regression detection
support EXPLAIN integration
support persistent runtimes safely
```

---

# 8. No objetivos

No deberá:

```text
rewrite queries
create indexes automatically
drop indexes
change transaction isolation
change replica routing
retry queries
change ORM mappings
modify hydration plans
change cache policies
kill queries automatically
change database configuration
```

---

# 9. QueryProfileId

Cada perfil tendrá identidad propia:

```php
final readonly class QueryProfileId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 10. QueryProfileId ≠ QueryOperationId

```text
QueryOperationId
```

identifica la operación ejecutada.

```text
QueryProfileId
```

identifica su representación analítica.

Normalmente:

```text
1 QueryOperation
→
1 QueryProfile
```

pero el diseño no deberá depender rígidamente de ello.

---

# 11. Profile Session

Para agrupar múltiples consultas:

```php
final readonly class QueryProfileSessionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 12. Profile Session examples

Una sesión podrá representar:

```text
HTTP request
CLI command
Queue job
Test
ORM flush
Import operation
Explicit profiling block
Developer toolbar request
```

---

# 13. QueryProfileSession ≠ Transaction

Una sesión puede contener:

```text
0 transactions
1 transaction
N transactions
```

---

# 14. QueryProfileSession ≠ Request

Puede existir fuera de HTTP.

---

# 15. Profile modes

```php
enum QueryProfilerMode
{
    case DISABLED;
    case METRICS;
    case STANDARD;
    case DETAILED;
    case DIAGNOSTIC;
}
```

---

# 16. DISABLED

Debe aproximarse a:

```text
near-zero profiler overhead
```

La telemetría base puede continuar según su propia configuración.

---

# 17. METRICS

Conserva principalmente:

```text
duration
count
fingerprint
outcome
```

sin reconstrucción detallada.

---

# 18. STANDARD

Añade:

```text
phase timing
connection context
transaction context
row counts
basic ORM correlation
```

---

# 19. DETAILED

Podrá añadir:

```text
query plan metadata
compiler information
routing information
hydration information
cache information
resource information
```

---

# 20. DIAGNOSTIC

Podrá permitir:

```text
source location
bounded stack information
EXPLAIN
extended diagnostics
```

sujeto a seguridad y presupuesto.

---

# 21. QueryProfile

Modelo conceptual:

```php
final readonly class QueryProfile
{
    public function __construct(
        public QueryProfileId $id,
        public QueryOperationId $queryId,
        public QueryFingerprint $fingerprint,
        public QueryProfileTimeline $timeline,
        public QueryProfileContext $context,
        public QueryProfileResult $result,
        public QueryProfileCost $cost,
        public QueryProfileDiagnostics $diagnostics,
    ) {}
}
```

---

# 22. Inmutabilidad

Un perfil completado deberá ser:

```text
immutable
```

para permitir:

```text
comparison
serialization
aggregation
debugging
testing
```

---

# 23. Profile lifecycle

```text
CREATED
  ↓
COLLECTING
  ↓
CORRELATING
  ↓
FINALIZING
  ↓
COMPLETED
```

Estados alternos:

```text
PARTIAL
FAILED
CANCELLED
UNKNOWN
```

---

# 24. Partial profile

Puede ocurrir cuando:

```text
sampling removed detail
telemetry budget exhausted
provider unavailable
operation still partially observed
trace context unavailable
```

---

# 25. Partial ≠ invalid

Un perfil parcial puede contener evidencia útil.

Debe declarar:

```text
coverage
```

---

# 26. Profile coverage

```php
enum QueryProfileCoverage
{
    case COMPLETE;
    case PARTIAL;
    case MINIMAL;
    case UNKNOWN;
}
```

---

# 27. UNKNOWN ≠ zero

Si una fase no fue observada:

```text
duration = UNKNOWN
```

No:

```text
duration = 0
```

---

# 28. QueryProfileContext

Podrá contener:

```text
logical database
persistence domain
tenant context
shard
endpoint role
connection
transaction
ORM operation
flush
request/job scope
runtime
```

---

# 29. Context security

No todo deberá exportarse.

Existirán:

```text
internal correlation context
external telemetry context
debug representation
```

separados.

---

# 30. Query fingerprint

El Profiler deberá utilizar una identidad semánticamente estable.

Ejemplo:

```sql
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 25;
```

deberán poder pertenecer a la misma familia:

```text
SELECT * FROM users WHERE id = ?
```

---

# 31. Fingerprint ≠ SQL string replacement

No deberá implementarse simplemente mediante regex sobre SQL.

VoltStack dispone de:

```text
Query Model
Query AST
Semantic Graph
Compiled Query
```

por lo que podrá generar fingerprints más robustos.

---

# 32. Semantic fingerprint

Idealmente:

```text
Query AST
+
semantic structure
+
logical operation
+
relevant execution characteristics
```

producirá:

```text
QueryFingerprint
```

---

# 33. Parameter values

Normalmente:

```text
parameter values
```

no formarán parte del fingerprint.

---

# 34. Parameter types

Los tipos sí podrán formar parte cuando afecten semántica/compilación.

---

# 35. Literal-sensitive queries

Algunas optimizaciones DB pueden depender de valores.

El profiler podrá mantener separadamente:

```text
semantic fingerprint
execution variant fingerprint
```

sin incluir valores sensibles directamente.

---

# 36. Query family

```php
final readonly class QueryFamily
{
    public function __construct(
        public QueryFingerprint $fingerprint,
        public QueryOperationType $operation,
    ) {}
}
```

---

# 37. Query profile timeline

Conceptualmente:

```text
t0
│
├── Query Construction
├── Normalization
├── Validation
├── Semantic Analysis
├── Optimization
├── Planning
├── Compilation
├── Connection Resolution
├── Connection Acquisition
├── Statement Preparation
├── Parameter Binding
├── Database Execution
├── Result Transfer
├── Result Consumption
├── Hydration
└── ORM Assembly
│
t1
```

---

# 38. No todas las fases existen siempre

Ejemplo:

```text
compiled query cache hit
```

puede eliminar:

```text
full compilation
```

Otro:

```text
raw execution
```

puede omitir parte del Query Builder pipeline.

---

# 39. Phase model

```php
enum QueryProfilePhase
{
    case CONSTRUCTION;
    case NORMALIZATION;
    case VALIDATION;
    case SEMANTIC_ANALYSIS;
    case OPTIMIZATION;
    case PLANNING;
    case COMPILATION;
    case CONNECTION_RESOLUTION;
    case CONNECTION_ACQUISITION;
    case STATEMENT_PREPARATION;
    case PARAMETER_BINDING;
    case DATABASE_EXECUTION;
    case RESULT_TRANSFER;
    case RESULT_CONSUMPTION;
    case HYDRATION;
    case ORM_ASSEMBLY;
}
```

---

# 40. Phase measurement

Cada fase podrá contener:

```php
final readonly class QueryProfilePhaseMeasurement
{
    public function __construct(
        public QueryProfilePhase $phase,
        public ?Duration $duration,
        public MeasurementConfidence $confidence,
        public MeasurementSource $source,
    ) {}
}
```

---

# 41. Confidence

```php
enum MeasurementConfidence
{
    case EXACT;
    case OBSERVED;
    case DERIVED;
    case ESTIMATED;
    case UNKNOWN;
}
```

---

# 42. Observed ≠ exact

Un tiempo medido por instrumentation puede incluir overhead.

Por tanto:

```text
OBSERVED
```

no necesita significar matemáticamente exacto.

---

# 43. Derived phase

Ejemplo:

```text
Other =
Total
-
KnownPhases
```

será:

```text
DERIVED
```

no `EXACT`.

---

# 44. Clock source

Debe utilizarse una abstracción monotónica para duraciones:

```php
interface ProfilerClock
{
    public function now(): ProfilerTimestamp;
}
```

---

# 45. Wall clock ≠ monotonic duration clock

Para duraciones:

```text
monotonic clock
```

es preferible.

---

# 46. Timeline overlap

Las fases pueden superponerse.

Ejemplo:

```text
streaming result transfer
+
hydration
```

pueden ocurrir intercaladamente.

---

# 47. No sumar ciegamente fases

No asumir:

```text
Total = Σ PhaseDuration
```

en todos los casos.

---

# 48. Critical path

El profiler podrá representar:

```text
critical-path duration
```

separadamente de:

```text
sum of work durations
```

---

# 49. Query latency decomposition

Modelo básico:

```text
T_application
=
T_pre_execution
+
T_connection
+
T_database
+
T_result
+
T_application_processing
```

---

# 50. Pre-execution

Incluye potencialmente:

```text
normalization
semantic analysis
optimization
planning
compilation
binding
```

---

# 51. Connection cost

Puede incluir:

```text
routing
pool wait
connection establishment
reconnect
```

---

# 52. Database cost

Puede incluir:

```text
server execution
lock waits
DB CPU
DB IO
```

pero VoltStack no necesariamente podrá separarlos sin soporte del servidor.

---

# 53. Database execution ≠ server CPU

Un `execute()` de 200 ms puede incluir:

```text
network
lock wait
server scheduling
IO
CPU
```

No deberá etiquetarse automáticamente como:

```text
200 ms CPU
```

---

# 54. Result cost

Puede incluir:

```text
network transfer
driver buffering
cursor consumption
row decoding
```

---

# 55. ORM cost

Puede incluir:

```text
type conversion
hydration
IdentityMap resolution
relationship assembly
snapshot creation
```

---

# 56. QueryProfileCost

```php
final readonly class QueryProfileCost
{
    public function __construct(
        public Duration $total,
        public ?Duration $database,
        public ?Duration $connection,
        public ?Duration $compilation,
        public ?Duration $resultConsumption,
        public ?Duration $hydration,
        public ?Duration $orm,
    ) {}
}
```

---

# 57. Unknown costs

Los campos desconocidos permanecerán:

```php
null
```

o utilizarán un `Known/Unknown` value object.

No se convertirán en cero.

---

# 58. Query execution timing

La medición deberá distinguir:

```text
execute call duration
```

de:

```text
full result consumption duration
```

---

# 59. Buffered driver

En un driver buffered:

```text
execute()
```

puede consumir gran parte del resultado inmediatamente.

---

# 60. Streaming driver

En streaming:

```text
execute()
```

puede ser corto mientras:

```text
iteration
```

consume tiempo posteriormente.

---

# 61. Query completed ≠ result consumed

Regla:

```text
Statement executed
≠
Result fully consumed
```

---

# 62. Result consumption profile

Podrá registrar:

```text
rows returned
rows consumed
bytes observed if available
cursor lifetime
time to first row
time to last row
```

---

# 63. Time To First Row

```text
TTFR
```

puede ser útil para streaming.

---

# 64. Time To Last Row

```text
TTLR
```

representa el tiempo hasta consumo completo.

---

# 65. TTFR ≠ database execution time

Debe mantenerse la distinción.

---

# 66. Row count

El profiler podrá observar:

```text
rows affected
rows returned
rows consumed
```

como conceptos distintos.

---

# 67. Rows returned ≠ rows hydrated

JOINs y projections hacen que sean diferentes.

---

# 68. Affected rows

Para:

```text
UPDATE
DELETE
INSERT
```

podrá correlacionarse con `affected rows`.

---

# 69. Affected rows ≠ entities changed

Un bulk update puede modificar:

```text
50,000 rows
```

sin cargar:

```text
50,000 entities
```

---

# 70. Query compilation profile

Podrá observar:

```text
AST nodes
optimization rules applied
plan creation
compiler duration
compiled query cache hit
```

---

# 71. AST node count

Puede ser diagnóstico útil.

No deberá convertirse automáticamente en métrica global de alta cardinalidad.

---

# 72. Optimization rule count

Podrá registrar:

```text
rules considered
rules applied
rewrite count
```

en diagnostic mode.

---

# 73. Query rewrite visibility

El Profiler podrá mostrar:

```text
Logical Query
↓
Optimized Query
```

como estructura resumida.

---

# 74. No raw sensitive predicates

Los valores deberán permanecer redactados.

---

# 75. Compilation cache

Se distinguirá:

```text
Compiled Query Cache Hit
```

de:

```text
Result Cache Hit
```

---

# 76. Compilation cache hit

Puede producir:

```text
compilation duration ≈ minimal
```

pero no necesariamente exactamente cero.

---

# 77. Result cache hit

Puede significar:

```text
database execution = none
```

---

# 78. Cached result profile

Un profile podrá indicar:

```text
source = RESULT_CACHE
```

y:

```text
databaseExecution = NOT_APPLICABLE
```

---

# 79. NOT_APPLICABLE ≠ UNKNOWN

Deben distinguirse.

```text
UNKNOWN
```

significa:

```text
no sabemos
```

`NOT_APPLICABLE`:

```text
esa fase no ocurrió
```

---

# 80. Measurement state

Podrá utilizarse:

```php
enum MeasurementState
{
    case OBSERVED;
    case NOT_APPLICABLE;
    case UNKNOWN;
}
```

---

# 81. Connection profiling

Correlacionará:

```text
logical database
endpoint
role
pool
acquisition
reuse
new connection
```

---

# 82. Pool wait

Un query lento por:

```text
PoolWait = 800 ms
DBExecution = 4 ms
```

no deberá diagnosticarse como slow SQL.

---

# 83. Connection establishment

Podrá separarse:

```text
pool wait
TCP/TLS/connect
authentication
session initialization
```

si el driver ofrece evidencia.

---

# 84. Reused connection

Debe poder indicarse:

```text
connection_reused=true
```

como diagnostic attribute.

---

# 85. Endpoint role

Dimensión bounded:

```text
WRITER
REPLICA
```

---

# 86. Replica identity

No necesariamente como metric label.

Puede aparecer en detailed profile mediante stable endpoint ID.

---

# 87. Replica lag context

Si disponible:

```text
lag evidence
freshness confidence
minimum required position
```

podrá asociarse al profile.

---

# 88. Replica lag ≠ query latency

Son dimensiones distintas.

---

# 89. Sticky routing

El profile podrá indicar:

```text
writer selected because sticky read policy
```

sin cambiar la decisión.

---

# 90. Routing explanation

Ejemplo:

```text
Read Query
↓
Transaction active
↓
Writer required
```

---

# 91. Shard profiling

Podrá incluir:

```text
shard ID
partition route
single/multi-shard
merge operation
```

---

# 92. Multi-shard query

Puede producir:

```text
1 logical query
↓
N physical queries
```

---

# 93. Logical profile vs physical profiles

VoltStack deberá soportar:

```text
LogicalQueryProfile
├── PhysicalQueryProfile shard-A
├── PhysicalQueryProfile shard-B
└── PhysicalQueryProfile shard-C
```

---

# 94. Fan-out cost

Podrá medirse:

```text
fanout count
max shard latency
merge latency
partial failure
```

---

# 95. Sum latency ≠ user latency

Con ejecución paralela:

```text
100 ms shard A
100 ms shard B
100 ms shard C
```

pueden producir:

```text
~100+ ms wall latency
```

no necesariamente:

```text
300 ms
```

---

# 96. Transaction profiling

Correlacionará:

```text
TransactionId
depth
isolation
savepoint context
transaction age
```

---

# 97. Query waiting in transaction

Una query puede ejecutarse dentro de una transacción larga.

El profiler podrá mostrar:

```text
transaction_age_at_query
```

en diagnostic mode.

---

# 98. Transaction age ≠ query duration

No mezclarlos.

---

# 99. Locking context

Podrá indicar:

```text
pessimistic lock requested
lock mode
```

si está disponible semánticamente.

---

# 100. Lock wait evidence

Si el driver/database permite obtenerlo:

```text
lock wait
```

podrá añadirse.

Si no:

```text
UNKNOWN
```

---

# 101. ORM correlation

Una query podrá pertenecer a:

```text
Repository operation
Model API operation
EntityManager find
Flush
Relationship load
Hydration sequence
```

---

# 102. ORM origin

Ejemplo:

```text
Query q_101
Origin:
  ORM relationship lazy load
```

es mucho más útil que mostrar solamente SQL.

---

# 103. Flush query profile

Un flush:

```text
flush_12
```

podrá agrupar:

```text
4 INSERT
7 UPDATE
2 DELETE
```

---

# 104. Query family aggregation

El profiler podrá agrupar por:

```text
QueryFingerprint
```

y calcular:

```text
count
total duration
average
minimum
maximum
percentiles
rows
failures
cache hits
```

---

# 105. High-frequency query

Ejemplo:

```text
Fingerprint A

Count: 10,000
Average: 1 ms
Total: 10 s
```

---

# 106. Individually fast ≠ globally cheap

La consulta anterior es rápida individualmente pero cara agregadamente.

---

# 107. Expensive individual query

```text
Fingerprint B

Count: 2
Average: 2.5 s
Total: 5 s
```

representa otro patrón.

---

# 108. Query ranking dimensions

El Profiler podrá ordenar por:

```text
total time
average time
max time
p95
p99
frequency
rows processed
hydration time
connection wait
failure count
```

---

# 109. Percentiles

Los percentiles requerirán estructuras bounded como:

```text
histogram
DDSketch-like provider
telemetry backend
```

El core no deberá guardar todas las muestras indefinidamente.

---

# 110. Profile aggregation

```php
final readonly class QueryProfileAggregate
{
    public function __construct(
        public QueryFingerprint $fingerprint,
        public int $count,
        public Duration $total,
        public Duration $average,
        public ?Duration $minimum,
        public ?Duration $maximum,
        public QueryProfileDistribution $distribution,
    ) {}
}
```

---

# 111. Average pitfalls

El promedio puede ocultar outliers.

Por ello:

```text
average
+
max
+
percentiles
```

son preferibles cuando estén disponibles.

---

# 112. QueryProfileAggregator

```php
interface QueryProfileAggregator
{
    public function add(QueryProfile $profile): void;

    public function snapshot(): QueryProfileAggregateSet;
}
```

---

# 113. Bounded aggregation

No conservará perfiles infinitamente.

Debe existir:

```text
max profiles
max fingerprints
time window
memory budget
```

---

# 114. Overflow policy

Opciones:

```text
DROP_DETAIL
AGGREGATE_ONLY
SAMPLE
EVICT_OLDEST
```

---

# 115. Query profile session summary

Podrá mostrar:

```text
queries
unique fingerprints
total DB time
total connection wait
total hydration time
cache hits
rows returned
rows affected
failures
```

---

# 116. Double-counted time

Si queries se ejecutan en paralelo:

```text
sum(query durations)
```

puede superar:

```text
session wall duration
```

Esto es válido.

---

# 117. Work time vs wall time

Se deberán distinguir:

```text
Aggregate Work Time
```

y:

```text
Wall Clock Critical Path
```

---

# 118. Query source

Podrá clasificarse:

```php
enum QuerySource
{
    case QUERY_BUILDER;
    case ORM;
    case SCHEMA;
    case MIGRATION;
    case BULK;
    case IMPORT;
    case RAW;
    case INTERNAL;
    case CUSTOM;
}
```

---

# 119. Source cardinality

Al ser enum bounded puede utilizarse en métricas.

---

# 120. Query operation

```text
SELECT
INSERT
UPDATE
DELETE
DDL
OTHER
```

será dimensión bounded.

---

# 121. Raw query profiling

Raw SQL seguirá siendo observable.

Pero puede carecer de:

```text
AST
semantic graph
optimizer metadata
```

---

# 122. Raw query profile coverage

Por tanto puede resultar:

```text
PARTIAL
```

respecto del pipeline completo.

---

# 123. EXPLAIN integration

El Profiler podrá integrar planes proporcionados por:

```text
EXPLAIN
EXPLAIN ANALYZE
platform-specific equivalents
```

pero mediante una capa especializada.

---

# 124. Query Profiler ≠ EXPLAIN

`EXPLAIN` describe principalmente cómo el DBMS planea/ejecuta una consulta.

El Profiler observa el pipeline completo de VoltStack.

---

# 125. ExplainPlanProvider

```php
interface ExplainPlanProvider
{
    public function explain(
        CompiledQuery $query,
        ExplainContext $context,
    ): ExplainResult;
}
```

---

# 126. Capability-driven EXPLAIN

Nunca:

```php
if ($database === 'postgres') {
    ...
}
```

en el profiler.

Usar:

```text
PlatformCapabilities
```

---

# 127. EXPLAIN support

Conceptualmente:

```text
supportsExplain()
supportsExplainAnalyze()
supportsStructuredExplain()
```

---

# 128. EXPLAIN ANALYZE danger

`EXPLAIN ANALYZE` puede ejecutar realmente la consulta.

Esto es crítico.

---

# 129. No automatic EXPLAIN ANALYZE

VoltStack no deberá ejecutar automáticamente:

```text
EXPLAIN ANALYZE DELETE ...
EXPLAIN ANALYZE UPDATE ...
```

---

# 130. Explain safety classification

```php
enum ExplainSafety
{
    case SAFE_METADATA_ONLY;
    case MAY_EXECUTE_QUERY;
    case MAY_MUTATE;
    case UNKNOWN;
}
```

---

# 131. UNKNOWN safety ≠ safe

Regla:

```text
UNKNOWN
≠
SAFE
```

---

# 132. Explain policy

```php
final readonly class QueryExplainPolicy
{
    public function __construct(
        public bool $enabled,
        public bool $allowAnalyze,
        public bool $allowWrites,
        public Duration $timeout,
    ) {}
}
```

---

# 133. Production default

Por default:

```text
automatic EXPLAIN ANALYZE = disabled
```

---

# 134. Plan source

Un plan podrá provenir de:

```text
database explain
database slow query tooling
cached diagnostic
external observability provider
```

y deberá indicar su fuente.

---

# 135. Execution plan ≠ VoltStack Query Plan

Distinción:

```text
VoltStack Logical/Physical Query Plan
≠
DBMS Execution Plan
```

Ambos pueden coexistir.

---

# 136. Plan correlation

Arquitectura ideal:

```text
Query AST
↓
VoltStack Logical Plan
↓
VoltStack Physical Plan
↓
Compiled SQL
↓
DBMS Execution Plan
```

---

# 137. Explain node model

VoltStack podrá normalizar ciertos conceptos:

```text
scan
index scan
join
sort
aggregate
limit
materialize
```

sin intentar eliminar todos los detalles vendor-specific.

---

# 138. Vendor plan preservation

El plan original podrá conservarse en:

```text
provider-specific diagnostic payload
```

bajo budget/security policy.

---

# 139. Index observations

El profiler podrá observar:

```text
sequential/full scan
index usage
sort
temporary structure
```

cuando el DBMS lo informe.

---

# 140. Index recommendation

Puede generar una recomendación:

```text
"evaluate an index on ..."
```

pero no crearla automáticamente.

---

# 141. Recommendation confidence

```php
enum DiagnosticConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 142. Recommendation ≠ fact

Siempre separar:

```text
Observed:
  full table scan

Inferred:
  filter has low selectivity or missing usable index

Recommended:
  evaluate index/selectivity
```

---

# 143. Query diagnostics

Categorías posibles:

```text
HIGH_DATABASE_LATENCY
HIGH_CONNECTION_WAIT
HIGH_COMPILATION_COST
HIGH_RESULT_CARDINALITY
HIGH_HYDRATION_COST
HIGH_QUERY_FREQUENCY
HIGH_TOTAL_COST
LARGE_RESULT_TRANSFER
POSSIBLE_INDEX_ISSUE
EXPENSIVE_SORT
EXPENSIVE_JOIN
REPEATED_QUERY
CACHE_MISS_PATTERN
LONG_TRANSACTION_CONTEXT
MULTI_SHARD_FANOUT
```

---

# 144. Profiler diagnostics ≠ automatic alarms

Los thresholds se definen por policy.

---

# 145. Slow Query Detector integration

El documento 222 recibirá:

```text
QueryProfile
QueryProfileAggregate
phase timings
query fingerprint
context
```

---

# 146. Slow query definition

No deberá limitarse a:

```text
DBExecution > threshold
```

Podrán existir diferentes categorías:

```text
slow database query
slow end-to-end query
slow hydration
slow acquisition
```

---

# 147. N+1 integration

El documento 223 utilizará:

```text
fingerprints
frequency
ORM origin
relationship identity
temporal proximity
root operation
```

---

# 148. Repeated query ≠ N+1

100 queries iguales podrían ser intencionales.

El profiler registra repetición.

N+1 detector interpreta semántica.

---

# 149. Profile comparison

VoltStack deberá poder comparar:

```text
Profile A
vs
Profile B
```

---

# 150. Comparison use cases

```text
before/after code change
before/after index
before/after ORM optimization
framework version comparison
driver comparison
database version comparison
```

---

# 151. QueryProfileComparator

```php
interface QueryProfileComparator
{
    public function compare(
        QueryProfileAggregateSet $baseline,
        QueryProfileAggregateSet $candidate,
    ): QueryProfileComparison;
}
```

---

# 152. Comparison dimensions

```text
count delta
average latency delta
p95 delta
DB time delta
hydration delta
rows delta
cache hit delta
failure delta
```

---

# 153. Regression ≠ slower single sample

Una sola muestra no debería establecer automáticamente regresión.

---

# 154. Regression evidence

Puede requerir:

```text
sample count
distribution
confidence
environment compatibility
```

---

# 155. Environment fingerprint

Para comparaciones:

```text
runtime
database platform
database version
driver
framework version
metadata generation
schema generation
```

pueden ser relevantes.

---

# 156. Version ≠ capability

Incluso en perfiles:

```text
DB version
```

no sustituye:

```text
effective capabilities
```

---

# 157. Profile normalization

Los perfiles podrán normalizarse para comparación.

Ejemplo:

```text
duration per 1,000 rows
queries per root entity
hydration time per entity
```

---

# 158. Normalized metrics caution

```text
time/entity
```

puede ser útil pero no necesariamente lineal.

---

# 159. Query profile tags

Internamente podrán existir:

```text
feature
component
operation
source
```

pero deberán ser bounded.

---

# 160. User-defined tags

Permitidos solo mediante registry/policy.

---

# 161. Arbitrary tags danger

No permitir:

```php
$profile->tag('user_id', $user->id);
```

como metric label indiscriminado.

---

# 162. Sensitive query parameters

No deberán aparecer por default.

---

# 163. SQL representation

Podrán existir tres representaciones:

```text
Raw SQL
Normalized SQL
Redacted SQL
```

---

# 164. Default diagnostic representation

Preferir:

```text
Normalized + Redacted
```

---

# 165. Parameter representation

Default:

```text
parameter count
parameter types
```

no valores.

---

# 166. Safe parameter metadata

Ejemplo:

```text
$1: int
$2: string(length_bucket=10-32)
```

si policy lo permite.

---

# 167. Length leakage

Incluso longitudes pueden ser sensibles.

Deberán poder deshabilitarse.

---

# 168. Query comments

Si VoltStack permite SQL comments para tracing, deberán:

```text
avoid PII
avoid secrets
remain bounded
```

---

# 169. Profiler security policy

```php
interface QueryProfilerSecurityPolicy
{
    public function allowSqlText(): bool;

    public function allowNormalizedSql(): bool;

    public function allowParameterTypes(): bool;

    public function allowSourceLocation(): bool;

    public function allowExplain(): bool;
}
```

---

# 170. Query profile redactor

```php
interface QueryProfileRedactor
{
    public function redact(
        QueryProfile $profile
    ): SafeQueryProfile;
}
```

---

# 171. Internal profile ≠ exported profile

Importante:

```text
Internal QueryProfile
↓
Redaction
↓
Exportable QueryProfile
```

---

# 172. Export boundaries

Podrán existir exporters hacia:

```text
OpenTelemetry
logs
developer toolbar
test recorder
custom telemetry backend
```

---

# 173. Provider agnosticism

El Query Profiler no dependerá directamente de:

```text
Datadog
New Relic
Grafana
Jaeger
Zipkin
```

---

# 174. OpenTelemetry integration

Podrá existir mediante adapter.

---

# 175. Profile storage

El core podrá utilizar:

```text
in-memory bounded session store
```

para debugging.

Persistencia histórica será extensión/provider.

---

# 176. QueryProfileStore

```php
interface QueryProfileStore
{
    public function record(QueryProfile $profile): void;

    public function session(
        QueryProfileSessionId $session
    ): QueryProfileCollection;
}
```

---

# 177. Production storage

No guardar todos los profiles indefinidamente.

---

# 178. Retention

Será responsabilidad de:

```text
telemetry backend
operations policy
```

---

# 179. Sampling architecture

```php
interface QueryProfilerSampler
{
    public function shouldProfile(
        QueryProfileCandidate $candidate
    ): QueryProfilingDecision;
}
```

---

# 180. Sampling decision

```php
enum QueryProfilingDecision
{
    case SKIP;
    case MINIMAL;
    case STANDARD;
    case DETAILED;
}
```

---

# 181. Adaptive sampling

Podrá incrementarse detalle para:

```text
failed queries
slow queries
unknown outcomes
rare operations
debug sessions
```

---

# 182. Adaptive sampling circularity

No siempre se sabe que una query será lenta antes de ejecutarla.

Puede aplicarse:

```text
lightweight timing first
↓
retain detailed post-execution context if threshold exceeded
```

---

# 183. Retrospective detail limitation

No se podrá recuperar información nunca capturada.

Por tanto el sistema deberá definir:

```text
always-captured lightweight evidence
```

---

# 184. Lightweight baseline

Idealmente:

```text
fingerprint
operation
start/end
outcome
connection role
rows
```

con overhead mínimo.

---

# 185. Detailed evidence

Solo cuando sea necesario:

```text
phase breakdown
source location
plan details
ORM relationship context
```

---

# 186. Profiling budget

```php
final readonly class QueryProfilerBudget
{
    public function __construct(
        public int $maxProfiles,
        public int $maxFingerprints,
        public int $maxDiagnostics,
        public int $maxExplainPlans,
        public int $maxSourceLocations,
        public int $maxMemoryBytes,
    ) {}
}
```

---

# 187. Budget exhaustion

Debe degradar:

```text
DETAILED
↓
STANDARD
↓
MINIMAL
↓
AGGREGATE ONLY
```

antes de afectar ejecución.

---

# 188. Profiler failure

Nunca deberá provocar:

```text
database query failure
```

por default.

---

# 189. Memory governance

El profiler deberá controlar:

```text
profile objects
SQL strings
plans
diagnostics
stack traces
aggregate maps
```

---

# 190. Fingerprint explosion

Queries generadas dinámicamente pueden producir demasiados fingerprints.

Debe existir:

```text
max fingerprints
overflow bucket
```

---

# 191. Overflow bucket

Conceptualmente:

```text
OTHER_QUERY_FAMILIES
```

sin perder conteos globales.

---

# 192. Query profile cardinality

No usar IDs únicos como métricas.

IDs son para traces/diagnostics.

---

# 193. Persistent runtime safety

Crítico en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 194. Request-scoped profiler session

Cada request/job deberá obtener:

```text
fresh QueryProfileSession
```

---

# 195. State que no deberá sobrevivir

```text
active profiles
query records
source locations
session aggregates
diagnostics
temporary explain plans
```

---

# 196. Shared immutable state

Puede sobrevivir:

```text
metric instruments
compiled policies
fingerprint algorithms
redaction rules
immutable descriptors
```

---

# 197. Static active profile

Prohibido:

```php
static ?QueryProfile $current;
```

---

# 198. Coroutine safety

En OpenSwoole:

```text
Coroutine A
→ Profile Session A

Coroutine B
→ Profile Session B
```

deberán permanecer aisladas.

---

# 199. Nested operations

Puede existir:

```text
ORM operation
└── query
    └── relationship hydration
```

Los contextos deberán manejar nesting explícitamente.

---

# 200. Async/concurrent queries

No depender de una simple pila global:

```text
push current query
pop current query
```

porque varias queries pueden coexistir.

---

# 201. Context propagation

Usar:

```text
scope-aware context
coroutine-aware context
fiber-aware context
explicit context propagation
```

según Runtime abstraction.

---

# 202. Connection pool profiling

El Profiler podrá responder:

```text
¿La query fue lenta porque esperó conexión?
```

Ejemplo:

```text
Total:          620 ms
Pool wait:      580 ms
DB execution:    18 ms
Hydration:       12 ms
Other:           10 ms
```

Diagnóstico:

```text
CONNECTION_POOL_CONTENTION
```

---

# 203. Compilation profiling

Ejemplo:

```text
Total:           45 ms
Compilation:     31 ms
DB execution:     4 ms
Hydration:        3 ms
```

Podría indicar:

```text
compiled query cache ineffective
high query shape churn
complex compilation
```

---

# 204. Hydration profiling

Ejemplo:

```text
Total:          500 ms
DB execution:    40 ms
Hydration:      390 ms
```

Diagnóstico:

```text
HIGH_HYDRATION_COST
```

---

# 205. Result cardinality profiling

Ejemplo:

```text
Rows returned: 250,000
Rows used:       2,000
```

podría indicar:

```text
over-fetching
```

pero deberá distinguir:

```text
observed
```

de:

```text
inferred
```

---

# 206. Projection recommendation

Si una entidad completa se hidrata para utilizar dos campos:

```text
Projection/DTO
```

puede ser recomendación.

Nunca transformación automática.

---

# 207. Query frequency profile

Ejemplo:

```text
Fingerprint:
  user_by_id

Executions:
  8,420

Average:
  0.8 ms

Total:
  6.7 s
```

Esto puede ser más importante que una query individual de 100 ms.

---

# 208. Total-cost ranking

El profiler deberá soportar:

```text
ORDER BY aggregate_total_cost DESC
```

conceptualmente.

---

# 209. Profile severity

Podrá clasificarse:

```php
enum QueryProfileSeverity
{
    case NORMAL;
    case NOTICE;
    case WARNING;
    case CRITICAL;
}
```

---

# 210. Severity policy

No estará hardcoded universalmente.

Dependerá de:

```text
environment
operation class
thresholds
budgets
```

---

# 211. CLI profiling

VoltStack podrá ofrecer:

```bash
php volt database:profile
```

en el futuro DX/CLI layer.

El core Query Profiler no dependerá de CLI.

---

# 212. Explicit profiling scope

API conceptual:

```php
DB::profile(function () {
    // database workload
});
```

---

# 213. Profile result

Podría devolver:

```php
$session = DB::profile(...);

$session->summary();
```

---

# 214. Framework independence

El Profiler no dependerá de:

```text
HTTP
CLI
Debug Toolbar
```

---

# 215. Query explain API

Conceptualmente:

```php
DB::profile($query)
    ->explain();
```

pero `explain()` deberá respetar safety policy.

---

# 216. Query comparison API

```php
$comparison = $profiler->compare(
    baseline: $before,
    candidate: $after,
);
```

---

# 217. Testing integration

Podrá permitir:

```php
QueryProfiler::assertQueryCount(5);
```

---

# 218. Query family assertion

```php
QueryProfiler::assertFingerprintExecuted(
    $fingerprint,
    times: 1,
);
```

---

# 219. Total DB time assertion

```php
QueryProfiler::assertDatabaseTimeBelow(
    milliseconds: 100,
);
```

---

# 220. Hydration assertion

```php
QueryProfiler::assertHydrationTimeBelow(
    milliseconds: 50,
);
```

---

# 221. Pool wait assertion

```php
QueryProfiler::assertConnectionWaitBelow(
    milliseconds: 10,
);
```

---

# 222. No N+1 assertion

La implementación real deberá delegar al sistema 223.

No duplicar detector.

---

# 223. Explain assertion

Tests de infraestructura podrán comprobar:

```text
no full scan
index used
```

solo mediante abstraction/platform capability.

---

# 224. Explain plan stability

No asumir que planes DB son idénticos entre versiones.

Tests deberán evitar assertions excesivamente frágiles.

---

# 225. Performance tests

El profiler podrá proporcionar evidencia a:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 226. Production-safe profiling

Configuración recomendada:

```text
lightweight metrics
low-cardinality fingerprints
sampling
redaction
no automatic EXPLAIN ANALYZE
bounded storage
```

---

# 227. Development profiling

Podrá habilitar:

```text
detailed timeline
normalized SQL
ORM origin
source location
EXPLAIN
diagnostics
```

---

# 228. Benchmark profiling

Para benchmarks precisos se deberá medir:

```text
with profiler
without profiler
```

para cuantificar overhead.

---

# 229. Observer effect

Regla:

> Medir una operación puede modificar ligeramente su costo observado.

El Profiler deberá minimizar y documentar este efecto.

---

# 230. Profiler overhead

Podrá medirse aproximadamente:

```text
T_instrumented
-
T_baseline
```

en benchmarks controlados.

---

# 231. Profiler overhead ≠ query cost

No mezclarlo con costo real de aplicación.

---

# 232. QueryProfileCollector

```php
interface QueryProfileCollector
{
    public function begin(
        QueryOperationDescriptor $operation,
        QueryProfileContext $context,
    ): QueryProfileScope;
}
```

---

# 233. QueryProfileScope

```php
interface QueryProfileScope
{
    public function phaseStarted(
        QueryProfilePhase $phase
    ): QueryPhaseScope;

    public function correlate(
        QueryProfileCorrelation $correlation
    ): void;

    public function finish(
        QueryProfileResult $result
    ): QueryProfile;
}
```

---

# 234. Null collector

```php
final class NullQueryProfileCollector
    implements QueryProfileCollector
{
    // near-zero overhead
}
```

---

# 235. QueryProfileCorrelation

```php
final readonly class QueryProfileCorrelation
{
    public function __construct(
        public ?ConnectionOperationId $connectionId,
        public ?TransactionId $transactionId,
        public ?ORMOperationId $ormOperationId,
        public ?PersistenceFlushId $flushId,
        public ?HydrationOperationId $hydrationId,
    ) {}
}
```

---

# 236. QueryProfileResult

```php
final readonly class QueryProfileResult
{
    public function __construct(
        public QueryProfileOutcome $outcome,
        public ?int $rowsReturned,
        public ?int $rowsConsumed,
        public ?int $rowsAffected,
        public QueryProfileCoverage $coverage,
    ) {}
}
```

---

# 237. QueryProfileOutcome

```php
enum QueryProfileOutcome
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

# 238. UNKNOWN outcome

Debe conservarse si el Execution Engine no puede afirmar el resultado.

---

# 239. Profile diagnostic

```php
final readonly class QueryProfileDiagnostic
{
    public function __construct(
        public QueryDiagnosticCode $code,
        public QueryProfileSeverity $severity,
        public DiagnosticConfidence $confidence,
        public string $summary,
    ) {}
}
```

---

# 240. Diagnostic text

No deberá incluir SQL/parameters sensibles salvo policy explícita.

---

# 241. QueryProfileAnalyzer

```php
interface QueryProfileAnalyzer
{
    public function analyze(
        QueryProfile $profile
    ): QueryProfileDiagnostics;
}
```

---

# 242. Analyzer architecture

Podrán existir analizadores independientes:

```text
ConnectionWaitAnalyzer
DatabaseLatencyAnalyzer
HydrationCostAnalyzer
CardinalityAnalyzer
QueryFrequencyAnalyzer
CompilationCostAnalyzer
TransactionContextAnalyzer
ShardFanoutAnalyzer
CacheBehaviorAnalyzer
```

---

# 243. Analyzer ≠ optimizer

Un analyzer genera:

```text
evidence
diagnostic
recommendation
```

no modifica query.

---

# 244. Explain analyzer

Separado:

```text
ExplainPlanAnalyzer
```

---

# 245. Analyzer extension system

Nuevos analizadores podrán registrarse mediante:

```text
ProfilerAnalyzerRegistry
```

con contratos estables.

---

# 246. Custom analyzer safety

No deberá recibir conexiones mutables ni EntityManager.

Trabajará sobre profiles/snapshots.

---

# 247. Profiler event integration

Podrán existir eventos:

```text
QueryProfileCompleted
QueryProfileDiagnosticProduced
QueryProfileSessionCompleted
```

---

# 248. Events ≠ canonical telemetry source

No deberán duplicar mediciones.

---

# 249. Event subscriber failure

No cambiará query outcome por default.

---

# 250. Query Profile Report

Ejemplo:

```text
QUERY PROFILE
────────────────────────────────────────

Fingerprint:
  select.users.by_status.v3

Operation:
  SELECT

Source:
  ORM Repository

Total:
  151.4 ms

Phases:
  Semantic Analysis       1.1 ms
  Optimization            0.8 ms
  Planning                0.7 ms
  Compilation             1.6 ms
  Connection Resolution   0.2 ms
  Pool Wait              34.7 ms
  Statement Preparation   0.5 ms
  Database Execution     11.8 ms
  Result Consumption      8.2 ms
  Hydration              73.9 ms
  ORM Assembly           17.1 ms

Rows:
  returned: 4,200
  consumed: 4,200

Entities:
  created: 850
  reused: 3,350

Connection:
  role: REPLICA
  reused: yes

Transaction:
  none

Diagnostics:
  WARNING — High hydration cost
  NOTICE  — Connection pool wait contributes 22.9%

Profile Coverage:
  COMPLETE
```

---

# 251. Aggregate report

```text
QUERY FAMILIES
────────────────────────────────────────

Fingerprint               Count    Avg      Total
───────────────────────────────────────────────
user_by_id                 8,420    0.8ms    6.7s
orders_for_user            8,100    1.1ms    8.9s
dashboard_aggregate          120   45.0ms    5.4s
report_full_scan               4  900.0ms    3.6s
```

Esto muestra que:

```text
orders_for_user
```

puede tener mayor costo agregado aunque ninguna ejecución individual sea lenta.

---

# 252. Timeline report

```text
0 ms
│
├─ Build
├─ Analyze
├─ Optimize
├─ Compile
│
├──────────── Pool Wait ─────────────┤
│
├── DB Execute ──┤
│
├──── Result Transfer ────┤
│
├──────────────── Hydration ────────────────┤
│
└──────────────────────────────────────────── 151 ms
```

---

# 253. ORM correlation report

```text
HTTP Request
│
└── UserRepository::findWithOrders()
    │
    ├── Query users
    │   └── 3.1 ms
    │
    ├── Hydration
    │   └── 1.8 ms
    │
    └── Relationship orders
        │
        ├── Query #1
        ├── Query #2
        ├── Query #3
        └── ...
```

El Query Profiler registra estructura.

El sistema N+1 decide si representa un patrón problemático.

---

# 254. Distributed query report

```text
LOGICAL QUERY PROFILE
────────────────────────────────

Strategy:
  MULTI_SHARD

Shards:
  4

Physical queries:

  shard-01   22 ms
  shard-02   18 ms
  shard-03   91 ms
  shard-04   20 ms

Merge:
  8 ms

Critical shard:
  shard-03

Wall duration:
  103 ms

Aggregate DB work:
  151 ms
```

---

# 255. Query profile diagnostics example

```text
OBSERVED
────────────────────────

Total duration:
  520 ms

Pool wait:
  6 ms

Database execution:
  42 ms

Rows:
  120,000

Hydration:
  431 ms


INFERRED
────────────────────────

Most application-visible latency occurs
after database execution.

High row cardinality contributes to
hydration cost.


RECOMMENDED
────────────────────────

Evaluate:
- narrower projection
- DTO/scalar hydration
- filtering earlier in the query
- reduced relationship graph
- chunk/lazy processing if full materialization
  is unnecessary
```

---

# 256. Directory structure

```text
src/Quantum/Database/Telemetry/Profiler/
│
├── Contract/
│   ├── QueryProfileCollector.php
│   ├── QueryProfileScope.php
│   ├── QueryPhaseScope.php
│   ├── QueryProfileAnalyzer.php
│   ├── QueryProfileStore.php
│   ├── QueryProfilerSampler.php
│   ├── QueryProfileRedactor.php
│   └── QueryProfilerSecurityPolicy.php
│
├── Identity/
│   ├── QueryProfileId.php
│   └── QueryProfileSessionId.php
│
├── Profile/
│   ├── QueryProfile.php
│   ├── QueryProfileContext.php
│   ├── QueryProfileResult.php
│   ├── QueryProfileCost.php
│   ├── QueryProfileCoverage.php
│   ├── QueryProfileOutcome.php
│   └── QueryProfileCorrelation.php
│
├── Phase/
│   ├── QueryProfilePhase.php
│   ├── QueryProfileTimeline.php
│   ├── QueryProfilePhaseMeasurement.php
│   ├── MeasurementState.php
│   ├── MeasurementSource.php
│   └── MeasurementConfidence.php
│
├── Fingerprint/
│   ├── QueryFingerprint.php
│   ├── QueryFingerprintFactory.php
│   ├── SemanticQueryFingerprint.php
│   ├── ExecutionVariantFingerprint.php
│   └── QueryFamily.php
│
├── Session/
│   ├── QueryProfileSession.php
│   ├── QueryProfileSessionContext.php
│   ├── QueryProfileSessionSummary.php
│   └── QueryProfileSessionManager.php
│
├── Aggregation/
│   ├── QueryProfileAggregator.php
│   ├── QueryProfileAggregate.php
│   ├── QueryProfileAggregateSet.php
│   ├── QueryProfileDistribution.php
│   └── BoundedQueryProfileAggregator.php
│
├── Analyzer/
│   ├── ConnectionWaitAnalyzer.php
│   ├── DatabaseLatencyAnalyzer.php
│   ├── CompilationCostAnalyzer.php
│   ├── ResultCardinalityAnalyzer.php
│   ├── HydrationCostAnalyzer.php
│   ├── QueryFrequencyAnalyzer.php
│   ├── TransactionContextAnalyzer.php
│   ├── ShardFanoutAnalyzer.php
│   ├── CacheBehaviorAnalyzer.php
│   └── QueryProfileAnalyzerPipeline.php
│
├── Diagnostics/
│   ├── QueryProfileDiagnostic.php
│   ├── QueryProfileDiagnostics.php
│   ├── QueryDiagnosticCode.php
│   ├── QueryProfileSeverity.php
│   ├── DiagnosticConfidence.php
│   └── QueryProfileExplainer.php
│
├── Explain/
│   ├── ExplainPlanProvider.php
│   ├── ExplainContext.php
│   ├── ExplainResult.php
│   ├── ExplainSafety.php
│   ├── QueryExplainPolicy.php
│   ├── ExplainPlanAnalyzer.php
│   └── ExplainPlanNormalizer.php
│
├── Comparison/
│   ├── QueryProfileComparator.php
│   ├── QueryProfileComparison.php
│   ├── QueryProfileDelta.php
│   └── QueryProfileEnvironment.php
│
├── Security/
│   ├── DefaultQueryProfilerSecurityPolicy.php
│   ├── QueryProfileRedactor.php
│   ├── SafeQueryProfile.php
│   └── SqlProfileRedactor.php
│
├── Policy/
│   ├── QueryProfilerPolicy.php
│   ├── CompiledQueryProfilerPolicy.php
│   ├── QueryProfilerMode.php
│   └── QueryProfilerBudget.php
│
├── Sampling/
│   ├── QueryProfilerSampler.php
│   ├── QueryProfilingDecision.php
│   ├── AdaptiveQueryProfilerSampler.php
│   └── AlwaysQueryProfilerSampler.php
│
├── Store/
│   ├── InMemoryQueryProfileStore.php
│   └── NullQueryProfileStore.php
│
├── Instrumentation/
│   ├── DefaultQueryProfileCollector.php
│   ├── DefaultQueryProfileScope.php
│   ├── DefaultQueryPhaseScope.php
│   └── NullQueryProfileCollector.php
│
├── Report/
│   ├── QueryProfileReport.php
│   ├── QueryProfileReportBuilder.php
│   ├── QueryProfileSessionReport.php
│   └── QueryProfileFormatter.php
│
├── Testing/
│   ├── RecordingQueryProfiler.php
│   ├── QueryProfilerAssertions.php
│   └── FakeProfilerClock.php
│
└── Exception/
    ├── QueryProfilerException.php
    ├── QueryProfileStateException.php
    ├── QueryProfileBudgetException.php
    ├── QueryExplainException.php
    └── QueryProfilerSecurityException.php
```

---

# 257. Flujo completo

```text
Application
    │
    ▼
Query Builder / ORM
    │
    ▼
Query Model
    │
    ├───────────────┐
    ▼               │
Semantic Engine     │
    ▼               │
Optimizer           │
    ▼               │
Planner             │
    ▼               │
Compiler            │
    ▼               │
Executor            │
    ▼               │
Connection          │
    ▼               │
Database            │
    ▼               │
Result              │
    ▼               │
Hydration / ORM     │
                    │
Telemetry Signals ──┘
    │
    ▼
Query Profile Collector
    │
    ▼
Correlation Engine
    │
    ▼
QueryProfile
    │
    ├── Analyzer Pipeline
    ├── Aggregator
    ├── Explain integration
    ├── Comparison
    └── Safe Export
```

---

# 258. Architectural invariants

## DB-QPROF-001
Query Profiler observará evidencia; no ejecutará queries por sí mismo salvo operaciones explícitas de EXPLAIN autorizadas.

## DB-QPROF-002
Query Profiler será distinto de Query Telemetry.

## DB-QPROF-003
Query Profiler será distinto de Query Optimizer.

## DB-QPROF-004
Query Profiler será distinto de Query Planner.

## DB-QPROF-005
Query Profiler será distinto de SQL Compiler.

## DB-QPROF-006
Query Profiler será distinto de Executor.

## DB-QPROF-007
Query Profiler será distinto de Slow Query Detector.

## DB-QPROF-008
Query Profiler será distinto de N+1 Detector.

## DB-QPROF-009
Query Profiler será distinto de DBMS EXPLAIN.

## DB-QPROF-010
Query Profile será distinto de Execution Plan.

## DB-QPROF-011
QueryFingerprint será distinto de raw SQL.

## DB-QPROF-012
Database execution time será distinto de application-visible query time.

## DB-QPROF-013
Slow SQL será distinto de slow ORM.

## DB-QPROF-014
High frequency será distinto de high individual cost.

## DB-QPROF-015
High individual cost será distinto de high aggregate cost.

## DB-QPROF-016
Cada subsystem conservará propiedad de su medición canonical.

## DB-QPROF-017
Profiler correlacionará mediciones sin duplicarlas.

## DB-QPROF-018
QueryProfileId será distinto de QueryOperationId.

## DB-QPROF-019
QueryProfileSession será distinta de Transaction.

## DB-QPROF-020
QueryProfileSession será distinta de HTTP Request.

## DB-QPROF-021
Profiler disabled tendrá overhead mínimo.

## DB-QPROF-022
Completed QueryProfile será immutable.

## DB-QPROF-023
Partial profile será representable.

## DB-QPROF-024
Partial profile no será automáticamente inválido.

## DB-QPROF-025
UNKNOWN measurement no será convertido en cero.

## DB-QPROF-026
NOT_APPLICABLE será distinto de UNKNOWN.

## DB-QPROF-027
Internal context será distinto de exported context.

## DB-QPROF-028
Fingerprint no dependerá de simple regex sobre SQL.

## DB-QPROF-029
Fingerprint deberá preferir estructura semántica.

## DB-QPROF-030
Parameter values no formarán parte del fingerprint por default.

## DB-QPROF-031
Sensitive parameter values no serán almacenados por default.

## DB-QPROF-032
Profiler soportará fases de ejecución.

## DB-QPROF-033
No todas las fases deberán existir para todas las queries.

## DB-QPROF-034
Phase measurement conservará confidence.

## DB-QPROF-035
Derived measurement será distinta de observed measurement.

## DB-QPROF-036
Duraciones usarán clock monotónico cuando sea posible.

## DB-QPROF-037
Wall clock será distinto de monotonic duration clock.

## DB-QPROF-038
Fases podrán superponerse.

## DB-QPROF-039
No se sumarán fases ciegamente.

## DB-QPROF-040
Critical path será distinto de aggregate work time.

## DB-QPROF-041
Database execution será distinto de DB CPU.

## DB-QPROF-042
Query executed será distinto de result consumed.

## DB-QPROF-043
Buffered y streaming results podrán tener perfiles distintos.

## DB-QPROF-044
Rows returned será distinto de rows consumed.

## DB-QPROF-045
Rows returned será distinto de hydrated entities.

## DB-QPROF-046
Rows affected será distinto de entities changed.

## DB-QPROF-047
Compilation telemetry podrá correlacionarse con compiled query cache.

## DB-QPROF-048
Compiled Query Cache será distinto de Result Cache.

## DB-QPROF-049
Result Cache hit podrá evitar DB execution.

## DB-QPROF-050
Pool wait será distinto de DB execution.

## DB-QPROF-051
Connection establishment será distinto de pool wait.

## DB-QPROF-052
Endpoint role será bounded.

## DB-QPROF-053
Replica identity no será metric label por default.

## DB-QPROF-054
Replica lag será distinto de query latency.

## DB-QPROF-055
Routing explanation no cambiará routing decision.

## DB-QPROF-056
Logical distributed query podrá producir múltiples physical query profiles.

## DB-QPROF-057
Fan-out aggregate work será distinto de wall latency.

## DB-QPROF-058
Transaction age será distinto de query duration.

## DB-QPROF-059
Lock wait desconocido permanecerá UNKNOWN.

## DB-QPROF-060
ORM operation podrá correlacionarse con query profile.

## DB-QPROF-061
Flush podrá agrupar múltiples query profiles.

## DB-QPROF-062
Query families se agruparán mediante fingerprints estables.

## DB-QPROF-063
Individually fast query podrá ser aggregate expensive.

## DB-QPROF-064
Average no será única estadística de performance.

## DB-QPROF-065
Percentiles deberán implementarse con almacenamiento bounded.

## DB-QPROF-066
Profile aggregation será bounded.

## DB-QPROF-067
Parallel query durations podrán sumar más que session wall time.

## DB-QPROF-068
Raw queries podrán tener profile coverage parcial.

## DB-QPROF-069
EXPLAIN será capability-driven.

## DB-QPROF-070
EXPLAIN ANALYZE podrá ejecutar consultas.

## DB-QPROF-071
EXPLAIN ANALYZE no será automático por default.

## DB-QPROF-072
Unsafe write EXPLAIN ANALYZE estará deshabilitado por default.

## DB-QPROF-073
UNKNOWN explain safety no será SAFE.

## DB-QPROF-074
VoltStack Query Plan será distinto de DBMS Execution Plan.

## DB-QPROF-075
Profiler podrá correlacionar ambos tipos de plan.

## DB-QPROF-076
Vendor plan details podrán preservarse bajo policy.

## DB-QPROF-077
Profiler no creará índices automáticamente.

## DB-QPROF-078
Profiler no eliminará índices automáticamente.

## DB-QPROF-079
Recommendation será distinta de observation.

## DB-QPROF-080
Diagnostic confidence será explícita.

## DB-QPROF-081
Slow Query Detector consumirá Query Profiles.

## DB-QPROF-082
Repeated Query será distinta de N+1.

## DB-QPROF-083
N+1 Detector interpretará semántica ORM separadamente.

## DB-QPROF-084
Profile comparison será soportada.

## DB-QPROF-085
Una muestra lenta no probará regresión.

## DB-QPROF-086
Regression analysis considerará sample confidence.

## DB-QPROF-087
Environment compatibility será relevante para comparaciones.

## DB-QPROF-088
Version será distinta de capability.

## DB-QPROF-089
Normalized metrics no asumirán linearidad universal.

## DB-QPROF-090
Arbitrary user tags no serán metric labels sin policy.

## DB-QPROF-091
Raw SQL no será exportado por default en producción.

## DB-QPROF-092
Normalized/redacted SQL será preferido para diagnostics.

## DB-QPROF-093
Parameter types podrán exponerse bajo policy.

## DB-QPROF-094
Parameter lengths podrán considerarse sensibles.

## DB-QPROF-095
Internal profile será distinto de exported profile.

## DB-QPROF-096
Redaction ocurrirá antes de external export.

## DB-QPROF-097
Profiler será provider-agnostic.

## DB-QPROF-098
Historical retention no será responsabilidad obligatoria del core.

## DB-QPROF-099
Sampling será configurable.

## DB-QPROF-100
Detailed sampling podrá activarse adaptativamente.

## DB-QPROF-101
Información no capturada no podrá reconstruirse mágicamente.

## DB-QPROF-102
Existirá lightweight baseline instrumentation.

## DB-QPROF-103
Profiling budget será bounded.

## DB-QPROF-104
Budget exhaustion degradará detalle antes de afectar ejecución.

## DB-QPROF-105
Profiler failure no deberá fallar queries por default.

## DB-QPROF-106
Profile memory será gobernada.

## DB-QPROF-107
Fingerprint explosion será controlada.

## DB-QPROF-108
Unique IDs no serán metric labels.

## DB-QPROF-109
Profiler session state será request/job scoped.

## DB-QPROF-110
Active profiles no sobrevivirán accidentalmente entre requests.

## DB-QPROF-111
Immutable profiler configuration podrá compartirse entre workers.

## DB-QPROF-112
No existirá static mutable current profile.

## DB-QPROF-113
Coroutine profiling será aislado.

## DB-QPROF-114
Concurrent queries no dependerán de global stack state.

## DB-QPROF-115
Context propagation será runtime-aware.

## DB-QPROF-116
High pool wait no será diagnosticado como slow SQL.

## DB-QPROF-117
High compilation cost será observable.

## DB-QPROF-118
High hydration cost será observable.

## DB-QPROF-119
High cardinality será observable.

## DB-QPROF-120
Over-fetching será inference, no fact automático.

## DB-QPROF-121
Projection recommendation no modificará query.

## DB-QPROF-122
Aggregate total cost será ranking soportado.

## DB-QPROF-123
Severity thresholds serán policy-driven.

## DB-QPROF-124
Core Profiler será independiente de CLI.

## DB-QPROF-125
Core Profiler será independiente de HTTP.

## DB-QPROF-126
Core Profiler será independiente de Debug Toolbar.

## DB-QPROF-127
Explain API respetará safety policy.

## DB-QPROF-128
Testing podrá inspeccionar query count.

## DB-QPROF-129
Testing podrá inspeccionar fingerprints.

## DB-QPROF-130
Testing podrá inspeccionar DB time.

## DB-QPROF-131
Testing podrá inspeccionar hydration time.

## DB-QPROF-132
Testing podrá inspeccionar connection wait.

## DB-QPROF-133
N+1 testing se delegará al sistema N+1.

## DB-QPROF-134
Explain assertions deberán considerar platform capabilities.

## DB-QPROF-135
Execution plans no se asumirán estables entre DB versions.

## DB-QPROF-136
Production profiling será redacted y bounded por default.

## DB-QPROF-137
Development profiling podrá tener mayor detalle.

## DB-QPROF-138
Profiler overhead deberá benchmarkearse.

## DB-QPROF-139
Observer effect será reconocido.

## DB-QPROF-140
Profiler overhead será distinto de application query cost.

## DB-QPROF-141
NullQueryProfileCollector estará disponible.

## DB-QPROF-142
Custom analyzers trabajarán sobre snapshots/profiles.

## DB-QPROF-143
Custom analyzers no recibirán mutable Connection por default.

## DB-QPROF-144
Custom analyzers no recibirán mutable EntityManager por default.

## DB-QPROF-145
Profiler events no duplicarán canonical metrics.

## DB-QPROF-146
Profiler event failure no cambiará query outcome por default.

## DB-QPROF-147
Query Profile podrá expresar UNKNOWN outcome.

## DB-QPROF-148
Query Profile podrá expresar CANCELLED.

## DB-QPROF-149
Query Profile podrá expresar TIMEOUT.

## DB-QPROF-150
Query Profile podrá expresar PARTIAL coverage.

## DB-QPROF-151
Diagnostics separarán observed/inferred/recommended.

## DB-QPROF-152
Profiler podrá identificar costo dominante.

## DB-QPROF-153
Costo dominante no implicará automáticamente causa raíz.

## DB-QPROF-154
Query source será bounded.

## DB-QPROF-155
Query operation type será bounded.

## DB-QPROF-156
Logical and physical distributed profiles estarán separados.

## DB-QPROF-157
Shard failures no producirán fake complete profile.

## DB-QPROF-158
Unknown shard outcome permanecerá UNKNOWN.

## DB-QPROF-159
Cache hit será observable sin fingir DB execution.

## DB-QPROF-160
Cache miss será distinto de DB failure.

## DB-QPROF-161
Compiled cache hit será distinto de result cache hit.

## DB-QPROF-162
Query Profile podrá representar zero-query ORM operations solo mediante ORM context, no como query ficticia.

## DB-QPROF-163
Profiler nunca inventará fases ausentes.

## DB-QPROF-164
Profiler nunca inventará DB server metrics no observadas.

## DB-QPROF-165
Profiler nunca asumirá que execute() representa consumo completo del resultado.

## DB-QPROF-166
Profiler nunca asumirá que query latency es exclusivamente DB latency.

## DB-QPROF-167
Profiler nunca asumirá que alta frecuencia implica error.

## DB-QPROF-168
Profiler nunca asumirá que full scan implica necesariamente falta de índice.

## DB-QPROF-169
Profiler preservará evidencia y confianza.

## DB-QPROF-170
Profiler será explicable.

## DB-QPROF-171
Profiler será determinista respecto de la misma evidencia y policy.

## DB-QPROF-172
Profiler permanecerá desacoplado de providers concretos.

## DB-QPROF-173
Profiler respetará tenant isolation.

## DB-QPROF-174
TenantId no será external metric label por default.

## DB-QPROF-175
Profiler respetará shard context.

## DB-QPROF-176
Profiler respetará transaction context.

## DB-QPROF-177
Profiler respetará persistent-runtime isolation.

## DB-QPROF-178
Profiler no retendrá runtime Connection objects.

## DB-QPROF-179
Profiler no retendrá runtime EntityManager objects.

## DB-QPROF-180
Profiler no retendrá hydrated entity graphs.

---

# 259. Modelo formal

Sea una operación de consulta:

```text
Q
```

y un conjunto de observaciones:

```text
E(Q) =
{
    e_query,
    e_connection,
    e_transaction,
    e_result,
    e_orm,
    e_cache,
    e_distribution
}
```

El profiler produce:

```text
P(Q) = Profile(E(Q), Policy)
```

---

# 260. Profile completeness

Sea:

```text
Required(P)
```

el conjunto de observaciones deseadas para el modo actual.

Entonces:

```text
Coverage(P)
=
Observed(P)
/
Required(P)
```

conceptualmente.

No necesariamente se representará como porcentaje numérico.

Preferir:

```text
COMPLETE
PARTIAL
MINIMAL
UNKNOWN
```

---

# 261. Costo total

Para un pipeline estrictamente secuencial:

```text
T_total
≈
T_build
+
T_semantic
+
T_optimize
+
T_plan
+
T_compile
+
T_connection
+
T_execute
+
T_result
+
T_hydration
+
T_orm
```

Pero para pipelines concurrentes/streaming:

```text
T_total
≠
Σ T_phase
```

necesariamente.

---

# 262. Costo agregado

Para fingerprint `F`:

```text
TotalCost(F)
=
Σ Duration(Q_i)
```

para:

```text
Fingerprint(Q_i) = F
```

---

# 263. Frecuencia

```text
Frequency(F)
=
Count(Q_i)
```

---

# 264. Costo promedio

```text
AverageCost(F)
=
TotalCost(F)
/
Frequency(F)
```

si:

```text
Frequency(F) > 0
```

---

# 265. Dominancia

Para fase `p`:

```text
Dominance(p)
=
T_p
/
T_total
```

solo cuando las mediciones sean semánticamente comparables y no se solapen de forma problemática.

---

# 266. Distributed critical path

Para ejecución paralela en shards:

```text
T_query
≈
max(T_shard_1 ... T_shard_n)
+
T_merge
+
T_coordination
```

no:

```text
Σ T_shard
```

como latencia de usuario.

---

# 267. Filosofía de diagnóstico

VoltStack seguirá:

```text
Measure
↓
Correlate
↓
Classify
↓
Explain
↓
Recommend
```

y no:

```text
Measure
↓
Guess
↓
Mutate Production
```

---

# 268. Ejemplo integral

Consulta ORM:

```php
$users = User::query()
    ->where('status', 'active')
    ->with('orders')
    ->get();
```

Pipeline:

```text
Model API
   │
   ▼
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
Planner
   │
   ▼
Compiler
   │
   ▼
Read Routing
   │
   ▼
Replica
   │
   ▼
Execution
   │
   ▼
Result
   │
   ▼
Hydration
   │
   ▼
IdentityMap
   │
   ▼
Relationship Loader
```

El profile podría producir:

```text
Root query:
  18 ms

Relationship batch:
  24 ms

Connection waits:
  2 ms

Hydration:
  37 ms

ORM assembly:
  11 ms

Total operation:
  92 ms
```

La conclusión correcta no sería simplemente:

```text
SQL took 92 ms
```

sino:

```text
Database work:        42 ms
Application ORM work: 48 ms
Connection overhead:   2 ms
```

---

# 269. Regla maestra

> **VoltStack Query Profiler deberá convertir telemetría fragmentada en una explicación coherente del costo de una consulta, preservando la diferencia entre tiempo de framework, espera de conexión, ejecución del DBMS, transferencia de resultados, hidratación, ORM, cache, transacción y distribución.**

La arquitectura final será:

```text
Query
↓
Telemetry
↓
Correlation
↓
Profile
↓
Aggregation
↓
Analysis
↓
Diagnostics
↓
Safe Recommendations
```

Nunca:

```text
Profiler
↓
Automatic Query Rewrite
```

---

# 270. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
✓ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
○ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 271. Siguiente documento

```text
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
```

El siguiente documento deberá definir el sistema especializado para detectar consultas y operaciones de base de datos cuyo costo exceda las expectativas establecidas por VoltStack.

Deberá distinguir explícitamente:

```text
Slow Query
Slow DB Execution
Slow End-to-End Query
Slow Connection Acquisition
Slow Result Consumption
Slow Hydration
Slow ORM Operation
High-Frequency Query
High Aggregate Cost
Query Regression
```

y diseñar:

```text
threshold policies
absolute thresholds
relative thresholds
adaptive thresholds
query-family baselines
percentile-based detection
environment-aware thresholds
phase-specific thresholds
slow-query classifications
severity levels
fingerprint aggregation
sampling
slow query evidence
source correlation
transaction context
connection/pool context
replica/shard context
ORM context
EXPLAIN integration
safe diagnostic capture
redaction
production policies
rate limiting
deduplication
alert suppression
persistent runtime isolation
testing
telemetry
debug toolbar integration
```

bajo la regla:

> **Una consulta lenta en VoltStack deberá clasificarse según dónde se consume realmente el tiempo; superar un umbral de duración no autoriza a concluir automáticamente que el DBMS, el SQL o la ausencia de un índice sean la causa.**