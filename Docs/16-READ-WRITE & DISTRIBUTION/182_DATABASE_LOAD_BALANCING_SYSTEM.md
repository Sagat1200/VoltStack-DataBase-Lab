# 182_DATABASE_LOAD_BALANCING_SYSTEM.md

# VoltStack Quantum Database
## Database Load Balancing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 182 — Database Load Balancing System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `181_DATABASE_FAILOVER_SYSTEM.md`  
**Siguiente documento:** `183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md`

---

# 1. Propósito

`Database Load Balancing System` define la arquitectura mediante la cual VoltStack distribuirá operaciones entre múltiples endpoints de base de datos **ya considerados elegibles** por las capas anteriores de routing, consistencia, replicación, sticky semantics y failover.

Los documentos anteriores establecieron la secuencia:

```text
176 Read/Write Connection System
    ↓
177 Read/Write Routing System
    ↓
178 Replica System
    ↓
179 Replica Lag Awareness System
    ↓
180 Sticky Connection System
    ↓
181 Failover System
```

Este documento añade:

```text
Eligible Candidate Set
        │
        ▼
Load Balancing Context
        │
        ▼
Load Balancing Policy
        │
        ▼
Candidate Metrics
        │
        ▼
Ranking / Selection
        │
        ▼
Selected Endpoint
```

La regla central será:

> **El Load Balancing System de VoltStack nunca decidirá si un endpoint es seguro, consistente o autoritativo; solo elegirá entre candidatos que ya hayan superado las restricciones de dominio, rol, transacción, salud, frescura, sticky consistency y failover.**

Formalmente:

```text
LoadBalancer(
    EligibleCandidates
)
→
One Selected Candidate
```

Nunca:

```text
LoadBalancer(
    All Known Endpoints
)
→
Hope Selected Endpoint Is Safe
```

---

# 2. Principio arquitectónico

La separación fundamental será:

```text
Eligibility
≠
Ranking
≠
Selection
```

La arquitectura completa:

```text
All Candidates
      │
      ▼
Hard Constraints
      │
      ▼
Eligible Candidates
      │
      ▼
Soft Preferences
      │
      ▼
Ranked Candidates
      │
      ▼
Load Balancer
      │
      ▼
Selected Endpoint
```

---

# 3. Objetivos

El sistema deberá proporcionar:

1. selección entre múltiples candidatos elegibles;
2. round-robin;
3. weighted round-robin;
4. random weighted;
5. least-load;
6. least-connections;
7. latency-aware;
8. locality-aware;
9. health-aware weighting;
10. replica freshness-aware preference;
11. adaptive balancing;
12. deterministic policy composition;
13. endpoint weights;
14. runtime metrics;
15. bounded state;
16. graceful degradation;
17. connection-pool awareness;
18. routing affinity integration;
19. failover integration;
20. sticky integration;
21. multi-region readiness;
22. observability;
23. persistent-runtime safety;
24. extensibilidad;
25. testing.

---

# 4. No objetivos

El sistema no deberá:

- determinar qué endpoints son replicas;
- medir replica lag;
- decidir write authority;
- promover nodos;
- ejecutar failover;
- cambiar shard;
- cambiar tenant;
- alterar transaction pinning;
- decidir consistency requirements;
- ejecutar queries;
- adquirir directamente conexiones físicas;
- sustituir Connection Manager.

---

# 5. Distinciones fundamentales

```text
Load Balancing
≠
Routing
≠
Failover
≠
Replica Eligibility
≠
Health Checking
≠
Connection Pooling
≠
Sharding
```

---

# 6. Routing vs load balancing

`ReadWriteRoutingSystem` responde:

> ¿Qué endpoints pueden ejecutar correctamente esta operación?

`LoadBalancingSystem` responde:

> Entre esos endpoints, ¿cuál conviene seleccionar?

---

# 7. Failover vs load balancing

Failover cambia:

```text
effective topology / authority
```

Load balancing cambia únicamente:

```text
which eligible endpoint receives this operation
```

---

# 8. Sharding vs load balancing

Sharding decide:

```text
which data partition/shard
```

Load balancing decide:

```text
which equivalent endpoint serving that shard
```

---

# 9. Arquitectura general

```text
Routing Engine
    │
    ▼
EligibleCandidateSet
    │
    ▼
LoadBalancingContext
    │
    ▼
LoadBalancingPolicyResolver
    │
    ▼
Candidate Metric Snapshot
    │
    ▼
Candidate Scoring / Ordering
    │
    ▼
Selection Algorithm
    │
    ▼
LoadBalancingDecision
    │
    ▼
Connection Manager
```

---

# 10. LoadBalancingContext

```php
final readonly class LoadBalancingContext
{
    public function __construct(
        public LoadBalancingOperationId $operationId,
        public DatabaseExecutionDomain $domain,
        public ConnectionIntent $intent,
        public ReadConsistencyRequirement $consistency,
        public ConnectionAffinity $affinity,
        public RoutingDecisionReason $routingReason,
        public LoadBalancingPolicy $policy,
        public LoadMetricsSnapshot $metrics,
    ) {}
}
```

---

# 11. Context immutability

`LoadBalancingContext` será immutable.

No deberá:

- ejecutar probes;
- abrir sockets;
- mutar pools;
- cambiar topology;
- modificar sticky state.

---

# 12. LoadBalancingOperationId

Identidad independiente:

```text
LoadBalancingOperationId
≠
RoutingOperationId
≠
TransactionId
```

aunque puedan correlacionarse.

---

# 13. LoadBalancingDecision

```php
final readonly class LoadBalancingDecision
{
    public function __construct(
        public EndpointId $selected,
        public LoadBalancingStrategy $strategy,
        public LoadBalancingDecisionReason $reason,
        public LoadBalancingDecisionMetadata $metadata,
    ) {}
}
```

---

# 14. Decision ≠ connection

```text
LoadBalancingDecision
≠
ConnectionLease
```

---

# 15. Candidate set

El input deberá ser:

```text
EligibleCandidateSet
```

no:

```text
AllTopologyEndpoints
```

---

# 16. Eligibility invariant

Todo candidato recibido ya deberá haber satisfecho:

```text
execution domain
role
transaction
consistency
sticky
replica state
lag/freshness
failover state
health hard constraints
```

---

# 17. Candidate revalidation

El balancer podrá validar defensivamente que:

```text
candidate ∈ eligible set
```

pero no reconstruirá toda la lógica de routing.

---

# 18. LoadBalancingStrategy

```php
enum LoadBalancingStrategy
{
    case FIRST;
    case ROUND_ROBIN;
    case RANDOM;
    case WEIGHTED_RANDOM;
    case WEIGHTED_ROUND_ROBIN;
    case LEAST_CONNECTIONS;
    case LEAST_LOAD;
    case LATENCY_AWARE;
    case LOCALITY_AWARE;
    case ADAPTIVE;
    case CUSTOM;
}
```

---

# 19. FIRST

Selecciona el primer candidato ordenado.

Útil para:

- tests;
- configuraciones simples;
- deterministic fallback.

No recomendado como estrategia general en grupos grandes.

---

# 20. ROUND_ROBIN

Distribuye solicitudes secuencialmente:

```text
A
B
C
A
B
C
...
```

---

# 21. Round-robin state

Requiere un cursor:

```text
nextIndex
```

que deberá ser:

```text
group-scoped/shared-safe
```

cuando se use entre requests.

---

# 22. Round-robin concurrency

Un contador mutable compartido deberá ser concurrency-safe.

No:

```php
static $i++;
```

sin garantías.

---

# 23. RANDOM

Selecciona uniformemente entre candidatos.

Ventajas:

- stateless;
- simple;
- evita coordinación de cursor.

Desventajas:

- distribución imperfecta en ventanas pequeñas.

---

# 24. WEIGHTED_RANDOM

Cada endpoint tiene peso:

```text
A = 50
B = 30
C = 20
```

Probabilidad aproximada:

```text
A → 50%
B → 30%
C → 20%
```

---

# 25. Weight ≠ percentage

El peso deberá interpretarse relativamente.

```text
1,1,1
```

es equivalente a:

```text
10,10,10
```

para selección proporcional.

---

# 26. Invalid weights

Pesos:

```text
negative
NaN
infinite
```

serán inválidos.

---

# 27. Zero weight

Podrá significar:

```text
do not select normally
```

pero el endpoint puede seguir siendo técnicamente elegible.

---

# 28. Weighted round-robin

Proporcionará distribución determinista aproximada según pesos.

Ejemplo:

```text
A weight 3
B weight 1
```

secuencia conceptual:

```text
A
A
A
B
A
A
A
B
```

---

# 29. Dynamic weights

Los pesos podrán derivarse de:

```text
capacity
health
load
latency
administrative priority
```

pero deberán diferenciarse de:

```text
hard eligibility
```

---

# 30. LEAST_CONNECTIONS

Selecciona el candidato con menor número de conexiones activas relevantes.

---

# 31. Active connection metric

```php
final readonly class ActiveConnectionCount
{
    public function __construct(
        public int $value,
        public Instant $observedAt,
    ) {}
}
```

---

# 32. Connection count ≠ actual database load

Un endpoint con:

```text
5 connections
```

puede estar más cargado que otro con:

```text
50 idle connections
```

---

# 33. LEAST_LOAD

Utiliza un `LoadSignal` más rico.

---

# 34. LoadSignal

```php
final readonly class EndpointLoadSignal
{
    public function __construct(
        public EndpointId $endpoint,
        public LoadLevel $level,
        public LoadConfidence $confidence,
        public Instant $observedAt,
    ) {}
}
```

---

# 35. LoadLevel

```php
enum LoadLevel
{
    case IDLE;
    case LOW;
    case MODERATE;
    case HIGH;
    case SATURATED;
    case UNKNOWN;
}
```

---

# 36. Load signal source

Podrá provenir de:

- pool utilization;
- query concurrency;
- DB telemetry;
- provider metrics;
- CPU;
- connection utilization;
- queue depth;
- application-side observations.

---

# 37. Load ≠ health

```text
HIGH_LOAD
≠
UNHEALTHY
```

Un nodo puede estar saludable pero ocupado.

---

# 38. SATURATED

Puede ser:

```text
soft rejection
```

o hard constraint según policy.

---

# 39. LATENCY_AWARE

Utiliza observaciones de latencia.

---

# 40. Latency types

Deben distinguirse:

```text
connection establishment latency
network round-trip latency
query execution latency
application-observed request latency
```

---

# 41. No mixed latency without semantics

No combinar arbitrariamente todas como una sola métrica.

---

# 42. EndpointLatencyProfile

```php
final readonly class EndpointLatencyProfile
{
    public function __construct(
        public Duration $network,
        public ?Duration $connection,
        public ?Duration $query,
        public Instant $observedAt,
        public MetricConfidence $confidence,
    ) {}
}
```

---

# 43. Latency freshness

Una medición vieja no deberá tratarse como actual.

---

# 44. Moving averages

Puede utilizarse:

```text
EWMA
```

para suavizar observaciones.

---

# 45. EWMA

Conceptualmente:

```text
Sₜ = αXₜ + (1-α)Sₜ₋₁
```

con `α` configurable.

---

# 46. No universal α

El framework no impondrá un único valor óptimo.

---

# 47. Outliers

Una única query lenta no deberá destruir necesariamente el score.

Policies podrán utilizar:

- EWMA;
- percentile;
- bounded window;
- outlier rejection.

---

# 48. LOCALITY_AWARE

Permite preferir candidatos cercanos al execution environment.

---

# 49. Locality dimensions

```text
same host
same zone
same region
same geography
same network domain
```

---

# 50. Locality ≠ correctness

Locality será:

```text
soft preference
```

después de safety/consistency.

---

# 51. Locality metadata

Podrá provenir de:

```text
EndpointMetadata
RuntimeLocationContext
```

---

# 52. RuntimeLocationContext

```php
final readonly class RuntimeLocationContext
{
    public function __construct(
        public ?RegionId $region,
        public ?ZoneId $zone,
    ) {}
}
```

---

# 53. No GPS semantics

Esto describe deployment locality, no ubicación personal del usuario.

---

# 54. ADAPTIVE strategy

Podrá combinar métricas dinámicas.

---

# 55. Adaptive strategy caution

No deberá convertirse en una caja negra impredecible.

Toda decisión deberá ser explicable.

---

# 56. Adaptive input

Ejemplo:

```text
capacity
connections
latency
load
error rate
replica freshness class
```

---

# 57. Hard vs soft metrics

`ReplicaFreshnessRequirement` ya se resolvió antes.

Aquí freshness puede utilizarse únicamente como:

```text
preference among already-valid replicas
```

---

# 58. Example

Requirement:

```text
lag <= 2s
```

Replicas:

```text
A = 50ms
B = 500ms
C = 1.5s
```

Todas son elegibles.

El balancer puede preferir A o B.

---

# 59. But load can matter

Si:

```text
A = 95% load
B = 30% load
```

B puede ser mejor.

---

# 60. Policy model

```php
final readonly class LoadBalancingPolicy
{
    public function __construct(
        public LoadBalancingStrategy $strategy,
        public EndpointWeightPolicy $weights,
        public LoadBalancingFallbackPolicy $fallback,
        public MetricFreshnessPolicy $metricFreshness,
        public LoadBalancingResourceBudget $budget,
    ) {}
}
```

---

# 61. Strategy per role

Puede configurarse:

```text
READ:
    adaptive

WRITE:
    first/authority

ANALYTICS:
    weighted
```

---

# 62. Write load balancing

Si existe más de un endpoint WRITE-capable:

eso no significa que puedan recibir escrituras arbitrariamente.

---

# 63. Authority constraint wins

Solo endpoints que el routing/failover layer considere:

```text
valid write authorities
```

entrarán al balancer.

---

# 64. Multi-primary

En una arquitectura realmente multi-writer, el sistema distribuido posterior deberá definir su semántica.

El balancer no inventará multi-primary support.

---

# 65. Read balancing primary use case

El principal uso será:

```text
multiple eligible replicas
```

---

# 66. Candidate weight

Modelo:

```php
final readonly class EndpointWeight
{
    public function __construct(
        public EndpointId $endpoint,
        public float $value,
    ) {}
}
```

---

# 67. Weight normalization

Los weights podrán normalizarse internamente:

```text
NormalizedWeightᵢ
=
Weightᵢ / ΣWeights
```

---

# 68. Weight zero set

Si todos los pesos son cero:

```text
selection failure
```

o fallback explícito.

No random undefined behavior.

---

# 69. Static weight

Configuración:

```php
'readers' => [
    ['host' => 'r1', 'weight' => 5],
    ['host' => 'r2', 'weight' => 3],
    ['host' => 'r3', 'weight' => 2],
],
```

---

# 70. Dynamic weight

Puede calcularse:

```text
EffectiveWeight
=
BaseWeight
×
HealthFactor
×
LoadFactor
×
LatencyFactor
```

solo si todos los factores tienen semántica definida.

---

# 71. Avoid magic formula

No deberá existir una fórmula opaca hardcoded universal.

---

# 72. WeightFactor

```php
interface EndpointWeightFactor
{
    public function factor(
        LoadBalancingCandidate $candidate,
        LoadBalancingContext $context,
    ): WeightFactorResult;
}
```

---

# 73. WeightFactorResult

```php
final readonly class WeightFactorResult
{
    public function __construct(
        public float $factor,
        public WeightFactorReason $reason,
    ) {}
}
```

---

# 74. Factor bounds

Podrá normalizarse, por ejemplo:

```text
0.0 ≤ factor ≤ 1.0
```

según implementación.

---

# 75. Unknown metric factor

Si no existe una métrica:

```text
UNKNOWN
```

deberá tratarse según policy.

No:

```text
unknown load = zero load
```

---

# 76. Graceful degradation

Si dynamic metrics no están disponibles:

```text
ADAPTIVE
→ weighted static
→ round robin
```

solo si fallback está explícitamente permitido.

---

# 77. No silent algorithm change

Requested:

```text
least_load
```

pero no existen load metrics.

No pasar silenciosamente a random sin dejar evidencia.

---

# 78. StrategyResolution

```php
final readonly class LoadBalancingStrategyResolution
{
    public function __construct(
        public LoadBalancingStrategy $requested,
        public LoadBalancingStrategy $effective,
        public StrategyResolutionStatus $status,
        public array $reasons,
    ) {}
}
```

---

# 79. StrategyResolutionStatus

```php
enum StrategyResolutionStatus
{
    case EXACT;
    case FALLBACK;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 80. Connection pool awareness

El balancer podrá consumir:

```text
pool utilization
available leases
wait queue size
```

---

# 81. Pool-aware ≠ pool manager

El balancer observa métricas.

No adquiere/libera leases.

---

# 82. Pool saturation

Un endpoint:

```text
healthy
fresh
eligible
```

puede tener pool saturado.

---

# 83. Saturation preference

Puede reducir su weight.

---

# 84. Pool unavailable

Si no puede adquirir lease:

```text
ConnectionManager
```

reporta acquisition failure.

---

# 85. Acquisition failure reroute

Antes de ejecutar una query, puede ser seguro:

```text
candidate A selected
↓
lease acquisition fails
↓
reroute among remaining eligible candidates
```

según policy.

---

# 86. Reroute budget

Debe ser bounded.

---

# 87. Candidate exclusion on acquisition failure

Dentro del mismo operation:

```text
failed candidate
```

podrá quedar temporalmente excluido.

---

# 88. No global health mutation from one failure

Una única acquisition failure no deberá marcar automáticamente endpoint global como unhealthy.

Eso corresponde al Health/Failover System.

---

# 89. Retry distinction

```text
LoadBalancer Reselect
≠
Query Retry
≠
Transaction Retry
≠
Failover
```

---

# 90. Transaction behavior

Dentro de active transaction:

```text
Load Balancing
```

normalmente no participa.

---

# 91. Why

La transaction ya posee:

```text
PinnedConnectionLease
```

---

# 92. Transaction invariant

```text
ActiveTransaction
→ exact pinned connection
```

No seleccionar un endpoint distinto por menor load.

---

# 93. Before transaction begins

Para una nueva transaction:

```text
eligible writer candidates
→ select endpoint
→ acquire lease
→ BEGIN
→ pin
```

---

# 94. Read-only transactions

Si existen varias replicas elegibles:

```text
load balancer
→ choose replica
→ BEGIN
→ pin
```

---

# 95. No rebalance mid-transaction

Aunque otra replica se vuelva más rápida:

```text
do not migrate active transaction
```

---

# 96. Sticky integration

Sticky System filtra candidatos primero.

---

# 97. Example

```text
required position P

A < P → rejected
B >= P → eligible
C >= P → eligible
```

Load balancer recibe:

```text
B, C
```

---

# 98. Sticky preference vs load balancing

Si sticky policy indica:

```text
prefer writer until scope end
```

writer puede recibir mayor preference.

---

# 99. Sticky hard requirement

Si sticky exige:

```text
AUTHORITATIVE
```

las replicas ni siquiera llegan al balancer.

---

# 100. Lag integration

Lag tiene dos roles distintos:

```text
Hard freshness constraint
```

en el Routing/Lag System.

Y:

```text
soft freshness preference
```

en el Load Balancer.

---

# 101. Never confuse roles

No utilizar:

```text
lag weighting
```

para aceptar una replica que viola un max-staleness hard requirement.

---

# 102. Failover integration

Durante failover:

```text
FailoverBarrier
```

puede reducir el candidate set.

---

# 103. Authority transition

En:

```text
WRITE_PAUSED
```

no habrá writer candidates elegibles.

---

# 104. New authority

Una vez verificada:

```text
new authority
```

podrá aparecer en candidate set.

---

# 105. Stale old writer

Nunca deberá seguir seleccionándose por:

```text
lower latency
```

si ya no es elegible.

---

# 106. Replica failure

Replica marcada:

```text
UNHEALTHY
```

debe haber sido eliminada antes.

El load balancer no necesita “penalizarla”.

---

# 107. Degraded replica

Una replica:

```text
DEGRADED but still eligible
```

puede recibir menor weight.

---

# 108. LoadBalancingCandidate

```php
final readonly class LoadBalancingCandidate
{
    public function __construct(
        public EndpointId $endpoint,
        public ConnectionRoleSet $roles,
        public EndpointLoadBalancingMetadata $metadata,
    ) {}
}
```

---

# 109. Metadata

Puede incluir:

```text
baseWeight
region
zone
capacityClass
poolId
```

---

# 110. Dynamic metrics separate

No mezclar immutable metadata con mutable observations.

---

# 111. LoadMetricsSnapshot

```php
final readonly class LoadMetricsSnapshot
{
    public function __construct(
        public LoadMetricsGeneration $generation,
        public EndpointLoadMetricMap $endpoints,
        public Instant $capturedAt,
    ) {}
}
```

---

# 112. Metrics snapshot immutable

Igual que topology/lag snapshots.

---

# 113. LoadMetricsGeneration

Permite diagnosticar con qué evidencia se tomó la decisión.

---

# 114. Metric freshness

Toda métrica deberá tener:

```text
observedAt
```

y opcionalmente:

```text
validUntil
```

---

# 115. Load metric age

```text
metric age
≠
load value
```

---

# 116. Stale load metrics

Policy:

```text
IGNORE
DEGRADE_CONFIDENCE
REJECT_STRATEGY
```

según estrategia.

---

# 117. Metric confidence

```php
enum LoadMetricConfidence
{
    case VERIFIED;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 118. No false precision

Si provider solo dice:

```text
HIGH_LOAD
```

VoltStack no inventará:

```text
83.42%
```

---

# 119. Least-load comparison

Dos candidatos con:

```text
HIGH
```

pueden quedar empatados.

---

# 120. Tie-breaking

Toda estrategia deberá tener una policy de desempate.

---

# 121. TieBreakStrategy

```php
enum TieBreakStrategy
{
    case ROUND_ROBIN;
    case RANDOM;
    case STABLE_ORDER;
    case WEIGHTED;
}
```

---

# 122. Stable order

Útil para deterministic tests.

---

# 123. Random tie-break

Puede reducir hot-spotting.

---

# 124. Adaptive feedback

El sistema podrá observar outcomes posteriores para mejorar dynamic metrics.

---

# 125. Outcome feedback

Ejemplos:

```text
connection acquisition latency
query latency
timeout
successful completion
```

---

# 126. Feedback ≠ eligibility mutation

El balancer no deberá decidir:

```text
endpoint permanently unhealthy
```

por sí solo.

Puede producir:

```text
LoadBalancingObservation
```

para otros subsistemas.

---

# 127. Error-rate aware balancing

Puede reducir weight ante:

```text
elevated transient failure rate
```

sin declararlo failover.

---

# 128. Circuit breaker integration

Si Circuit Breaker está:

```text
OPEN
```

ese endpoint debe ser ineligible antes del balancer.

---

# 129. HALF_OPEN

Puede haber una policy de tráfico limitado.

---

# 130. Half-open probing

Debe pertenecer al resilience/circuit-breaker layer.

No al random balancer.

---

# 131. Capacity classes

Endpoints pueden declarar:

```text
SMALL
MEDIUM
LARGE
XL
```

o capacidad relativa.

---

# 132. CapacityWeightPolicy

Podrá mapear:

```text
SMALL → 1
MEDIUM → 2
LARGE → 4
```

según configuración.

---

# 133. Capacity ≠ current load

Una replica grande puede seguir saturada.

---

# 134. Locality-aware policy

Ejemplo:

```text
same zone
↓
same region
↓
remote region
```

como preference chain.

---

# 135. Zone outage

Failover/health constraints eliminarán endpoints afectados.

Load balancer no necesita deducir outage.

---

# 136. Cross-region cost

Policies podrán considerar:

```text
network cost
```

como soft factor.

---

# 137. Cost-aware balancing

Podrá existir como custom strategy.

---

# 138. Cost does not override consistency

Siempre:

```text
Consistency > Cost
```

---

# 139. Read traffic shaping

Una policy podrá limitar cuánto tráfico va al writer.

Ejemplo:

```text
prefer replicas
writer weight = 0.1
```

solo si writer fallback es elegible.

---

# 140. Writer protection

Puede ser útil para reservar capacidad de escritura.

---

# 141. But hard sticky semantics win

Si sticky exige writer:

```text
writer traffic shaping
```

no podrá impedir la operación si el writer es el único endpoint consistente.

---

# 142. Fair distribution

Load balancing buscará evitar hot spots.

---

# 143. Fairness ≠ equal distribution

Con weights:

```text
fair
```

puede significar proporcional a capacidad.

---

# 144. Short-term imbalance

Random/weighted algorithms pueden mostrar desbalance temporal.

No implica necesariamente fallo.

---

# 145. Long-term distribution

Telemetry podrá evaluar distribución observada.

---

# 146. Selection history

Podrá mantenerse bounded history para diagnostics.

Nunca:

```text
every decision forever
```

---

# 147. Selection counters

Per-endpoint counters pueden utilizarse internamente.

---

# 148. Cardinality control

En clusters con miles de replicas, per-endpoint telemetry deberá estar gobernada.

---

# 149. LoadBalancingState

Algunas estrategias requieren mutable state:

- round-robin cursor;
- smooth weighted counters;
- moving metrics;
- local penalties.

---

# 150. State ownership

Ese estado deberá pertenecer a:

```text
LoadBalancingGroupState
```

no al request.

---

# 151. LoadBalancingGroupId

```php
final readonly class LoadBalancingGroupId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 152. Group key

Puede derivarse de:

```text
ExecutionDomain
+
RoleGroup
+
TopologyGeneration
```

---

# 153. Topology generation change

El state del balancer deberá reconciliarse.

Ejemplo:

```text
A,B,C
```

pasa a:

```text
A,C,D
```

---

# 154. Stale cursor

No deberá producir endpoint B removido.

---

# 155. State reconciliation

```text
old group state
+
new candidate identity set
→ reconciled group state
```

---

# 156. Weight changes

Smooth weighted algorithms deberán adaptarse sin unbounded drift.

---

# 157. Persistent workers

Shared group state puede vivir más allá de una request.

Debe ser:

- bounded;
- concurrency-safe;
- generation-aware;
- resettable.

---

# 158. Multi-process limitation

Un round-robin cursor local por worker no produce round-robin global.

---

# 159. Important distinction

```text
Process-local fairness
≠
Cluster-global fairness
```

---

# 160. Global balancing

Si se necesita distribución global exacta puede requerirse:

```text
external proxy
service mesh
database proxy
central coordinator
```

---

# 161. Framework default

VoltStack no deberá introducir coordinación distribuida solo para obtener round-robin perfecto.

---

# 162. Statistical balancing

Process-local randomized/weighted strategies pueden ser suficientes en muchos deployments.

---

# 163. Recommended default

Para múltiples read replicas:

```text
WEIGHTED_RANDOM
```

o:

```text
ROUND_ROBIN
```

con posibilidad futura de adaptive policies.

Una recomendación concreta dependerá del deployment, pero el core debe mantener ambas.

---

# 164. Persistent runtime safety

En FrankenPHP:

```text
immutable policy
+
bounded shared strategy state
+
scope-local decision context
```

---

# 165. RoadRunner

Misma arquitectura.

---

# 166. OpenSwoole

Mutable balancing state compartido entre coroutines deberá ser concurrency-safe.

---

# 167. No request leakage

Request A puede seleccionar:

```text
Replica B
```

pero Request B no debe quedar forzado a B salvo affinity explícita.

---

# 168. Affinity

Connection affinity puede restringir o preferir un endpoint.

---

# 169. Required affinity

Si:

```text
AffinityStrength = REQUIRE
```

y endpoint sigue siendo elegible:

load balancing se omite.

---

# 170. Preferred affinity

Si:

```text
PREFER
```

podrá aplicar un weight boost.

---

# 171. Affinity boost

Debe ser explícito y bounded.

---

# 172. No affinity resurrection

Un endpoint ineligible no podrá volver al set porque affinity lo prefiere.

---

# 173. Load balancing policy registry

```php
interface LoadBalancingPolicyRegistry
{
    public function resolve(
        LoadBalancingPolicyId $id,
    ): CompiledLoadBalancingPolicy;
}
```

---

# 174. Registry lifecycle

```text
collect
↓
validate
↓
compile
↓
freeze
```

---

# 175. Custom strategy

```php
interface CustomLoadBalancingStrategy
{
    public function select(
        EligibleCandidateSet $candidates,
        LoadBalancingContext $context,
    ): EndpointId;
}
```

---

# 176. Custom strategy safety

El resultado deberá validarse:

```text
selected endpoint ∈ eligible candidate set
```

---

# 177. No arbitrary endpoint injection

Un custom strategy no podrá devolver endpoints externos al set.

---

# 178. Strategy sandbox boundaries

No deberá:

- mutar transaction;
- cambiar tenant;
- consultar ORM;
- ejecutar SQL;
- promover replicas;
- modificar sticky context.

---

# 179. Determinism

Estrategias deterministas deberán producir mismo resultado para:

```text
same candidates
same metrics
same state
same policy
```

---

# 180. Random strategies

La variabilidad será intencional.

Para tests deberá poder inyectarse:

```text
RandomSource
```

determinista.

---

# 181. RandomSource

```php
interface RandomSource
{
    public function nextFloat(): float;
}
```

---

# 182. Cryptographic randomness unnecessary

Load balancing no necesita por default RNG criptográfico.

---

# 183. Test reproducibility

Seeded random source permitirá:

```text
reproducible selection tests
```

---

# 184. Time abstraction

Latency/load algorithms deberán recibir:

```text
Clock
```

inyectable.

---

# 185. No direct `microtime()` everywhere

Centralizar clocks mejora:

- testing;
- monotonicity;
- portability.

---

# 186. Selection budget

```php
final readonly class LoadBalancingResourceBudget
{
    public function __construct(
        public int $maxCandidates,
        public int $maxFactors,
        public int $maxReselections,
        public Duration $maxDecisionTime,
    ) {}
}
```

---

# 187. Candidate bound

Un pathological candidate set no deberá causar trabajo sin límites.

---

# 188. Algorithm complexity

Idealmente estrategias base:

```text
O(N)
```

o mejor.

---

# 189. Sorting

Si una estrategia requiere ranking completo:

```text
O(N log N)
```

puede ser aceptable con bounded `N`.

---

# 190. Hot-path reflection

Deberá evitarse.

Policies/strategies compiladas en boot.

---

# 191. No network I/O in selection

El balancer no hará:

```text
ping every replica
```

antes de cada decisión.

---

# 192. Snapshot-driven selection

Consumirá:

```text
LoadMetricsSnapshot
RoutingEvidenceSnapshot
```

---

# 193. Metric collection

Un componente separado:

```php
interface EndpointLoadMetricsCollector
{
    public function collect(): LoadMetricsSnapshot;
}
```

---

# 194. Collection frequency

Será policy-driven.

---

# 195. Probe amplification prevention

Similar al Lag System:

```text
single-flight
throttling
bounded concurrency
```

---

# 196. Metric collector failure

No deberá tumbar necesariamente queries.

Puede degradar strategy.

---

# 197. Metric fallback example

Requested:

```text
LEAST_LOAD
```

metrics unavailable.

Explicit policy:

```text
fallback → WEIGHTED_RANDOM
```

---

# 198. No fallback configured

Resultado:

```text
LoadBalancingMetricsUnavailableException
```

o equivalent.

---

# 199. Selection failure

Si existe candidato elegible pero strategy no puede elegir:

```text
LoadBalancingSelectionException
```

---

# 200. Empty candidates

Eso es principalmente un routing error:

```text
RoutingNoEligibleCandidateException
```

No un load-balancing failure.

---

# 201. Telemetry events

```text
LoadBalancingStarted
LoadBalancingStrategyResolved
LoadBalancingCandidateScored
LoadBalancingDecisionMade
LoadBalancingFallbackApplied
LoadBalancingReselection
LoadBalancingFailed
```

---

# 202. Candidate-scored event caution

Puede generar alto volumen.

Deshabilitable/muestreable en producción.

---

# 203. Metrics

Ejemplos:

```text
db.load_balancing.decisions
db.load_balancing.strategy
db.load_balancing.fallbacks
db.load_balancing.reselections
db.load_balancing.selection_duration
db.load_balancing.pool_saturation
db.load_balancing.metric_staleness
```

---

# 204. Distribution metrics

Podrá medirse:

```text
selection distribution by endpoint class
```

con cardinalidad gobernada.

---

# 205. Endpoint IDs

No necesariamente adecuados como metric labels en clusters grandes.

---

# 206. Telemetry dimensions

Bounded:

```text
strategy
role
result
fallback
affinity
metric_source
```

---

# 207. Trace example

```text
db.lb.strategy = weighted_random
db.lb.candidates = 3
db.lb.reselected = false
db.lb.affinity = none
db.lb.metrics_generation = 72
```

---

# 208. Diagnostics API

Conceptualmente:

```php
DB::loadBalancer()->inspect('default');
```

---

# 209. Example diagnostics

```text
DATABASE LOAD BALANCER

Connection:
    default

Role:
    READ

Strategy:
    ADAPTIVE

Effective Strategy:
    LEAST_LOAD

Candidate Count:
    3

Metrics Generation:
    72

Candidates:

    replica-a
        eligible: yes
        base weight: 1
        load: HIGH
        latency: 18ms
        effective preference: LOW

    replica-b
        eligible: yes
        base weight: 1
        load: LOW
        latency: 24ms
        effective preference: HIGH

    replica-c
        eligible: yes
        base weight: 2
        load: MODERATE
        latency: 60ms
        effective preference: MEDIUM

Selected:
    replica-b
```

---

# 210. Explain API

```php
DB::loadBalancer()->explain($routingDecision);
```

---

# 211. Explain result

```text
LOAD BALANCING EXPLANATION

Eligible Candidates:
    replica-a
    replica-b

Excluded Earlier By Routing:
    replica-c
        reason: sticky position not satisfied

Strategy:
    WEIGHTED_RANDOM

Weights:
    replica-a = 1
    replica-b = 3

Selected:
    replica-b

Important:
    replica-c was never considered by the load balancer.
```

---

# 212. This distinction matters

Diagnostics deberán mostrar:

```text
Rejected By Routing
```

separadamente de:

```text
Not Selected By Load Balancer
```

---

# 213. Rejection ≠ non-selection

`Replica A` puede ser perfectamente elegible y simplemente no seleccionada esta vez.

---

# 214. Error hierarchy

```text
DatabaseLoadBalancingException
├── LoadBalancingPolicyException
├── LoadBalancingStrategyUnsupportedException
├── LoadBalancingStrategyResolutionException
├── LoadBalancingMetricsUnavailableException
├── LoadBalancingMetricsStaleException
├── LoadBalancingInvalidWeightException
├── LoadBalancingNoPositiveWeightException
├── LoadBalancingSelectionException
├── LoadBalancingDecisionInvalidException
├── LoadBalancingStateException
├── LoadBalancingStateReconciliationException
├── LoadBalancingBudgetExceededException
├── LoadBalancingReselectionException
└── LoadBalancingInvariantViolationException
```

---

# 215. Testing architecture

Suite propuesta:

```text
LoadBalancingCandidateTests
LoadBalancingPolicyTests
LoadBalancingStrategyTests
RoundRobinTests
WeightedRoundRobinTests
RandomStrategyTests
WeightedRandomTests
LeastConnectionsTests
LeastLoadTests
LatencyAwareTests
LocalityAwareTests
AdaptiveLoadBalancingTests
LoadBalancingMetricTests
LoadBalancingAffinityTests
LoadBalancingStickyIntegrationTests
LoadBalancingLagIntegrationTests
LoadBalancingFailoverIntegrationTests
LoadBalancingPoolTests
LoadBalancingPersistentRuntimeTests
LoadBalancingConcurrencyTests
LoadBalancingTelemetryTests
LoadBalancingDiagnosticsTests
```

---

# 216. Eligibility test

Candidate marcado ineligible por router jamás deberá llegar al strategy.

---

# 217. Round-robin test

Con:

```text
A,B,C
```

esperar:

```text
A,B,C,A,B,C
```

en single-state deterministic test.

---

# 218. Concurrent round-robin test

Múltiples workers/coroutines no deberán corromper el cursor local compartido.

---

# 219. Random test

Con seeded RNG deberá ser reproducible.

---

# 220. Weighted random distribution test

Muchas selecciones deberán aproximar weights estadísticamente.

No exigir secuencia exacta.

---

# 221. Zero weight test

Endpoint zero-weight no deberá seleccionarse normalmente.

---

# 222. All zero weights test

Debe fallar o usar fallback explícito.

---

# 223. Invalid weight test

Negative/NaN/infinite deberá rechazarse.

---

# 224. Least-connections test

Debe seleccionar lowest observed active count.

---

# 225. Stale metric test

Stale metrics deberán seguir configured policy.

---

# 226. Unknown load test

UNKNOWN no será tratado como IDLE.

---

# 227. Latency test

Lower fresh latency podrá ser preferida.

---

# 228. Outlier test

Una observación extrema no deberá destruir necesariamente metric history según configured smoothing.

---

# 229. Locality test

Same-zone candidate puede preferirse cuando todos los demás requisitos son equivalentes.

---

# 230. Consistency precedence test

Lower latency replica que viola freshness nunca llegará al balancer.

---

# 231. Sticky test

Solo replicas que satisfacen sticky requirement participarán.

---

# 232. Failover test

Old writer removido de topology no deberá seleccionarse aunque conserve best historical metrics.

---

# 233. Transaction test

Active transaction deberá bypass normal load-balancing endpoint selection.

---

# 234. Read-only transaction test

Balancer selecciona antes de `BEGIN`, luego connection queda pinned.

---

# 235. Pool saturation test

Pool-saturated candidate podrá recibir penalty/fallback según policy.

---

# 236. Acquisition failure test

Antes de execution podrá ocurrir bounded reselection.

---

# 237. Reselection loop test

No deberá exceder budget.

---

# 238. Topology change test

Round-robin state se reconciliará cuando candidate set cambie.

---

# 239. Removed candidate test

Stale balancer state no devolverá candidate ya eliminado.

---

# 240. Persistent runtime test

Miles de requests deberán mantener:

```text
bounded state
no per-request leakage
no history growth
```

---

# 241. Coroutine isolation test

Request-specific affinity/context permanecerá aislada aunque group balancing state sea shared-safe.

---

# 242. Proposed directory structure

```text
src/Quantum/Database/Routing/LoadBalancing/
│
├── LoadBalancer.php
├── DefaultLoadBalancer.php
├── LoadBalancingOperationId.php
├── LoadBalancingContext.php
├── LoadBalancingDecision.php
├── LoadBalancingDecisionReason.php
│
├── Candidate/
│   ├── LoadBalancingCandidate.php
│   ├── LoadBalancingCandidateSet.php
│   └── EndpointLoadBalancingMetadata.php
│
├── Strategy/
│   ├── LoadBalancingStrategy.php
│   ├── LoadBalancingStrategyResolver.php
│   ├── LoadBalancingStrategyResolution.php
│   ├── FirstStrategy.php
│   ├── RoundRobinStrategy.php
│   ├── RandomStrategy.php
│   ├── WeightedRandomStrategy.php
│   ├── WeightedRoundRobinStrategy.php
│   ├── LeastConnectionsStrategy.php
│   ├── LeastLoadStrategy.php
│   ├── LatencyAwareStrategy.php
│   ├── LocalityAwareStrategy.php
│   ├── AdaptiveStrategy.php
│   └── CustomLoadBalancingStrategy.php
│
├── Policy/
│   ├── LoadBalancingPolicy.php
│   ├── LoadBalancingPolicyRegistry.php
│   ├── LoadBalancingPolicyContributor.php
│   ├── LoadBalancingFallbackPolicy.php
│   ├── EndpointWeightPolicy.php
│   ├── CapacityWeightPolicy.php
│   ├── TieBreakStrategy.php
│   └── MetricFreshnessPolicy.php
│
├── Weight/
│   ├── EndpointWeight.php
│   ├── EndpointWeightFactor.php
│   ├── WeightFactorResult.php
│   └── EffectiveWeightCalculator.php
│
├── Metrics/
│   ├── LoadMetricsSnapshot.php
│   ├── LoadMetricsGeneration.php
│   ├── EndpointLoadMetricMap.php
│   ├── EndpointLoadSignal.php
│   ├── LoadLevel.php
│   ├── LoadMetricConfidence.php
│   ├── EndpointLatencyProfile.php
│   ├── ActiveConnectionCount.php
│   └── EndpointLoadMetricsCollector.php
│
├── State/
│   ├── LoadBalancingGroupId.php
│   ├── LoadBalancingGroupState.php
│   ├── RoundRobinState.php
│   ├── WeightedRoundRobinState.php
│   └── LoadBalancingStateReconciler.php
│
├── Random/
│   ├── RandomSource.php
│   └── DefaultRandomSource.php
│
├── Clock/
│   └── LoadBalancingClock.php
│
├── Budget/
│   └── LoadBalancingResourceBudget.php
│
├── Diagnostics/
│   ├── LoadBalancingInspector.php
│   ├── LoadBalancingExplainer.php
│   └── LoadBalancingDiagnosticReport.php
│
├── Telemetry/
│   ├── LoadBalancingTelemetry.php
│   ├── LoadBalancingStarted.php
│   ├── LoadBalancingDecisionMade.php
│   ├── LoadBalancingFallbackApplied.php
│   └── LoadBalancingFailed.php
│
└── Exception/
    ├── DatabaseLoadBalancingException.php
    ├── LoadBalancingPolicyException.php
    ├── LoadBalancingStrategyUnsupportedException.php
    ├── LoadBalancingMetricsUnavailableException.php
    ├── LoadBalancingInvalidWeightException.php
    ├── LoadBalancingSelectionException.php
    ├── LoadBalancingStateException.php
    ├── LoadBalancingBudgetExceededException.php
    └── LoadBalancingInvariantViolationException.php
```

---

# 243. Invariantes arquitectónicos

## DB-LB-001
Load balancing será distinto de routing.

## DB-LB-002
Load balancing será distinto de failover.

## DB-LB-003
Load balancing será distinto de replica eligibility.

## DB-LB-004
Load balancing será distinto de sharding.

## DB-LB-005
Load balancing será distinto de health checking.

## DB-LB-006
Load balancing será distinto de connection pooling.

## DB-LB-007
LoadBalancer recibirá únicamente candidatos elegibles.

## DB-LB-008
LoadBalancer no reintroducirá candidatos rechazados.

## DB-LB-009
Hard constraints habrán sido resueltas antes del load balancer.

## DB-LB-010
Load balancing no decidirá tenant.

## DB-LB-011
Load balancing no decidirá shard.

## DB-LB-012
Load balancing no decidirá write authority.

## DB-LB-013
Load balancing no cambiará transaction pinning.

## DB-LB-014
Load balancing no degradará consistency requirements.

## DB-LB-015
Load balancing no medirá replica lag directamente.

## DB-LB-016
Load balancing no ejecutará failover.

## DB-LB-017
Load balancing no abrirá conexiones directamente.

## DB-LB-018
LoadBalancingDecision será distinta de ConnectionLease.

## DB-LB-019
LoadBalancingContext será immutable.

## DB-LB-020
LoadBalancingContext no realizará network I/O.

## DB-LB-021
Selection strategy será explícita.

## DB-LB-022
FIRST será determinista.

## DB-LB-023
ROUND_ROBIN mantendrá state bounded.

## DB-LB-024
Round-robin state será concurrency-safe cuando sea compartido.

## DB-LB-025
Random strategy podrá ser stateless.

## DB-LB-026
Weighted strategies utilizarán pesos válidos.

## DB-LB-027
Negative weights serán inválidos.

## DB-LB-028
NaN weights serán inválidos.

## DB-LB-029
Infinite weights serán inválidos.

## DB-LB-030
Zero weight será semántica explícita.

## DB-LB-031
All-zero weight set no producirá selección indefinida.

## DB-LB-032
Weights serán relativos.

## DB-LB-033
Dynamic weights estarán separados de hard eligibility.

## DB-LB-034
Health factor no convertirá unhealthy candidate en eligible.

## DB-LB-035
Lag factor no convertirá stale replica en eligible.

## DB-LB-036
Affinity factor no convertirá invalid endpoint en eligible.

## DB-LB-037
Least-connections utilizará observed metrics.

## DB-LB-038
Connection count no se interpretará como exact DB load.

## DB-LB-039
Least-load tendrá semántica de load explícita.

## DB-LB-040
UNKNOWN load no equivaldrá a IDLE.

## DB-LB-041
Load será distinto de health.

## DB-LB-042
Saturated candidate podrá seguir siendo healthy.

## DB-LB-043
Latency será distinta de replica lag.

## DB-LB-044
Latency será distinta de freshness.

## DB-LB-045
Network latency será distinta de query latency.

## DB-LB-046
Metrics deberán declarar freshness.

## DB-LB-047
Stale metrics no serán current metrics.

## DB-LB-048
Metrics podrán declarar confidence.

## DB-LB-049
Low-confidence metric no se presentará con falsa precisión.

## DB-LB-050
Adaptive strategy deberá ser explicable.

## DB-LB-051
Adaptive strategy no será una caja negra sin diagnostics.

## DB-LB-052
Freshness hard requirement se resolverá antes del adaptive balancer.

## DB-LB-053
Freshness podrá ser soft preference entre replicas válidas.

## DB-LB-054
Correctness tendrá prioridad sobre load optimization.

## DB-LB-055
Consistency tendrá prioridad sobre latency.

## DB-LB-056
Transaction affinity tendrá prioridad sobre load balancing.

## DB-LB-057
Active transaction normalmente bypassará endpoint balancing.

## DB-LB-058
Read-only transaction elegirá endpoint antes de BEGIN.

## DB-LB-059
Active transaction no se moverá por load changes.

## DB-LB-060
Sticky System filtrará candidatos antes del balancer.

## DB-LB-061
Sticky AUTHORITATIVE podrá eliminar replicas completamente.

## DB-LB-062
Failover barrier podrá reducir candidate set.

## DB-LB-063
Old writer removido no será seleccionable.

## DB-LB-064
New writer solo será seleccionable después de authority eligibility.

## DB-LB-065
Pool metrics podrán influir como soft signals.

## DB-LB-066
Pool Manager seguirá siendo dueño de leases.

## DB-LB-067
Pool acquisition failure será distinto de health failure.

## DB-LB-068
Acquisition failure podrá permitir bounded reselection antes de execution.

## DB-LB-069
Reselection será distinta de query retry.

## DB-LB-070
Reselection será distinta de transaction retry.

## DB-LB-071
Reselection será distinta de failover.

## DB-LB-072
Reselection count será bounded.

## DB-LB-073
Candidate failed in current operation podrá excluirse temporalmente.

## DB-LB-074
Una failure local no mutará global health automáticamente.

## DB-LB-075
LoadMetricsSnapshot será immutable.

## DB-LB-076
Load metrics publication será atomic/build-and-swap.

## DB-LB-077
LoadMetricsGeneration será diagnosticable.

## DB-LB-078
Metric history será bounded.

## DB-LB-079
Selection history será bounded.

## DB-LB-080
Per-endpoint state será generation-aware.

## DB-LB-081
Topology change requerirá balancer state reconciliation.

## DB-LB-082
Removed endpoint no permanecerá seleccionable por stale state.

## DB-LB-083
Changed weight deberá reconciliarse.

## DB-LB-084
Process-local round-robin no se presentará como globally exact.

## DB-LB-085
Cluster-global fairness requerirá infraestructura adicional cuando sea necesaria.

## DB-LB-086
VoltStack no introducirá distributed coordination solo para round-robin perfecto por default.

## DB-LB-087
Locality será soft preference.

## DB-LB-088
Locality no superará consistency.

## DB-LB-089
Locality no superará health hard constraints.

## DB-LB-090
Region metadata no será latency guarantee.

## DB-LB-091
Cross-region cost podrá influir solo como preference.

## DB-LB-092
Capacity será distinta de current load.

## DB-LB-093
Capacity weights serán configurables.

## DB-LB-094
Writer protection podrá reducir ordinary read preference hacia writer.

## DB-LB-095
Writer protection no podrá negar strict sticky writer requirement.

## DB-LB-096
Fairness no significará equal traffic cuando existan weights.

## DB-LB-097
Short-window imbalance será posible con random strategies.

## DB-LB-098
Long-term distribution será observable.

## DB-LB-099
Tie-breaking será explícito.

## DB-LB-100
Random tie-break podrá utilizar RandomSource inyectable.

## DB-LB-101
Tests podrán utilizar seeded RandomSource.

## DB-LB-102
Load balancing no requerirá cryptographic RNG.

## DB-LB-103
Time-dependent strategies utilizarán Clock abstractions.

## DB-LB-104
Hot path evitará reflection innecesaria.

## DB-LB-105
Hot path evitará network probes.

## DB-LB-106
Candidate evaluation será bounded.

## DB-LB-107
Factor evaluation será bounded.

## DB-LB-108
Selection time tendrá budget.

## DB-LB-109
Policy registration ocurrirá principalmente en bootstrap.

## DB-LB-110
Compiled policy registry será frozen.

## DB-LB-111
Custom strategy no podrá devolver endpoint externo al eligible set.

## DB-LB-112
Custom strategy no podrá mutar transaction.

## DB-LB-113
Custom strategy no podrá cambiar tenant.

## DB-LB-114
Custom strategy no podrá cambiar shard.

## DB-LB-115
Custom strategy no podrá ejecutar promotion.

## DB-LB-116
Custom strategy no podrá mutar sticky context.

## DB-LB-117
Strategy fallback será explícito.

## DB-LB-118
Unsupported strategy no degradará silenciosamente.

## DB-LB-119
Metric-unavailable fallback será explícito.

## DB-LB-120
Effective strategy será diagnosticable.

## DB-LB-121
Telemetry será observational.

## DB-LB-122
Telemetry listener no modificará selection.

## DB-LB-123
Candidate-scoring telemetry podrá ser sampled.

## DB-LB-124
Metrics evitarán high-cardinality labels por default.

## DB-LB-125
Diagnostics distinguirán rejected-by-routing de not-selected-by-balancer.

## DB-LB-126
Not selected no significará unhealthy.

## DB-LB-127
Not selected no significará ineligible.

## DB-LB-128
Diagnostics mostrarán strategy requested/effective.

## DB-LB-129
Diagnostics mostrarán metric generation.

## DB-LB-130
Diagnostics mostrarán weights cuando sea útil.

## DB-LB-131
Diagnostics serán sanitizables.

## DB-LB-132
Persistent workers podrán compartir immutable policies.

## DB-LB-133
Shared mutable strategy state será concurrency-safe.

## DB-LB-134
Request-local balancing context no se compartirá.

## DB-LB-135
FrankenPHP no filtrará request-specific affinity entre requests.

## DB-LB-136
RoadRunner no filtrará request-specific selection state.

## DB-LB-137
OpenSwoole coroutine contexts estarán aislados.

## DB-LB-138
Shared group state será bounded.

## DB-LB-139
Global static last-selected endpoint estará prohibido como request affinity.

## DB-LB-140
Load balancer state podrá resetearse/reconciliarse.

## DB-LB-141
Metric collectors serán separados del selector.

## DB-LB-142
Metric collectors tendrán rate limits.

## DB-LB-143
Metric collectors evitarán probe amplification.

## DB-LB-144
Metric collector failure no implicará automáticamente query failure.

## DB-LB-145
Metric collector failure podrá activar configured strategy fallback.

## DB-LB-146
UNKNOWN metrics permanecerán UNKNOWN.

## DB-LB-147
No se fabricarán numeric load percentages.

## DB-LB-148
Load balancer no otorgará transaction atomicity.

## DB-LB-149
Load balancer no otorgará transaction isolation.

## DB-LB-150
Load balancer no otorgará read-your-writes.

## DB-LB-151
Load balancer no otorgará failover safety.

## DB-LB-152
Load balancer no otorgará replication durability.

## DB-LB-153
Load balancer no transformará multiple writers en safe multi-primary topology.

## DB-LB-154
Load balancer nunca sacrificará una hard consistency guarantee para mejorar distribución.

## DB-LB-155
La existencia de un endpoint más rápido no hará elegible a un endpoint que las capas anteriores hayan rechazado.

---

# 244. Modelo formal

Sea:

```text
E(O) = conjunto de candidatos elegibles para operación O
```

El Load Balancer opera únicamente sobre:

```text
E(O)
```

La selección:

```text
Selected(O)
=
LB(
    E(O),
    Metrics,
    Policy,
    Affinity
)
```

con la condición obligatoria:

```text
Selected(O) ∈ E(O)
```

---

# 245. Modelo weighted

Para candidatos:

```text
C = {c₁, c₂, ..., cₙ}
```

con pesos positivos:

```text
w₁, w₂, ..., wₙ
```

la probabilidad weighted-random será:

```text
P(cᵢ)
=
wᵢ
/
Σⱼ wⱼ
```

Esto solo aplica después de:

```text
cᵢ ∈ EligibleCandidates
```

---

# 246. Modelo adaptive

Conceptualmente:

```text
EffectivePreference(c)
=
Policy(
    BaseWeight(c),
    Load(c),
    Latency(c),
    Capacity(c),
    Locality(c),
    HealthPreference(c),
    FreshnessPreference(c)
)
```

sujeto a:

```text
Eligible(c) = true
```

---

# 247. Flujo completo

```text
                     QUERY / OPERATION
                            │
                            ▼
                     Semantic Routing
                            │
                            ▼
                   Execution Domain Check
                            │
                            ▼
                    Role / Intent Check
                            │
                            ▼
                   Transaction Constraints
                            │
                            ▼
                    Replica Eligibility
                            │
                            ▼
                       Lag/Freshness
                            │
                            ▼
                    Sticky Requirement
                            │
                            ▼
                    Failover / Authority
                            │
                            ▼
                     Health Hard Gates
                            │
                            ▼
                  ELIGIBLE CANDIDATE SET
                            │
                            ▼
                  LOAD BALANCING SYSTEM
                     /      |       \
                    /       |        \
              Weights     Load     Latency
                  \         |         /
                   \        |        /
                    ▼       ▼       ▼
                     Selection Policy
                            │
                            ▼
                     Selected Endpoint
                            │
                            ▼
                     Connection Manager
                            │
                            ▼
                      Connection Lease
```

---

# 248. Ejemplo integrado

Supongamos:

```text
READ requirement:
    max staleness = 2s

Sticky:
    none

Replicas:

A
    lag: 100ms
    load: HIGH
    latency: 10ms

B
    lag: 500ms
    load: LOW
    latency: 20ms

C
    lag: 5s
    load: LOW
    latency: 5ms
```

Primero:

```text
Lag Awareness
```

elimina:

```text
C
```

porque:

```text
5s > 2s
```

El Load Balancer recibe:

```text
A
B
```

No:

```text
A
B
C
```

Después puede elegir:

```text
B
```

por menor carga.

Esto preserva:

```text
Consistency First
Optimization Second
```

---

# 249. Arquitectura resultante del bloque

Con los documentos:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
```

VoltStack dispone ya de:

```text
Logical Database
      │
      ▼
Read/Write Roles
      │
      ▼
Routing Requirements
      │
      ▼
Replica Topology
      │
      ▼
Replica Freshness
      │
      ▼
Sticky Consistency
      │
      ▼
Failover Authority
      │
      ▼
Eligible Candidate Set
      │
      ▼
Load Balancing
      │
      ▼
Selected Endpoint
```

La siguiente etapa deberá elevar estos componentes a una arquitectura distribuida coherente.

---

# 250. Regla maestra final

> **El Database Load Balancing System de VoltStack optimizará distribución, utilización y latencia únicamente dentro del espacio de decisiones que las capas de seguridad, consistencia y topología ya hayan declarado válidas. El balancer nunca tendrá autoridad para relajar una garantía, restaurar un endpoint rechazado, cambiar de shard, saltarse transaction affinity o elegir un writer cuya autoridad no esté demostrada.**

En forma compacta:

```text
Safety
   ↓
Consistency
   ↓
Eligibility
   ↓
Load Balancing
   ↓
Performance
```

Nunca:

```text
Performance
   ↓
Hope It Is Safe
```

---

# 251. Siguiente documento

```text
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
```

El siguiente documento consolidará:

```text
Read/Write Connections
Replica Topology
Replica Lag Awareness
Sticky Consistency
Failover
Load Balancing
```

en una arquitectura distribuida global:

```text
Application
    │
    ▼
Database Execution Domain
    │
    ▼
Distributed Database Topology
    │
    ├── Cluster
    ├── Region
    ├── Replication Group
    ├── Authority
    ├── Replicas
    ├── Failure Domains
    ├── Read/Write Routing
    ├── Consistency Requirements
    ├── Failover
    └── Load Balancing
```

y establecerá los límites entre:

```text
Distributed Database
≠
Sharding
≠
Replication
≠
Multitenancy
≠
Distributed Transaction
```

preparando los documentos:

```text
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

para extender la distribución desde **múltiples copias del mismo dataset** hacia **múltiples particiones del dataset**.