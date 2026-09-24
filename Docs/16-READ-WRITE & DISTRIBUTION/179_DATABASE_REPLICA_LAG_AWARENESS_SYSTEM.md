# 179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md

# VoltStack Quantum Database
## Database Replica Lag Awareness System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 179 — Database Replica Lag Awareness System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `178_DATABASE_REPLICA_SYSTEM.md`  
**Siguiente documento:** `180_DATABASE_STICKY_CONNECTION_SYSTEM.md`

---

# 1. Propósito

`Database Replica Lag Awareness System` define la arquitectura mediante la cual VoltStack podrá observar, representar, clasificar y utilizar el **atraso de replicación** de una replica respecto de su fuente de datos.

Los documentos anteriores establecieron:

```text
176 Read/Write Connection System
    ↓
Connection roles and capabilities

177 Read/Write Routing System
    ↓
Semantic endpoint selection

178 Replica System
    ↓
Replication topology and replica identity
```

Este documento añade:

```text
Replication Progress
        │
        ▼
Lag Observation
        │
        ▼
Lag Measurement
        │
        ▼
Lag Evidence
        │
        ▼
Freshness Classification
        │
        ▼
Consistency Evaluation
        │
        ▼
Replica Eligibility
        │
        ▼
Read/Write Routing
```

La regla central será:

> **VoltStack nunca asumirá que una replica está suficientemente actualizada por el simple hecho de estar disponible, saludable o marcada como `READY`; la elegibilidad de una replica para una lectura dependiente de frescura deberá evaluarse utilizando evidencia explícita de progreso o lag, junto con su antigüedad, confianza y relación con los requisitos de consistencia de la operación.**

Por tanto:

```text
ReplicaAvailable
≠
ReplicaHealthy
≠
ReplicaReady
≠
ReplicaFresh
≠
ReplicaCaughtUp
≠
ReplicaEligibleFor(Read)
```

---

# 2. Problema fundamental

Supongamos:

```text
Primary
   │
   ├── Replica A
   ├── Replica B
   └── Replica C
```

Todas responden correctamente:

```text
A = HEALTHY
B = HEALTHY
C = HEALTHY
```

pero:

```text
A = 20 ms behind
B = 2 s behind
C = 3 min behind
```

Una política:

```text
EVENTUAL
```

podría aceptar las tres.

Una política:

```text
MAX_STALENESS = 5 seconds
```

solo debería aceptar:

```text
A
B
```

Una lectura que requiere observar una escritura concreta puede exigir algo aún más fuerte:

```text
ReplicaPosition >= RequiredWritePosition
```

En ese caso incluso una medición:

```text
lag = 1 ms
```

no necesariamente demuestra que la replica ya contiene esa escritura.

---

# 3. Distinciones fundamentales

VoltStack mantendrá:

```text
Replica Lag
≠
Replication Position
≠
Replica Health
≠
Replica State
≠
Read Consistency
≠
Transaction Isolation
≠
Network Latency
≠
Query Latency
```

---

# 4. Lag ≠ network latency

Una query puede tardar:

```text
5 ms
```

contra una replica atrasada:

```text
5 minutes
```

Por tanto:

```text
Fast Query
≠
Fresh Data
```

---

# 5. Lag ≠ health

Una replica puede estar:

```text
HEALTHY
READY
```

y aun así presentar:

```text
ReplicationLag = 45 seconds
```

---

# 6. Lag ≠ position

`ReplicaLag` responde aproximadamente:

> ¿Qué tan atrasada parece estar esta replica?

`ReplicationPosition` responde:

> ¿Hasta qué punto del stream de replicación ha avanzado?

---

# 7. Position is stronger evidence

Para read-after-write, cuando sea soportado:

```text
ReplicaPosition >= RequiredPosition
```

es una evidencia semánticamente más fuerte que:

```text
EstimatedLag < 100 ms
```

---

# 8. Arquitectura general

```text
Replication Platform / Provider
              │
              ▼
       Lag Observation
              │
              ▼
       Normalization
              │
              ▼
         Lag Evidence
              │
      ┌───────┴────────┐
      ▼                ▼
Time Lag          Position Evidence
      │                │
      └───────┬────────┘
              ▼
      Freshness Evaluator
              │
              ▼
       Freshness Profile
              │
              ▼
      Consistency Matcher
              │
              ▼
     Replica Eligibility
              │
              ▼
      Read/Write Router
```

---

# 9. Responsabilidades

El sistema será responsable de:

- recopilar evidencia de lag;
- normalizarla;
- indicar su origen;
- registrar cuándo fue observada;
- expresar confidence;
- detectar evidencia expirada;
- clasificar freshness;
- comparar lag contra budgets;
- evaluar replication position cuando exista;
- producir eligibility evidence;
- integrarse con routing;
- proporcionar telemetry;
- proporcionar diagnostics.

---

# 10. No responsabilidades

No deberá:

- ejecutar queries de aplicación;
- seleccionar directamente una replica;
- implementar load balancing;
- promover replicas;
- hacer failover;
- modificar TransactionContext;
- decidir isolation levels;
- fingir consistencia;
- bloquear indefinidamente esperando sincronización.

---

# 11. ReplicaLag

Abstracción principal:

```php
interface ReplicaLag
{
    public function kind(): ReplicaLagKind;
}
```

---

# 12. ReplicaLagKind

```php
enum ReplicaLagKind
{
    case TIME;
    case POSITION;
    case HYBRID;
    case PROVIDER_ESTIMATE;
    case UNKNOWN;
}
```

---

# 13. Time-based lag

Representa una estimación temporal:

```php
final readonly class TimeReplicaLag implements ReplicaLag
{
    public function __construct(
        public Duration $duration,
    ) {}

    public function kind(): ReplicaLagKind
    {
        return ReplicaLagKind::TIME;
    }
}
```

Ejemplos:

```text
25 ms
500 ms
2 s
30 s
```

---

# 14. Time lag limitation

Una duración:

```text
2 seconds
```

no demuestra necesariamente que:

```text
every transaction committed >2s ago
```

sea visible.

Depende de cómo se mida.

---

# 15. Position-based lag

Cuando existe información de progreso:

```php
final readonly class PositionReplicaLag implements ReplicaLag
{
    public function __construct(
        public ReplicationCoordinate $source,
        public ReplicationCoordinate $replica,
        public ReplicationPositionComparison $comparison,
    ) {}
}
```

---

# 16. Position lag is not necessarily distance

En muchas plataformas puede saberse:

```text
replica BEFORE source
```

sin poder calcular:

```text
exact distance = 37 units
```

VoltStack no deberá inventarla.

---

# 17. Hybrid evidence

Puede existir:

```text
time lag estimate
+
replication coordinate
```

simultáneamente.

Esto se modelará como:

```text
HYBRID
```

sin colapsar prematuramente ambas evidencias.

---

# 18. Provider estimate

Un proveedor administrado puede reportar:

```text
replication lag = 1.7 s
```

sin exponer posiciones.

VoltStack podrá representarlo como:

```text
PROVIDER_ESTIMATE
```

---

# 19. UNKNOWN lag

Si no existe medición:

```text
UNKNOWN
```

Nunca:

```text
UNKNOWN = 0
```

---

# 20. ReplicaLagObservation

Modelo:

```php
final readonly class ReplicaLagObservation
{
    public function __construct(
        public ReplicaId $replica,
        public ReplicaLag $lag,
        public ReplicaLagSource $source,
        public ReplicaLagConfidence $confidence,
        public Instant $observedAt,
        public ?Instant $validUntil,
        public TopologyGeneration $topologyGeneration,
    ) {}
}
```

---

# 21. Lag source

```php
enum ReplicaLagSource
{
    case PLATFORM;
    case PROVIDER;
    case REPLICATION_POSITION;
    case ACTIVE_PROBE;
    case PASSIVE_OBSERVATION;
    case CONFIGURED;
    case UNKNOWN;
}
```

---

# 22. Confidence

```php
enum ReplicaLagConfidence
{
    case VERIFIED;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 23. Confidence matters

Dos mediciones:

```text
A: lag = 200 ms, VERIFIED
B: lag = 100 ms, LOW
```

no deberán tratarse necesariamente como:

```text
B is definitely fresher
```

---

# 24. Measurement freshness

Toda medición dinámica deberá considerar:

```text
observedAt
```

y opcionalmente:

```text
validUntil
```

---

# 25. Evidence age

```text
EvidenceAge
=
Now - ObservedAt
```

---

# 26. Lag measurement age ≠ replication lag

Muy importante:

```text
EvidenceAge
≠
ReplicaLag
```

Ejemplo:

```text
measured lag: 100 ms
measurement age: 30 s
```

La evidencia puede ser demasiado vieja para confiar en ella.

---

# 27. EvidenceFreshness

```php
enum LagEvidenceFreshness
{
    case CURRENT;
    case AGING;
    case STALE;
    case EXPIRED;
    case UNKNOWN;
}
```

---

# 28. Expired evidence

Una medición expirada no deberá ser usada como evidencia actual bajo políticas estrictas.

---

# 29. LagMeasurementPolicy

```php
final readonly class LagMeasurementPolicy
{
    public function __construct(
        public Duration $maxEvidenceAge,
        public Duration $measurementTimeout,
        public ReplicaLagConfidence $minimumConfidence,
    ) {}
}
```

---

# 30. Lag observer

Contrato:

```php
interface ReplicaLagObserver
{
    public function observe(
        ReplicaSnapshot $replica,
        ReplicaLagObservationContext $context,
    ): ReplicaLagObservation;
}
```

---

# 31. Observer adapters

Podrán existir:

```text
MySqlReplicaLagObserver
MariaDbReplicaLagObserver
PostgreSqlReplicaLagObserver
ProviderReplicaLagObserver
PositionBasedReplicaLagObserver
```

---

# 32. Core independence

El core no deberá contener:

```php
if ($driver === 'mysql') {
    // MySQL replication query
}
```

---

# 33. Capability-driven observation

Preferir:

```text
ReplicaLagObservationCapability
```

resuelta desde:

```text
Platform Capability System
```

---

# 34. Measurement execution boundary

Algunas plataformas requieren consultar metadata del DB.

Ese I/O pertenece al observer/collector especializado.

No al:

```text
ReadWriteRoutingEngine
```

---

# 35. Hot path principle

Idealmente:

```text
routing
→ consume recent snapshot
```

y no:

```text
routing
→ query every replica for lag
→ wait
→ route
```

---

# 36. Lag collector

```php
interface ReplicaLagCollector
{
    public function collect(
        ReplicationTopologySnapshot $topology,
    ): ReplicaLagSnapshot;
}
```

---

# 37. ReplicaLagSnapshot

```php
final readonly class ReplicaLagSnapshot
{
    public function __construct(
        public TopologyGeneration $topologyGeneration,
        public LagObservationMap $observations,
        public Instant $capturedAt,
        public LagSnapshotGeneration $generation,
    ) {}
}
```

---

# 38. Snapshot immutability

El snapshot será immutable.

---

# 39. Atomic publication

```text
Observe
↓
Normalize
↓
Validate
↓
Build snapshot
↓
Freeze
↓
Atomic publish
```

---

# 40. Partial observations

No será necesario que todas las replicas produzcan una medición.

Ejemplo:

```text
Replica A → 10 ms
Replica B → UNKNOWN
Replica C → 500 ms
```

es válido.

---

# 41. Partial ≠ failed snapshot

El snapshot podrá indicar cobertura:

```php
enum LagSnapshotCoverage
{
    case COMPLETE;
    case PARTIAL;
    case NONE;
    case UNKNOWN;
}
```

---

# 42. Not observed ≠ zero lag

Regla:

```text
NotObserved
≠
ZeroLag
```

---

# 43. Measurement errors

Una medición puede fallar por:

- timeout;
- permissions;
- unsupported capability;
- replica unavailable;
- topology changed;
- malformed provider response;
- incompatible positions.

Esto deberá producir evidencia explícita, no un valor falso.

---

# 44. LagObservationOutcome

```php
enum LagObservationOutcome
{
    case OBSERVED;
    case UNSUPPORTED;
    case UNAVAILABLE;
    case TIMED_OUT;
    case INCOMPARABLE;
    case FAILED;
    case UNKNOWN;
}
```

---

# 45. Zero lag

Incluso:

```text
lag = 0
```

debe interpretarse dentro de la semántica del observer.

No significa universalmente:

```text
bit-for-bit identical at this instant
```

---

# 46. Negative lag

Un valor temporal negativo deberá considerarse:

```text
invalid observation
```

salvo que un adapter especializado tenga una semántica explícita.

---

# 47. Clock skew

Las mediciones temporales pueden verse afectadas por:

```text
clock skew
```

entre sistemas.

---

# 48. Clock reliability

Un observer deberá declarar si su medición depende de:

```text
local clock
remote clock
provider clock
```

cuando sea relevante.

---

# 49. Monotonic clock

Para:

```text
measurement duration
timeouts
evidence age within process
```

VoltStack deberá preferir monotonic clocks donde sea posible.

---

# 50. Wall clock

Para timestamps diagnósticos:

```text
observedAt
```

podrá utilizarse wall-clock time.

---

# 51. Freshness requirement

Una operación de lectura podrá expresar:

```php
interface ReplicaFreshnessRequirement
{
}
```

---

# 52. Requirement types

Propuesta:

```text
ANY
MAX_STALENESS
REQUIRED_POSITION
AUTHORITATIVE
CUSTOM
```

---

# 53. ANY

No exige una frescura concreta más allá de eligibility básica.

Útil para:

```text
EVENTUAL
```

---

# 54. MAX_STALENESS

Ejemplo:

```php
new MaxReplicaStaleness(
    Duration::seconds(5),
);
```

---

# 55. REQUIRED_POSITION

Ejemplo:

```php
new RequiredReplicationCoordinate(
    $writeObservation->coordinate(),
);
```

---

# 56. AUTHORITATIVE

Exige una fuente considerada autoritativa.

No debe simularse mediante:

```text
lag < X
```

---

# 57. FreshnessRequirement ≠ ReadConsistencyRequirement

La consistencia es semántica de alto nivel.

Freshness requirement es una condición operacional utilizada para satisfacerla.

Ejemplo:

```text
ReadConsistency::SESSION
        ↓
RequiredReplicationCoordinate(P)
```

---

# 58. Consistency translation

Un componente especializado podrá resolver:

```text
ReadConsistencyRequirement
        │
        ▼
ReplicaFreshnessRequirement
```

---

# 59. Eventual consistency

Podría resolver:

```text
EVENTUAL
→ ANY
```

---

# 60. Bounded staleness

Podría resolver:

```text
BOUNDED_STALENESS(5s)
→ MAX_STALENESS(5s)
```

---

# 61. Session consistency

Si existe write coordinate:

```text
SESSION
→ REQUIRED_POSITION(P)
```

---

# 62. Session without coordinate

Podría utilizar:

```text
STICKY_WRITER
```

como fallback.

Esto se desarrolla en 180.

---

# 63. Authoritative read

```text
AUTHORITATIVE
→ AUTHORITATIVE
```

No:

```text
AUTHORITATIVE
→ MAX_STALENESS(0)
```

porque no son semánticamente equivalentes.

---

# 64. ReplicaFreshnessEvaluator

```php
interface ReplicaFreshnessEvaluator
{
    public function evaluate(
        ReplicaSnapshot $replica,
        ReplicaLagObservation $observation,
        ReplicaFreshnessRequirement $requirement,
    ): ReplicaFreshnessEvaluation;
}
```

---

# 65. Evaluation result

```php
final readonly class ReplicaFreshnessEvaluation
{
    public function __construct(
        public ReplicaFreshnessStatus $status,
        public ReplicaFreshnessReason $reason,
        public ReplicaLagConfidence $confidence,
    ) {}
}
```

---

# 66. Freshness statuses

```php
enum ReplicaFreshnessStatus
{
    case SATISFIED;
    case UNSATISFIED;
    case UNKNOWN;
}
```

---

# 67. UNKNOWN is essential

Ejemplo:

```text
Requirement:
    max lag 5 seconds

Evidence:
    unavailable
```

Resultado:

```text
UNKNOWN
```

No:

```text
UNSATISFIED
```

necesariamente.

Tampoco:

```text
SATISFIED
```

---

# 68. Policy handles UNKNOWN

Posteriormente una policy decide:

```text
UNKNOWN
→ reject replica
```

o para consistencia débil:

```text
UNKNOWN
→ allow with degraded confidence
```

si explícitamente configurado.

---

# 69. Strict consistency

Para garantías fuertes:

```text
UNKNOWN
```

deberá fallar cerrado.

---

# 70. Max staleness evaluation

Formalmente:

```text
SATISFIED
iff
ObservedLag <= MaximumLag
AND
EvidenceFreshEnough
AND
Confidence >= RequiredConfidence
```

---

# 71. Boundary semantics

Si:

```text
ObservedLag = MaximumLag
```

por default:

```text
SATISFIED
```

si el resto de condiciones se cumplen.

---

# 72. Position evaluation

Sea:

```text
R = required coordinate
P = replica coordinate
```

Si:

```text
P AFTER R
OR
P EQUAL R
```

la requirement puede considerarse satisfecha.

---

# 73. Position BEFORE

```text
P BEFORE R
```

→ `UNSATISFIED`.

---

# 74. Position INCOMPARABLE

→ `UNKNOWN` o error de capability según contexto.

Nunca:

```text
SATISFIED
```

---

# 75. Position CONCURRENT

Para garantías que exigen haber observado un write específico:

```text
CONCURRENT
```

no demuestra satisfacción.

---

# 76. Position UNKNOWN

→ `UNKNOWN`.

---

# 77. FreshnessProfile

El sistema podrá producir:

```php
final readonly class ReplicaFreshnessProfile
{
    public function __construct(
        public ReplicaId $replica,
        public ReplicaFreshnessClass $class,
        public ReplicaLagObservation $observation,
        public LagEvidenceFreshness $evidenceFreshness,
    ) {}
}
```

---

# 78. Freshness classes

Ejemplo configurable:

```php
enum ReplicaFreshnessClass
{
    case CURRENT;
    case NEAR_CURRENT;
    case DELAYED;
    case STALE;
    case EXPIRED;
    case UNKNOWN;
}
```

---

# 79. Classes are policy-derived

No deberán tener thresholds universales.

---

# 80. Example policy

```text
CURRENT:
    <= 100 ms

NEAR_CURRENT:
    <= 1 s

DELAYED:
    <= 10 s

STALE:
    > 10 s
```

podría ser una policy de una aplicación.

No un significado universal del framework.

---

# 81. LagThresholdPolicy

```php
final readonly class LagThresholdPolicy
{
    public function __construct(
        public Duration $current,
        public Duration $nearCurrent,
        public Duration $delayed,
    ) {}
}
```

---

# 82. Per-connection policy

Distintos clusters podrán tener thresholds diferentes.

---

# 83. Per-workload policy

También:

```text
web request
analytics
reporting
background processing
```

podrían aceptar diferente staleness.

---

# 84. Per-query requirements

API avanzada podría permitir:

```php
User::query()
    ->maxReplicaStaleness(Duration::seconds(2))
    ->get();
```

---

# 85. API semantics

Esto deberá generar:

```text
ReplicaFreshnessRequirement
```

No seleccionar una replica directamente.

---

# 86. Default policy

La aplicación podrá definir:

```php
'database' => [
    'replicas' => [
        'default_max_staleness' => '5s',
    ],
],
```

si desea bounded-staleness reads.

---

# 87. No hidden guarantee

Si no se configura un límite:

VoltStack no deberá presentar las lecturas de replica como bounded-staleness.

---

# 88. Lag-aware eligibility

La eligibility completa:

```text
Replica Base Eligibility
        │
        ▼
Lag Evidence Available?
        │
        ▼
Evidence Fresh?
        │
        ▼
Confidence Sufficient?
        │
        ▼
Freshness Requirement Satisfied?
        │
        ▼
Lag-Aware Eligibility
```

---

# 89. ReplicaLagEligibilityEvaluator

```php
interface ReplicaLagEligibilityEvaluator
{
    public function evaluate(
        ReplicaSnapshot $replica,
        ReplicaFreshnessRequirement $requirement,
        ReplicaLagSnapshot $lag,
    ): ReplicaLagEligibility;
}
```

---

# 90. Eligibility result

```php
final readonly class ReplicaLagEligibility
{
    public function __construct(
        public ReplicaId $replica,
        public ReplicaEligibilityStatus $status,
        public ReplicaFreshnessEvaluation $freshness,
    ) {}
}
```

---

# 91. Router integration

```text
Read Operation
      │
      ▼
Consistency Requirement
      │
      ▼
Freshness Requirement
      │
      ▼
Replica Registry
      │
      ▼
Base Eligible Replicas
      │
      ▼
Lag Snapshot
      │
      ▼
Lag-Aware Eligibility
      │
      ▼
ReadWrite Router
```

---

# 92. Router does not measure lag

Regla:

```text
ReadWriteRouter
≠
ReplicaLagObserver
```

---

# 93. Router consumes evidence

El router recibe:

```text
RoutingEvidenceSnapshot
```

que puede incluir:

```text
freshness class
lag status
required-position satisfaction
```

---

# 94. Lag as hard constraint

Si la operación exige:

```text
MAX_STALENESS = 1 second
```

lag será una:

```text
Hard Constraint
```

---

# 95. Lag as preference

Para una lectura eventual sin límite:

```text
lower lag
```

puede utilizarse como:

```text
Soft Preference
```

---

# 96. Constraint before ranking

Nunca:

```text
replica A lag = 20 s
replica B lag = 1 s

requirement <= 5 s

load balancer selects A because less loaded
```

A debe eliminarse antes del load balancing.

---

# 97. Ranking by freshness

Entre replicas elegibles:

```text
A = 20 ms
B = 100 ms
```

la policy podría preferir A.

Pero no necesariamente.

---

# 98. Freshest replica ≠ best replica

Porque también importan:

```text
load
latency
region
health
affinity
capacity
```

---

# 99. Correctness before optimization

Orden:

```text
Requirement Satisfaction
        ↓
Eligibility
        ↓
Health
        ↓
Routing Policy
        ↓
Load Balancing
```

---

# 100. Waiting for replica catch-up

Algunas operaciones podrían permitir:

```text
wait until replica reaches required position
```

---

# 101. Wait policy

Debe ser explícita:

```php
final readonly class ReplicaCatchUpWaitPolicy
{
    public function __construct(
        public bool $enabled,
        public Duration $timeout,
        public Duration $pollInterval,
    ) {}
}
```

---

# 102. No unbounded waiting

Prohibido:

```text
while replica behind:
    wait forever
```

---

# 103. Catch-up wait ≠ transaction wait

Es un mecanismo de consistencia de lectura.

No una transaction.

---

# 104. Wait location

La espera no deberá implementarse dentro del generic router.

Preferir:

```text
ReplicaCatchUpCoordinator
```

---

# 105. Catch-up coordinator

```php
interface ReplicaCatchUpCoordinator
{
    public function await(
        ReplicaId $replica,
        ReplicationCoordinate $required,
        ReplicaCatchUpWaitPolicy $policy,
    ): ReplicaCatchUpResult;
}
```

---

# 106. Result

```php
enum ReplicaCatchUpResult
{
    case SATISFIED;
    case TIMED_OUT;
    case REPLICA_UNAVAILABLE;
    case POSITION_INCOMPARABLE;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 107. Better fallback

En aplicaciones web, frecuentemente será mejor:

```text
replica behind
→ route to writer
```

que:

```text
wait several seconds
```

---

# 108. Policy decides

Core no impondrá una estrategia universal.

---

# 109. Cancellation

Catch-up waiting deberá respetar:

```text
CancellationToken
```

---

# 110. Deadline

También:

```text
Query/Request Deadline
```

---

# 111. Remaining budget

La espera no deberá superar el deadline restante.

---

# 112. Transaction integration

Dentro de una transaction:

```text
PinnedConnection
```

tiene precedencia.

---

# 113. Write transaction

No se hará:

```text
READ
→ inspect replica lag
→ choose replica
```

dentro de una write transaction pinned.

---

# 114. Read-only replica transaction

Si una transaction read-only comenzó sobre una replica:

la transaction permanece pinned.

No deberá migrarse a otra replica simplemente porque su lag cambió.

---

# 115. Consistency deterioration mid-transaction

Si la replica deja de cumplir una expectativa operacional:

VoltStack no podrá mover transparentemente la physical transaction.

Deberá aplicar la política/error correspondiente.

---

# 116. Transaction isolation distinction

```text
Replica freshness
≠
Transaction isolation
```

`REPEATABLE_READ` sobre una replica atrasada puede dar una vista consistente pero antigua.

---

# 117. ORM IdentityMap

También:

```text
IdentityMap
≠
Replica Freshness
```

---

# 118. Repeated entity lookup

Si:

```php
$user = $em->find(User::class, 10);
```

ya está en IdentityMap, una segunda llamada puede no consultar la DB.

Por tanto lag awareness no participa.

---

# 119. Refresh

Una operación explícita:

```php
$em->refresh($user);
```

sí deberá producir una nueva query con requisitos de routing/consistencia definidos.

---

# 120. Query cache

Un cache hit puede evitar DB access.

Por tanto:

```text
ReplicaLag
```

no determina la frescura del resultado cacheado.

---

# 121. Cache freshness separate

```text
CacheFreshness
≠
ReplicaFreshness
```

---

# 122. Cache consistency policy

El sistema de Cache deberá determinar si un cached result puede satisfacer la consistency requirement antes de evitar routing.

---

# 123. Sticky integration

Documento 180 utilizará este sistema para decidir cuándo una lectura puede abandonar el writer.

---

# 124. Example sticky evolution

```text
WRITE committed
      │
      ▼
Required Position P
      │
      ▼
Sticky Writer
      │
      ▼
Replica reaches P?
   │        │
  NO       YES
   │        │
Writer    Replica eligible
```

---

# 125. Without positions

Puede utilizarse:

```text
time-based sticky window
```

como estrategia aproximada.

Pero:

```text
StickyDuration
≠
ProofReplicaCaughtUp
```

---

# 126. Failover integration

Durante failover:

```text
replication topology
```

puede cambiar radicalmente.

---

# 127. Generation mismatch

Lag evidence de:

```text
TopologyGeneration 41
```

no deberá aplicarse ciegamente a:

```text
TopologyGeneration 42
```

---

# 128. Evidence generation check

Por default:

```text
Observation.TopologyGeneration
=
CurrentTopologyGeneration
```

deberá verificarse.

---

# 129. Epoch changes

Tras promotion/failover:

```text
ReplicationEpoch
```

puede cambiar.

Old position evidence puede quedar:

```text
INCOMPARABLE
```

---

# 130. No cross-epoch freshness assumption

Prohibido:

```text
old lag 0 ms
→ new primary topology must also be current
```

---

# 131. Load balancing integration

El documento 182 podrá utilizar:

```text
FreshnessClass
```

como input de candidate ranking.

---

# 132. Example

```text
A:
  eligible
  lag 20ms
  load 95%

B:
  eligible
  lag 80ms
  load 20%
```

Una policy puede elegir B si ambos cumplen la freshness requirement.

---

# 133. Analytics workloads

Para reporting:

```text
max staleness = 5 minutes
```

puede ser aceptable.

---

# 134. Interactive workloads

Para web:

```text
max staleness = 2 seconds
```

podría configurarse.

---

# 135. Critical reads

Para:

```text
payment state
permission changes
security-sensitive state
```

la aplicación podría exigir:

```text
AUTHORITATIVE_READ
```

en vez de depender de lag estimado.

---

# 136. Security-sensitive consistency

VoltStack deberá permitir elevar la consistency requirement explícitamente.

---

# 137. No automatic semantic guessing

El framework no deberá decidir:

```text
table name = payments
→ authoritative
```

mediante heurísticas ocultas.

---

# 138. Query metadata

Una capa superior podrá declarar:

```text
consistency profile
```

mediante metadata explícita.

---

# 139. ConsistencyProfile

Ejemplo:

```php
enum DatabaseConsistencyProfile
{
    case EVENTUAL;
    case SESSION;
    case BOUNDED_STALENESS;
    case AUTHORITATIVE;
}
```

---

# 140. Profile parameters

`BOUNDED_STALENESS` requiere:

```text
maximum duration
```

y no deberá almacenarse solo como enum si necesita parámetros.

---

# 141. Parameterized requirement

Preferir value objects:

```php
new BoundedStalenessRequirement(
    maximum: Duration::seconds(2),
);
```

---

# 142. Observation scheduling

Lag collection podrá ser:

```text
periodic
event-driven
on-demand
hybrid
```

---

# 143. Periodic

Adecuado para topologías estables.

---

# 144. Event-driven

Puede reaccionar a:

```text
replication state changes
provider events
failover
```

---

# 145. On-demand

Podrá utilizarse para una consistency requirement especial.

Pero deberá ser explícito por su costo.

---

# 146. Hybrid

Ejemplo:

```text
periodic snapshots
+
on-demand verification for REQUIRED_POSITION
```

---

# 147. Probe amplification

Debe evitarse:

```text
100 concurrent requests
→ 100 lag probes to each replica
```

---

# 148. Single-flight measurement

Podrá existir:

```text
ReplicaLagSingleFlightCoordinator
```

para compartir un refresh concurrente.

---

# 149. Single-flight result

Comparte:

```text
measurement operation
```

No mutable request state.

---

# 150. Throttling

Collectors deberán soportar:

```text
minimum refresh interval
```

---

# 151. Backpressure

Si monitoring infrastructure está saturada:

```text
lag evidence = unavailable/stale
```

es preferible a saturar la DB.

---

# 152. Resource budgets

Definir:

```text
max replicas per collection
max concurrent probes
max observation duration
max provider payload
max retry attempts
```

---

# 153. Retry

Lag observation retry será distinta de:

```text
Query Retry
Transaction Retry
```

---

# 154. Observation retry safety

Normalmente una metadata read puede ser retryable, pero dependerá del adapter.

---

# 155. Bounded retries

Nunca infinitos.

---

# 156. Telemetry architecture

Eventos conceptuales:

```text
ReplicaLagObserved
ReplicaLagObservationFailed
ReplicaLagThresholdExceeded
ReplicaFreshnessChanged
ReplicaFreshnessRequirementRejected
ReplicaCatchUpStarted
ReplicaCatchUpCompleted
```

---

# 157. Events observational

No deberán mutar:

```text
ReplicaLagSnapshot
```

ya publicado.

---

# 158. Metrics

Ejemplos:

```text
db.replica.lag.seconds
db.replica.lag.observation.duration
db.replica.lag.observation.failures
db.replica.lag.evidence.age
db.replica.freshness.rejections
db.replica.catchup.wait.duration
db.replica.catchup.timeouts
```

---

# 159. Cardinality control

No usar por default:

```text
tenant_id
user_id
query_id
full_endpoint
```

como labels.

---

# 160. Replica label

Incluso `ReplicaId` puede ser de alta cardinalidad en sistemas grandes.

La telemetry policy deberá decidir.

---

# 161. Histograms

Lag temporal puede exponerse mediante histogramas.

---

# 162. Unknown metrics

No representar:

```text
UNKNOWN lag
```

como:

```text
0 seconds
```

---

# 163. Unknown counter

Preferir:

```text
db.replica.lag.unknown
```

---

# 164. Tracing

Ejemplo:

```text
db.replica.lag.kind = position
db.replica.lag.confidence = verified
db.replica.freshness.status = satisfied
db.replica.freshness.requirement = session
```

---

# 165. Sensitive data

No registrar:

```text
raw SQL
query parameters
credentials
provider tokens
```

como parte del lag subsystem.

---

# 166. Diagnostics API

Conceptualmente:

```php
DB::replicas('default')
    ->lag()
    ->inspect();
```

---

# 167. Example diagnostics

```text
REPLICA LAG STATUS

Topology Generation:
    42

Lag Snapshot:
    128

Replica A:
    state: READY
    lag: 24 ms
    evidence age: 400 ms
    confidence: VERIFIED
    freshness: CURRENT

Replica B:
    state: READY
    lag: 2.4 s
    evidence age: 500 ms
    confidence: HIGH
    freshness: DELAYED

Replica C:
    state: READY
    lag: UNKNOWN
    evidence age: N/A
    confidence: UNKNOWN
    freshness: UNKNOWN
```

---

# 168. Explain requirement

```php
DB::replicas('default')
    ->lag()
    ->explain(
        replica: 'replica-b',
        requirement: new MaxReplicaStaleness(
            Duration::seconds(1),
        ),
    );
```

---

# 169. Explain result

```text
Replica:
    replica-b

Requirement:
    MAX_STALENESS <= 1s

Observed Lag:
    2.4s

Evidence:
    CURRENT

Confidence:
    HIGH

Result:
    UNSATISFIED

Routing Eligibility:
    REJECT
```

---

# 170. Position diagnostics

```text
Required Coordinate:
    domain: cluster-1
    epoch: 17
    position: [redacted]

Replica Coordinate:
    domain: cluster-1
    epoch: 17
    position: [redacted]

Comparison:
    BEFORE

Result:
    UNSATISFIED
```

---

# 171. Unknown diagnostics

```text
Requirement:
    SESSION

Required Position:
    available

Replica Position:
    unavailable

Result:
    UNKNOWN

Policy:
    FAIL_CLOSED

Recommended Routing:
    AUTHORITATIVE
```

---

# 172. Security

Lag observers pueden requerir acceso a metadata administrativa.

---

# 173. Least privilege

Las credenciales utilizadas para observación deberán tener únicamente los permisos necesarios.

---

# 174. Credential separation

```text
ApplicationConnectionCredentials
≠
ReplicaMonitoringCredentials
```

cuando la infraestructura lo permita.

---

# 175. Provider credentials

Nunca deberán incorporarse a:

```text
ReplicaLagObservation
ReplicaLagSnapshot
RoutingEvidenceSnapshot
```

---

# 176. Untrusted provider values

Deben validarse:

```text
durations
timestamps
positions
replica identities
topology generation
```

---

# 177. Overflow protection

Valores absurdos:

```text
lag = 999999999999999 years
```

deberán rechazarse o normalizarse mediante límites explícitos.

---

# 178. Parsing

No deberá aceptarse parsing ambiguo:

```text
"1:20"
```

sin formato definido.

---

# 179. Position opacity

Raw replication coordinates deberán tratarse como datos opacos por capas que no entienden su tipo.

---

# 180. Persistent runtime safety

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el sistema deberá separar:

```text
shared immutable lag snapshots
```

de:

```text
scope-local requirements and decisions
```

---

# 181. No global consistency requirement

Prohibido:

```php
static $maxReplicaLag = 2;
```

como estado mutable por request.

---

# 182. Snapshot publication

La referencia al último snapshot podrá ser shared y actualizada atómicamente.

---

# 183. Concurrent readers

Readers deberán observar:

```text
snapshot N
```

o:

```text
snapshot N+1
```

nunca un snapshot parcialmente actualizado.

---

# 184. Observation generation

Cada nuevo snapshot tendrá:

```text
LagSnapshotGeneration
```

---

# 185. Old snapshot retention

Bounded.

---

# 186. Request A vs B

```text
Request A:
requires <= 500ms

Request B:
accepts eventual
```

sus requirements deberán permanecer aisladas.

---

# 187. Coroutine isolation

Dos coroutines no compartirán:

```text
current freshness requirement
catch-up wait state
routing fallback state
```

---

# 188. Cancellation cleanup

Una espera cancelada deberá liberar:

```text
timers
wait registrations
temporary observation handles
```

---

# 189. Error hierarchy

Propuesta:

```text
DatabaseReplicaLagException
├── ReplicaLagObservationException
├── ReplicaLagObservationTimeoutException
├── ReplicaLagUnsupportedException
├── ReplicaLagEvidenceUnavailableException
├── ReplicaLagEvidenceExpiredException
├── ReplicaLagInvalidValueException
├── ReplicaLagThresholdExceededException
├── ReplicaFreshnessUnsatisfiedException
├── ReplicaFreshnessUnknownException
├── ReplicationPositionUnavailableException
├── ReplicationPositionIncomparableException
├── ReplicaCatchUpTimeoutException
├── ReplicaCatchUpCancelledException
├── ReplicaLagTopologyGenerationMismatchException
└── ReplicaLagInvariantViolationException
```

---

# 190. Error semantics

`ReplicaLagThresholdExceededException` no deberá implicar:

```text
replica broken
```

Solo:

```text
replica unsuitable for this requirement
```

---

# 191. Unknown exception

`ReplicaFreshnessUnknownException` deberá conservar la diferencia entre:

```text
known stale
```

y:

```text
cannot determine
```

---

# 192. Testing architecture

Suite propuesta:

```text
ReplicaLagModelTests
ReplicaLagObservationTests
ReplicaLagCollectorTests
ReplicaLagSnapshotTests
ReplicaLagFreshnessTests
ReplicaLagThresholdTests
ReplicaPositionFreshnessTests
ReplicaLagEligibilityTests
ReplicaLagRoutingTests
ReplicaLagCatchUpTests
ReplicaLagTopologyTests
ReplicaLagFailureTests
ReplicaLagTelemetryTests
ReplicaLagDiagnosticsTests
ReplicaLagPersistentRuntimeTests
ReplicaLagConcurrencyTests
ReplicaLagSecurityTests
```

---

# 193. Time lag test

```text
lag = 500ms
requirement <= 1s
```

→ `SATISFIED`.

---

# 194. Threshold boundary test

```text
lag = 1s
requirement <= 1s
```

→ `SATISFIED`.

---

# 195. Exceeded test

```text
lag = 1.001s
requirement <= 1s
```

→ `UNSATISFIED`.

---

# 196. Unknown test

```text
lag = UNKNOWN
requirement <= 1s
```

→ `UNKNOWN`.

---

# 197. Expired evidence test

```text
lag = 100ms
measurement age = 1 minute
max evidence age = 5 seconds
```

no deberá ser `SATISFIED`.

---

# 198. Low confidence test

```text
lag = 50ms
confidence = LOW
required confidence = HIGH
```

no deberá satisfacer strict requirement.

---

# 199. Position equality test

```text
ReplicaPosition = RequiredPosition
```

→ `SATISFIED`.

---

# 200. Position ahead test

```text
ReplicaPosition AFTER RequiredPosition
```

→ `SATISFIED`.

---

# 201. Position behind test

```text
ReplicaPosition BEFORE RequiredPosition
```

→ `UNSATISFIED`.

---

# 202. Position incomparable test

Different epoch:

```text
INCOMPARABLE
```

→ never `SATISFIED`.

---

# 203. Topology generation test

Observation:

```text
generation 41
```

Current topology:

```text
generation 42
```

deberá invalidarse/reconciliarse según policy.

---

# 204. Partial snapshot test

Una replica sin observación no deberá impedir que otras replicas tengan evidencia válida.

---

# 205. Zero lag test

`0ms` deberá conservar su confidence/source y no transformarse en autoridad absoluta.

---

# 206. Negative lag test

Debe rechazarse.

---

# 207. Catch-up timeout test

Replica no alcanza posición antes del deadline:

```text
TIMED_OUT
```

---

# 208. Catch-up cancellation test

Cancellation:

```text
CANCELLED
```

con cleanup correcto.

---

# 209. Router test

Replica que no satisface hard freshness requirement deberá eliminarse antes del load balancer.

---

# 210. Transaction test

Active write transaction deberá ignorar ordinary replica lag routing.

---

# 211. Sticky integration test

Después de write:

```text
required position P
```

mientras ninguna replica cubra P:

```text
writer remains required
```

cuando la sticky policy use position awareness.

---

# 212. Persistent worker test

Miles de measurements deberán mantener:

```text
bounded memory
bounded history
no request leakage
```

---

# 213. Concurrency test

Concurrent collectors no deberán publicar una generación vieja sobre una nueva.

---

# 214. Proposed directory structure

```text
src/Quantum/Database/Replication/Lag/
│
├── ReplicaLag.php
├── ReplicaLagKind.php
├── TimeReplicaLag.php
├── PositionReplicaLag.php
├── HybridReplicaLag.php
├── UnknownReplicaLag.php
│
├── Observation/
│   ├── ReplicaLagObservation.php
│   ├── ReplicaLagSource.php
│   ├── ReplicaLagConfidence.php
│   ├── LagObservationOutcome.php
│   ├── ReplicaLagObserver.php
│   └── ReplicaLagObservationContext.php
│
├── Collection/
│   ├── ReplicaLagCollector.php
│   ├── ReplicaLagSnapshot.php
│   ├── LagSnapshotGeneration.php
│   ├── LagSnapshotCoverage.php
│   ├── LagObservationMap.php
│   └── ReplicaLagSingleFlightCoordinator.php
│
├── Freshness/
│   ├── ReplicaFreshnessRequirement.php
│   ├── AnyReplicaFreshness.php
│   ├── MaxReplicaStaleness.php
│   ├── RequiredReplicationCoordinate.php
│   ├── AuthoritativeFreshnessRequirement.php
│   ├── ReplicaFreshnessEvaluator.php
│   ├── ReplicaFreshnessEvaluation.php
│   ├── ReplicaFreshnessStatus.php
│   ├── ReplicaFreshnessClass.php
│   ├── ReplicaFreshnessProfile.php
│   └── LagEvidenceFreshness.php
│
├── Policy/
│   ├── LagMeasurementPolicy.php
│   ├── LagThresholdPolicy.php
│   ├── ReplicaLagEligibilityEvaluator.php
│   └── ReplicaLagEligibility.php
│
├── CatchUp/
│   ├── ReplicaCatchUpCoordinator.php
│   ├── ReplicaCatchUpWaitPolicy.php
│   └── ReplicaCatchUpResult.php
│
├── Platform/
│   ├── ReplicaLagObservationCapability.php
│   ├── MySqlReplicaLagObserver.php
│   ├── MariaDbReplicaLagObserver.php
│   ├── PostgreSqlReplicaLagObserver.php
│   └── ProviderReplicaLagObserver.php
│
├── Routing/
│   └── ReplicaLagRoutingEvidenceProvider.php
│
├── Diagnostics/
│   ├── ReplicaLagInspector.php
│   ├── ReplicaLagExplainer.php
│   └── ReplicaLagReport.php
│
├── Telemetry/
│   ├── ReplicaLagTelemetry.php
│   ├── ReplicaLagObserved.php
│   ├── ReplicaLagThresholdExceeded.php
│   └── ReplicaFreshnessChanged.php
│
└── Exception/
    ├── DatabaseReplicaLagException.php
    ├── ReplicaLagObservationException.php
    ├── ReplicaLagObservationTimeoutException.php
    ├── ReplicaLagUnsupportedException.php
    ├── ReplicaLagEvidenceUnavailableException.php
    ├── ReplicaLagEvidenceExpiredException.php
    ├── ReplicaLagInvalidValueException.php
    ├── ReplicaLagThresholdExceededException.php
    ├── ReplicaFreshnessUnsatisfiedException.php
    ├── ReplicaFreshnessUnknownException.php
    ├── ReplicationPositionUnavailableException.php
    ├── ReplicaCatchUpTimeoutException.php
    └── ReplicaLagInvariantViolationException.php
```

---

# 215. Architectural invariants

## DB-RLA-001
Replica lag será distinto de network latency.

## DB-RLA-002
Replica lag será distinto de query latency.

## DB-RLA-003
Replica lag será distinto de replica health.

## DB-RLA-004
Replica lag será distinto de replica lifecycle state.

## DB-RLA-005
Replica lag será distinto de replication position.

## DB-RLA-006
Replica lag será distinto de transaction isolation.

## DB-RLA-007
Replica availability no implicará freshness.

## DB-RLA-008
Replica READY no implicará caught-up.

## DB-RLA-009
Healthy replica no implicará acceptable lag.

## DB-RLA-010
UNKNOWN lag nunca equivaldrá a zero lag.

## DB-RLA-011
Not observed nunca equivaldrá a zero lag.

## DB-RLA-012
Time lag será una medición con semántica explícita.

## DB-RLA-013
Time lag no probará automáticamente una required position.

## DB-RLA-014
Position evidence podrá ser más fuerte para read-after-write.

## DB-RLA-015
Replication position no se asumirá numérica.

## DB-RLA-016
Position lag no requerirá una distancia calculable.

## DB-RLA-017
Hybrid evidence conservará sus componentes.

## DB-RLA-018
Provider estimate se identificará como tal.

## DB-RLA-019
Lag observation tendrá timestamp.

## DB-RLA-020
Lag observation tendrá source.

## DB-RLA-021
Lag observation tendrá confidence.

## DB-RLA-022
Lag observation pertenecerá a topology generation.

## DB-RLA-023
Evidence age será distinta de measured lag.

## DB-RLA-024
Expired evidence no será current evidence.

## DB-RLA-025
Stale evidence no se usará como fresh bajo strict policy.

## DB-RLA-026
Observation confidence será considerada por requirements fuertes.

## DB-RLA-027
Low-confidence evidence no satisfará high-confidence requirement.

## DB-RLA-028
Core no tendrá vendor-specific lag logic dispersa.

## DB-RLA-029
Platform adapters normalizarán lag semantics.

## DB-RLA-030
ReadWriteRouter no medirá lag directamente.

## DB-RLA-031
QueryExecutor no medirá lag directamente.

## DB-RLA-032
ORM no medirá lag directamente.

## DB-RLA-033
Lag collector podrá ejecutar specialized monitoring I/O.

## DB-RLA-034
Lag monitoring I/O estará fuera del normal query hot path cuando sea posible.

## DB-RLA-035
ReplicaLagSnapshot será immutable.

## DB-RLA-036
Snapshot publication será atómica.

## DB-RLA-037
Partial lag coverage será representable.

## DB-RLA-038
Una replica sin evidence no invalidará evidence de otras replicas.

## DB-RLA-039
Observation failure será distinta de zero lag.

## DB-RLA-040
Unsupported observation será distinta de failed observation.

## DB-RLA-041
Timeout será representable.

## DB-RLA-042
Incomparable position será representable.

## DB-RLA-043
Negative lag será inválido por default.

## DB-RLA-044
Clock skew deberá considerarse cuando afecte medición.

## DB-RLA-045
Monotonic clock será preferido para durations/deadlines.

## DB-RLA-046
Wall clock podrá utilizarse para diagnostic timestamps.

## DB-RLA-047
FreshnessRequirement será distinta de ReadConsistencyRequirement.

## DB-RLA-048
Consistency podrá resolverse a operational freshness requirement.

## DB-RLA-049
EVENTUAL podrá aceptar ANY freshness.

## DB-RLA-050
BOUNDED_STALENESS podrá generar MAX_STALENESS.

## DB-RLA-051
SESSION podrá generar REQUIRED_POSITION cuando exista write coordinate.

## DB-RLA-052
SESSION sin coordinate podrá requerir sticky writer.

## DB-RLA-053
AUTHORITATIVE no se reducirá a zero-lag replica.

## DB-RLA-054
Max staleness será parametrizable.

## DB-RLA-055
Framework no impondrá un max staleness universal.

## DB-RLA-056
Freshness evaluation producirá SATISFIED, UNSATISFIED o UNKNOWN.

## DB-RLA-057
UNKNOWN nunca será SATISFIED implícitamente.

## DB-RLA-058
Strict consistency tratará UNKNOWN de forma conservadora.

## DB-RLA-059
Max-staleness evaluation considerará evidence freshness.

## DB-RLA-060
Max-staleness evaluation considerará confidence.

## DB-RLA-061
Boundary equality será válida salvo policy contraria explícita.

## DB-RLA-062
Position EQUAL required podrá satisfacer requirement.

## DB-RLA-063
Position AFTER required podrá satisfacer requirement.

## DB-RLA-064
Position BEFORE required no satisfará requirement.

## DB-RLA-065
Position CONCURRENT no probará required-write visibility.

## DB-RLA-066
Position INCOMPARABLE no satisfará requirement fuerte.

## DB-RLA-067
Position UNKNOWN no satisfará requirement fuerte.

## DB-RLA-068
Freshness classes serán policy-derived.

## DB-RLA-069
Freshness class thresholds no serán universales.

## DB-RLA-070
Different workloads podrán tener diferentes staleness budgets.

## DB-RLA-071
Per-query freshness requirements serán explícitas.

## DB-RLA-072
Per-query requirement no seleccionará endpoint directamente.

## DB-RLA-073
Lag-aware eligibility será distinta de routing.

## DB-RLA-074
Lag-aware eligibility ocurrirá antes de load balancing.

## DB-RLA-075
Replica que viola hard lag requirement será eliminada.

## DB-RLA-076
Load balancer no podrá reintroducir una replica rechazada.

## DB-RLA-077
Lower lag podrá ser soft preference cuando no sea constraint.

## DB-RLA-078
Freshest replica no será automáticamente best replica.

## DB-RLA-079
Correctness tendrá prioridad sobre load optimization.

## DB-RLA-080
Waiting for catch-up será explícito.

## DB-RLA-081
Catch-up waiting será bounded.

## DB-RLA-082
Catch-up waiting respetará timeout.

## DB-RLA-083
Catch-up waiting respetará cancellation.

## DB-RLA-084
Catch-up waiting respetará outer deadline.

## DB-RLA-085
Catch-up coordinator será distinto del router.

## DB-RLA-086
Writer fallback podrá preferirse a waiting mediante policy.

## DB-RLA-087
Core no impondrá wait-vs-writer universalmente.

## DB-RLA-088
Active write transaction bypassará normal replica lag routing.

## DB-RLA-089
Transaction pinning tendrá precedencia.

## DB-RLA-090
Read-only transaction no migrará de replica por lag change.

## DB-RLA-091
Replica freshness no alterará transaction isolation.

## DB-RLA-092
REPEATABLE_READ sobre stale replica podrá observar datos viejos.

## DB-RLA-093
IdentityMap no será replica freshness mechanism.

## DB-RLA-094
IdentityMap hit podrá evitar lag-aware DB routing.

## DB-RLA-095
Explicit refresh deberá declarar routing/consistency semantics.

## DB-RLA-096
Query cache freshness será distinta de replica freshness.

## DB-RLA-097
Cache hit no demostrará replica freshness.

## DB-RLA-098
Sticky routing podrá consumir position evidence.

## DB-RLA-099
Sticky duration no probará replica catch-up.

## DB-RLA-100
Topology generation mismatch será detectable.

## DB-RLA-101
Old-generation lag evidence no se aplicará ciegamente a new generation.

## DB-RLA-102
Replication epoch changes podrán invalidar position evidence.

## DB-RLA-103
Cross-epoch freshness no se asumirá.

## DB-RLA-104
Failover podrá invalidar lag snapshots.

## DB-RLA-105
Load balancing consumirá solo eligible replicas.

## DB-RLA-106
Critical reads podrán exigir authoritative source.

## DB-RLA-107
Framework no inferirá critical consistency por table name.

## DB-RLA-108
Consistency metadata deberá ser explícita.

## DB-RLA-109
Lag collection podrá ser periodic.

## DB-RLA-110
Lag collection podrá ser event-driven.

## DB-RLA-111
Lag collection podrá ser on-demand.

## DB-RLA-112
Lag collection podrá ser hybrid.

## DB-RLA-113
On-demand measurement será bounded.

## DB-RLA-114
Concurrent requests no deberán causar probe amplification.

## DB-RLA-115
Single-flight measurement será soportable.

## DB-RLA-116
Measurement throttling será soportable.

## DB-RLA-117
Monitoring backpressure no deberá saturar database infrastructure.

## DB-RLA-118
Resource budgets serán explícitos.

## DB-RLA-119
Observation retry será distinto de query retry.

## DB-RLA-120
Observation retry será distinto de transaction retry.

## DB-RLA-121
Observation retries serán bounded.

## DB-RLA-122
Telemetry será observational.

## DB-RLA-123
Telemetry listener no mutará published lag snapshot.

## DB-RLA-124
UNKNOWN lag no se reportará como zero en metrics.

## DB-RLA-125
Metrics evitarán high-cardinality dimensions por default.

## DB-RLA-126
Diagnostics distinguirán lag y evidence age.

## DB-RLA-127
Diagnostics distinguirán confidence.

## DB-RLA-128
Diagnostics distinguirán SATISFIED, UNSATISFIED y UNKNOWN.

## DB-RLA-129
Diagnostics podrán explicar routing rejection.

## DB-RLA-130
Monitoring credentials seguirán least privilege.

## DB-RLA-131
Monitoring credentials no estarán en lag snapshots.

## DB-RLA-132
Provider secrets no estarán en routing evidence.

## DB-RLA-133
Provider lag values serán validados.

## DB-RLA-134
Lag parsing será no ambiguo.

## DB-RLA-135
Replication positions permanecerán opaque fuera de adapters.

## DB-RLA-136
Persistent runtime podrá compartir immutable lag snapshots.

## DB-RLA-137
Scope-local requirements no se compartirán.

## DB-RLA-138
No existirá mutable global max-lag por request.

## DB-RLA-139
Snapshot publication será concurrency-safe.

## DB-RLA-140
Old snapshot no sobrescribirá new snapshot.

## DB-RLA-141
Lag snapshot history será bounded.

## DB-RLA-142
Request consistency state se limpiará al finalizar scope.

## DB-RLA-143
Coroutine consistency state estará aislado.

## DB-RLA-144
Cancelled catch-up wait limpiará sus recursos.

## DB-RLA-145
ReplicaLagThresholdExceeded no significará replica failure.

## DB-RLA-146
Known stale será distinto de unknown freshness.

## DB-RLA-147
Lag subsystem no ejecutará application queries.

## DB-RLA-148
Lag subsystem no promoverá replicas.

## DB-RLA-149
Lag subsystem no implementará failover.

## DB-RLA-150
Lag subsystem no implementará load balancing.

## DB-RLA-151
Lag subsystem no cambiará transaction state.

## DB-RLA-152
Lag subsystem no garantizará distributed consistency.

## DB-RLA-153
Lag subsystem no convertirá asynchronous replication en synchronous semantics.

## DB-RLA-154
Lag estimate nunca será presentado como certeza mayor que su source/confidence.

## DB-RLA-155
Freshness decisions deberán estar sustentadas por evidence explícita.

## DB-RLA-156
Evidence insufficient deberá permanecer UNKNOWN.

## DB-RLA-157
Routing fallback deberá ser explícito.

## DB-RLA-158
Fallback a writer no significará replica failure.

## DB-RLA-159
Required position deberá pertenecer al replication domain correcto.

## DB-RLA-160
Required position deberá pertenecer a epoch compatible.

## DB-RLA-161
Commit UNKNOWN no producirá required position confiable.

## DB-RLA-162
Rollback no producirá write freshness requirement.

## DB-RLA-163
Only confirmed write outcome podrá alimentar strong read-after-write evidence.

## DB-RLA-164
Measurement timestamps no sustituirán replication coordinates.

## DB-RLA-165
Replication coordinates no sustituirán health evidence.

## DB-RLA-166
Health evidence no sustituirá lag evidence.

## DB-RLA-167
Lag evidence no sustituirá topology evidence.

## DB-RLA-168
Topology evidence no sustituirá transaction guarantees.

## DB-RLA-169
Freshness evaluation deberá ser deterministic para mismo snapshot/policy.

## DB-RLA-170
Published lag snapshots serán immutable.

## DB-RLA-171
No se expondrán mutable observation maps.

## DB-RLA-172
No se retendrán application entities en lag subsystem.

## DB-RLA-173
No se retendrán query results en lag snapshots.

## DB-RLA-174
No se retendrán TransactionContext objects en shared lag state.

## DB-RLA-175
VoltStack nunca afirmará que una replica satisface una lectura cuando la evidencia disponible no pueda demostrar la garantía requerida.

---

# 216. Modelo formal

Sea:

```text
R = replica
L(R,t) = evidencia de lag observada para R en tiempo t
F = freshness requirement
```

La elegibilidad de frescura será:

```text
FreshnessEligible(R, F)
=
BaseReplicaEligible(R)
∧
EvidenceUsable(L(R,t))
∧
RequirementSatisfied(L(R,t), F)
```

Para bounded staleness:

```text
RequirementSatisfied
=
ObservedLag <= MaxStaleness
∧
EvidenceAge <= MaxEvidenceAge
∧
Confidence >= MinimumConfidence
```

Para required position:

```text
RequirementSatisfied
=
Compare(
    ReplicaCoordinate,
    RequiredCoordinate
) ∈ {EQUAL, AFTER}
```

Si la comparación es:

```text
UNKNOWN
INCOMPARABLE
CONCURRENT
```

no se demostrará la garantía requerida.

---

# 217. Modelo de read-after-write

```text
              WRITE
                │
                ▼
         Authoritative Node
                │
                ▼
         COMMIT confirmed
                │
                ▼
     Write Observation = P
                │
                ▼
          Later READ
                │
                ▼
     SESSION consistency
                │
                ▼
     Required Position = P
                │
        ┌───────┴────────┐
        ▼                ▼
Replica A < P        Replica B >= P
        │                │
        ▼                ▼
     REJECT            ACCEPT
        │                │
        └───────┬────────┘
                ▼
             Router
```

Si ninguna replica demuestra `>= P`:

```text
fallback → writer
```

o:

```text
bounded catch-up wait
```

según policy.

---

# 218. Modelo de bounded staleness

```text
Read Requirement:
    MAX_STALENESS = 2s

Replica A:
    lag = 100ms
    evidence = current
    → eligible

Replica B:
    lag = 1.5s
    evidence = current
    → eligible

Replica C:
    lag = 8s
    → reject

Replica D:
    lag = unknown
    → unknown/reject under strict policy
```

Después:

```text
{A, B}
   │
   ▼
Load Balancer
```

Nunca:

```text
{A, B, C, D}
→ load balancer
→ hope selected replica is fresh
```

---

# 219. Resultado arquitectónico

Con:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
```

VoltStack dispone de la cadena:

```text
Physical/Logical Connections
          │
          ▼
Connection Roles
          │
          ▼
Replication Topology
          │
          ▼
Replica State
          │
          ▼
Replication Progress
          │
          ▼
Lag/Freshness Evidence
          │
          ▼
Consistency Requirement
          │
          ▼
Replica Eligibility
          │
          ▼
Read/Write Routing
```

Esto permite evitar uno de los errores más comunes de los sistemas read/write split:

```text
SELECT
→ replica
```

y sustituirlo por:

```text
READ
+
Execution Domain
+
Transaction State
+
Consistency Requirement
+
Replica State
+
Lag/Freshness Evidence
+
Routing Policy
        │
        ▼
Eligible Endpoint
```

---

# 220. Regla maestra final

> **El Replica Lag Awareness System será una capa de evidencia, no una fábrica de garantías. VoltStack podrá afirmar que una replica satisface un requisito de frescura únicamente cuando la evidencia disponible, su antigüedad, confianza, topología y semántica de comparación permitan demostrarlo; en cualquier otro caso preservará `UNKNOWN`, rechazará la replica cuando la garantía sea obligatoria o aplicará un fallback explícitamente configurado.**

En forma compacta:

```text
Observation
    +
Freshness
    +
Confidence
    +
Topology
    +
Requirement
      │
      ▼
Evidence-Based Eligibility
```

Nunca:

```text
Replica responds
      ↓
Assume current
```

---

# 221. Siguiente documento

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

El siguiente documento definirá el sistema mediante el cual VoltStack preservará **read-after-write consistency** manteniendo temporalmente determinadas lecturas sobre una fuente autoritativa cuando las replicas todavía no puedan demostrar suficiente frescura:

```text
WRITE
  │
  ▼
Commit confirmed
  │
  ▼
Write Observation
  │
  ▼
Sticky State
  │
  ├── Time Window
  ├── Required Position
  ├── Scope
  ├── Domain
  └── Expiration
          │
          ▼
Later READ
          │
          ▼
Can replica satisfy required freshness?
      │                 │
     NO                YES
      │                 │
      ▼                 ▼
   Writer             Replica
```

manteniendo la distinción:

```text
Sticky Connection
≠
Transaction Pinning
≠
Connection Pooling
≠
Session Affinity
≠
Replica Lag
```

y evitando que estado sticky mutable se filtre entre requests, jobs, Fibers, coroutines o workers persistentes.