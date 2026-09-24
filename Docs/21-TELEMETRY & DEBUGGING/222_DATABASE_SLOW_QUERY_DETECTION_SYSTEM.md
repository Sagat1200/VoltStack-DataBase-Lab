# 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md

# VoltStack Quantum Database
## Slow Query Detection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 222 — Slow Query Detection System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `221_DATABASE_QUERY_PROFILER_SYSTEM.md`  
**Siguiente documento:** `223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md`

---

# 1. Propósito

`Slow Query Detection System` define la arquitectura mediante la cual VoltStack identificará, clasificará, contextualizará y reportará consultas u operaciones de base de datos cuyo costo observado exceda los límites esperados.

La regla central será:

> **Una consulta lenta en VoltStack deberá clasificarse según dónde se consume realmente el tiempo; superar un umbral de duración no autoriza a concluir automáticamente que el DBMS, el SQL o la ausencia de un índice sean la causa.**

VoltStack deberá poder distinguir:

```text
Slow Query
├── Slow Compilation
├── Slow Connection Resolution
├── Slow Pool Acquisition
├── Slow Database Execution
├── Slow Result Transfer
├── Slow Result Consumption
├── Slow Hydration
├── Slow ORM Assembly
├── Slow Distributed Merge
└── Slow End-to-End Operation
```

Por tanto:

```text
Slow Application Query
≠
Slow SQL
```

y:

```text
Slow SQL
≠
Missing Index
```

---

# 2. Posición arquitectónica

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry
├── ORM Telemetry
│
├── Query Profiler
│      │
│      ▼
│  QueryProfile
│      │
│      ▼
├── Slow Query Detection        ← este documento
│
├── N+1 Telemetry
├── Debug Information
└── Developer Debug Toolbar
```

El sistema 222 consume principalmente:

```text
217 Query Telemetry
218 Connection Telemetry
219 Transaction Telemetry
220 ORM Telemetry
221 Query Profiler
```

---

# 3. Responsabilidad

El detector deberá:

```text
observe profile
↓
evaluate policy
↓
compare thresholds/baselines
↓
classify
↓
assign severity
↓
attach evidence
↓
deduplicate/rate-limit
↓
emit diagnostic
```

No deberá:

```text
rewrite query
create index
kill connection
retry query
change replica
change transaction isolation
change ORM mapping
modify SQL
```

---

# 4. Distinciones fundamentales

```text
Slow Query Detection
≠
Query Profiling

Slow Query Detection
≠
Query Optimization

Slow Query Detection
≠
N+1 Detection

Slow Query Detection
≠
Database EXPLAIN

Slow Query Detection
≠
Performance Benchmarking

Slow Query Detection
≠
Alert Delivery

Slow Query Detection
≠
Query Cancellation

Slow Query
≠
Failed Query

Slow Query
≠
High-Frequency Query

Slow Query
≠
High Aggregate Cost

Slow Query
≠
Performance Regression

Slow Query
≠
Lock Wait

Slow Query
≠
Pool Contention

Slow Query
≠
Large Result Set
```

Una operación puede pertenecer a varias categorías simultáneamente.

---

# 5. Ejemplo del problema

Supongamos:

```text
Total query operation:    1,200 ms

Compilation:                  3 ms
Pool wait:                1,050 ms
DB execution:                20 ms
Result transfer:             15 ms
Hydration:                   70 ms
ORM assembly:                42 ms
```

Un detector simple diría:

```text
Slow SQL: 1.2 seconds
```

VoltStack deberá producir algo conceptualmente equivalente a:

```text
SLOW END-TO-END QUERY

Primary contributor:
  CONNECTION_POOL_WAIT

Pool wait:
  1,050 ms

Database execution:
  20 ms

Confidence:
  HIGH
```

---

# 6. Modelo conceptual

```text
QueryProfile
    │
    ▼
SlowQueryPolicyResolver
    │
    ▼
SlowQueryEvaluationContext
    │
    ▼
Threshold Evaluators
    │
    ├── Absolute
    ├── Relative
    ├── Baseline
    ├── Percentile
    ├── Phase
    └── Resource
    │
    ▼
SlowQueryClassifier
    │
    ▼
SlowQueryDetection
    │
    ▼
Deduplication
    │
    ▼
Rate Limiting
    │
    ▼
Diagnostic / Telemetry / Debug
```

---

# 7. SlowQueryDetection

Modelo principal:

```php
final readonly class SlowQueryDetection
{
    public function __construct(
        public SlowQueryDetectionId $id,
        public QueryProfileId $profileId,
        public QueryFingerprint $fingerprint,
        public SlowQueryClassificationSet $classifications,
        public SlowQuerySeverity $severity,
        public SlowQueryEvidence $evidence,
        public DetectionConfidence $confidence,
        public SlowQueryDiagnosticContext $context,
    ) {}
}
```

---

# 8. Detection ID

```php
final readonly class SlowQueryDetectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

No deberá utilizarse como metric label.

---

# 9. Query fingerprint

Las detecciones deberán agruparse principalmente mediante:

```text
QueryFingerprint
```

y no mediante:

```text
raw SQL + literal parameters
```

Esto permite reconocer familias como:

```sql
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 500;
```

como una misma familia semántica.

---

# 10. Clasificación principal

```php
enum SlowQueryClassification
{
    case END_TO_END;
    case QUERY_COMPILATION;
    case CONNECTION_RESOLUTION;
    case CONNECTION_ACQUISITION;
    case CONNECTION_ESTABLISHMENT;
    case DATABASE_EXECUTION;
    case RESULT_TRANSFER;
    case RESULT_CONSUMPTION;
    case HYDRATION;
    case ORM_PROCESSING;
    case TRANSACTION_CONTEXT;
    case DISTRIBUTED_EXECUTION;
    case DISTRIBUTED_MERGE;
    case RESOURCE_PRESSURE;
    case UNKNOWN_DOMINANT_COST;
}
```

---

# 11. Múltiples clasificaciones

Una query puede ser:

```text
END_TO_END
+
DATABASE_EXECUTION
+
HYDRATION
```

simultáneamente.

Por tanto:

```text
SlowQueryClassification
```

no deberá modelarse necesariamente como un único valor.

---

# 12. Primary classification

Podrá existir:

```text
primaryClassification
```

para presentación.

Pero conservará:

```text
allClassifications
```

para análisis.

---

# 13. Slow End-to-End Query

Se detecta cuando:

```text
T_application > threshold
```

donde:

```text
T_application
```

representa el costo visible de la operación completa.

---

# 14. Slow Database Execution

Se detecta cuando:

```text
T_database > DB_threshold
```

independientemente de que el total también sea lento.

---

# 15. Slow Compilation

```text
T_compile > compile_threshold
```

podría indicar:

```text
complex query shape
compiler cache miss
high query-shape churn
optimizer complexity
```

pero ninguna de ellas deberá declararse automáticamente causa raíz.

---

# 16. Slow Connection Acquisition

```text
T_pool_wait > acquisition_threshold
```

podría indicar:

```text
pool contention
resource exhaustion
connection leak
insufficient pool capacity
long-held connections
```

pero estas serán hipótesis diagnósticas.

---

# 17. Slow Connection Establishment

Separado de pool wait:

```text
DNS
TCP
TLS
authentication
session initialization
```

pueden contribuir.

---

# 18. Slow Result Transfer

Una consulta puede ejecutarse rápido pero devolver enormes cantidades de datos:

```text
DB execution:      30 ms
Result transfer:  900 ms
```

Clasificación:

```text
RESULT_TRANSFER
```

---

# 19. Slow Result Consumption

Especialmente relevante con:

```text
ResultCursor
StreamingResult
LazyCollection
```

donde:

```text
execute()
```

no representa el costo total.

---

# 20. Slow Hydration

Ejemplo:

```text
DB execution:  35 ms
Hydration:    620 ms
```

Clasificación:

```text
HYDRATION
```

No:

```text
SLOW_SQL
```

---

# 21. Slow ORM Processing

Puede involucrar:

```text
IdentityMap
relationship assembly
snapshot creation
change tracking setup
entity lifecycle processing
```

---

# 22. Transaction-context slowdown

Una consulta puede estar asociada a:

```text
long transaction
lock contention
savepoint-heavy workflow
transaction retry
```

pero el detector deberá distinguir evidencia observada de inferencia.

---

# 23. Distributed execution

En sharding:

```text
Logical Query
├── shard-A  20 ms
├── shard-B  22 ms
├── shard-C 800 ms
└── shard-D  19 ms
```

el problema puede clasificarse:

```text
DISTRIBUTED_EXECUTION
```

con:

```text
dominant shard = shard-C
```

---

# 24. Distributed merge

Otro caso:

```text
Shard execution:
  max = 45 ms

Merge:
  700 ms
```

Clasificación:

```text
DISTRIBUTED_MERGE
```

---

# 25. Unknown dominant cost

Si:

```text
Total = 900 ms
Known phases = 120 ms
Unaccounted = 780 ms
```

VoltStack no deberá inventar causa.

Clasificación:

```text
UNKNOWN_DOMINANT_COST
```

---

# 26. SlowQuerySeverity

```php
enum SlowQuerySeverity
{
    case NOTICE;
    case WARNING;
    case HIGH;
    case CRITICAL;
}
```

---

# 27. Severity ≠ classification

Ejemplo:

```text
Classification:
  HYDRATION

Severity:
  WARNING
```

---

# 28. Threshold policy

No existirá un único:

```text
slow_query_threshold = 100ms
```

como modelo completo.

VoltStack deberá soportar políticas por:

```text
query class
operation type
source
phase
environment
database role
workload
query family
resource budget
```

---

# 29. SlowQueryPolicy

```php
final readonly class SlowQueryPolicy
{
    public function __construct(
        public bool $enabled,
        public SlowQueryThresholdSet $thresholds,
        public SlowQuerySamplingPolicy $sampling,
        public SlowQueryDeduplicationPolicy $deduplication,
        public SlowQueryRateLimitPolicy $rateLimit,
        public SlowQueryDiagnosticPolicy $diagnostics,
    ) {}
}
```

---

# 30. Policy compilation

Configuración deberá convertirse durante bootstrap:

```text
Configuration
↓
Validation
↓
CompiledSlowQueryPolicy
```

evitando interpretar configuración compleja en cada query.

---

# 31. Absolute threshold

Modelo simple:

```text
if duration >= 500ms
→ slow
```

---

# 32. Ventaja

Es:

```text
simple
predictable
cheap
```

---

# 33. Limitación

No considera que:

```text
5 ms
```

pueda ser anormal para una query que normalmente tarda:

```text
100 μs
```

ni que:

```text
700 ms
```

pueda ser esperado para un reporte pesado.

---

# 34. Relative threshold

Podrá expresarse:

```text
current_duration
>
baseline × multiplier
```

Ejemplo:

```text
baseline = 20 ms
current = 90 ms
multiplier = 3
```

resultado:

```text
possible anomaly
```

---

# 35. Relative threshold ≠ regression proof

Una observación aislada puede deberse a:

```text
cold cache
network jitter
replica state
resource contention
parameter selectivity
```

---

# 36. Baseline threshold

Cada:

```text
QueryFingerprint
```

podrá mantener un baseline estadístico.

---

# 37. Baseline identity

Deberá considerar dimensiones relevantes como:

```text
query fingerprint
database platform
logical database class
execution role
environment
query source
```

sin explotar cardinalidad.

---

# 38. Tenant-specific baseline

No deberá utilizarse por default.

```text
TenantId
```

puede producir alta cardinalidad y filtración de información.

---

# 39. Shard-specific baseline

Puede ser útil en diagnósticos internos, pero no necesariamente como métrica global.

---

# 40. Baseline model

Conceptualmente:

```php
final readonly class QueryPerformanceBaseline
{
    public function __construct(
        public QueryFingerprint $fingerprint,
        public int $samples,
        public DurationDistribution $total,
        public DurationDistribution $database,
        public DurationDistribution $hydration,
        public BaselineConfidence $confidence,
    ) {}
}
```

---

# 41. Cold baseline

Con pocas muestras:

```text
confidence = LOW
```

---

# 42. No baseline

Debe representarse:

```text
NO_BASELINE
```

No:

```text
baseline = 0
```

---

# 43. Baseline poisoning

Eventos anómalos no deberán incorporarse indiscriminadamente al baseline.

De lo contrario:

```text
slow behavior
↓
baseline grows
↓
slow behavior becomes "normal"
```

---

# 44. Baseline update policy

Podrá usar:

```text
rolling window
exponential weighting
bounded histogram
provider-based statistics
```

---

# 45. Baseline history

El core no necesita almacenar historial infinito.

---

# 46. Percentile threshold

Ejemplo:

```text
current > historical p99
```

o:

```text
p95 > service objective
```

---

# 47. Percentile detector

Requiere suficientes muestras.

Con:

```text
samples = 3
```

un `p99` carece de suficiente significado.

---

# 48. Minimum sample count

```php
final readonly class PercentileThreshold
{
    public function __construct(
        public float $percentile,
        public int $minimumSamples,
        public float $multiplier,
    ) {}
}
```

---

# 49. Adaptive threshold

VoltStack podrá combinar:

```text
absolute floor
+
baseline
+
percentile
+
resource policy
```

---

# 50. Ejemplo adaptativo

```text
Detect if:

duration > 250 ms

AND

duration > baseline_p95 × 2
```

Esto reduce ruido.

---

# 51. Adaptive ≠ machine learning requirement

El sistema deberá poder implementarse inicialmente con:

```text
deterministic statistics
```

sin requerir IA.

---

# 52. Future anomaly detector

Podrá añadirse mediante extensión:

```text
SlowQueryAnomalyDetector
```

sin acoplar el core a modelos ML.

---

# 53. Phase-specific thresholds

Ejemplo:

```yaml
database:
  slow_query:
    thresholds:
      total: 500ms
      execution: 250ms
      pool_wait: 100ms
      compilation: 50ms
      hydration: 150ms
```

---

# 54. Threshold precedence

Conceptualmente:

```text
Query Family Override
        ↓
Operation Policy
        ↓
Source Policy
        ↓
Database Policy
        ↓
Global Default
```

---

# 55. Policy resolution

Debe ser determinista.

---

# 56. Query family override

Ejemplo:

```text
report.monthly_financial_summary
```

puede aceptar:

```text
2 seconds
```

mientras una query interactiva puede limitarse a:

```text
100 ms
```

---

# 57. Workload class

VoltStack podrá introducir:

```php
enum DatabaseWorkloadClass
{
    case INTERACTIVE;
    case BACKGROUND;
    case REPORTING;
    case BULK;
    case MIGRATION;
    case MAINTENANCE;
    case CUSTOM;
}
```

---

# 58. Workload class ≠ Query source

Una query ORM puede ejecutarse como:

```text
INTERACTIVE
```

o:

```text
BACKGROUND
```

---

# 59. Source classification

```text
ORM
QUERY_BUILDER
RAW
MIGRATION
SCHEMA
BULK
IMPORT
INTERNAL
```

permite políticas adicionales.

---

# 60. Threshold evaluator contract

```php
interface SlowQueryThresholdEvaluator
{
    public function evaluate(
        QueryProfile $profile,
        SlowQueryEvaluationContext $context,
    ): ThresholdEvaluation;
}
```

---

# 61. Evaluator implementations

```text
AbsoluteThresholdEvaluator
RelativeThresholdEvaluator
BaselineThresholdEvaluator
PercentileThresholdEvaluator
PhaseThresholdEvaluator
ResourceThresholdEvaluator
```

---

# 62. ThresholdEvaluation

```php
final readonly class ThresholdEvaluation
{
    public function __construct(
        public bool $exceeded,
        public DetectionConfidence $confidence,
        public ThresholdEvidence $evidence,
    ) {}
}
```

---

# 63. Threshold evidence

Deberá poder explicar:

```text
Observed:
  742 ms

Threshold:
  250 ms

Baseline p95:
  84 ms

Ratio:
  8.83×
```

---

# 64. Explainability

Cada detección deberá responder:

```text
¿Por qué fue marcada como lenta?
```

---

# 65. Detection confidence

```php
enum DetectionConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 66. Confidence sources

Dependerá de:

```text
profile coverage
measurement confidence
sample count
baseline quality
threshold type
context completeness
```

---

# 67. SlowQueryEvidence

```php
final readonly class SlowQueryEvidence
{
    public function __construct(
        public Duration $observedDuration,
        public Duration $threshold,
        public ?Duration $baseline,
        public ?float $baselineRatio,
        public QueryProfileCoverage $profileCoverage,
        public SlowPhaseEvidenceSet $phases,
    ) {}
}
```

---

# 68. Evidence ≠ diagnosis

Ejemplo:

```text
Evidence:
  DB execution = 900 ms
```

Diagnóstico:

```text
Database execution dominates latency
```

Hipótesis:

```text
possible inefficient plan
```

Recomendación:

```text
inspect EXPLAIN
```

---

# 69. Tres niveles de información

VoltStack deberá separar:

```text
OBSERVED
INFERRED
RECOMMENDED
```

---

# 70. Ejemplo

```text
OBSERVED
DB execution: 1.2 s

INFERRED
DB execution dominates total latency.

RECOMMENDED
Inspect the database execution plan.
```

No:

```text
Missing index detected.
```

sin evidencia.

---

# 71. Dominant phase analysis

Podrá calcular:

```text
dominant phase
```

si existe evidencia suficiente.

---

# 72. Dominance threshold

Ejemplo:

```text
phase / total >= 0.60
```

podrá considerarse dominante bajo policy.

---

# 73. Overlapping phases

Si las fases se solapan:

```text
dominance
```

deberá considerar la semántica de timeline del Query Profiler.

---

# 74. Unknown phase dominance

Si gran parte del tiempo no está explicado:

```text
UNKNOWN_DOMINANT_COST
```

será preferible.

---

# 75. Query regression detection

Una query puede no superar threshold absoluto y aun haber empeorado.

Ejemplo:

```text
Before:
  p95 = 10 ms

After:
  p95 = 45 ms

Threshold:
  100 ms
```

No es `slow` absoluto, pero puede ser:

```text
PERFORMANCE_REGRESSION
```

---

# 76. Slow query ≠ regression

Se mantendrán separados.

---

# 77. Regression integration

El sistema podrá producir evidencia para:

```text
Performance Testing
Query Profile Comparison
```

sin fusionar completamente ambos sistemas.

---

# 78. High aggregate cost

Ejemplo:

```text
Query:
  1 ms

Executions:
  100,000

Aggregate:
  100 seconds
```

No es slow query individual.

Pero deberá poder producir:

```text
HIGH_AGGREGATE_COST
```

como diagnóstico complementario.

---

# 79. High frequency

Igualmente:

```text
HIGH_FREQUENCY
```

será distinto de:

```text
SLOW_QUERY
```

---

# 80. N+1

Un patrón:

```text
1 ms × 500 queries
```

podría ser N+1.

El sistema 222 no deberá decidirlo por repetición solamente.

---

# 81. Integración con 223

Se enviará evidencia como:

```text
QueryFingerprint
count
ORM origin
relationship metadata
timing
root operation
```

al sistema N+1.

---

# 82. Connection pool context

El detector deberá correlacionar:

```text
pool wait
pool saturation evidence
connection reuse
new connection
```

cuando esté disponible.

---

# 83. Pool contention classification

Podrá generar:

```text
SLOW_CONNECTION_ACQUISITION
```

y diagnóstico:

```text
POSSIBLE_POOL_CONTENTION
```

---

# 84. Pool contention ≠ insufficient pool size

Una saturación puede deberse a:

```text
connection leaks
long transactions
slow DB
unexpected concurrency
small pool
```

No deberá asumir una sola causa.

---

# 85. Transaction context

Podrá incluir:

```text
transaction age
isolation
nested depth
savepoint depth
retry attempt
```

---

# 86. Long transaction

Una query lenta dentro de transacción antigua podrá recibir:

```text
LONG_TRANSACTION_CONTEXT
```

como contexto.

No necesariamente causa.

---

# 87. Deadlock retry

Si una operación tarda:

```text
2 seconds
```

porque hubo:

```text
deadlock
+
retry
```

el detector deberá distinguir:

```text
logical operation latency
```

de:

```text
individual attempt latency
```

---

# 88. Retry-aware profiling

Conceptualmente:

```text
Logical Query Operation
├── attempt 1 → deadlock
└── attempt 2 → success
```

---

# 89. Attempt ≠ logical operation

Ambos podrán tener profiles separados/correlacionados.

---

# 90. Unknown transaction outcome

Nunca deberá convertirse en:

```text
slow success
```

si el resultado es desconocido.

Puede existir:

```text
SLOW + UNKNOWN_OUTCOME
```

---

# 91. Replica context

El detector podrá incluir:

```text
writer/replica
replica lag evidence
freshness
routing reason
```

---

# 92. Slow replica

Una replica lenta no implica que todas las replicas sean lentas.

---

# 93. Endpoint-specific anomaly

Podrá existir internamente:

```text
endpoint anomaly
```

pero los endpoint IDs deberán mantenerse fuera de labels globales de alta cardinalidad.

---

# 94. Sharding context

Podrá registrar:

```text
logical query
physical queries
fan-out
critical shard
merge duration
```

---

# 95. Slow shard

Si:

```text
shard A = 20ms
shard B = 21ms
shard C = 1.4s
```

la detección deberá preservar:

```text
critical shard evidence
```

---

# 96. No fake global diagnosis

No afirmar:

```text
all shards slow
```

cuando solo existe evidencia de uno.

---

# 97. Multi-shard partial evidence

Si un shard no responde:

```text
coverage = PARTIAL
```

y el resultado deberá preservar incertidumbre.

---

# 98. Result cardinality

Una consulta puede ser lenta por procesar:

```text
1,000,000 rows
```

El detector podrá añadir:

```text
LARGE_RESULT_SET
```

como contexto.

---

# 99. Large result ≠ bad query

Reportes, exports y procesamiento masivo pueden requerirlo.

La workload class importa.

---

# 100. Rows-per-duration

Podrán calcularse métricas diagnósticas:

```text
rows / second
```

pero no deberán considerarse universalmente comparables.

---

# 101. Hydration cost per entity

Podrá calcularse:

```text
hydration_time / hydrated_entities
```

cuando ambos valores sean confiables.

---

# 102. Zero entities

Evitar división por cero y conclusiones incorrectas.

---

# 103. Query compilation cost

Podrá correlacionarse con:

```text
compiled query cache hit/miss
AST complexity
optimization rules
plan complexity
```

---

# 104. Cache context

El detector deberá distinguir:

```text
Compiled Query Cache
Result Cache
Metadata Cache
Entity Cache
```

---

# 105. Result cache hit

Si una operación lenta proviene de cache:

```text
database execution = NOT_APPLICABLE
```

No se marcará:

```text
SLOW_DATABASE_EXECUTION
```

---

# 106. Cache lookup slowdown

Puede clasificarse como:

```text
RESOURCE/INTEGRATION COST
```

según la arquitectura general de Telemetry.

---

# 107. Cache miss ≠ problem

Un miss no implica anomalía.

---

# 108. EXPLAIN integration

Cuando exista:

```text
SLOW_DATABASE_EXECUTION
```

el sistema podrá solicitar un:

```text
ExplainDiagnosticRequest
```

---

# 109. Detector no ejecuta EXPLAIN directamente

Flujo:

```text
Slow Detection
↓
Diagnostic Policy
↓
Explain Integration
↓
Capability/Safety Check
↓
Explain Provider
```

---

# 110. EXPLAIN safety

Deberá reutilizar las reglas del documento 221.

---

# 111. EXPLAIN ANALYZE

No se ejecutará automáticamente en producción por default.

Especialmente:

```text
INSERT
UPDATE
DELETE
DDL
```

---

# 112. Explain budget

Incluso `EXPLAIN` seguro tendrá:

```text
rate limit
timeout
max captures
```

---

# 113. Explain deduplication

No solicitar:

```text
EXPLAIN
```

miles de veces para el mismo fingerprint en una ventana corta.

---

# 114. Explain cache

Podrá existir un pequeño cache diagnóstico:

```text
Fingerprint + relevant generation
→ ExplainResult
```

---

# 115. Explain cache ≠ Query Result Cache

Será estrictamente diagnóstico.

---

# 116. Schema generation

Un cambio de schema puede invalidar:

```text
Explain diagnostic cache
```

---

# 117. Capability generation

Cambios relevantes de plataforma/capabilities también podrán invalidarlo.

---

# 118. Slow query diagnostic capture

Al detectar una query lenta podrán conservarse, según policy:

```text
fingerprint
normalized SQL
phase timings
query source
ORM origin
transaction context
connection role
row counts
cache state
distribution context
source location
EXPLAIN reference
```

---

# 119. Raw parameters

No se capturarán por default.

---

# 120. Sensitive data

No deberán aparecer:

```text
passwords
tokens
API keys
PII
tenant secrets
financial data
```

como parte de diagnostics por default.

---

# 121. Redaction pipeline

```text
Detection
↓
Internal Evidence
↓
Redaction
↓
Safe Diagnostic
↓
Exporter
```

---

# 122. Raw SQL

En producción:

```text
disabled/redacted by default
```

---

# 123. Development SQL

Podrá permitirse:

```text
normalized SQL
```

y opcionalmente raw SQL bajo configuración explícita.

---

# 124. Source location

En desarrollo:

```text
app/Repository/UserRepository.php:84
```

puede ser extremadamente útil.

---

# 125. Source location production

Podrá deshabilitarse por:

```text
performance
security
path disclosure
```

---

# 126. Stack trace

No deberá capturarse en todas las queries.

---

# 127. Conditional stack capture

Puede capturarse únicamente:

```text
after slow detection
```

cuando la arquitectura permita recuperar una ubicación útil.

---

# 128. Limitation

Una stack completa pasada puede no ser recuperable después.

Por tanto podrá utilizarse:

```text
lightweight source token
```

durante ejecución.

---

# 129. Source token

Ejemplo:

```text
application call-site identifier
```

sin conservar toda la stack.

---

# 130. Deduplication

Una aplicación puede ejecutar la misma query lenta:

```text
10,000 times/minute
```

No deberá generar:

```text
10,000 logs completos
```

---

# 131. Deduplication key

Conceptualmente:

```text
fingerprint
+
classification
+
severity bucket
+
environment
```

---

# 132. Tenant ID

No deberá incluirse por default en dedup key global.

---

# 133. Deduplication window

Ejemplo:

```text
60 seconds
```

durante los cuales se agregan:

```text
occurrences
max duration
average duration
first seen
last seen
```

---

# 134. SlowQueryOccurrenceAggregate

```php
final readonly class SlowQueryOccurrenceAggregate
{
    public function __construct(
        public QueryFingerprint $fingerprint,
        public int $occurrences,
        public Duration $maximum,
        public Duration $average,
        public Instant $firstSeen,
        public Instant $lastSeen,
    ) {}
}
```

---

# 135. Deduplication ≠ dropping evidence

Se preservarán agregados aunque se reduzca logging.

---

# 136. Rate limiting

Separado de deduplication.

```text
Deduplication
=
combine equivalent detections

Rate Limiting
=
bound emitted diagnostics
```

---

# 137. Rate limit scopes

Podrán existir:

```text
global
classification
fingerprint
exporter
```

---

# 138. Rate limit overflow

Opciones:

```text
DROP_DETAIL
AGGREGATE
SAMPLE
SUPPRESS_EXPORT
```

---

# 139. Suppression

Una detección podrá estar:

```text
DETECTED
+
SUPPRESSED
```

---

# 140. Suppressed ≠ not slow

Debe conservarse conceptualmente.

---

# 141. Suppression reason

```php
enum SlowQuerySuppressionReason
{
    case DEDUPLICATED;
    case RATE_LIMITED;
    case POLICY;
    case SAMPLING;
    case BUDGET_EXHAUSTED;
}
```

---

# 142. Sampling

No todas las slow queries necesitan full diagnostics.

---

# 143. Always-detect vs sample-details

Arquitectura preferida:

```text
cheap timing
↓
detect threshold
↓
sample expensive diagnostic detail
```

---

# 144. Critical query

Para severidad:

```text
CRITICAL
```

la policy puede elevar sampling.

---

# 145. First occurrence

Puede capturarse siempre:

```text
first occurrence per fingerprint/window
```

y luego muestrear.

---

# 146. Worst occurrence

El agregador podrá conservar:

```text
worst observed profile
```

bajo memory budget.

---

# 147. Worst profile retention

No deberá retener graph/connection/entity objects.

Solo snapshot diagnóstico seguro.

---

# 148. Alert integration

El detector emitirá:

```text
SlowQueryDetected
```

pero no será responsable de:

```text
email
Slack
PagerDuty
SMS
```

---

# 149. Alerting provider

Será integración superior.

---

# 150. Alert storm prevention

Deduplication y rate limiting deberán ocurrir antes de exportadores ruidosos.

---

# 151. Telemetry metrics

Métricas posibles:

```text
database.slow_query.count
database.slow_query.duration
database.slow_query.ratio
database.slow_query.phase.duration
```

---

# 152. Bounded labels

Permitidos:

```text
operation
classification
severity
source
role
platform
```

---

# 153. Labels prohibidos por default

```text
raw SQL
user ID
tenant ID
connection ID
transaction ID
query ID
full URL
email
```

---

# 154. Slow query ratio

Formalmente:

```text
SlowRatio
=
SlowQueryCount
/
ObservedQueryCount
```

dentro de una ventana y policy definidas.

---

# 155. Slow ratio ≠ error rate

Una query puede ser lenta y exitosa.

---

# 156. Error + slow

Puede ocurrir:

```text
SLOW
+
FAILED
```

simultáneamente.

---

# 157. Timeout

Un timeout será:

```text
FAILED/TIMEOUT
```

y puede también clasificarse como:

```text
SLOW
```

si existe evidencia temporal suficiente.

---

# 158. Cancelled query

Una query cancelada antes del threshold:

```text
CANCELLED
```

no deberá inventarse como slow.

---

# 159. Cancellation after threshold

Puede ser:

```text
SLOW
+
CANCELLED
```

---

# 160. Detection lifecycle

```text
PROFILE_RECEIVED
      ↓
POLICY_RESOLVED
      ↓
THRESHOLDS_EVALUATED
      ↓
CLASSIFIED
      ↓
EVIDENCE_ATTACHED
      ↓
SEVERITY_ASSIGNED
      ↓
DEDUPLICATED
      ↓
RATE_LIMITED
      ↓
EXPORTED
```

Estados alternos:

```text
SUPPRESSED
PARTIAL
UNKNOWN
```

---

# 161. Detection timing

Preferentemente:

```text
after sufficient query profile evidence exists
```

---

# 162. Streaming query complication

Una query streaming puede tardar:

```text
10 minutes
```

porque el consumidor procesa lentamente cada fila.

Esto no implica:

```text
10-minute SQL execution
```

---

# 163. Streaming classification

Deberá distinguir:

```text
DATABASE_EXECUTION
RESULT_CONSUMPTION
CONSUMER_PROCESSING
```

según evidencia disponible.

---

# 164. Lazy Collection

Igualmente:

```text
LazyCollection lifetime
```

puede ser largo sin que una query individual sea lenta.

---

# 165. Chunk processing

Cada chunk podrá evaluarse individualmente.

La operación completa podrá tener su propio:

```text
large dataset processing profile
```

pero no se etiquetará como slow query solo por duración total.

---

# 166. Bulk operations

Un:

```text
bulk update
```

de 5 segundos sobre millones de filas puede ser esperado.

Workload class y policy son fundamentales.

---

# 167. Migrations

Migraciones tendrán thresholds distintos.

No deben usar automáticamente thresholds interactivos.

---

# 168. Background jobs

Igualmente:

```text
BACKGROUND
```

puede permitir latencias mayores.

---

# 169. Interactive workload

Queries de request HTTP pueden tener thresholds más estrictos.

---

# 170. Resource budgets

El detector podrá considerar:

```text
deadline
query timeout
request budget
transaction budget
resource governance policy
```

---

# 171. Budget violation

Una query de:

```text
90 ms
```

puede ser lenta respecto de un presupuesto de:

```text
50 ms
```

aunque el threshold global sea:

```text
500 ms
```

---

# 172. Budget-aware detection

Clasificación adicional:

```text
BUDGET_EXCEEDED
```

podrá coexistir.

---

# 173. Slow threshold ≠ timeout

Ejemplo:

```text
slow threshold = 250ms
timeout = 5s
```

Entre ambos:

```text
query is slow but allowed to continue
```

---

# 174. Detector no cancela

La cancelación pertenece a:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
```

---

# 175. Persistent runtime

El sistema deberá ser seguro en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 176. Scoped detection state

Por request/job/operation:

```text
active dedup context
temporary aggregates
diagnostic budget
sampling state
```

deberán permanecer aislados.

---

# 177. Global baselines

Podrán compartirse mediante provider thread/process-safe.

Pero no mediante mutable static PHP improvisado.

---

# 178. Static state prohibited

Evitar:

```php
static array $slowQueries = [];
```

como almacenamiento de runtime.

---

# 179. Worker leak

Después de request:

```text
request slow-query state
→ reset/release
```

---

# 180. Coroutine isolation

```text
Coroutine A
→ DetectionContext A

Coroutine B
→ DetectionContext B
```

---

# 181. Sampling isolation

Una decisión de sampling de un request no deberá contaminar otro.

---

# 182. Tenant isolation

La detección deberá respetar:

```text
tenant context
database domain
shard context
```

---

# 183. Tenant observability

Administradores de un tenant no deberán recibir diagnostics de otro tenant.

---

# 184. Cross-tenant aggregate

Solo podrá producirse en:

```text
authorized operational telemetry
```

y preferentemente anonimizado/agregado.

---

# 185. Security context

Una query lenta no deberá provocar bypass de:

```text
authorization
tenant isolation
data redaction
```

para obtener diagnostics.

---

# 186. Diagnostic authorization

Debug toolbar/API de diagnóstico deberá aplicar permisos superiores.

---

# 187. Production defaults

Configuración recomendada:

```text
enabled = true
lightweight detection = true
raw SQL = false
parameters = false
source stack = false
automatic explain analyze = false
deduplication = true
rate limiting = true
sampling = true
```

---

# 188. Development defaults

Podrán ser:

```text
normalized SQL = true
source location = true
phase details = true
ORM context = true
EXPLAIN metadata = opt-in
```

---

# 189. Query Slow Log

VoltStack podrá exponer un:

```text
SlowQueryLogExporter
```

---

# 190. Ejemplo de log

```text
[database.slow_query]

fingerprint:
  select.users.by_status.v3

classification:
  HYDRATION

severity:
  WARNING

total:
  482 ms

database:
  31 ms

hydration:
  392 ms

rows:
  18,400

source:
  ORM

confidence:
  HIGH
```

---

# 191. No parameter dump

No:

```text
email = user@example.com
token = abc123...
```

---

# 192. Developer diagnostic

En desarrollo:

```text
SLOW DATABASE OPERATION
────────────────────────────────────

Total                  782 ms

Connection
  Pool wait             12 ms

Database
  Execution            621 ms

Result
  Transfer              41 ms

Application
  Hydration             82 ms
  ORM                    26 ms

Primary classification
  DATABASE_EXECUTION

Threshold
  250 ms

Baseline p95
  74 ms

Deviation
  8.39×

Confidence
  HIGH

Recommendation
  Inspect execution plan.
```

---

# 193. Pool diagnostic

```text
SLOW DATABASE OPERATION
────────────────────────────────────

Total                  910 ms
Pool wait              842 ms
Database execution      14 ms

Classification:
  CONNECTION_ACQUISITION

Observation:
  92.5% of observed latency occurred
  while acquiring a connection.

Recommendation:
  Inspect pool saturation, connection
  lifetime and transaction duration.
```

---

# 194. Hydration diagnostic

```text
Total                   680 ms
DB execution             28 ms
Rows returned         65,000
Hydration                590 ms

Classification:
  HYDRATION

Possible actions:
  - reduce projection
  - use scalar/DTO hydration
  - process lazily/chunked
  - inspect relationship graph
```

---

# 195. Distributed diagnostic

```text
Logical query:
  440 ms

Shard A:
   31 ms

Shard B:
   29 ms

Shard C:
  401 ms

Shard D:
   34 ms

Merge:
   18 ms

Classification:
  DISTRIBUTED_EXECUTION

Critical component:
  shard C
```

---

# 196. Baseline anomaly example

```text
Current:
  94 ms

Absolute threshold:
  250 ms

Historical:
  p50 = 8 ms
  p95 = 14 ms
  p99 = 20 ms

Result:
  Not absolute-slow.
  Strong relative anomaly.
```

El sistema podrá reportarlo como:

```text
PERFORMANCE_ANOMALY
```

sin mezclarlo necesariamente con `SLOW_QUERY`.

---

# 197. SlowQueryDetector

```php
interface SlowQueryDetector
{
    public function detect(
        QueryProfile $profile,
        SlowQueryEvaluationContext $context,
    ): SlowQueryDetectionResult;
}
```

---

# 198. Detection result

```php
final readonly class SlowQueryDetectionResult
{
    public function __construct(
        public bool $slow,
        public ?SlowQueryDetection $detection,
        public SlowQueryEvaluationSummary $evaluation,
    ) {}
}
```

---

# 199. SlowQueryClassifier

```php
interface SlowQueryClassifier
{
    public function classify(
        QueryProfile $profile,
        ThresholdEvaluationSet $evaluations,
    ): SlowQueryClassificationSet;
}
```

---

# 200. Severity resolver

```php
interface SlowQuerySeverityResolver
{
    public function resolve(
        QueryProfile $profile,
        SlowQueryClassificationSet $classifications,
        SlowQueryEvidence $evidence,
    ): SlowQuerySeverity;
}
```

---

# 201. Baseline provider

```php
interface QueryPerformanceBaselineProvider
{
    public function baseline(
        QueryFingerprint $fingerprint,
        QueryBaselineContext $context,
    ): ?QueryPerformanceBaseline;
}
```

---

# 202. Baseline recorder

Separado:

```php
interface QueryPerformanceBaselineRecorder
{
    public function observe(
        QueryProfile $profile,
        QueryBaselineContext $context,
    ): void;
}
```

---

# 203. Read/write separation

Esto permite providers:

```text
local
shared
telemetry backend
testing
```

---

# 204. Deduplicator

```php
interface SlowQueryDeduplicator
{
    public function evaluate(
        SlowQueryDetection $detection,
    ): DeduplicationDecision;
}
```

---

# 205. Rate limiter

```php
interface SlowQueryRateLimiter
{
    public function allow(
        SlowQueryDetection $detection,
    ): RateLimitDecision;
}
```

---

# 206. Diagnostic enricher

```php
interface SlowQueryDiagnosticEnricher
{
    public function enrich(
        SlowQueryDetection $detection,
        QueryProfile $profile,
    ): SlowQueryDiagnostic;
}
```

---

# 207. Explain integration contract

```php
interface SlowQueryExplainCoordinator
{
    public function explainIfAllowed(
        SlowQueryDetection $detection,
        QueryProfile $profile,
    ): ?ExplainResult;
}
```

---

# 208. Directory structure

```text
src/Quantum/Database/Telemetry/SlowQuery/
│
├── Contract/
│   ├── SlowQueryDetector.php
│   ├── SlowQueryClassifier.php
│   ├── SlowQueryThresholdEvaluator.php
│   ├── SlowQuerySeverityResolver.php
│   ├── QueryPerformanceBaselineProvider.php
│   ├── QueryPerformanceBaselineRecorder.php
│   ├── SlowQueryDeduplicator.php
│   ├── SlowQueryRateLimiter.php
│   └── SlowQueryDiagnosticEnricher.php
│
├── Detection/
│   ├── SlowQueryDetection.php
│   ├── SlowQueryDetectionId.php
│   ├── SlowQueryDetectionResult.php
│   ├── SlowQueryClassification.php
│   ├── SlowQueryClassificationSet.php
│   ├── SlowQuerySeverity.php
│   └── DetectionConfidence.php
│
├── Evidence/
│   ├── SlowQueryEvidence.php
│   ├── SlowPhaseEvidence.php
│   ├── SlowPhaseEvidenceSet.php
│   ├── ThresholdEvidence.php
│   └── SlowQueryDiagnosticContext.php
│
├── Threshold/
│   ├── SlowQueryThresholdSet.php
│   ├── ThresholdEvaluation.php
│   ├── ThresholdEvaluationSet.php
│   ├── AbsoluteThresholdEvaluator.php
│   ├── RelativeThresholdEvaluator.php
│   ├── BaselineThresholdEvaluator.php
│   ├── PercentileThresholdEvaluator.php
│   ├── PhaseThresholdEvaluator.php
│   └── ResourceThresholdEvaluator.php
│
├── Baseline/
│   ├── QueryPerformanceBaseline.php
│   ├── QueryBaselineContext.php
│   ├── BaselineConfidence.php
│   ├── DurationDistribution.php
│   ├── RollingBaselineProvider.php
│   └── NullBaselineProvider.php
│
├── Classification/
│   ├── DefaultSlowQueryClassifier.php
│   ├── DominantPhaseAnalyzer.php
│   └── SlowQueryCauseCandidate.php
│
├── Policy/
│   ├── SlowQueryPolicy.php
│   ├── CompiledSlowQueryPolicy.php
│   ├── SlowQueryPolicyResolver.php
│   ├── SlowQueryDiagnosticPolicy.php
│   ├── SlowQuerySamplingPolicy.php
│   ├── SlowQueryDeduplicationPolicy.php
│   └── SlowQueryRateLimitPolicy.php
│
├── Sampling/
│   ├── SlowQuerySampler.php
│   └── SlowQuerySamplingDecision.php
│
├── Deduplication/
│   ├── DefaultSlowQueryDeduplicator.php
│   ├── SlowQueryDeduplicationKey.php
│   ├── SlowQueryOccurrenceAggregate.php
│   └── DeduplicationDecision.php
│
├── RateLimit/
│   ├── DefaultSlowQueryRateLimiter.php
│   └── RateLimitDecision.php
│
├── Diagnostic/
│   ├── SlowQueryDiagnostic.php
│   ├── SlowQueryDiagnosticBuilder.php
│   ├── SlowQueryRecommendation.php
│   ├── SlowQuerySuppressionReason.php
│   └── SlowQueryDiagnosticFormatter.php
│
├── Explain/
│   ├── SlowQueryExplainCoordinator.php
│   └── DefaultSlowQueryExplainCoordinator.php
│
├── Telemetry/
│   ├── SlowQueryTelemetry.php
│   ├── SlowQueryDetectedEvent.php
│   └── SlowQueryMetricRecorder.php
│
├── Export/
│   ├── SlowQueryExporter.php
│   ├── SlowQueryLogExporter.php
│   └── NullSlowQueryExporter.php
│
├── Runtime/
│   ├── SlowQueryRuntimeContext.php
│   └── SlowQueryRuntimeResetter.php
│
├── Testing/
│   ├── RecordingSlowQueryDetector.php
│   ├── FakeBaselineProvider.php
│   └── SlowQueryAssertions.php
│
└── Exception/
    ├── SlowQueryDetectionException.php
    ├── SlowQueryPolicyException.php
    ├── SlowQueryBaselineException.php
    └── SlowQueryDiagnosticException.php
```

---

# 209. Configuración conceptual

```php
return [

    'slow_query' => [

        'enabled' => true,

        'thresholds' => [
            'total' => '500ms',
            'database' => '250ms',
            'connection_wait' => '100ms',
            'compilation' => '50ms',
            'hydration' => '150ms',
        ],

        'baseline' => [
            'enabled' => true,
            'minimum_samples' => 100,
        ],

        'sampling' => [
            'enabled' => true,
        ],

        'deduplication' => [
            'enabled' => true,
            'window' => '60s',
        ],

        'rate_limit' => [
            'enabled' => true,
        ],

        'diagnostics' => [
            'normalized_sql' => true,
            'parameters' => false,
            'source_location' => false,
            'explain' => false,
            'explain_analyze' => false,
        ],

    ],

];
```

Los valores son ilustrativos; los defaults finales deberán definirse mediante benchmarks y políticas de producto.

---

# 210. Flujo completo

```text
Query Execution
      │
      ▼
Query Telemetry
      │
      ▼
Query Profiler
      │
      ▼
QueryProfile
      │
      ▼
SlowQueryPolicyResolver
      │
      ▼
Threshold Evaluation
      │
      ├── Absolute
      ├── Relative
      ├── Baseline
      ├── Percentile
      ├── Phase
      └── Resource Budget
      │
      ▼
Classification
      │
      ▼
Severity
      │
      ▼
Evidence
      │
      ▼
Deduplication
      │
      ▼
Rate Limiting
      │
      ▼
Optional Diagnostic Enrichment
      │
      ├── Source
      ├── ORM Context
      ├── Transaction
      ├── Connection
      ├── Shard
      └── EXPLAIN
      │
      ▼
Redaction
      │
      ▼
Telemetry / Log / Debug Toolbar
```

---

# 211. Anti-patrones

## 211.1 Medir solo SQL

Incorrecto:

```text
execute() > 500ms
→ slow query
```

como única definición.

---

## 211.2 Asumir índice faltante

Incorrecto:

```text
slow
→ missing index
```

---

## 211.3 Loggear parámetros completos

Incorrecto:

```text
SQL + passwords + emails + tokens
```

---

## 211.4 Capturar stack en cada query

Genera overhead innecesario.

---

## 211.5 Un threshold universal

Incorrecto para:

```text
interactive
bulk
reporting
migration
background
```

---

## 211.6 Alertar cada ocurrencia

Produce alert storms.

---

## 211.7 Tenant ID como metric label

Produce:

```text
high cardinality
+
possible information exposure
```

---

## 211.8 Confundir total time con DB time

Uno de los errores principales que esta arquitectura evita.

---

## 211.9 Baseline sin mínimo de muestras

Produce conclusiones estadísticas débiles.

---

## 211.10 Permitir que anomalías redefinan rápidamente el baseline

Produce baseline poisoning.

---

## 211.11 Ejecutar EXPLAIN ANALYZE automáticamente

Especialmente peligroso para writes.

---

## 211.12 Guardar Query objects completos

Los diagnostics deberán conservar snapshots, no objetos runtime.

---

## 211.13 Retener Connection

Prohibido.

---

## 211.14 Retener EntityManager

Prohibido.

---

## 211.15 Detector modificando queries

Viola separación arquitectónica.

---

# 212. Testing strategy

El sistema deberá probar:

```text
absolute threshold detection
phase threshold detection
baseline detection
percentile detection
minimum sample enforcement
unknown measurement handling
partial profile handling
classification
severity
dominant phase
deduplication
rate limiting
sampling
redaction
EXPLAIN safety
transaction correlation
connection correlation
replica context
shard context
streaming result semantics
persistent runtime reset
coroutine isolation
```

---

# 213. Ejemplo de test

```php
$profile = QueryProfileFactory::make([
    'total' => Duration::milliseconds(800),
    'database' => Duration::milliseconds(40),
    'connection_wait' => Duration::milliseconds(700),
]);

$result = $detector->detect(
    $profile,
    $context,
);

expect($result->slow)->toBeTrue();

expect(
    $result->detection->classifications
)->toContain(
    SlowQueryClassification::CONNECTION_ACQUISITION
);
```

---

# 214. Test de incertidumbre

```php
$profile = QueryProfileFactory::make([
    'total' => Duration::milliseconds(900),
    'database' => null,
    'coverage' => QueryProfileCoverage::PARTIAL,
]);
```

El detector no deberá afirmar:

```text
DATABASE_EXECUTION
```

sin evidencia.

---

# 215. Test de result cache

Si:

```text
source = RESULT_CACHE
database_execution = NOT_APPLICABLE
total = 600ms
```

no deberá generar:

```text
SLOW_DATABASE_EXECUTION
```

---

# 216. Test de persistent runtime

```text
Request A
→ slow fingerprint A

reset

Request B
→ no inherited request-local detections
```

---

# 217. Architectural invariants

## DB-SLOW-001
Slow Query Detection será distinto de Query Profiling.

## DB-SLOW-002
Slow Query Detection será distinto de Query Optimization.

## DB-SLOW-003
Slow Query Detection será distinto de N+1 Detection.

## DB-SLOW-004
Slow Query Detection será distinto de EXPLAIN.

## DB-SLOW-005
Slow Query será distinto de Failed Query.

## DB-SLOW-006
Slow Query será distinto de High-Frequency Query.

## DB-SLOW-007
Slow Query será distinto de High Aggregate Cost.

## DB-SLOW-008
Slow Query será distinto de Performance Regression.

## DB-SLOW-009
Slow End-to-End Query será distinto de Slow DB Execution.

## DB-SLOW-010
Slow SQL no implicará missing index.

## DB-SLOW-011
Una query podrá tener múltiples clasificaciones.

## DB-SLOW-012
Primary classification no eliminará clasificaciones secundarias.

## DB-SLOW-013
UNKNOWN dominant cost permanecerá UNKNOWN.

## DB-SLOW-014
Thresholds serán policy-driven.

## DB-SLOW-015
No existirá dependencia conceptual de un threshold universal.

## DB-SLOW-016
Absolute threshold será soportado.

## DB-SLOW-017
Relative threshold será soportado.

## DB-SLOW-018
Baseline threshold será soportado.

## DB-SLOW-019
Percentile threshold será soportado.

## DB-SLOW-020
Phase-specific threshold será soportado.

## DB-SLOW-021
Resource budget threshold será soportado.

## DB-SLOW-022
No baseline será distinto de zero baseline.

## DB-SLOW-023
Baseline confidence será explícita.

## DB-SLOW-024
Percentiles requerirán mínimo de muestras.

## DB-SLOW-025
Baselines serán bounded.

## DB-SLOW-026
Baseline poisoning deberá mitigarse.

## DB-SLOW-027
Adaptive detection no requerirá ML.

## DB-SLOW-028
Future anomaly detectors serán extensibles.

## DB-SLOW-029
Policy precedence será determinista.

## DB-SLOW-030
Workload class será distinta de query source.

## DB-SLOW-031
Threshold evaluation producirá evidencia explicable.

## DB-SLOW-032
Detection confidence será explícita.

## DB-SLOW-033
Evidence será distinta de diagnosis.

## DB-SLOW-034
Diagnosis será distinta de recommendation.

## DB-SLOW-035
Observed será distinto de inferred.

## DB-SLOW-036
Inferred será distinto de recommended.

## DB-SLOW-037
Dominant phase solo se afirmará con evidencia suficiente.

## DB-SLOW-038
Overlapping phases deberán respetarse.

## DB-SLOW-039
Slow query será distinta de regression.

## DB-SLOW-040
High aggregate cost será distinta de slow individual query.

## DB-SLOW-041
High frequency será distinta de slow query.

## DB-SLOW-042
Repeated query será distinta de N+1.

## DB-SLOW-043
Connection pool wait será observable separadamente.

## DB-SLOW-044
Pool contention no implicará automáticamente insufficient pool size.

## DB-SLOW-045
Transaction age será contexto, no causa automática.

## DB-SLOW-046
Retry attempt será distinto de logical operation.

## DB-SLOW-047
UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-SLOW-048
Replica context podrá correlacionarse.

## DB-SLOW-049
Slow replica no implicará slow topology.

## DB-SLOW-050
Shard context podrá correlacionarse.

## DB-SLOW-051
Critical shard podrá identificarse con evidencia.

## DB-SLOW-052
Partial shard evidence no producirá fake global diagnosis.

## DB-SLOW-053
Large result set será distinto de bad query.

## DB-SLOW-054
Hydration cost podrá medirse separadamente.

## DB-SLOW-055
Compilation cost podrá medirse separadamente.

## DB-SLOW-056
Compiled Query Cache será distinto de Result Cache.

## DB-SLOW-057
Result Cache hit no generará DB execution ficticia.

## DB-SLOW-058
Cache miss no será automáticamente problema.

## DB-SLOW-059
Detector no ejecutará EXPLAIN directamente.

## DB-SLOW-060
EXPLAIN respetará capability model.

## DB-SLOW-061
EXPLAIN respetará safety policy.

## DB-SLOW-062
EXPLAIN ANALYZE no será automático por default.

## DB-SLOW-063
Write EXPLAIN ANALYZE estará bloqueado por default.

## DB-SLOW-064
Explain capture será bounded.

## DB-SLOW-065
Explain requests serán deduplicables.

## DB-SLOW-066
Diagnostic capture será policy-driven.

## DB-SLOW-067
Raw parameters no serán capturados por default.

## DB-SLOW-068
Sensitive data deberá redactarse.

## DB-SLOW-069
Raw SQL estará deshabilitado/redactado por default en producción.

## DB-SLOW-070
Source location será configurable.

## DB-SLOW-071
Stack traces no se capturarán indiscriminadamente.

## DB-SLOW-072
Deduplication será distinta de rate limiting.

## DB-SLOW-073
Deduplication no eliminará agregados.

## DB-SLOW-074
Suppressed detection seguirá siendo detection.

## DB-SLOW-075
Suppression reason será explícita.

## DB-SLOW-076
Sampling podrá afectar detalle sin eliminar necesariamente detección básica.

## DB-SLOW-077
Critical detections podrán elevar sampling.

## DB-SLOW-078
Diagnostic retention será bounded.

## DB-SLOW-079
Runtime objects no se retendrán en diagnostics.

## DB-SLOW-080
Alert delivery estará fuera del core detector.

## DB-SLOW-081
Alert storms deberán mitigarse.

## DB-SLOW-082
Telemetry labels serán bounded.

## DB-SLOW-083
Query IDs no serán metric labels.

## DB-SLOW-084
Transaction IDs no serán metric labels.

## DB-SLOW-085
Connection IDs no serán metric labels.

## DB-SLOW-086
Tenant IDs no serán metric labels por default.

## DB-SLOW-087
Raw SQL no será metric label.

## DB-SLOW-088
Slow ratio será distinto de error rate.

## DB-SLOW-089
Slow query podrá ser exitosa.

## DB-SLOW-090
Slow query podrá fallar.

## DB-SLOW-091
Timeout podrá coexistir con slow detection.

## DB-SLOW-092
Cancelled query no será automáticamente slow.

## DB-SLOW-093
Streaming lifetime será distinto de DB execution duration.

## DB-SLOW-094
LazyCollection lifetime será distinto de query duration.

## DB-SLOW-095
Chunk traversal duration será distinta de individual slow query.

## DB-SLOW-096
Bulk workload tendrá policy propia.

## DB-SLOW-097
Migration workload tendrá policy propia.

## DB-SLOW-098
Background workload podrá tener policy propia.

## DB-SLOW-099
Interactive workload podrá tener policy propia.

## DB-SLOW-100
Slow threshold será distinto de timeout.

## DB-SLOW-101
Detector no cancelará queries.

## DB-SLOW-102
Detector respetará resource budgets.

## DB-SLOW-103
Budget exceeded podrá coexistir con slow classification.

## DB-SLOW-104
Request-local detection state será scoped.

## DB-SLOW-105
No habrá mutable static detection registry.

## DB-SLOW-106
Persistent workers deberán resetear estado request-local.

## DB-SLOW-107
Coroutine state será aislado.

## DB-SLOW-108
Sampling state será aislado.

## DB-SLOW-109
Tenant isolation será preservado.

## DB-SLOW-110
Cross-tenant diagnostics requerirán autorización.

## DB-SLOW-111
Slow detection no podrá bypass authorization.

## DB-SLOW-112
Production defaults serán seguros.

## DB-SLOW-113
Development diagnostics podrán ser más detallados.

## DB-SLOW-114
Normalized SQL será preferible a raw SQL.

## DB-SLOW-115
Diagnostic exporters recibirán representación redactada.

## DB-SLOW-116
Profiler failure no deberá convertirse en slow query falsa.

## DB-SLOW-117
Partial profile reducirá confidence cuando corresponda.

## DB-SLOW-118
UNKNOWN measurement no será cero.

## DB-SLOW-119
NOT_APPLICABLE será distinto de UNKNOWN.

## DB-SLOW-120
Result consumption será distinto de statement execution.

## DB-SLOW-121
Rows returned será distinto de rows consumed.

## DB-SLOW-122
Rows affected será distinto de entities changed.

## DB-SLOW-123
High hydration cost no será slow SQL.

## DB-SLOW-124
High pool wait no será slow SQL.

## DB-SLOW-125
High result transfer no será slow SQL.

## DB-SLOW-126
High compilation cost no será slow SQL.

## DB-SLOW-127
Distributed merge cost no será slow SQL.

## DB-SLOW-128
Unknown unaccounted time no será atribuido al DBMS.

## DB-SLOW-129
Detection será explainable.

## DB-SLOW-130
Detection será deterministic bajo misma evidencia/policy.

## DB-SLOW-131
Severity será policy-driven.

## DB-SLOW-132
Severity será distinta de classification.

## DB-SLOW-133
Baseline storage será provider-agnostic.

## DB-SLOW-134
Detector será provider-agnostic.

## DB-SLOW-135
Detector será independiente de HTTP.

## DB-SLOW-136
Detector será independiente de CLI.

## DB-SLOW-137
Detector será independiente de Debug Toolbar.

## DB-SLOW-138
Detector será independiente de alert vendors.

## DB-SLOW-139
SlowQueryDetection será immutable una vez finalizada.

## DB-SLOW-140
Detection IDs no serán labels métricos.

## DB-SLOW-141
Fingerprints no contendrán parameter values por default.

## DB-SLOW-142
Fingerprint aggregation será bounded.

## DB-SLOW-143
Dedup keys serán bounded.

## DB-SLOW-144
First-seen y last-seen podrán conservarse en aggregates.

## DB-SLOW-145
Worst occurrence podrá conservarse bajo budget.

## DB-SLOW-146
Worst occurrence no retendrá runtime objects.

## DB-SLOW-147
Diagnostic source location será opcional.

## DB-SLOW-148
Explain result será distinto de slow detection.

## DB-SLOW-149
Explain result podrá enriquecer diagnosis.

## DB-SLOW-150
Missing index recommendation requerirá evidencia apropiada.

## DB-SLOW-151
Full scan no implicará necesariamente missing index.

## DB-SLOW-152
Slow database execution no implicará necesariamente inefficient SQL.

## DB-SLOW-153
Slow connection acquisition no implicará necesariamente pool too small.

## DB-SLOW-154
Long transaction no implicará necesariamente query fault.

## DB-SLOW-155
Large result no implicará necesariamente application bug.

## DB-SLOW-156
Query baseline será contextual.

## DB-SLOW-157
Baseline dimensions evitarán cardinalidad no acotada.

## DB-SLOW-158
Tenant-specific baseline no será default.

## DB-SLOW-159
Endpoint-specific evidence será diagnostic-only por default.

## DB-SLOW-160
Platform capabilities prevalecerán sobre vendor conditionals.

## DB-SLOW-161
MySQL y MariaDB seguirán siendo plataformas diferenciadas.

## DB-SLOW-162
No se inferirán capacidades exclusivamente por versión.

## DB-SLOW-163
Detector no generará SQL.

## DB-SLOW-164
Detector no ejecutará SQL de aplicación.

## DB-SLOW-165
Detector no modificará Query AST.

## DB-SLOW-166
Detector no modificará Query Plan.

## DB-SLOW-167
Detector no modificará CompiledQuery.

## DB-SLOW-168
Detector no modificará EntityManager.

## DB-SLOW-169
Detector no modificará UnitOfWork.

## DB-SLOW-170
Detector no cambiará connection routing.

## DB-SLOW-171
Detector no cambiará transaction isolation.

## DB-SLOW-172
Detector no hará retry.

## DB-SLOW-173
Detector no hará failover.

## DB-SLOW-174
Detector no creará índices.

## DB-SLOW-175
Detector no modificará schema.

## DB-SLOW-176
Detector no vaciará caches.

## DB-SLOW-177
Detector no ocultará UNKNOWN outcome.

## DB-SLOW-178
Detector preservará evidence confidence.

## DB-SLOW-179
Detector preservará profile coverage.

## DB-SLOW-180
Slow Query Detection será una capa diagnóstica, no una segunda capa de ejecución.

---

# 218. Modelo formal

Sea una consulta:

```text
Q
```

con perfil:

```text
P(Q)
```

y una política:

```text
Π
```

El detector produce:

```text
D(Q) = Detect(P(Q), Π)
```

---

# 219. Detección absoluta

Para una dimensión temporal `x`:

```text
Slow_x(Q)
⇔
T_x(Q) ≥ Θ_x
```

donde:

```text
Θ_x
```

es el threshold efectivo para esa dimensión.

---

# 220. Detección relativa

```text
RelativeSlow(Q)
⇔
T(Q) ≥ B(Q) × α
```

donde:

```text
B(Q)
```

es baseline y:

```text
α
```

multiplicador configurado.

---

# 221. Detección combinada

Podrá utilizarse:

```text
Slow(Q)
⇔
T(Q) ≥ Θ_absolute
∧
T(Q) ≥ B(Q) × α
```

según policy.

---

# 222. Dominant phase

Para fase `p`:

```text
Dominance(p)
=
T_p / T_total
```

cuando las mediciones sean compatibles.

Si:

```text
Dominance(p) ≥ δ
```

la fase podrá considerarse dominante.

---

# 223. Confianza

Conceptualmente:

```text
Confidence
=
f(
  profile coverage,
  measurement confidence,
  baseline samples,
  threshold quality,
  context completeness
)
```

No será necesario convertirlo inicialmente en una fórmula numérica.

---

# 224. Filosofía de diseño

El sistema deberá seguir:

```text
Measure
↓
Compare
↓
Classify
↓
Explain
↓
Recommend
```

Nunca:

```text
Duration > threshold
↓
Guess missing index
↓
Modify database
```

---

# 225. Regla maestra

> **VoltStack Slow Query Detection deberá detectar lentitud como una propiedad observable de una operación y clasificarla utilizando evidencia del Query Profiler; la detección no deberá confundirse con diagnóstico causal, optimización automática, N+1 detection, timeout, failure ni ejecución de EXPLAIN.**

Arquitectura final:

```text
QueryProfile
      ↓
Policy
      ↓
Threshold Evaluation
      ↓
Classification
      ↓
Severity
      ↓
Evidence
      ↓
Deduplication
      ↓
Rate Limiting
      ↓
Diagnostic Enrichment
      ↓
Redaction
      ↓
Telemetry / Debugging
```

con la separación:

```text
Observation
≠
Inference
≠
Recommendation
≠
Automatic Mutation
```

---

# 226. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
✓ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
✓ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 227. Siguiente documento

```text
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

El siguiente documento deberá diseñar el sistema especializado para detectar patrones N+1 utilizando correlación semántica entre:

```text
root query
ORM operation
entity hydration
relationship metadata
relationship loading
query fingerprint
query frequency
temporal locality
IdentityMap
eager/lazy loading
batch loading
transaction context
tenant/shard context
```

preservando como regla fundamental:

> **Repetir una consulta muchas veces no demuestra por sí mismo un problema N+1; VoltStack deberá detectar una relación causal entre una operación raíz, una colección de entidades y consultas repetitivas dependientes de cada entidad o relación.**

Deberá distinguir explícitamente:

```text
Repeated Query
≠
N+1

High Query Count
≠
N+1

Lazy Loading
≠
N+1

Relationship Loading
≠
N+1

N+1 Detection
≠
Automatic Eager Loading
```