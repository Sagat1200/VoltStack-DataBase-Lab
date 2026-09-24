# 181_DATABASE_FAILOVER_SYSTEM.md

# VoltStack Quantum Database
## Database Failover System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 181 — Database Failover System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `180_DATABASE_STICKY_CONNECTION_SYSTEM.md`  
**Siguiente documento:** `182_DATABASE_LOAD_BALANCING_SYSTEM.md`

---

# 1. Propósito

`Database Failover System` define la arquitectura mediante la cual VoltStack detectará, representará y reaccionará ante la pérdida de disponibilidad o autoridad de un endpoint de base de datos, permitiendo que una topología pueda transicionar hacia otro endpoint válido sin fabricar garantías sobre datos, transacciones o consistencia que no puedan demostrarse.

Los documentos anteriores establecieron:

```text
176 Read/Write Connection System
    ↓
roles de conexión

177 Read/Write Routing System
    ↓
selección semántica

178 Replica System
    ↓
topología de replicación

179 Replica Lag Awareness System
    ↓
progreso y frescura

180 Sticky Connection System
    ↓
read-after-write consistency
```

Ahora se añade:

```text
Endpoint Failure
      │
      ▼
Failure Evidence
      │
      ▼
Failover Decision
      │
      ▼
Authority Candidate Evaluation
      │
      ▼
Fencing / Promotion
      │
      ▼
Topology Transition
      │
      ▼
New Authority
      │
      ▼
Connection + Routing Reconciliation
```

La regla central será:

> **VoltStack nunca interpretará la pérdida de una conexión como autorización suficiente para promover otra replica. Todo failover deberá basarse en evidencia explícita de fallo, autoridad, topología y seguridad de promoción; cuando no pueda demostrarse qué escrituras sobrevivieron o qué nodo posee autoridad exclusiva, el sistema preservará `UNKNOWN` en lugar de fabricar continuidad.**

Por tanto:

```text
ConnectionFailure
≠
NodeFailure
≠
PrimaryFailure
≠
FailoverRequired
≠
PromotionSafe
≠
NewAuthorityEstablished
```

---

# 2. Objetivos

El sistema deberá proporcionar:

- detección estructurada de fallos;
- clasificación de failure evidence;
- evaluación de autoridad;
- selección de candidatos;
- coordinación de failover;
- fencing del writer anterior;
- protección contra split-brain;
- transición atómica de topología;
- invalidación de conexiones obsoletas;
- reconciliación de transactions;
- reconciliación de sticky requirements;
- integración con replica lag;
- integración con routing;
- soporte para proveedores administrados;
- failover manual y automático;
- recovery y failback controlados;
- telemetry;
- diagnostics;
- seguridad en runtimes persistentes.

---

# 3. No responsabilidades

`FailoverSystem` no será responsable de:

- ejecutar queries ORM;
- generar SQL;
- administrar UnitOfWork;
- hacer hydration;
- balancear carga normal;
- implementar replication protocol;
- reemplazar el sistema de backup;
- garantizar distributed consensus por sí solo;
- implementar un cluster database completo;
- asumir que una replica contiene todas las escrituras.

---

# 4. Distinciones fundamentales

VoltStack mantendrá:

```text
Failover
≠
Retry
≠
Reconnect
≠
Load Balancing
≠
Replica Routing
≠
Connection Pooling
≠
Recovery
≠
Failback
```

---

# 5. Reconnect ≠ failover

Si una conexión TCP se pierde:

```text
Connection A
   ↓
disconnect
```

puede bastar:

```text
reconnect to same endpoint
```

No necesariamente:

```text
promote another database
```

---

# 6. Retry ≠ failover

Un query retry puede utilizar:

```text
same authority
```

mientras que failover modifica:

```text
effective database authority/topology
```

---

# 7. Load balancing ≠ failover

Load balancing selecciona entre:

```text
already eligible endpoints
```

Failover modifica cuáles endpoints:

```text
are authoritative / eligible
```

---

# 8. Recovery ≠ failover

`Failover`:

```text
failed authority
→ alternate authority
```

`Recovery`:

```text
failed component
→ restored component
```

---

# 9. Failback ≠ recovery

Que el antiguo primary vuelva a funcionar no significa que deba recuperar inmediatamente su rol.

```text
OldPrimaryRecovered
≠
OldPrimaryMayBecomePrimary
```

---

# 10. Arquitectura general

```text
                    Database Topology
                           │
                           ▼
                    Health Evidence
                           │
                           ▼
                  Failure Detector
                           │
                           ▼
                   Failure Evidence
                           │
                           ▼
                  Failover Evaluator
                           │
                  ┌────────┴────────┐
                  │                 │
                NOOP            FAILOVER
                                    │
                                    ▼
                          Candidate Discovery
                                    │
                                    ▼
                         Candidate Evaluation
                                    │
                                    ▼
                             Fencing Gate
                                    │
                                    ▼
                           Promotion Adapter
                                    │
                                    ▼
                         Authority Verification
                                    │
                                    ▼
                        Topology Transition
                                    │
                                    ▼
                      Connection Invalidation
                                    │
                                    ▼
                        Routing Reconciliation
```

---

# 11. Failure model

VoltStack no utilizará un único booleano:

```text
failed = true
```

Se utilizará evidencia estructurada.

---

# 12. DatabaseFailureKind

```php
enum DatabaseFailureKind
{
    case CONNECTION;
    case ENDPOINT;
    case AUTHORITY;
    case REPLICATION;
    case NETWORK_PARTITION;
    case PLATFORM;
    case PROVIDER;
    case RESOURCE_EXHAUSTION;
    case ADMINISTRATIVE;
    case UNKNOWN;
}
```

---

# 13. Failure scope

```php
enum DatabaseFailureScope
{
    case CONNECTION;
    case ENDPOINT;
    case AVAILABILITY_ZONE;
    case REGION;
    case CLUSTER;
    case UNKNOWN;
}
```

---

# 14. Failure confidence

```php
enum FailureConfidence
{
    case CONFIRMED;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 15. FailureEvidence

```php
final readonly class DatabaseFailureEvidence
{
    public function __construct(
        public DatabaseEndpointId $endpoint,
        public DatabaseFailureKind $kind,
        public DatabaseFailureScope $scope,
        public FailureConfidence $confidence,
        public Instant $observedAt,
        public TopologyGeneration $topologyGeneration,
        public array $signals,
    ) {}
}
```

---

# 16. Evidence ≠ truth absoluta

Incluso múltiples timeouts pueden deberse a:

```text
application network partition
```

y no a:

```text
database primary failure
```

---

# 17. Network partition problem

Supongamos:

```text
Application A ──X── Primary

Application B ───── Primary
```

Desde A:

```text
Primary appears dead
```

Desde B:

```text
Primary remains alive
```

Promover una replica sin fencing puede producir:

```text
Primary 1 accepts writes
+
Primary 2 accepts writes
=
SPLIT BRAIN
```

---

# 18. Regla de seguridad

> **La incapacidad de contactar al writer anterior no demuestra que haya dejado de aceptar escrituras.**

Formalmente:

```text
Unreachable(OldWriter)
≠
Fenced(OldWriter)
```

---

# 19. FailoverTrigger

```php
enum FailoverTrigger
{
    case AUTOMATIC_HEALTH;
    case PROVIDER_EVENT;
    case ADMINISTRATIVE;
    case ORCHESTRATOR;
    case TEST;
}
```

---

# 20. Automatic failover

Solo deberá activarse cuando:

```text
PolicyAllowsAutomaticFailover
AND
EvidenceSufficient
AND
CandidateExists
AND
FencingRequirementSatisfied
```

---

# 21. Manual failover

Permitirá:

```text
operator
→ request transition
```

pero seguirá pasando por invariants de seguridad.

---

# 22. Manual ≠ unsafe

Una acción administrativa no deberá omitir silenciosamente:

- topology validation;
- candidate validation;
- fencing;
- generation checks;
- authority verification.

---

# 23. Failover policy

```php
final readonly class DatabaseFailoverPolicy
{
    public function __construct(
        public FailoverMode $mode,
        public FailureConfidence $minimumConfidence,
        public FencingPolicy $fencing,
        public DataLossPolicy $dataLoss,
        public Duration $decisionTimeout,
        public Duration $promotionTimeout,
    ) {}
}
```

---

# 24. FailoverMode

```php
enum FailoverMode
{
    case DISABLED;
    case MANUAL;
    case AUTOMATIC;
    case PROVIDER_MANAGED;
}
```

---

# 25. PROVIDER_MANAGED

En servicios administrados, VoltStack puede no ejecutar la promoción.

Flujo:

```text
Provider
   │
   ▼
performs failover
   │
   ▼
VoltStack observes topology change
   │
   ▼
verifies new authority
   │
   ▼
reconciles local state
```

---

# 26. Provider-managed ≠ invisible

Aunque el proveedor realice el failover, VoltStack todavía necesita saber:

- topology generation;
- authority identity;
- connection invalidation;
- transaction uncertainty;
- replication epoch;
- sticky reconciliation.

---

# 27. Failover state machine

```text
IDLE
 │
 ▼
SUSPECTED
 │
 ▼
EVALUATING
 │
 ├── insufficient evidence ──→ IDLE
 │
 ▼
PREPARING
 │
 ▼
FENCING
 │
 ▼
PROMOTING
 │
 ▼
VERIFYING
 │
 ├── uncertain ──→ UNKNOWN
 │
 ▼
TRANSITIONING
 │
 ▼
COMPLETED
```

Failure paths:

```text
FAILED
ABORTED
UNKNOWN
```

---

# 28. FailoverState

```php
enum FailoverState
{
    case IDLE;
    case SUSPECTED;
    case EVALUATING;
    case PREPARING;
    case FENCING;
    case PROMOTING;
    case VERIFYING;
    case TRANSITIONING;
    case COMPLETED;
    case FAILED;
    case ABORTED;
    case UNKNOWN;
}
```

---

# 29. UNKNOWN state

`UNKNOWN` será first-class.

Ejemplo:

```text
promotion command sent
↓
orchestrator connection lost
↓
cannot verify result
```

No deberá marcarse:

```text
COMPLETED
```

ni:

```text
FAILED
```

sin evidencia.

---

# 30. FailoverOperationId

Cada transición tendrá:

```php
final readonly class FailoverOperationId
{
    // globally unique diagnostic identity
}
```

---

# 31. FailoverContext

```php
final readonly class FailoverContext
{
    public function __construct(
        public FailoverOperationId $operationId,
        public ReplicationTopologySnapshot $originalTopology,
        public DatabaseFailureEvidence $trigger,
        public DatabaseFailoverPolicy $policy,
        public Instant $startedAt,
    ) {}
}
```

---

# 32. Mutable execution state

El estado mutable de una operación pertenecerá a:

```text
FailoverExecutionContext
```

No a un singleton global.

---

# 33. Candidate discovery

Los candidatos deberán provenir de:

```text
current replication topology
```

o de una autoridad externa confiable.

---

# 34. Candidate ≠ eligible candidate

Una replica registrada:

```text
REPLICA
```

no es automáticamente:

```text
PROMOTION_CANDIDATE
```

---

# 35. PromotionCandidate

```php
final readonly class PromotionCandidate
{
    public function __construct(
        public DatabaseEndpointId $endpoint,
        public ReplicaId $replica,
        public PromotionCandidateEvidence $evidence,
    ) {}
}
```

---

# 36. Candidate evaluation

Debe considerar, cuando la plataforma lo permita:

```text
Replica State
Replication Health
Replication Position
Replica Lag
Durability Evidence
Topology Generation
Replication Epoch
Promotion Capability
Region / Failure Domain
Provider State
Administrative Constraints
```

---

# 37. Candidate evaluation result

```php
enum PromotionCandidateStatus
{
    case ELIGIBLE;
    case DEGRADED;
    case INELIGIBLE;
    case UNKNOWN;
}
```

---

# 38. PromotionCandidateEvaluation

```php
final readonly class PromotionCandidateEvaluation
{
    public function __construct(
        public PromotionCandidate $candidate,
        public PromotionCandidateStatus $status,
        public array $reasons,
        public DataContinuityEvidence $continuity,
    ) {}
}
```

---

# 39. Data continuity

Una pregunta central será:

> ¿Qué tan seguro es afirmar que este candidato contiene las escrituras confirmadas relevantes del writer anterior?

---

# 40. DataContinuityStatus

```php
enum DataContinuityStatus
{
    case VERIFIED;
    case HIGH_CONFIDENCE;
    case POSSIBLE_GAP;
    case KNOWN_GAP;
    case UNKNOWN;
}
```

---

# 41. Replica lag alone insufficient

Una replica con:

```text
lag = 0ms
```

no necesariamente constituye prueba absoluta de que contiene todas las escrituras relevantes.

---

# 42. Position evidence

Cuando exista:

```text
CandidatePosition >= RequiredDurablePosition
```

puede aportar evidencia más fuerte.

---

# 43. Required durable position

El sistema podrá mantener:

```text
LastKnownDurableAuthorityPosition
```

si la plataforma puede demostrarlo.

---

# 44. Known committed writes

Sticky requirements activas también pueden contener:

```text
RequiredPosition(P)
```

que deberán reconciliarse con el candidato.

---

# 45. Candidate behind required position

Si:

```text
CandidatePosition < P
```

y `P` representa una escritura confirmada:

```text
candidate cannot satisfy continuity requirement
```

---

# 46. Data-loss policy

```php
enum DataLossPolicy
{
    case ZERO_KNOWN_LOSS;
    case REQUIRE_EXPLICIT_APPROVAL;
    case ALLOW_BOUNDED_LOSS;
    case PROVIDER_DEFINED;
}
```

---

# 47. Default

Para VoltStack:

```text
ZERO_KNOWN_LOSS
```

deberá ser el default conceptual para failover controlado por framework.

---

# 48. Bounded loss

Si alguna infraestructura admite RPO:

```text
RPO = 5 seconds
```

deberá configurarse explícitamente.

---

# 49. RPO ≠ measured lag

```text
Recovery Point Objective
≠
Current Replica Lag
```

RPO es una política/objetivo.

Lag es evidencia operacional.

---

# 50. Candidate ranking

Primero:

```text
Safety
```

después:

```text
Freshness
```

después:

```text
Operational Preference
```

Nunca al revés.

---

# 51. Candidate ranking example

```text
Replica A
    position: current
    region: remote

Replica B
    position: behind
    region: local
```

No deberá elegirse B únicamente por menor network latency si ello viola continuity policy.

---

# 52. Promotion score

Si se usa scoring:

```text
Score
=
SafetyGate
×
PreferenceScore
```

No:

```text
Safety + Performance
```

donde suficiente performance pueda compensar falta de seguridad.

---

# 53. Hard gates

Ejemplos:

```text
promotion capable?
topology compatible?
fencing satisfied?
continuity acceptable?
not administratively excluded?
```

---

# 54. Fencing

`Fencing` evita que el antiguo writer siga aceptando escrituras después de establecer una nueva autoridad.

---

# 55. Fencing strategies

Conceptualmente:

```php
enum FencingStrategy
{
    case PROVIDER_MANAGED;
    case LEASE;
    case EPOCH_TOKEN;
    case NETWORK_ISOLATION;
    case READ_ONLY_DEMOTION;
    case EXTERNAL_COORDINATOR;
    case MANUAL_CONFIRMATION;
}
```

---

# 56. Core does not invent fencing

VoltStack deberá utilizar capabilities/adapters.

No implementará una falsa garantía genérica.

---

# 57. Fencing result

```php
enum FencingStatus
{
    case CONFIRMED;
    case NOT_REQUIRED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 58. UNKNOWN fencing

Para automatic failover estricto:

```text
FencingStatus::UNKNOWN
→ promotion prohibited
```

---

# 59. Split-brain protection

Regla:

```text
Two writable authorities
=
critical invariant violation
```

---

# 60. Authority lease

Una arquitectura compatible puede usar:

```text
AuthorityLease
```

con:

```text
LeaseId
Epoch
Holder
ExpiresAt
```

---

# 61. Lease ≠ DB transaction

```text
AuthorityLease
≠
Transaction
≠
ConnectionLease
```

---

# 62. Epoch fencing

Cada autoridad puede pertenecer a:

```text
AuthorityEpoch N
```

Tras failover:

```text
N → N+1
```

Operaciones con epoch anterior podrán rechazarse si la infraestructura lo soporta.

---

# 63. Topology epoch

Debe distinguirse:

```text
TopologyGeneration
```

de:

```text
AuthorityEpoch
```

---

# 64. TopologyGeneration

Representa una versión del snapshot/topología conocida por VoltStack.

---

# 65. AuthorityEpoch

Representa una generación de autoridad/fencing cuando la infraestructura ofrece esa semántica.

---

# 66. ReplicationEpoch

Puede ser una tercera identidad:

```text
ReplicationEpoch
```

usada para comparar replication positions.

---

# 67. No conflation

```text
TopologyGeneration
≠
AuthorityEpoch
≠
ReplicationEpoch
```

---

# 68. Promotion adapter

```php
interface DatabasePromotionAdapter
{
    public function promote(
        PromotionCandidate $candidate,
        FailoverExecutionContext $context,
    ): PromotionResult;
}
```

---

# 69. PromotionResult

```php
final readonly class PromotionResult
{
    public function __construct(
        public PromotionStatus $status,
        public ?AuthorityEvidence $authority,
        public ?ReplicationEpoch $replicationEpoch,
    ) {}
}
```

---

# 70. PromotionStatus

```php
enum PromotionStatus
{
    case CONFIRMED;
    case REJECTED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 71. Promotion command success ≠ authority verified

```text
Promotion API returned 200
≠
New writer verified
```

VoltStack deberá realizar:

```text
Authority Verification
```

cuando sea posible.

---

# 72. AuthorityEvidence

```php
final readonly class AuthorityEvidence
{
    public function __construct(
        public DatabaseEndpointId $endpoint,
        public AuthorityEpoch $epoch,
        public AuthorityConfidence $confidence,
        public Instant $observedAt,
    ) {}
}
```

---

# 73. AuthorityConfidence

```php
enum AuthorityConfidence
{
    case VERIFIED;
    case PROVIDER_CONFIRMED;
    case HIGH;
    case LOW;
    case UNKNOWN;
}
```

---

# 74. Authority verification

Puede incluir:

```text
endpoint writable?
provider identifies endpoint as primary?
authority epoch correct?
old authority fenced?
topology agrees?
```

---

# 75. Writable probe caution

Un successful write probe por sí solo no prueba exclusividad.

```text
CanWrite(NewNode)
≠
OnlyWriter(NewNode)
```

---

# 76. Topology transition

Una promoción confirmada deberá generar un nuevo:

```text
ReplicationTopologySnapshot
```

---

# 77. Atomic publication

```text
Old Snapshot
    │
    ▼
Build New Snapshot
    │
    ▼
Validate
    │
    ▼
Freeze
    │
    ▼
Atomic Publish
```

---

# 78. No partial topology

Concurrent requests deberán observar:

```text
Generation N
```

o:

```text
Generation N+1
```

Nunca una mezcla parcial.

---

# 79. New generation

Cada failover exitoso deberá incrementar:

```text
TopologyGeneration
```

o producir una nueva identidad equivalente.

---

# 80. Stale topology consumers

Routing decisions basadas en generación vieja deberán ser detectables.

---

# 81. Routing reconciliation

Tras publicar nueva topología:

```text
ReadWriteRouter
```

deberá dejar de seleccionar endpoints con roles obsoletos.

---

# 82. Connection pool reconciliation

Las conexiones existentes al old writer son peligrosas.

---

# 83. Connection invalidation

Se deberá poder invalidar:

```text
connections associated with old authority generation
```

---

# 84. PoolGeneration

Una estrategia:

```text
ConnectionPoolGeneration
```

podrá asociarse a topology/authority generation.

---

# 85. Borrow-time validation

Al adquirir una conexión:

```text
ConnectionGeneration
<
RequiredGeneration
```

→ no reutilizar.

---

# 86. Return-time validation

Una conexión retornada al pool después de failover podrá:

```text
close
quarantine
revalidate
```

según policy.

---

# 87. No silent role mutation

Una conexión creada como:

```text
WRITE
```

hacia old writer no deberá convertirse simplemente en:

```text
READ
```

cambiando una propiedad local.

La autoridad real cambió.

---

# 88. In-flight queries

Failover no puede garantizar que queries ya enviadas:

```text
did or did not execute
```

si la conexión se pierde.

---

# 89. In-flight write outcome

Puede ser:

```text
SUCCESS
FAILED
UNKNOWN
```

---

# 90. UNKNOWN preservation

Nunca:

```text
connection lost during INSERT
→ assume failed
→ retry on new writer
```

sin analizar idempotencia/outcome.

---

# 91. Duplicate-write risk

Si la escritura sí ocurrió en old writer y se repite en new writer:

```text
duplicate side effects
```

pueden producirse.

---

# 92. Transaction integration

Failover durante una active transaction será especialmente delicado.

---

# 93. No transparent transaction migration

Regla:

> **Una transacción física activa nunca será trasladada transparentemente a otra conexión durante failover.**

---

# 94. Why

Una transaction contiene:

```text
server-side transaction state
locks
snapshot
temporary state
savepoints
session state
```

que no existe en otra conexión.

---

# 95. Transaction connection loss

Resultado típico:

```text
Transaction
    ↓
Connection Lost
    ↓
Transaction Outcome?
```

Puede ser:

```text
ROLLED_BACK
```

si existe evidencia suficiente.

O:

```text
UNKNOWN
```

---

# 96. Commit during failover

Caso crítico:

```text
COMMIT sent to old writer
↓
failover occurs
↓
connection lost before response
```

Resultado:

```text
UNKNOWN
```

---

# 97. No auto-replay commit

No:

```text
COMMIT unknown
→ rerun transaction on new writer automatically
```

sin una estrategia explícita de idempotencia/reconciliation.

---

# 98. TransactionManager integration

`TransactionManager` deberá recibir:

```text
ConnectionFailure / AuthorityChange
```

y marcar el contexto apropiadamente.

---

# 99. Tainted persistence context

Cuando outcome sea desconocido:

```text
EntityManager / UnitOfWork
```

podrán quedar:

```text
TAINTED
```

de acuerdo con los documentos de consistencia anteriores.

---

# 100. Database rollback ≠ object rewind

Failover tampoco modifica esta regla:

```text
Database rollback
≠
automatic PHP object graph rewind
```

---

# 101. Transaction retry

El documento 170 estableció transaction retry.

Después de failover, un retry solo será permitido si:

- el error es retryable;
- outcome anterior es suficientemente conocido;
- policy lo permite;
- callback es retry-safe;
- nueva authority está establecida.

---

# 102. UNKNOWN outcome blocks blind retry

Por default:

```text
UNKNOWN
→ no blind transaction replay
```

---

# 103. Sticky reconciliation

Documento 180 puede mantener:

```text
RequiredPosition(P)
```

después de un write confirmado.

Failover deberá reconciliarlo.

---

# 104. Candidate satisfies sticky position

Si:

```text
NewAuthorityPosition >= P
```

en dominio/epoch comparable:

```text
requirement can remain satisfiable
```

---

# 105. Candidate behind sticky position

Si:

```text
NewAuthorityPosition < P
```

VoltStack tiene evidencia de una posible pérdida respecto de esa requirement.

---

# 106. Candidate incomparable

Si la nueva topología utiliza un nuevo replication epoch:

```text
P(old epoch)
vs
P(new epoch)
```

puede resultar:

```text
INCOMPARABLE
```

---

# 107. Incomparable ≠ satisfied

Nunca:

```text
new writer exists
→ clear sticky requirement
```

---

# 108. Sticky reconciliation status

```php
enum StickyFailoverReconciliationStatus
{
    case SATISFIED;
    case PRESERVED;
    case POSSIBLE_DATA_LOSS;
    case INCOMPARABLE;
    case UNKNOWN;
}
```

---

# 109. Consistency token after failover

Tokens propagados entre requests también deberán validarse contra:

```text
new topology
new replication epoch
new authority
```

---

# 110. Token cannot fabricate continuity

Si una position ya no es comparable:

el token conserva evidencia histórica, pero no prueba que el new writer contenga el write.

---

# 111. Replica lag integration

Lag snapshots previos al failover pueden quedar obsoletos.

---

# 112. Lag generation invalidation

```text
LagSnapshot.TopologyGeneration = N

CurrentTopologyGeneration = N+1
```

deberá forzar revalidación.

---

# 113. Replica state transition

Una antigua replica promovida puede cambiar:

```text
REPLICA
→ PROMOTING
→ AUTHORITATIVE
```

---

# 114. Old primary transition

Puede cambiar:

```text
AUTHORITATIVE
→ UNREACHABLE
```

o después:

```text
AUTHORITATIVE
→ FENCED
→ RECOVERING
→ REPLICA
```

---

# 115. Promotion transitional state

Durante:

```text
PROMOTING
```

el endpoint no deberá recibir tráfico normal de escritura hasta que la autoridad esté confirmada.

---

# 116. Failover routing barrier

Podrá existir:

```text
WriteRoutingBarrier
```

durante una transición crítica.

---

# 117. Barrier purpose

Evita:

```text
new writes
```

mientras:

```text
authority is unresolved
```

---

# 118. WriteAvailabilityState

```php
enum WriteAvailabilityState
{
    case AVAILABLE;
    case DEGRADED;
    case PAUSED;
    case UNAVAILABLE;
    case UNKNOWN;
}
```

---

# 119. Prefer temporary unavailability

Ante autoridad incierta:

```text
temporary write unavailability
```

es preferible a:

```text
two possible writers
```

---

# 120. CAP tradeoff

En presencia de network partition, una arquitectura de failover debe decidir qué garantías prioriza.

VoltStack no esconderá esa decisión detrás de:

```text
auto_failover = true
```

---

# 121. Framework responsibility

VoltStack deberá:

```text
represent policy
enforce known safety gates
preserve uncertainty
integrate provider capabilities
```

No resolver mágicamente consensus.

---

# 122. Provider capability model

```php
interface DatabaseFailoverCapability
{
    public function mode(): FailoverCapabilityMode;

    public function supportsPromotion(): bool;

    public function supportsFencing(): bool;

    public function supportsAuthorityEpoch(): bool;

    public function supportsContinuityVerification(): bool;
}
```

---

# 123. CapabilityMode

```php
enum FailoverCapabilityMode
{
    case FRAMEWORK_COORDINATED;
    case PROVIDER_MANAGED;
    case EXTERNAL_ORCHESTRATOR;
    case MANUAL_ONLY;
    case UNSUPPORTED;
}
```

---

# 124. MySQL/MariaDB/PostgreSQL

VoltStack podrá disponer de adapters específicos.

Pero el core nunca deberá asumir:

```php
if ($platform === 'postgresql') {
    // generic failover truth
}
```

---

# 125. Deployment topology matters

El mismo DBMS puede desplegarse mediante:

```text
self-managed replication
cloud managed service
orchestrator
proxy
cluster manager
```

con semánticas distintas.

---

# 126. Database vendor ≠ failover mechanism

Regla:

```text
DBMS
≠
Deployment Topology
≠
Failover Provider
```

---

# 127. FailoverProvider

```php
interface FailoverProvider
{
    public function capabilities(): DatabaseFailoverCapability;

    public function observeAuthority(): AuthorityObservation;

    public function fence(
        DatabaseEndpointId $endpoint,
    ): FencingResult;

    public function promote(
        PromotionCandidate $candidate,
    ): PromotionResult;
}
```

---

# 128. Provider plugin model

Podrán existir paquetes como:

```text
voltstack/database-failover-*
```

sin acoplar el core a infraestructura concreta.

---

# 129. External orchestrator

VoltStack podrá funcionar en modo:

```text
observe-only
```

donde Kubernetes, cloud provider o DB orchestrator ejecuta el failover.

---

# 130. Observe-only flow

```text
External System
      │
      ▼
Changes Authority
      │
      ▼
VoltStack detects new topology
      │
      ▼
Validates generation
      │
      ▼
Invalidates stale connections
      │
      ▼
Updates routing
```

---

# 131. Failover coordinator

```php
interface DatabaseFailoverCoordinator
{
    public function evaluate(
        DatabaseFailureEvidence $failure,
    ): FailoverDecision;

    public function execute(
        FailoverDecision $decision,
    ): FailoverResult;
}
```

---

# 132. Coordinator ≠ state singleton

El coordinator podrá ser compartido si es stateless.

La operación mutable pertenecerá a:

```text
FailoverExecutionContext
```

---

# 133. FailoverDecision

```php
final readonly class FailoverDecision
{
    public function __construct(
        public FailoverDecisionType $type,
        public ?PromotionCandidate $candidate,
        public array $reasons,
        public DataContinuityStatus $continuity,
    ) {}
}
```

---

# 134. FailoverDecisionType

```php
enum FailoverDecisionType
{
    case NO_ACTION;
    case RECONNECT;
    case WAIT;
    case FAILOVER;
    case REQUIRE_OPERATOR;
    case UNAVAILABLE;
}
```

---

# 135. WAIT

Puede utilizarse para evitar flapping ante fallos transitorios.

---

# 136. Detection threshold

Una policy podrá requerir:

```text
N consecutive failures
```

o:

```text
failure duration > threshold
```

antes de considerar failover.

---

# 137. Threshold ≠ proof

Solo aumenta confidence.

No convierte automáticamente network partition en confirmed primary failure.

---

# 138. Failure detector

```php
interface DatabaseFailureDetector
{
    public function evaluate(
        DatabaseHealthEvidence $evidence,
    ): DatabaseFailureAssessment;
}
```

---

# 139. Passive evidence

Puede provenir de:

```text
connection failures
timeouts
query failures
pool errors
```

---

# 140. Active evidence

Puede provenir de:

```text
health probes
provider APIs
orchestrator status
replication metadata
```

---

# 141. Multiple evidence sources

Preferir:

```text
passive + active + provider
```

sobre un único signal cuando se requiere automatic promotion.

---

# 142. FailureEvidenceSet

```php
final readonly class FailureEvidenceSet
{
    public function __construct(
        public array $observations,
        public FailureConfidence $combinedConfidence,
    ) {}
}
```

---

# 143. Evidence freshness

Old failure evidence no deberá iniciar un nuevo failover.

---

# 144. Failover debounce

El sistema deberá evitar:

```text
failure
recover
failure
recover
```

produciendo promociones repetidas.

---

# 145. Failover cooldown

```php
final readonly class FailoverCooldownPolicy
{
    public function __construct(
        public Duration $minimumInterval,
    ) {}
}
```

---

# 146. Cooldown is not safety proof

Cooldown evita flapping.

No reemplaza fencing.

---

# 147. Single-flight failover

Para una topología:

```text
one effective failover operation
```

deberá ejecutarse a la vez.

---

# 148. Concurrent failover attempts

Dos workers:

```text
Worker A → promote R1
Worker B → promote R2
```

es un riesgo crítico.

---

# 149. Coordination requirement

En deployment multi-process/multi-node, el lock local PHP no es suficiente.

---

# 150. External coordination

Automatic framework-coordinated failover puede requerir:

```text
distributed coordinator
provider atomic API
lease service
orchestrator
```

---

# 151. No fake distributed lock

VoltStack no deberá presentar un mutex local como protección distribuida.

---

# 152. Failover lock

Si existe:

```text
FailoverCoordinationLease
```

deberá tener:

- owner;
- generation;
- expiration;
- fencing token;
- renewal policy.

---

# 153. Coordination lease loss

Si se pierde durante promoción:

```text
operation must stop where safely possible
```

y pasar a:

```text
UNKNOWN
```

cuando el outcome no pueda establecerse.

---

# 154. Idempotent provider operations

Los adapters deberían utilizar operation IDs cuando el provider los soporte.

---

# 155. Repeated promotion request

Debe distinguirse:

```text
same operation replay
```

de:

```text
new promotion attempt
```

---

# 156. Recovery architecture

Después de failover:

```text
Old Writer
```

puede reaparecer.

No se reincorpora inmediatamente.

---

# 157. Recovery states

```php
enum RecoveredEndpointState
{
    case DISCOVERED;
    case FENCED;
    case VALIDATING;
    case RESYNCHRONIZING;
    case READY_AS_REPLICA;
    case REJECTED;
    case UNKNOWN;
}
```

---

# 158. Rejoin safety

Antes de reincorporarlo:

```text
old divergent writes?
replication lineage compatible?
resync completed?
authority disabled?
```

deberá evaluarse.

---

# 159. Divergent old primary

Si old primary aceptó escrituras durante partition:

puede existir:

```text
divergent history
```

---

# 160. Never auto-merge arbitrary histories

VoltStack no intentará fusionar automáticamente historiales divergentes de bases relacionales.

---

# 161. Recovery may require rebuild

El endpoint puede necesitar:

```text
reseed
restore
reclone
resynchronize
```

según provider.

---

# 162. Failback

`Failback` significa devolver autoridad a un endpoint previamente preferido.

---

# 163. No automatic failback by default

Default recomendado:

```text
automatic failover
≠
automatic failback
```

---

# 164. Why

El failback puede causar:

- nueva indisponibilidad;
- cache invalidation;
- connection churn;
- replication reversal;
- nuevos consistency risks.

---

# 165. Failback policy

```php
enum FailbackPolicy
{
    case DISABLED;
    case MANUAL;
    case SCHEDULED;
    case AUTOMATIC_WHEN_VERIFIED;
}
```

---

# 166. Failback safety

Deberá pasar por controles equivalentes a una transición de autoridad normal.

---

# 167. Read-only failover

La pérdida de una replica de lectura no requiere promotion.

---

# 168. Replica failure

Normalmente:

```text
Replica unavailable
→ remove from eligible read set
```

No:

```text
promote another replica
```

---

# 169. All replicas unavailable

Si writer sigue disponible:

```text
read routing
→ writer fallback
```

según policy.

---

# 170. Writer failure

Este es el principal trigger para authority failover.

---

# 171. Multi-region

En sistemas multi-region:

```text
region failure
```

puede afectar varios endpoints simultáneamente.

---

# 172. Failure domains

Topology metadata deberá poder representar:

```text
host
rack
zone
region
provider
```

sin obligar al core a una nube específica.

---

# 173. Candidate diversity

La selección puede preferir un candidate fuera del failure domain afectado, después de pasar safety gates.

---

# 174. Latency is secondary

No sacrificar continuity por elegir el endpoint geográficamente más cercano.

---

# 175. Read-only degraded mode

Si no existe autoridad segura para writes pero existen replicas legibles:

una aplicación podrá optar por:

```text
READ_ONLY_DEGRADED
```

---

# 176. Explicit policy

Nunca activar read-only degraded mode silenciosamente si la aplicación espera writes.

---

# 177. DatabaseAvailabilityMode

```php
enum DatabaseAvailabilityMode
{
    case NORMAL;
    case DEGRADED;
    case READ_ONLY;
    case WRITE_PAUSED;
    case UNAVAILABLE;
    case UNKNOWN;
}
```

---

# 178. Routing integration

El router deberá recibir:

```text
DatabaseAvailabilityMode
+
CurrentTopologyGeneration
+
CurrentAuthority
```

---

# 179. Write paused

Cuando:

```text
WRITE_PAUSED
```

write queries deberán fallar antes de adquirir una conexión si es posible.

---

# 180. Query planner independence

Query Planner no deberá conocer failover.

---

# 181. Compiler independence

SQL Compiler no deberá conocer failover.

---

# 182. Executor integration

Execution Engine sí podrá recibir:

```text
routing/availability context
```

para adquirir una conexión válida.

---

# 183. Retry integration

Un connection failure puede desencadenar:

```text
reconnect
```

o:

```text
wait for failover
```

antes de retry.

---

# 184. No retry storm

Durante failover, miles de requests no deberán:

```text
retry immediately in tight loop
```

---

# 185. Backoff

Utilizar:

```text
bounded exponential backoff
+
jitter
```

en capas donde retry sea apropiado.

---

# 186. Failover barrier

Podrá existir un shared immutable:

```text
FailoverBarrierSnapshot
```

que indique:

```text
NORMAL
TRANSITIONING
WRITE_PAUSED
RECOVERING
```

---

# 187. Hot-path behavior

Requests deberían consultar:

```text
current topology snapshot
+
current failover barrier
```

sin ejecutar probes de failover en cada query.

---

# 188. Control plane vs data plane

Separación:

```text
Control Plane
    ├── failure detection
    ├── promotion
    ├── fencing
    └── topology publication

Data Plane
    ├── query routing
    ├── connection acquisition
    └── execution
```

---

# 189. Fundamental rule

```text
Data Plane
≠
Failover Coordinator
```

---

# 190. Why

No queremos:

```text
SELECT request
→ discovers failure
→ directly promotes database
```

en el query hot path.

---

# 191. Failure signal propagation

El data plane puede publicar:

```text
FailureSignal
```

al control plane.

---

# 192. Failure signal boundedness

Signals deberán ser:

- bounded;
- deduplicated;
- rate-limited;
- sanitized.

---

# 193. Persistent runtime

En FrankenPHP:

```text
TopologySnapshot
FailoverBarrierSnapshot
```

pueden compartirse como immutable state.

---

# 194. Mutable operation state

No deberá compartirse accidentalmente entre requests:

```text
current failover attempt
candidate list mutation
temporary errors
request-specific retry counters
```

---

# 195. RoadRunner

Misma regla de worker reset.

---

# 196. OpenSwoole

Concurrent coroutines no deberán mutar snapshots publicados.

---

# 197. Atomic snapshot swap

Ideal:

```text
AtomicReference<TopologySnapshot>
```

conceptualmente.

---

# 198. No partial mutation

Evitar:

```php
$topology->primary = $new;
$topology->replicas[] = $old;
```

sobre un objeto compartido mutable.

---

# 199. Build and swap

Preferir:

```text
old immutable snapshot
        ↓
new immutable snapshot
        ↓
atomic swap
```

---

# 200. Process-local snapshots

En deployment multi-node, cada process puede tener su propia copia.

---

# 201. Distribution

La fuente de verdad deberá ser:

```text
provider/orchestrator/coordinator/topology service
```

según deployment.

---

# 202. Local snapshot ≠ global coordination

Un shared PHP object dentro de un worker no coordina otros hosts.

---

# 203. Configuration

Ejemplo conceptual:

```php
'database' => [
    'failover' => [
        'mode' => 'provider_managed',

        'failure_detection' => [
            'minimum_confidence' => 'high',
            'grace_period' => '3s',
        ],

        'fencing' => [
            'required' => true,
        ],

        'data_loss' => [
            'policy' => 'zero_known_loss',
        ],

        'cooldown' => '30s',

        'failback' => 'manual',
    ],
],
```

---

# 204. Validation

Configuración:

```text
automatic failover
+
no fencing capability
+
strict exclusive writer requirement
```

deberá producir:

```text
configuration error
```

o exigir provider guarantee explícita.

---

# 205. Secure defaults

Defaults recomendados:

```text
Failover:
    provider-managed when available

Framework automatic promotion:
    disabled unless fencing capability exists

Data loss:
    zero-known-loss

Unknown authority:
    pause writes

Automatic failback:
    disabled
```

---

# 206. Diagnostics API

Conceptualmente:

```php
DB::failover()->status();
```

---

# 207. Example diagnostics

```text
DATABASE FAILOVER STATUS

Connection:
    default

Availability:
    NORMAL

Topology Generation:
    42

Authority Epoch:
    17

Current Authority:
    db-primary-02

Failover State:
    COMPLETED

Last Operation:
    fo_01H...

Trigger:
    PROVIDER_EVENT

Previous Authority:
    db-primary-01

Fencing:
    CONFIRMED

Promotion:
    CONFIRMED

Data Continuity:
    VERIFIED

Transactions With Unknown Outcome:
    0

Sticky Requirements Reconciled:
    24
```

---

# 208. Failure diagnostics

```php
DB::failover()->explainLast();
```

---

# 209. Explain output

```text
FAILOVER EXPLANATION

Trigger:
    Primary endpoint unavailable

Evidence:
    connection failures: HIGH
    provider state: CONFIRMED
    replication state: AVAILABLE

Candidate:
    replica-02

Candidate Position:
    >= last known durable authority position

Fencing:
    provider confirmed old writer disabled

Promotion:
    confirmed

Authority Verification:
    verified

Topology:
    generation 41 → 42

Result:
    COMPLETED
```

---

# 210. Uncertain diagnostics

```text
FAILOVER EXPLANATION

State:
    UNKNOWN

Reason:
    promotion request accepted but provider status
    could not be verified before timeout

Write Routing:
    PAUSED

Old Authority:
    unreachable

Old Authority Fenced:
    UNKNOWN

New Authority:
    UNKNOWN

Safety Decision:
    no writes routed until authority is verified
```

---

# 211. Telemetry events

```text
DatabaseFailureSuspected
DatabaseFailureConfirmed
FailoverEvaluationStarted
FailoverCandidateSelected
FailoverFencingStarted
FailoverFencingCompleted
FailoverPromotionStarted
FailoverPromotionCompleted
FailoverAuthorityVerified
FailoverTopologyPublished
FailoverCompleted
FailoverFailed
FailoverUnknown
DatabaseRecoveryStarted
DatabaseRecoveryCompleted
DatabaseFailbackStarted
DatabaseFailbackCompleted
```

---

# 212. Telemetry observational

Telemetry listeners no deberán:

```text
promote
fence
change topology
retry transactions
```

---

# 213. Metrics

Ejemplos:

```text
db.failover.operations
db.failover.success
db.failover.failed
db.failover.unknown
db.failover.duration
db.failover.fencing.duration
db.failover.promotion.duration
db.failover.write_pause.duration
db.failover.recovery.duration
db.failover.topology.generation
```

---

# 214. Recovery metrics

También:

```text
db.failover.rto
db.failover.data_continuity_unknown
db.failover.sticky_reconciliation_failures
db.failover.stale_connections_invalidated
```

---

# 215. RTO

Puede medirse:

```text
RTOObserved
=
WriteAvailabilityRestoredAt
-
FailureEffectiveAt
```

cuando ambos puntos puedan determinarse.

---

# 216. RTO ≠ configured RTO objective

```text
ObservedRecoveryTime
≠
RecoveryTimeObjective
```

---

# 217. Sensitive telemetry

No incluir:

- credentials;
- connection strings;
- provider tokens;
- raw SQL;
- query parameters;
- sensitive topology secrets.

---

# 218. Error hierarchy

```text
DatabaseFailoverException
├── FailoverDisabledException
├── FailoverNotRequiredException
├── FailoverEvidenceInsufficientException
├── FailoverCandidateNotFoundException
├── FailoverCandidateIneligibleException
├── FailoverCandidateContinuityException
├── FailoverFencingException
├── FailoverFencingUnknownException
├── FailoverPromotionException
├── FailoverPromotionUnknownException
├── FailoverAuthorityVerificationException
├── FailoverSplitBrainRiskException
├── FailoverCoordinationException
├── FailoverLeaseLostException
├── FailoverTopologyConflictException
├── FailoverGenerationMismatchException
├── FailoverDataLossRiskException
├── FailoverStickyReconciliationException
├── FailoverRecoveryException
├── FailbackException
└── FailoverInvariantViolationException
```

---

# 219. SplitBrainRiskException

Debe ser una excepción crítica.

Nunca deberá degradarse a warning silencioso.

---

# 220. DataLossRiskException

Debe diferenciar:

```text
KNOWN_GAP
POSSIBLE_GAP
UNKNOWN
```

---

# 221. Testing architecture

Suite:

```text
FailoverFailureDetectionTests
FailoverEvidenceTests
FailoverPolicyTests
FailoverCandidateTests
FailoverContinuityTests
FailoverFencingTests
FailoverPromotionTests
FailoverAuthorityTests
FailoverTopologyTests
FailoverConnectionInvalidationTests
FailoverTransactionTests
FailoverStickyTests
FailoverReplicaLagTests
FailoverRoutingTests
FailoverRecoveryTests
FailbackTests
FailoverPersistentRuntimeTests
FailoverConcurrencyTests
FailoverTelemetryTests
FailoverDiagnosticsTests
FailoverSecurityTests
```

---

# 222. Connection failure test

Una única connection failure no deberá implicar automáticamente promotion.

---

# 223. Reconnect test

Si same authority continúa disponible:

```text
RECONNECT
```

deberá ser posible sin failover.

---

# 224. Candidate freshness test

Candidate atrasado respecto de required durable position deberá ser rechazado bajo zero-loss policy.

---

# 225. Fencing test

Old writer no fenced:

```text
automatic promotion
→ reject
```

bajo strict fencing policy.

---

# 226. Split-brain test

Dos endpoints reportan writable authority:

```text
write routing
→ PAUSED
```

hasta resolver autoridad.

---

# 227. Promotion unknown test

Promotion request sin resultado verificable:

```text
state = UNKNOWN
```

---

# 228. Atomic topology test

Concurrent readers observarán solo generación N o N+1.

---

# 229. Stale connection test

Conexiones pertenecientes al old authority generation no deberán volver al pool normal.

---

# 230. Transaction test

Active transaction no migrará a new writer.

---

# 231. Unknown commit test

```text
COMMIT
→ connection loss
→ failover
```

deberá preservar transaction outcome `UNKNOWN`.

---

# 232. Retry safety test

UNKNOWN transaction no deberá ser replayed ciegamente.

---

# 233. Sticky continuity test

Required position P deberá verificarse contra new authority.

---

# 234. Sticky incomparable test

Old/new replication epochs incomparables deberán producir uncertainty.

---

# 235. Lag snapshot test

Lag snapshot de old topology generation deberá invalidarse.

---

# 236. Recovery test

Old writer recuperado no deberá recibir writes automáticamente.

---

# 237. Divergence test

Endpoint con divergent history deberá quedar fuera de topology activa hasta reconciliación.

---

# 238. Failback test

Old preferred writer recuperado no deberá provocar automatic failback por default.

---

# 239. Persistent worker test

Failover state de una operación no deberá filtrarse a requests posteriores.

---

# 240. Multi-worker coordination test

Dos workers intentando failover deberán converger mediante coordinator/provider, no mediante mutex local independiente.

---

# 241. Proposed directory structure

```text
src/Quantum/Database/Failover/
│
├── DatabaseFailoverCoordinator.php
├── DatabaseFailoverPolicy.php
├── FailoverMode.php
├── FailoverState.php
├── FailoverTrigger.php
├── FailoverOperationId.php
│
├── Context/
│   ├── FailoverContext.php
│   ├── FailoverExecutionContext.php
│   └── FailoverContextFactory.php
│
├── Failure/
│   ├── DatabaseFailureDetector.php
│   ├── DatabaseFailureEvidence.php
│   ├── DatabaseFailureKind.php
│   ├── DatabaseFailureScope.php
│   ├── FailureConfidence.php
│   ├── FailureEvidenceSet.php
│   └── DatabaseFailureAssessment.php
│
├── Decision/
│   ├── FailoverDecision.php
│   ├── FailoverDecisionType.php
│   └── FailoverEvaluator.php
│
├── Candidate/
│   ├── PromotionCandidate.php
│   ├── PromotionCandidateSelector.php
│   ├── PromotionCandidateEvaluator.php
│   ├── PromotionCandidateEvaluation.php
│   ├── PromotionCandidateStatus.php
│   ├── DataContinuityEvidence.php
│   ├── DataContinuityStatus.php
│   └── DataLossPolicy.php
│
├── Fencing/
│   ├── DatabaseFencingManager.php
│   ├── FencingStrategy.php
│   ├── FencingPolicy.php
│   ├── FencingResult.php
│   ├── FencingStatus.php
│   ├── AuthorityLease.php
│   └── AuthorityEpoch.php
│
├── Promotion/
│   ├── DatabasePromotionAdapter.php
│   ├── PromotionResult.php
│   ├── PromotionStatus.php
│   ├── AuthorityEvidence.php
│   └── AuthorityConfidence.php
│
├── Provider/
│   ├── FailoverProvider.php
│   ├── DatabaseFailoverCapability.php
│   └── FailoverCapabilityMode.php
│
├── Topology/
│   ├── FailoverTopologyTransition.php
│   ├── TopologyTransitionValidator.php
│   └── TopologyPublisher.php
│
├── Connection/
│   ├── FailoverConnectionInvalidator.php
│   ├── ConnectionGenerationValidator.php
│   └── StaleConnectionQuarantine.php
│
├── Routing/
│   ├── FailoverRoutingBarrier.php
│   ├── FailoverBarrierSnapshot.php
│   ├── WriteAvailabilityState.php
│   └── DatabaseAvailabilityMode.php
│
├── Coordination/
│   ├── FailoverCoordinationLease.php
│   ├── FailoverCoordinationProvider.php
│   └── FailoverSingleFlightCoordinator.php
│
├── Reconciliation/
│   ├── StickyFailoverReconciler.php
│   ├── StickyFailoverReconciliationStatus.php
│   ├── TransactionFailoverReconciler.php
│   └── ReplicaLagFailoverReconciler.php
│
├── Recovery/
│   ├── DatabaseRecoveryManager.php
│   ├── RecoveredEndpointState.php
│   ├── FailbackManager.php
│   └── FailbackPolicy.php
│
├── Diagnostics/
│   ├── FailoverInspector.php
│   ├── FailoverExplainer.php
│   └── FailoverDiagnosticReport.php
│
├── Telemetry/
│   ├── FailoverTelemetry.php
│   ├── DatabaseFailureSuspected.php
│   ├── FailoverPromotionStarted.php
│   ├── FailoverTopologyPublished.php
│   ├── FailoverCompleted.php
│   └── FailoverUnknown.php
│
└── Exception/
    ├── DatabaseFailoverException.php
    ├── FailoverEvidenceInsufficientException.php
    ├── FailoverCandidateNotFoundException.php
    ├── FailoverFencingException.php
    ├── FailoverPromotionException.php
    ├── FailoverPromotionUnknownException.php
    ├── FailoverSplitBrainRiskException.php
    ├── FailoverTopologyConflictException.php
    ├── FailoverDataLossRiskException.php
    └── FailoverInvariantViolationException.php
```

---

# 242. Invariantes arquitectónicos

## DB-FOV-001
Connection failure no equivaldrá a node failure.

## DB-FOV-002
Node failure no equivaldrá automáticamente a primary failure.

## DB-FOV-003
Primary failure no equivaldrá automáticamente a safe failover.

## DB-FOV-004
Failover será distinto de reconnect.

## DB-FOV-005
Failover será distinto de retry.

## DB-FOV-006
Failover será distinto de load balancing.

## DB-FOV-007
Failover será distinto de recovery.

## DB-FOV-008
Failover será distinto de failback.

## DB-FOV-009
Failure evidence tendrá confidence explícita.

## DB-FOV-010
Failure evidence tendrá timestamp.

## DB-FOV-011
Failure evidence pertenecerá a topology generation.

## DB-FOV-012
Old failure evidence no iniciará automáticamente new failover.

## DB-FOV-013
Timeout aislado no demostrará primary failure.

## DB-FOV-014
Unreachable old writer no significará fenced old writer.

## DB-FOV-015
Automatic promotion requerirá fencing adecuado.

## DB-FOV-016
UNKNOWN fencing impedirá strict automatic promotion.

## DB-FOV-017
Split-brain será una critical invariant violation.

## DB-FOV-018
Temporary write unavailability será preferible a uncertain dual authority.

## DB-FOV-019
Failover no fingirá distributed consensus.

## DB-FOV-020
Manual failover seguirá validando invariants.

## DB-FOV-021
Provider-managed failover seguirá requiriendo local reconciliation.

## DB-FOV-022
Provider success response no equivaldrá automáticamente a verified authority.

## DB-FOV-023
Writable new node no probará exclusive authority.

## DB-FOV-024
Authority evidence será explícita.

## DB-FOV-025
Authority confidence será explícita.

## DB-FOV-026
Promotion candidate será distinto de ordinary replica.

## DB-FOV-027
Candidate discovery utilizará current topology.

## DB-FOV-028
Candidate evaluation precederá candidate ranking.

## DB-FOV-029
Safety gates precederán performance preferences.

## DB-FOV-030
Low latency no compensará unsafe promotion.

## DB-FOV-031
Replica lag zero no probará automáticamente complete continuity.

## DB-FOV-032
Replication position será utilizada cuando ofrezca evidencia superior.

## DB-FOV-033
Data continuity tendrá estado explícito.

## DB-FOV-034
UNKNOWN continuity no será VERIFIED.

## DB-FOV-035
Known gap será distinto de possible gap.

## DB-FOV-036
Possible gap será distinto de unknown.

## DB-FOV-037
Zero-known-loss será default conceptual.

## DB-FOV-038
Bounded data loss requerirá opt-in explícito.

## DB-FOV-039
RPO será distinto de current lag.

## DB-FOV-040
Failover operation tendrá identidad propia.

## DB-FOV-041
Failover operation tendrá state machine explícita.

## DB-FOV-042
UNKNOWN failover outcome será first-class.

## DB-FOV-043
Promotion timeout no implicará promotion failure.

## DB-FOV-044
Promotion command accepted no implicará completed failover.

## DB-FOV-045
Authority verification ocurrirá después de promotion cuando sea posible.

## DB-FOV-046
Topology transition será validada.

## DB-FOV-047
Topology snapshot será immutable.

## DB-FOV-048
Topology publication será atómica.

## DB-FOV-049
Requests no observarán partial topology.

## DB-FOV-050
Successful failover producirá new topology generation.

## DB-FOV-051
TopologyGeneration será distinta de AuthorityEpoch.

## DB-FOV-052
AuthorityEpoch será distinta de ReplicationEpoch.

## DB-FOV-053
ReplicationEpoch será distinta de TopologyGeneration.

## DB-FOV-054
Stale routing generation será detectable.

## DB-FOV-055
Old authority connections serán invalidables.

## DB-FOV-056
Stale pooled connections no serán reutilizadas ciegamente.

## DB-FOV-057
Connection local role no se mutará para fingir topology transition.

## DB-FOV-058
Connection failure durante write podrá producir UNKNOWN.

## DB-FOV-059
UNKNOWN write no será asumido FAILED.

## DB-FOV-060
UNKNOWN write no será asumido SUCCESS.

## DB-FOV-061
Unknown write no será replayed ciegamente.

## DB-FOV-062
Active physical transaction no migrará entre connections.

## DB-FOV-063
Failover no recreará server-side transaction state.

## DB-FOV-064
Failover no recreará transaction locks.

## DB-FOV-065
Failover no recreará savepoints.

## DB-FOV-066
Failover no recreará transaction snapshot.

## DB-FOV-067
Commit sent + lost connection podrá permanecer UNKNOWN.

## DB-FOV-068
UNKNOWN commit no será replayed automáticamente.

## DB-FOV-069
TransactionManager recibirá authority-loss evidence.

## DB-FOV-070
Persistence context podrá quedar TAINTED.

## DB-FOV-071
Database rollback no rebobinará object graph.

## DB-FOV-072
Transaction retry seguirá las reglas del Transaction Retry System.

## DB-FOV-073
Retry después de failover requerirá stable new authority.

## DB-FOV-074
UNKNOWN prior outcome bloqueará blind retry.

## DB-FOV-075
Sticky requirements sobrevivirán topology transition hasta reconciliación.

## DB-FOV-076
New writer no satisfará sticky requirement solo por ser writer.

## DB-FOV-077
Required position deberá compararse cuando sea posible.

## DB-FOV-078
Candidate behind required confirmed position indicará continuity problem.

## DB-FOV-079
Incomparable position no será considerada satisfied.

## DB-FOV-080
Failover no limpiará sticky state silenciosamente.

## DB-FOV-081
Portable consistency tokens deberán revalidarse.

## DB-FOV-082
Old replication epoch podrá invalidar token comparability.

## DB-FOV-083
Old lag snapshot será invalidado por topology generation change.

## DB-FOV-084
Promoting replica no recibirá normal writes antes de authority verification.

## DB-FOV-085
Write routing podrá pausarse durante authority transition.

## DB-FOV-086
Write pause será explícita.

## DB-FOV-087
Read-only degraded mode será explícito.

## DB-FOV-088
Failover System no será Query Planner.

## DB-FOV-089
Failover System no será SQL Compiler.

## DB-FOV-090
Failover System no será ORM.

## DB-FOV-091
Failover System no será TransactionManager.

## DB-FOV-092
Failover System no será LoadBalancer.

## DB-FOV-093
Failover System no será ReplicaLagObserver.

## DB-FOV-094
Data plane no ejecutará promotion directamente.

## DB-FOV-095
Control plane será separado del query hot path.

## DB-FOV-096
Query failure podrá producir bounded failure signal.

## DB-FOV-097
Failure signals serán rate-limited.

## DB-FOV-098
Failure signals serán deduplicables.

## DB-FOV-099
Failover probes no se ejecutarán en cada query.

## DB-FOV-100
Automatic failover será single-flight por authority domain.

## DB-FOV-101
Local mutex no se presentará como distributed coordination.

## DB-FOV-102
Multi-node failover requerirá external/atomic coordination.

## DB-FOV-103
Coordination lease tendrá expiration.

## DB-FOV-104
Coordination lease podrá tener fencing token.

## DB-FOV-105
Lost coordination lease detendrá operación donde sea seguro.

## DB-FOV-106
Uncertain operation después de lease loss permanecerá UNKNOWN.

## DB-FOV-107
Repeated provider operation deberá distinguir same operation de new attempt.

## DB-FOV-108
Failover cooldown reducirá flapping.

## DB-FOV-109
Cooldown no sustituirá fencing.

## DB-FOV-110
Failure grace period no sustituirá authority proof.

## DB-FOV-111
Recovered old writer no será promovido automáticamente.

## DB-FOV-112
Recovered old writer deberá permanecer fenced durante validation.

## DB-FOV-113
Rejoin requerirá replication compatibility.

## DB-FOV-114
Divergent histories no se fusionarán automáticamente.

## DB-FOV-115
Recovery podrá requerir full rebuild/resync.

## DB-FOV-116
Recovery será distinta de failback.

## DB-FOV-117
Automatic failback estará deshabilitado por default.

## DB-FOV-118
Failback pasará por authority safety checks.

## DB-FOV-119
Replica read failure normalmente eliminará candidate, no promoverá autoridad.

## DB-FOV-120
All replicas failed no implicará writer failover si writer está sano.

## DB-FOV-121
Writer fallback podrá servir reads cuando replicas estén indisponibles.

## DB-FOV-122
Failure domains podrán representarse semánticamente.

## DB-FOV-123
Candidate outside failed domain podrá preferirse tras safety gates.

## DB-FOV-124
Region preference no superará data continuity.

## DB-FOV-125
DBMS vendor será distinto de failover mechanism.

## DB-FOV-126
Deployment topology será distinta de DBMS.

## DB-FOV-127
Provider capability determinará failover operations disponibles.

## DB-FOV-128
Core no contendrá vendor failover conditionals dispersos.

## DB-FOV-129
Unsupported automatic failover será explícito.

## DB-FOV-130
Configuration incompatible fallará temprano.

## DB-FOV-131
Strict fencing + unsupported fencing no activará unsafe automation.

## DB-FOV-132
Immutable shared topology será segura para persistent workers.

## DB-FOV-133
Mutable failover execution state será operation-scoped.

## DB-FOV-134
FrankenPHP no filtrará mutable failover state entre requests.

## DB-FOV-135
RoadRunner no filtrará mutable failover state entre jobs/requests.

## DB-FOV-136
OpenSwoole no permitirá concurrent mutation de published topology.

## DB-FOV-137
Snapshot updates utilizarán build-and-swap.

## DB-FOV-138
Shared snapshot no será mutado field-by-field.

## DB-FOV-139
Process-local snapshot no equivaldrá a global coordination.

## DB-FOV-140
External source of truth será explícita.

## DB-FOV-141
Telemetry será observational.

## DB-FOV-142
Telemetry listener no podrá promover database.

## DB-FOV-143
Telemetry listener no podrá fencear endpoint.

## DB-FOV-144
Telemetry listener no cambiará topology.

## DB-FOV-145
Telemetry no expondrá credentials.

## DB-FOV-146
Telemetry no expondrá provider secrets.

## DB-FOV-147
Telemetry no expondrá query parameters.

## DB-FOV-148
Diagnostics distinguirán failure, promotion y authority verification.

## DB-FOV-149
Diagnostics distinguirán known y unknown continuity.

## DB-FOV-150
Diagnostics mostrarán write availability state.

## DB-FOV-151
Diagnostics mostrarán topology generation.

## DB-FOV-152
Diagnostics mostrarán fencing status.

## DB-FOV-153
Diagnostics mostrarán failover operation state.

## DB-FOV-154
Metrics de UNKNOWN no se registrarán como success o failure.

## DB-FOV-155
RTO observado será distinto de RTO objetivo.

## DB-FOV-156
Failover history será bounded.

## DB-FOV-157
Failure evidence history será bounded.

## DB-FOV-158
Candidate diagnostics no retendrán application data.

## DB-FOV-159
Failover context no retendrá entities.

## DB-FOV-160
Failover context no retendrá QueryResults.

## DB-FOV-161
Failover context no retendrá UnitOfWork.

## DB-FOV-162
Failover context no retendrá credentials.

## DB-FOV-163
Connection invalidation será idempotente.

## DB-FOV-164
Topology publication será idempotente por generation/operation.

## DB-FOV-165
Recovery operations serán idempotentes cuando provider lo permita.

## DB-FOV-166
Write barrier será fail-closed ante unknown authority.

## DB-FOV-167
Read routing no asumirá que new writer contiene every old confirmed write.

## DB-FOV-168
Confirmed old write + new authority continuity unknown permanecerá una inconsistencia observable.

## DB-FOV-169
Failover nunca fabricará una replication position.

## DB-FOV-170
Failover nunca fabricará authority epoch.

## DB-FOV-171
Failover nunca fabricará fencing confirmation.

## DB-FOV-172
Failover nunca fabricará successful transaction outcome.

## DB-FOV-173
Correctness tendrá prioridad sobre transparent availability.

## DB-FOV-174
Exclusive write authority tendrá prioridad sobre automatic convenience.

## DB-FOV-175
Cuando VoltStack no pueda demostrar una transición segura de autoridad, deberá preservar la incertidumbre y bloquear operaciones que dependan de una garantía no demostrada.

---

# 243. Modelo formal de failover

Sea:

```text
A₀ = current authority
C  = candidate
F  = failure evidence
P  = failover policy
```

Una promoción automática solo será válida si:

```text
MayPromote(C)
=
FailureSufficient(F, P)
∧
CandidateEligible(C, P)
∧
ContinuityAcceptable(C, P)
∧
FencingSatisfied(A₀, P)
∧
CoordinationOwned()
```

Después de promotion:

```text
AuthorityEstablished(C)
=
PromotionConfirmed(C)
∧
AuthorityVerified(C)
∧
TopologyPublished(C)
```

No basta:

```text
PromotionCommandSent(C)
```

---

# 244. Modelo de continuidad

Sea:

```text
D = last required durable position
C = candidate replication position
```

Cuando son comparables:

```text
C >= D
→ continuity potentially verified
```

Si:

```text
C < D
```

entonces:

```text
Known/Possible Data Gap
```

dependiendo de la calidad de la evidencia.

Si:

```text
Compare(C,D) = INCOMPARABLE
```

entonces:

```text
Continuity = UNKNOWN
```

Nunca:

```text
UNKNOWN → VERIFIED
```

---

# 245. Modelo de seguridad contra split-brain

Para una autoridad de escritura:

```text
∀ t:
Count(ActiveWritableAuthorities(t)) <= 1
```

Cuando VoltStack no pueda demostrar esa propiedad:

```text
WriteAvailability(t)
=
PAUSED | UNAVAILABLE | UNKNOWN
```

según policy.

No:

```text
route writes optimistically
```

---

# 246. Flujo completo

```text
                         FAILURE SIGNALS
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
             Queries       Health Probe     Provider
                │              │              │
                └──────────────┼──────────────┘
                               ▼
                        Failure Evidence
                               │
                               ▼
                       Failover Evaluator
                               │
                   ┌───────────┴───────────┐
                   │                       │
               NO FAILOVER             FAILOVER
                   │                       │
                   ▼                       ▼
              Keep topology       Discover Candidates
                                           │
                                           ▼
                                    Safety Evaluation
                                           │
                               ┌───────────┴──────────┐
                               │                      │
                           NO SAFE                 SAFE
                           CANDIDATE              CANDIDATE
                               │                      │
                               ▼                      ▼
                        Writes unavailable          Fence
                                                      │
                                                      ▼
                                                   Promote
                                                      │
                                                      ▼
                                                Verify Authority
                                                      │
                                        ┌─────────────┴────────────┐
                                        │                          │
                                     UNKNOWN                    VERIFIED
                                        │                          │
                                        ▼                          ▼
                                  Pause Writes              Build Topology
                                                                   │
                                                                   ▼
                                                             Atomic Publish
                                                                   │
                                                                   ▼
                                                        Invalidate Connections
                                                                   │
                                                                   ▼
                                                         Reconcile Transactions
                                                                   │
                                                                   ▼
                                                          Reconcile Sticky State
                                                                   │
                                                                   ▼
                                                               Resume Routing
```

---

# 247. Arquitectura resultante

Con:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
```

el subsistema distribuido de VoltStack comienza a formar:

```text
                    Database Distribution Layer
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    Connection Roles    Replica Topology      Failover
          │                   │                   │
          ▼                   ▼                   ▼
    Read/Write Route      Lag/Freshness       Authority
          │                   │              Transitions
          └──────────────┬────┴───────┬───────────┘
                         ▼            ▼
                  Sticky Consistency
                         │
                         ▼
                  Candidate Eligibility
                         │
                         ▼
                    Query Execution
```

Esto prepara el siguiente componente:

```text
Eligible Candidates
        │
        ▼
Load Balancing
```

---

# 248. Regla maestra final

> **El Database Failover System de VoltStack tratará el cambio de autoridad como una transición distribuida de seguridad y consistencia, no como un simple cambio de hostname. La promoción de un nuevo writer deberá considerar evidencia de fallo, fencing, continuidad de datos, coordinación, generación de topología y estado transaccional. Cuando cualquiera de esas propiedades esenciales sea incierta, VoltStack preservará explícitamente esa incertidumbre y preferirá pausar escrituras antes que crear split-brain, duplicar operaciones o afirmar que datos no demostrados sobrevivieron.**

En forma compacta:

```text
Failure
   +
Evidence
   +
Candidate
   +
Continuity
   +
Fencing
   +
Coordination
   +
Authority Verification
        │
        ▼
Safe Topology Transition
```

Nunca:

```text
connection failed
      ↓
pick another host
      ↓
call it primary
```

---

# 249. Siguiente documento

```text
182_DATABASE_LOAD_BALANCING_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack distribuirá operaciones entre múltiples endpoints **después** de aplicar todos los filtros de seguridad y consistencia:

```text
All Endpoints
      │
      ▼
Role Filtering
      │
      ▼
Transaction Constraints
      │
      ▼
Tenant / Shard Constraints
      │
      ▼
Replica State
      │
      ▼
Freshness / Lag
      │
      ▼
Sticky Requirements
      │
      ▼
Failover / Authority State
      │
      ▼
Eligible Candidate Set
      │
      ▼
LOAD BALANCER
      │
      ├── Round Robin
      ├── Weighted
      ├── Least Load
      ├── Latency Aware
      ├── Locality Aware
      ├── Health Weighted
      └── Adaptive
      │
      ▼
Selected Endpoint
```

manteniendo como regla:

```text
Load Balancing
≠
Eligibility
```

y especialmente:

```text
A load balancer may choose
only among endpoints that
have already been proven eligible.
```