# 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md

# VoltStack Quantum Database
## Database Failover and Recovery System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 240 — Database Failover and Recovery System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de **Failover y Recovery** de VoltStack Database.

El sistema deberá permitir que VoltStack responda de forma segura ante:

- pérdida de conexiones;
- caída de endpoints;
- degradación de réplicas;
- pérdida del writer;
- cambios de topología;
- failover administrado por infraestructura externa;
- promoción de réplicas;
- reemplazo de nodos;
- fallos parciales de shards;
- recuperación posterior;
- reintegración de nodos;
- resultados transaccionales inciertos.

La regla central será:

> **Failover en VoltStack no significa simplemente “usar otro servidor”. Significa seleccionar una nueva ruta únicamente cuando esa ruta posee autoridad válida para la operación y cuando el cambio no inventa el resultado de operaciones anteriores, no viola consistencia, no rompe shard ownership y no crea una segunda autoridad concurrente.**

En particular:

```text
Failover
≠
Retry

Failover
≠
Reconnect

Failover
≠
Load Balancing

Failover
≠
Circuit Breaker

Failover
≠
Replica Selection

Failover
≠
Recovery

Failover
≠
Automatic Writer Promotion
```

---

# 2. Problema fundamental

Considérese:

```text
Application
    │
    ▼
Writer A
    │
    ├── Replica B
    └── Replica C
```

Writer A deja de responder.

Una implementación ingenua podría hacer:

```text
Writer A fails
      ↓
choose Replica B
      ↓
send writes to B
```

Esto es incorrecto.

La aplicación no sabe necesariamente:

```text
¿A realmente dejó de ser writer?

¿B fue promovida?

¿B está sincronizada?

¿existe un nuevo authority epoch?

¿A todavía acepta writes?

¿el último COMMIT llegó a A?

¿la infraestructura ya realizó failover?

¿B conoce ese COMMIT?

¿A y B podrían aceptar writes simultáneamente?
```

El resultado podría ser:

```text
Split Brain
```

o:

```text
Lost Writes
```

o:

```text
Duplicate Operations
```

---

# 3. Objetivos

El sistema deberá modelar explícitamente:

```text
Failure Detection
        ↓
Failure Classification
        ↓
Failover Eligibility
        ↓
Authority Validation
        ↓
Candidate Selection
        ↓
Topology Transition
        ↓
Routing Reconciliation
        ↓
Connection Reconciliation
        ↓
Recovery
        ↓
Reintegration
```

---

# 4. Principio arquitectónico

VoltStack separará cuatro conceptos:

```text
Failure
Failover
Recovery
Failback
```

---

# 5. Failure

Significa:

> Existe evidencia de que un recurso no puede satisfacer correctamente una operación o función determinada.

No implica automáticamente que exista un reemplazo.

---

# 6. Failover

Significa:

> Transferir nuevas operaciones hacia un recurso alternativo que posee autoridad y capacidades válidas para asumirlas.

---

# 7. Recovery

Significa:

> Restablecer la capacidad operativa después de una degradación o failover.

---

# 8. Failback

Significa:

> Retornar tráfico o autoridad hacia un recurso previamente utilizado después de demostrar que puede reincorporarse de forma segura.

---

# 9. Arquitectura general

```text
                    Database Operation
                           │
                           ▼
                    Routing System
                           │
                           ▼
                   Current Topology
                           │
                           ▼
                    Target Endpoint
                           │
                           ▼
                      Execution
                           │
                    ┌──────┴──────┐
                    │             │
                 SUCCESS        FAILURE
                                  │
                                  ▼
                       Failure Classification
                                  │
                                  ▼
                        Resilience Evidence
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
                   Retry       Circuit       Failover
                     │            │            │
                     └────────────┼────────────┘
                                  ▼
                         Failover Coordinator
                                  │
                                  ▼
                         Authority Validator
                                  │
                                  ▼
                         Candidate Resolver
                                  │
                                  ▼
                         Failover Decision
                                  │
                  ┌───────────────┼───────────────┐
                  │               │               │
               REJECT          WAIT           SWITCH
                                                  │
                                                  ▼
                                      Topology Reconciliation
                                                  │
                                                  ▼
                                         Routing Update
                                                  │
                                                  ▼
                                           Recovery
```

---

# 10. Failover domains

VoltStack deberá distinguir el dominio del fallo.

```php
enum FailureDomain
{
    case CONNECTION;
    case ENDPOINT;
    case REPLICA;
    case WRITER;
    case REPLICATION_GROUP;
    case SHARD;
    case LOGICAL_DATABASE;
    case NETWORK_PATH;
    case TOPOLOGY;
    case UNKNOWN;
}
```

---

# 11. Connection failure

Una conexión rota no demuestra que:

```text
endpoint is down
```

Puede tratarse de:

```text
idle timeout
network reset
connection corruption
server-side termination
```

Por tanto:

```text
Connection Failure
≠
Endpoint Failure
```

---

# 12. Endpoint failure

Existe evidencia suficiente de que un endpoint específico no debe recibir nuevas operaciones.

Ejemplo:

```text
Replica B
    ↓
UNAVAILABLE
```

---

# 13. Replica failure

Una replica fallida normalmente afecta:

```text
read capacity
```

pero no necesariamente:

```text
write authority
```

---

# 14. Writer failure

Es una condición mucho más delicada.

```text
Writer Unreachable
```

no equivale a:

```text
Writer No Longer Authoritative
```

---

# 15. Replication group failure

Ejemplo:

```text
Replication Group
├── Writer A     DOWN
├── Replica B    DOWN
└── Replica C    DOWN
```

Puede requerir elevar el fallo al nivel:

```text
REPLICATION_GROUP
```

---

# 16. Shard failure

En una arquitectura sharded:

```text
Shard 1 → Healthy
Shard 2 → Failed
Shard 3 → Healthy
```

VoltStack deberá preservar el aislamiento.

```text
Shard 2 Failure
≠
Global Database Failure
```

---

# 17. Logical database failure

Solo se utilizará cuando la infraestructura que representa una base lógica completa no pueda satisfacer operaciones válidas.

---

# 18. Failure evidence

El Failover System no dependerá de una sola señal.

Podrá consumir:

```text
Connection Failure Evidence
Query Failure Evidence
Circuit Breaker State
Health Evidence
Replication Evidence
Topology Evidence
Authority Evidence
Administrative Evidence
```

---

# 19. Evidence model

```php
final readonly class FailoverEvidence
{
    public function __construct(
        public FailureDomain $domain,
        public EndpointId|null $endpoint,
        public FailureCategory $category,
        public Confidence $confidence,
        public Instant $observedAt,
        public array $attributes = [],
    ) {}
}
```

---

# 20. Evidence confidence

```php
enum FailoverEvidenceConfidence
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case AUTHORITATIVE;
    case UNKNOWN;
}
```

---

# 21. UNKNOWN ≠ FAILED

Una regla esencial:

```text
UNKNOWN
≠
UNAVAILABLE
```

Si VoltStack perdió conectividad con un writer:

```text
Writer state = UNKNOWN
```

puede ser más correcto que:

```text
Writer state = FAILED
```

---

# 22. Endpoint operational state

```php
enum EndpointOperationalState
{
    case HEALTHY;
    case DEGRADED;
    case SUSPECTED;
    case UNAVAILABLE;
    case RECOVERING;
    case QUARANTINED;
    case UNKNOWN;
}
```

---

# 23. Endpoint role

Separado del estado:

```php
enum EndpointRole
{
    case WRITER;
    case REPLICA;
    case STANDBY;
    case UNKNOWN;
}
```

---

# 24. Role ≠ Authority

Un endpoint configurado como:

```text
WRITER
```

no necesariamente posee actualmente autoridad válida.

---

# 25. Authority Model

Para operaciones mutables, VoltStack deberá modelar explícitamente:

```text
Write Authority
```

---

# 26. Authority record

Conceptualmente:

```php
final readonly class AuthorityRecord
{
    public function __construct(
        public LogicalDatabaseId $database,
        public EndpointId $endpoint,
        public AuthorityEpoch $epoch,
        public AuthorityStatus $status,
        public Instant $observedAt,
        public Confidence $confidence,
    ) {}
}
```

---

# 27. AuthorityStatus

```php
enum AuthorityStatus
{
    case AUTHORITATIVE;
    case NON_AUTHORITATIVE;
    case TRANSITIONING;
    case UNKNOWN;
}
```

---

# 28. Authority epoch

Cada cambio válido de writer podrá incrementar:

```text
AuthorityEpoch
```

Ejemplo:

```text
Epoch 17 → Writer A
Epoch 18 → Writer B
```

---

# 29. AuthorityEpoch

```php
final readonly class AuthorityEpoch
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 30. Purpose of epoch

Permite distinguir:

```text
old authority
```

de:

```text
current authority
```

---

# 31. Fencing

Cuando la infraestructura lo soporte, deberá utilizarse:

```text
Fencing
```

para impedir que el writer anterior continúe realizando writes.

---

# 32. Fencing token

Conceptualmente:

```php
final readonly class FencingToken
{
    public function __construct(
        public string $value,
        public AuthorityEpoch $epoch,
    ) {}
}
```

---

# 33. FencingToken ≠ Credential

El token representa autoridad temporal/generacional, no identidad de usuario.

---

# 34. Split-brain prevention

La prioridad para write failover será:

```text
Prevent Split Brain
>
Restore Write Availability
```

---

# 35. Fundamental write rule

VoltStack no deberá hacer:

```text
Writer A unreachable
        ↓
Replica B reachable
        ↓
send write to B
```

sin evidencia de que B posee:

```text
valid write authority
```

---

# 36. External failover manager

En muchos despliegues, la promoción real será realizada por infraestructura externa:

```text
PostgreSQL HA manager
MySQL HA manager
cloud managed database
Kubernetes operator
proxy/router
cluster manager
```

VoltStack deberá poder integrarse con ella.

---

# 37. Application-level failover

VoltStack podrá realizar:

```text
application routing failover
```

sin asumir necesariamente:

```text
database cluster promotion
```

---

# 38. Important separation

```text
Database Infrastructure Failover
≠
Application Routing Failover
```

---

# 39. FailoverAuthorityProvider

```php
interface FailoverAuthorityProvider
{
    public function currentAuthority(
        LogicalDatabaseId $database
    ): AuthorityRecord;
}
```

---

# 40. Static authority provider

Para instalaciones simples:

```text
single writer
```

podrá existir un provider basado en configuración.

---

# 41. Dynamic authority provider

Para HA:

```text
cluster topology
managed DB
service discovery
external HA controller
```

podrá aportar autoridad dinámica.

---

# 42. Authority provider failure

Si no puede determinarse quién es writer:

```text
Authority = UNKNOWN
```

VoltStack no deberá inventarlo.

---

# 43. Failover candidate

```php
final readonly class FailoverCandidate
{
    public function __construct(
        public EndpointId $endpoint,
        public EndpointRole $role,
        public EndpointOperationalState $state,
        public AuthorityStatus $authority,
        public ReplicationState|null $replication,
        public CapabilitySet $capabilities,
    ) {}
}
```

---

# 44. Candidate ≠ Eligible Candidate

Un endpoint descubierto no significa que pueda utilizarse.

---

# 45. Candidate eligibility

Conceptualmente:

```text
Eligible =
    RoleValid
  ∧ OperationallyAvailable
  ∧ CircuitEligible
  ∧ AuthorityValid
  ∧ ConsistencyCompatible
  ∧ ShardCompatible
  ∧ CapabilityCompatible
```

---

# 46. Read failover

Es generalmente menos peligroso.

Ejemplo:

```text
Replica A fails
      ↓
Replica B healthy
      ↓
read may route to B
```

si:

```text
freshness requirement satisfied
consistency requirement satisfied
circuit closed
shard correct
```

---

# 47. Read failover ≠ arbitrary replica switch

Si una operación requiere:

```text
READ_YOUR_WRITES
```

no puede enviarse a una replica atrasada simplemente porque está disponible.

---

# 48. Writer fallback for reads

Dependiendo de policy:

```text
Replica unavailable
      ↓
Writer
```

podrá ser válido para reads.

---

# 49. Writer fallback policy

```php
enum ReadFailoverPolicy
{
    case OTHER_REPLICA;
    case WRITER;
    case OTHER_REPLICA_THEN_WRITER;
    case FAIL;
    case CUSTOM;
}
```

---

# 50. Write failover

Mucho más restrictivo.

Requiere:

```text
Authority validation
```

antes del routing.

---

# 51. Write failover eligibility

Conceptualmente:

```text
WriteFailoverEligible(T)
=
T.authority = AUTHORITATIVE
∧
T.role = WRITER
∧
T.operationalState allows writes
∧
T.circuit allows writes
∧
T.shard ownership valid
```

---

# 52. Replica promotion

VoltStack core no deberá asumir que puede ejecutar:

```text
PROMOTE REPLICA
```

en cualquier plataforma.

---

# 53. Promotion abstraction

Si se soporta:

```php
interface DatabasePromotionProvider
{
    public function promote(
        PromotionRequest $request
    ): PromotionResult;
}
```

Será una integración opcional.

---

# 54. Promotion ≠ failover complete

Después de promoción se requiere:

```text
Authority Confirmation
Topology Reconciliation
Routing Update
Connection Reconciliation
```

---

# 55. Planned switchover

Failover planificado:

```text
Writer A
   ↓
drain
   ↓
replication catch-up
   ↓
fence A
   ↓
promote B
   ↓
authority epoch++
   ↓
route to B
```

---

# 56. Emergency failover

Puede ocurrir cuando:

```text
Writer A unavailable
```

sin oportunidad de graceful drain.

Mayor riesgo de:

```text
unknown writes
replication loss
authority ambiguity
```

---

# 57. Automatic failover

VoltStack podrá permitirlo únicamente cuando el deployment pueda aportar suficientes garantías.

---

# 58. Manual failover

En infraestructuras sin autoridad automatizada, VoltStack podrá requerir intervención administrativa.

---

# 59. FailoverMode

```php
enum FailoverMode
{
    case DISABLED;
    case MANUAL;
    case EXTERNAL;
    case AUTOMATIC;
}
```

---

# 60. EXTERNAL

Significa:

> La infraestructura cambia la autoridad; VoltStack detecta/reconcilia el nuevo topology state.

Este será un modelo importante para servicios administrados.

---

# 61. Failover coordinator

```php
interface FailoverCoordinator
{
    public function evaluate(
        FailoverRequest $request
    ): FailoverDecision;
}
```

---

# 62. FailoverRequest

```php
final readonly class FailoverRequest
{
    public function __construct(
        public FailureDomain $domain,
        public OperationIntent $intent,
        public FailoverEvidenceSet $evidence,
        public DatabaseContext $context,
    ) {}
}
```

---

# 63. FailoverDecision

```php
enum FailoverDecisionType
{
    case KEEP_CURRENT;
    case USE_ALTERNATIVE;
    case WAIT_FOR_RECOVERY;
    case REQUIRE_AUTHORITY_REFRESH;
    case REQUIRE_MANUAL_INTERVENTION;
    case FAIL;
    case UNKNOWN;
}
```

---

# 64. Decision result

```php
final readonly class FailoverDecision
{
    public function __construct(
        public FailoverDecisionType $type,
        public EndpointId|null $target,
        public string $reason,
        public Confidence $confidence,
    ) {}
}
```

---

# 65. UNKNOWN decision

No deberá convertirse silenciosamente en:

```text
USE_ALTERNATIVE
```

---

# 66. Failover state machine

```text
                 ┌────────────┐
                 │  HEALTHY   │
                 └─────┬──────┘
                       │
                  failure evidence
                       ▼
                 ┌────────────┐
                 │ DEGRADED   │
                 └─────┬──────┘
                       │
                 failover required
                       ▼
                ┌──────────────┐
                │ FAILING_OVER │
                └──────┬───────┘
                       │
             authority/topology valid
                       ▼
                 ┌────────────┐
                 │ RECOVERING │
                 └─────┬──────┘
                       │
                 recovery proven
                       ▼
                 ┌────────────┐
                 │  HEALTHY   │
                 └────────────┘
```

Con ramas:

```text
DEGRADED
   ↓
UNKNOWN

FAILING_OVER
   ↓
FAILED

RECOVERING
   ↓
DEGRADED
```

---

# 67. RecoveryState

```php
enum RecoveryState
{
    case HEALTHY;
    case DEGRADED;
    case FAILING_OVER;
    case RECOVERING;
    case FAILED;
    case UNKNOWN;
}
```

---

# 68. Failover generation

Cada transición importante podrá poseer:

```text
FailoverGeneration
```

para evitar aplicar información obsoleta.

---

# 69. Topology snapshot

```php
final readonly class DatabaseTopologySnapshot
{
    public function __construct(
        public TopologyGeneration $generation,
        public AuthorityEpoch $authorityEpoch,
        public array $endpoints,
        public Instant $observedAt,
    ) {}
}
```

---

# 70. Stale topology

Una decisión basada en:

```text
generation 52
```

no deberá aplicarse ciegamente si la topología actual es:

```text
generation 57
```

---

# 71. Compare-and-transition

Cuando exista estado compartido:

```text
expected generation
→ transition
→ new generation
```

deberá ser atómico o equivalentemente protegido.

---

# 72. Connection reconciliation

Después de failover:

```text
old connections
```

pueden apuntar al writer anterior.

---

# 73. Old connection risk

Aunque routing ya conozca:

```text
Writer B
```

un pool podría contener:

```text
Connection → Writer A
```

---

# 74. Pool reconciliation

El Connection Pool deberá poder:

```text
mark old authority connections stale
stop leasing them for writes
drain them
close them according to policy
```

---

# 75. Connection generation

Cada conexión podrá asociarse conceptualmente con:

```text
TopologyGeneration
AuthorityEpoch
```

observados al crearse.

---

# 76. Connection validation

Antes de utilizar una conexión para write:

```text
ConnectionAuthorityEpoch
```

podrá compararse con:

```text
CurrentAuthorityEpoch
```

cuando el deployment lo permita.

---

# 77. Stale writer connection

Nunca deberá utilizarse silenciosamente para nuevos writes.

---

# 78. DNS failover

Un hostname puede cambiar:

```text
db-writer.internal
```

de:

```text
10.0.0.10
```

a:

```text
10.0.0.20
```

---

# 79. DNS ≠ authority proof

Resolver una IP diferente no demuestra por sí solo:

```text
new writer authority
```

---

# 80. Proxy failover

Cuando VoltStack se conecta a:

```text
database proxy
```

la topología física puede quedar oculta.

En ese caso:

```text
proxy endpoint
```

puede ser la unidad de routing visible.

---

# 81. Managed database

Un proveedor cloud puede mantener:

```text
stable writer endpoint
```

y cambiar internamente el nodo.

VoltStack deberá permitir este modelo sin exigir conocer cada nodo físico.

---

# 82. Abstraction boundary

```text
Application-visible endpoint
≠
Physical database node
```

---

# 83. Retry integration

Failover y retry deberán coordinarse.

```text
Operation fails
      ↓
classify
      ↓
retry?
      │
      ├── same target
      └── alternate target
                │
                ▼
          failover eligibility
```

---

# 84. Retry ≠ Failover

Repetir:

```text
SELECT
```

sobre otra replica puede ser:

```text
retry + read rerouting
```

sin constituir writer failover.

---

# 85. Transaction retry after failover

Una transacción completa podrá ser reintentada sobre el nuevo writer únicamente cuando:

```text
previous outcome is known safe for replay
```

---

# 86. UNKNOWN commit

Caso crítico:

```text
COMMIT sent
    ↓
connection lost
```

VoltStack puede no saber:

```text
committed?
rolled back?
```

---

# 87. Fundamental rule

```text
UNKNOWN COMMIT
+
NEW WRITER
≠
SAFE RETRY
```

---

# 88. Commit ambiguity

Si se repite automáticamente:

```text
charge customer
```

podría ocurrir dos veces.

---

# 89. Recovery from UNKNOWN

Requiere mecanismos superiores como:

```text
idempotency keys
business operation identifiers
transaction status lookup
outbox
deduplication
application reconciliation
```

según el caso.

---

# 90. Failover never rewrites history

Cambiar de writer no permite a VoltStack declarar:

```text
previous transaction failed
```

si el resultado era UNKNOWN.

---

# 91. Transaction pinning

Una transaction activa permanece vinculada a:

```text
connection
endpoint
authority context
```

---

# 92. Mid-transaction failover

No se realizará:

```text
BEGIN on A
UPDATE on A
A fails
UPDATE on B
COMMIT on B
```

Esto sería incorrecto.

---

# 93. Correct behavior

```text
transaction on A
      ↓
A fails
      ↓
transaction fails / outcome classified
      ↓
rollback if possible
      ↓
whole-transaction retry decision
      ↓
possibly restart on B
```

solo cuando sea seguro.

---

# 94. Savepoints

No permiten continuar una transaction en otro servidor.

```text
Savepoint
≠
Distributed Transaction Migration
```

---

# 95. Circuit Breaker integration

Un endpoint:

```text
Circuit = OPEN
```

puede excluirse de nuevos targets.

---

# 96. Circuit OPEN ≠ Failover command

El Circuit Breaker no promoverá writers.

---

# 97. Recovery and circuit

Después de recuperación:

```text
Failover Recovery
```

y:

```text
Circuit HALF_OPEN
```

pueden coordinarse.

---

# 98. Recovered endpoint

Un endpoint no deberá recibir inmediatamente 100% del tráfico.

Podrá pasar por:

```text
QUARANTINED
→ PROBING
→ ELIGIBLE
```

---

# 99. Gradual recovery

Para replicas podrá utilizarse:

```text
0%
→ probes
→ low traffic
→ normal traffic
```

si el Load Balancer soporta weighted recovery.

---

# 100. Recovery ≠ Circuit Close

Son conceptos relacionados pero distintos.

---

# 101. Replica recovery

Una replica que vuelve debe demostrar:

```text
connection health
replication health
acceptable lag
correct topology membership
```

antes de recibir reads.

---

# 102. Replica catch-up

Estado conceptual:

```php
enum ReplicaRecoveryState
{
    case DISCONNECTED;
    case RECONNECTING;
    case CATCHING_UP;
    case VERIFYING;
    case READY;
    case FAILED;
    case UNKNOWN;
}
```

---

# 103. Catch-up ≠ ready

Una replica conectada pero atrasada:

```text
CATCHING_UP
```

no deberá participar en reads con freshness estricta.

---

# 104. Reintegration

Proceso:

```text
endpoint discovered
      ↓
health verification
      ↓
topology verification
      ↓
replication verification
      ↓
circuit recovery
      ↓
routing eligibility
      ↓
load balancing
```

---

# 105. Failback

Supongamos:

```text
Writer A failed
Writer B promoted
Writer A repaired
```

No deberá ocurrir:

```text
A returns
   ↓
automatically become writer again
```

---

# 106. Former writer

A deberá reincorporarse normalmente como:

```text
NON_AUTHORITATIVE
```

hasta que la infraestructura determine otra cosa.

---

# 107. Automatic failback

Deberá estar deshabilitado por default para writers.

---

# 108. Failback policy

```php
enum FailbackPolicy
{
    case NEVER_AUTOMATIC;
    case MANUAL;
    case EXTERNAL;
    case AUTOMATIC_WHEN_VERIFIED;
}
```

---

# 109. Replica failback

Es menos riesgoso.

Una replica recuperada puede volver al read pool después de demostrar:

```text
health
freshness
replication correctness
```

---

# 110. Writer failback

Requiere nuevamente:

```text
authority transition
fencing
epoch change
topology reconciliation
```

---

# 111. Flapping protection

Un endpoint puede oscilar:

```text
UP
DOWN
UP
DOWN
```

---

# 112. Stability window

VoltStack podrá exigir:

```text
healthy for N seconds
```

antes de reintegrarlo.

---

# 113. Recovery hysteresis

La condición para volver a HEALTHY puede ser más estricta que la condición para degradar.

Esto evita:

```text
state flapping
```

---

# 114. RecoveryPolicy

```php
interface RecoveryPolicy
{
    public function evaluate(
        RecoveryEvidence $evidence
    ): RecoveryDecision;
}
```

---

# 115. Recovery evidence

Podrá incluir:

```text
successful probes
replication lag
authority status
circuit status
connection success rate
query success rate
stability duration
```

---

# 116. RecoveryDecision

```php
enum RecoveryDecision
{
    case KEEP_QUARANTINED;
    case CONTINUE_PROBING;
    case REINTEGRATE;
    case REQUIRE_RESYNC;
    case REQUIRE_MANUAL_ACTION;
    case FAILED;
    case UNKNOWN;
}
```

---

# 117. Replica lag integration

Una replica recuperada:

```text
lag = UNKNOWN
```

no deberá tratarse como:

```text
lag = 0
```

---

# 118. Replication position

Cuando sea posible se utilizará:

```text
replication position
LSN
GTID
equivalent platform position
```

a través de abstracciones platform-specific.

---

# 119. Platform abstraction

El core trabajará con:

```text
ReplicationPosition
```

no con detalles de PostgreSQL/MySQL directamente.

---

# 120. Version ≠ Capability

VoltStack preguntará:

```php
$capabilities->supportsReplicationPosition();
```

en lugar de:

```php
if ($database === 'postgres') { ... }
```

en capas genéricas.

---

# 121. MySQL y MariaDB

Continuarán tratándose como plataformas separadas cuando sus semánticas diverjan.

---

# 122. PostgreSQL

Sus capacidades específicas se implementarán en su plataforma/adapter.

---

# 123. SQLite

Normalmente no tendrá failover de servidor tradicional.

El sistema deberá reconocer:

```text
FailoverCapability = UNSUPPORTED / NOT_APPLICABLE
```

en lugar de simularlo.

---

# 124. Capability model

```php
interface FailoverCapabilities
{
    public function supportsReplicaDiscovery(): bool;

    public function supportsPromotion(): bool;

    public function supportsAuthorityEpoch(): bool;

    public function supportsFencing(): bool;

    public function supportsReplicationPosition(): bool;

    public function supportsAutomatedFailover(): bool;
}
```

---

# 125. Capability status

Idealmente no será solo boolean.

```php
enum CapabilitySupport
{
    case SUPPORTED;
    case WITH_LIMITATIONS;
    case EXTERNAL;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 126. Read consistency

Failover nunca reducirá silenciosamente:

```text
STRONG
```

a:

```text
EVENTUAL
```

solo para encontrar un endpoint.

---

# 127. Consistency downgrade

Si se desea permitir:

```text
consistency downgrade
```

deberá ser policy explícita de una capa autorizada.

---

# 128. Read-your-writes

El sistema de:

```text
Sticky Connection
Replica Lag Awareness
Minimum Replication Position
```

deberá seguir aplicando después de failover.

---

# 129. Sticky state after writer change

Un sticky marker vinculado al writer anterior puede requerir reinterpretación.

---

# 130. Minimum observed position

Cuando sea posible deberá utilizarse una posición lógica comparable.

---

# 131. Incompatible position

Si no puede compararse la posición anterior con la nueva autoridad:

```text
Consistency = UNKNOWN
```

No se fingirá equivalencia.

---

# 132. Sharding

Failover se evaluará dentro de:

```text
Shard Ownership
```

---

# 133. Cross-shard substitution forbidden

Si:

```text
Customer 42 → Shard A
```

y Shard A falla, VoltStack no enviará la operación a:

```text
Shard B
```

solo porque B esté disponible.

---

# 134. Shard replica failover

Sí puede existir:

```text
Shard A / Replica 1
        ↓ failure
Shard A / Replica 2
```

si cumple las garantías requeridas.

---

# 135. Shard writer failover

Debe preservar:

```text
same shard ownership
+
new valid writer authority
```

---

# 136. Shard map generation

El failover deberá conservar:

```text
ShardMapGeneration
```

cuando sea relevante.

---

# 137. Resharding interaction

Failover durante:

```text
resharding
```

es especialmente complejo.

No se deberá inferir ownership únicamente de la topología anterior.

---

# 138. Distributed operations

Una operación sobre varios shards puede obtener:

```text
Shard A → SUCCESS
Shard B → FAILURE
Shard C → SUCCESS
```

---

# 139. Partial distributed outcome

No deberá presentarse como:

```text
SUCCESS
```

global.

---

# 140. No fake distributed rollback

Si no existe transacción distribuida:

```text
success on A
```

no puede deshacerse mágicamente porque B falló.

---

# 141. Recovery coordinator

Para operaciones distribuidas podrá requerirse una capa superior de:

```text
compensation
reconciliation
workflow recovery
```

pero no será fingida por Database core.

---

# 142. Multitenancy

Tenant context seguirá siendo obligatorio.

---

# 143. Failover does not change tenant

```text
Tenant A
→ Writer A
→ failover
→ Writer B
```

continúa siendo:

```text
Tenant A
```

---

# 144. Dedicated tenant databases

Si cada tenant posee su propia infraestructura:

```text
Tenant A DB → failure
Tenant B DB → healthy
```

los dominios de failover deberán mantenerse separados.

---

# 145. Shared tenant database

Si varios tenants comparten el mismo writer, el failover podrá afectar a todos.

---

# 146. Tenant metadata ≠ authority

No deberá utilizarse tenant metadata no confiable para declarar writers.

---

# 147. Cache integration

Failover puede afectar:

```text
Result Cache
Entity Cache
routing metadata cache
topology cache
```

---

# 148. Cache invalidation

No deberá ejecutarse:

```text
FLUSHALL
```

por default.

---

# 149. Topology cache

Sí deberá invalidarse/refrescarse cuando cambie:

```text
writer
replicas
authority epoch
topology generation
```

---

# 150. Result cache

Un cambio de writer no hace automáticamente inválidos todos los resultados cacheados.

La validez seguirá dependiendo del:

```text
Cache Consistency System
```

---

# 151. Entity cache

Misma regla:

```text
Failover
≠
Global Entity Cache Flush
```

---

# 152. Metadata cache

ORM metadata normalmente no cambia por failover.

---

# 153. Query compiler cache

Tampoco debería invalidarse solo porque cambió el endpoint, salvo cambio de:

```text
platform capabilities
server compatibility
schema generation
```

---

# 154. Topology capability change

Si el nuevo writer posee capacidades distintas:

```text
capability generation
```

podrá requerir invalidación de planes/compiled queries relevantes.

---

# 155. Failover safety levels

VoltStack podrá clasificar decisiones:

```php
enum FailoverSafetyLevel
{
    case SAFE;
    case CONDITIONALLY_SAFE;
    case REQUIRES_CONFIRMATION;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 156. SAFE

Existe evidencia suficiente de:

```text
valid alternative
correct role
authority
consistency
ownership
```

---

# 157. CONDITIONALLY_SAFE

Puede depender de:

```text
read-only operation
consistency policy
replication freshness
```

---

# 158. REQUIRES_CONFIRMATION

Ejemplo:

```text
writer authority cannot be proven automatically
```

---

# 159. UNSAFE

Ejemplo:

```text
candidate is replica without promotion
```

para un write.

---

# 160. UNKNOWN

Evidencia insuficiente.

```text
UNKNOWN ≠ SAFE
```

---

# 161. OperationIntent

La decisión dependerá del tipo de operación.

```php
enum DatabaseOperationIntent
{
    case READ;
    case WRITE;
    case LOCKING_READ;
    case TRANSACTION_START;
    case TRANSACTION_CONTINUATION;
    case ADMINISTRATION;
}
```

---

# 162. Locking reads

Un:

```text
SELECT ... FOR UPDATE
```

deberá seguir reglas de writer/transaction authority.

No será tratado como read replica normal.

---

# 163. Transaction continuation

No puede hacer failover transparente.

---

# 164. Administration operations

Pueden requerir target específico y no ser failoverable.

---

# 165. DDL operations

Migraciones/schema operations normalmente deberán dirigirse a autoridad explícita.

No deberán saltar a otro target arbitrariamente.

---

# 166. Migration integration

Durante failover:

```text
migration execution
```

podrá pausarse o fallar de forma segura.

---

# 167. Zero-downtime migrations

Deberán coordinarse con topology changes.

Un failover no invalida automáticamente la estrategia ZDT, pero puede alterar sus precondiciones.

---

# 168. Failover policy

```php
interface FailoverPolicy
{
    public function decide(
        FailoverContext $context
    ): FailoverDecision;
}
```

---

# 169. Default policy

La política default será conservadora:

```text
READ:
    alternate eligible replica
    or writer if allowed

WRITE:
    only confirmed authoritative writer

TRANSACTION:
    no mid-transaction migration

UNKNOWN OUTCOME:
    no blind replay
```

---

# 170. Failover timeout

La búsqueda de recuperación no será infinita.

---

# 171. Recovery budget

```php
final readonly class RecoveryBudget
{
    public function __construct(
        public Duration $maxDuration,
        public int $maxTopologyRefreshes,
        public int $maxCandidateEvaluations,
    ) {}
}
```

---

# 172. Recovery Budget ≠ Retry Budget

Son independientes.

---

# 173. Deadline propagation

El Failover System respetará:

```text
request deadline
query deadline
transaction deadline
```

---

# 174. Failover cannot extend request forever

Si quedan:

```text
200ms
```

del request budget, no deberá iniciar una recuperación de:

```text
30 seconds
```

sin policy explícita.

---

# 175. Cancellation

Failover evaluation deberá soportar:

```text
CancellationToken
```

---

# 176. Cancellation ≠ endpoint failure

Cancelar recuperación por deadline no demuestra que el nuevo endpoint esté fallido.

---

# 177. Concurrency

Múltiples workers pueden detectar simultáneamente la caída del writer.

---

# 178. Failover storm

Debe evitarse:

```text
100 workers
→ 100 promotions
```

---

# 179. Coordination

Si VoltStack controla promoción, deberá existir:

```text
FailoverCoordinator
+
distributed coordination
```

---

# 180. Single-flight transition

Idealmente:

```text
one active failover transition
```

por:

```text
logical database / shard authority domain
```

---

# 181. Failover lock

Puede existir conceptualmente:

```php
interface FailoverLease
{
    public function generation(): FailoverGeneration;

    public function release(): void;
}
```

---

# 182. Lease ≠ authority

Poseer el lock de coordinación no convierte a un endpoint en writer.

---

# 183. Shared coordination optional

VoltStack core no dependerá obligatoriamente de:

```text
Redis
etcd
Consul
ZooKeeper
```

---

# 184. External HA coordination

Cuando el cluster manager externo ya coordina failover, VoltStack no deberá duplicar esa función.

---

# 185. Persistent runtimes

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

la topología puede cambiar mientras el worker sigue vivo.

---

# 186. No boot-time-only topology

No será válido asumir:

```text
topology loaded at worker boot
=
topology forever
```

---

# 187. Topology refresh

El sistema deberá permitir:

```text
event-driven refresh
TTL refresh
failure-triggered refresh
administrative refresh
```

---

# 188. Topology snapshot immutability

Cada operación utilizará un snapshot coherente.

---

# 189. Mid-operation topology mutation

No deberá modificar silenciosamente las decisiones ya tomadas dentro de una transaction.

---

# 190. Scoped operation context

```php
final readonly class FailoverOperationContext
{
    public function __construct(
        public TopologyGeneration $topologyGeneration,
        public AuthorityEpoch|null $authorityEpoch,
        public EndpointId|null $originalEndpoint,
        public OperationIntent $intent,
    ) {}
}
```

---

# 191. Long-lived shared state

Podrá ser worker-long-lived:

```text
Topology Registry
Health Registry
Circuit Registry
immutable configuration
```

---

# 192. Scoped state

Será operation/request scoped:

```text
FailoverAttempt
Candidate Evaluation
Recovery Budget
Cancellation State
Transaction Binding
```

---

# 193. No static current writer

Evitar:

```php
Database::$currentWriter = 'db01';
```

como estado mutable global sin generation/control.

---

# 194. Topology registry

```php
interface DatabaseTopologyRegistry
{
    public function snapshot(
        LogicalDatabaseId $database
    ): DatabaseTopologySnapshot;

    public function refresh(
        LogicalDatabaseId $database
    ): DatabaseTopologySnapshot;
}
```

---

# 195. Topology source

Podrá provenir de:

```text
static configuration
DNS/service discovery
cloud provider integration
database cluster metadata
proxy metadata
custom provider
```

---

# 196. Topology source trust

Cada provider deberá declarar:

```text
authority
freshness
confidence
```

de la información.

---

# 197. Conflicting topology evidence

Si dos fuentes afirman:

```text
Writer = A
```

y:

```text
Writer = B
```

VoltStack no elegirá arbitrariamente.

Resultado:

```text
Authority = UNKNOWN
```

hasta resolución.

---

# 198. Recovery after network partition

Caso:

```text
Application ↔ Writer A
```

pierde conectividad.

No significa necesariamente que:

```text
Writer A is dead
```

Puede existir:

```text
network partition
```

---

# 199. Network partition safety

Este es uno de los principales motivos por los cuales:

```text
Unreachable
≠
Non-authoritative
```

---

# 200. Quorum-aware infrastructure

Cuando el cluster externo utiliza quorum, VoltStack podrá confiar en su authority provider.

---

# 201. VoltStack will not invent quorum

El Database layer no implementará consenso distribuido genérico para decidir writers.

---

# 202. Recovery phases

Proceso recomendado:

```text
1. Detect
2. Classify
3. Contain
4. Refresh topology
5. Establish authority
6. Resolve candidate
7. Reconcile routing
8. Reconcile pools
9. Probe
10. Resume traffic
11. Monitor
12. Reintegrate
```

---

# 203. Detect

Recopilar evidencia sin declarar conclusiones prematuras.

---

# 204. Classify

Determinar:

```text
connection?
endpoint?
replica?
writer?
shard?
network?
unknown?
```

---

# 205. Contain

Circuit Breaker puede impedir nuevos requests hacia el target degradado.

---

# 206. Refresh topology

Consultar la fuente de autoridad.

---

# 207. Establish authority

Para writes, identificar writer válido.

---

# 208. Resolve candidate

Aplicar:

```text
role
health
circuit
authority
consistency
shard
capability
```

---

# 209. Reconcile routing

Actualizar el routing efectivo para nuevas operaciones.

---

# 210. Reconcile pools

Evitar reutilizar conexiones incompatibles.

---

# 211. Probe

Verificar nuevo target sin generar thundering herd.

---

# 212. Resume traffic

Solo después de satisfacer policy.

---

# 213. Monitor

Observar:

```text
failure rate
latency
replication
circuit
authority
```

---

# 214. Reintegrate

Nodos recuperados vuelven únicamente después de validación.

---

# 215. Recovery outcome

```php
enum RecoveryOutcome
{
    case RECOVERED;
    case RECOVERED_WITH_DEGRADED_CAPACITY;
    case WAITING;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 216. RECOVERED_WITH_DEGRADED_CAPACITY

Ejemplo:

```text
Writer healthy
1 of 3 replicas healthy
```

El sistema funciona, pero no está completamente recuperado.

---

# 217. UNKNOWN recovery

VoltStack conservará UNKNOWN cuando no pueda demostrar el estado.

---

# 218. Error hierarchy

```text
DatabaseResilienceException
└── DatabaseFailoverException
    ├── FailoverUnavailableException
    ├── NoEligibleFailoverTargetException
    ├── AuthorityUnknownException
    ├── AuthorityConflictException
    ├── StaleTopologyException
    ├── FailoverCoordinationException
    ├── PromotionFailedException
    ├── FencingFailedException
    ├── RecoveryFailedException
    ├── RecoveryTimeoutException
    ├── ReplicaNotReadyException
    ├── FailbackRejectedException
    └── DistributedRecoveryException
```

---

# 219. Error information

Los errores deberán transportar datos estructurados como:

```text
failure domain
original endpoint
candidate endpoint
topology generation
authority epoch
operation intent
safety level
recovery state
reason
```

sin exponer secretos.

---

# 220. Telemetry architecture

Eventos conceptuales:

```text
DatabaseFailoverEvaluationStarted
DatabaseFailoverCandidateEvaluated
DatabaseFailoverStarted
DatabaseFailoverCompleted
DatabaseFailoverRejected
DatabaseFailoverFailed
DatabaseAuthorityChanged
DatabaseTopologyChanged
DatabaseRecoveryStarted
DatabaseRecoveryCompleted
DatabaseRecoveryFailed
DatabaseEndpointQuarantined
DatabaseEndpointReintegrated
DatabaseFailbackStarted
DatabaseFailbackCompleted
```

---

# 221. Event volume

No deberá emitirse un evento costoso por cada lectura normal.

---

# 222. Metrics

Ejemplos:

```text
db.failover.attempts
db.failover.success
db.failover.failure
db.failover.duration
db.failover.authority_unknown
db.failover.no_candidate
db.recovery.duration
db.recovery.failures
db.endpoint.quarantined
db.endpoint.reintegrated
db.topology.generation
db.authority.epoch
```

---

# 223. Cardinality

Evitar labels como:

```text
raw SQL
request ID
user ID
transaction ID
```

en métricas globales.

---

# 224. Tracing

Un trace podrá mostrar:

```text
Query
 ↓
Connection failure
 ↓
Circuit opened
 ↓
Topology refresh
 ↓
Writer changed
 ↓
Retry evaluation
 ↓
New operation
```

pero sin fingir que ambas ejecuciones son la misma transaction física.

---

# 225. Diagnostics

Ejemplo:

```text
Logical Database:
    primary

Failure Domain:
    WRITER

Original Writer:
    db-a

Original Authority Epoch:
    41

Observed State:
    UNAVAILABLE

Circuit:
    OPEN

Topology Refresh:
    SUCCESS

Current Authority:
    db-b

Current Authority Epoch:
    42

Candidate:
    db-b

Authority:
    AUTHORITATIVE

Failover Safety:
    SAFE

Routing:
    UPDATED

Old Pool:
    DRAINING

Recovery:
    RECOVERED
```

---

# 226. Unknown authority diagnostics

```text
Original Writer:
    db-a

Status:
    UNREACHABLE

Candidate:
    db-b

Candidate Role:
    REPLICA

Authority:
    UNKNOWN

Decision:
    WRITE FAILOVER REJECTED

Reason:
    No authoritative writer could be established.
```

---

# 227. Explain API

Conceptualmente:

```php
$report = $db
    ->resilience()
    ->failover()
    ->explain('primary');
```

---

# 228. Security

Failover diagnostics no expondrán:

```text
database password
TLS private key
cloud credentials
service discovery secrets
fencing secrets
```

---

# 229. Administrative permissions

Operaciones como:

```text
manual failover
force topology refresh
force quarantine
force reintegration
manual failback
promotion
```

deberán requerir permisos administrativos explícitos.

---

# 230. Audit

Cambios manuales de autoridad deberán registrarse.

---

# 231. Promotion audit

Como mínimo:

```text
who requested
when
database/shard
old writer
new writer
old epoch
new epoch
reason
result
```

cuando la capa de identidad/auditoría esté disponible.

---

# 232. Testing

El sistema deberá proporcionar:

```text
FakeTopologyProvider
FakeAuthorityProvider
FakePromotionProvider
FakeRecoveryPolicy
FakeClock
FailureInjector
TopologyScenarioBuilder
```

---

# 233. Replica failure test

```text
Replica A → failed
Replica B → healthy
```

Esperado:

```text
reads may route to B
```

si satisface consistency.

---

# 234. Replica lag test

```text
Replica B → healthy
lag → unacceptable
```

Esperado:

```text
B not eligible
```

para una operación con freshness estricta.

---

# 235. Writer unreachable test

```text
Writer A → unreachable
Replica B → reachable
Authority B → UNKNOWN
```

Esperado:

```text
no write failover
```

---

# 236. Writer authority test

```text
Writer A → failed
Writer B → AUTHORITATIVE
Epoch → incremented
```

Esperado:

```text
new writes may route to B
```

---

# 237. Split-brain test

```text
A claims writer
B claims writer
```

sin autoridad superior.

Esperado:

```text
AuthorityConflict
writes rejected
```

---

# 238. Stale topology test

Failover plan generado con:

```text
generation 5
```

pero current:

```text
generation 6
```

Esperado:

```text
plan revalidation
```

---

# 239. Mid-transaction failure test

```text
BEGIN A
UPDATE A
A fails
```

Esperado:

```text
no continuation on B
```

---

# 240. Unknown commit test

```text
COMMIT sent
connection lost
writer failover
```

Esperado:

```text
UNKNOWN preserved
no blind retry
```

---

# 241. Circuit integration test

Endpoint OPEN.

Esperado:

```text
not eligible for ordinary failover routing
```

---

# 242. Recovery test

Replica returns but lag remains high.

Esperado:

```text
CATCHING_UP
not READY
```

---

# 243. Reintegration test

Después de:

```text
health success
replication caught up
stability window satisfied
circuit recovered
```

Esperado:

```text
READY
```

---

# 244. Former writer test

A era writer.

B fue promovido.

A vuelve.

Esperado:

```text
A does not automatically regain write authority
```

---

# 245. Shard isolation test

Shard 2 falla.

Esperado:

```text
Shard 1 and Shard 3 remain unaffected
```

---

# 246. Cross-shard safety test

No existe target válido para Shard A.

Esperado:

```text
operation fails
```

no:

```text
route to Shard B
```

---

# 247. Persistent worker test

Topología cambia entre Request 1 y Request 2.

Esperado:

```text
Request 2 can observe refreshed topology
```

---

# 248. Stale connection test

Pool contiene conexión al authority epoch anterior.

Esperado:

```text
not leased for incompatible new write
```

---

# 249. Failover storm test

100 workers detectan fallo.

Esperado:

```text
bounded/coordinated failover transition
```

cuando VoltStack controla transición.

---

# 250. Recovery flapping test

Endpoint alterna UP/DOWN.

Esperado:

```text
stability policy prevents immediate reintegration
```

---

# 251. Directory structure

```text
src/Quantum/Database/Resilience/Failover/
│
├── Contract/
│   ├── FailoverCoordinator.php
│   ├── FailoverPolicy.php
│   ├── FailoverAuthorityProvider.php
│   ├── DatabasePromotionProvider.php
│   ├── DatabaseTopologyRegistry.php
│   ├── DatabaseTopologyProvider.php
│   ├── RecoveryPolicy.php
│   └── FailoverCoordinationProvider.php
│
├── Model/
│   ├── FailureDomain.php
│   ├── FailoverEvidence.php
│   ├── FailoverEvidenceSet.php
│   ├── FailoverEvidenceConfidence.php
│   ├── FailoverRequest.php
│   ├── FailoverContext.php
│   ├── FailoverCandidate.php
│   ├── FailoverDecision.php
│   ├── FailoverDecisionType.php
│   ├── FailoverMode.php
│   ├── FailoverSafetyLevel.php
│   ├── FailoverGeneration.php
│   ├── RecoveryState.php
│   ├── RecoveryOutcome.php
│   ├── RecoveryBudget.php
│   └── FailbackPolicy.php
│
├── Authority/
│   ├── AuthorityRecord.php
│   ├── AuthorityStatus.php
│   ├── AuthorityEpoch.php
│   ├── FencingToken.php
│   ├── StaticAuthorityProvider.php
│   └── DynamicAuthorityProvider.php
│
├── Topology/
│   ├── DatabaseTopologySnapshot.php
│   ├── TopologyGeneration.php
│   ├── EndpointTopology.php
│   ├── ReplicationGroupTopology.php
│   ├── ShardTopology.php
│   └── DefaultDatabaseTopologyRegistry.php
│
├── Candidate/
│   ├── FailoverCandidateResolver.php
│   ├── FailoverEligibilityAnalyzer.php
│   ├── ReadFailoverCandidateResolver.php
│   └── WriteFailoverCandidateResolver.php
│
├── Policy/
│   ├── DefaultFailoverPolicy.php
│   ├── ConservativeWriteFailoverPolicy.php
│   ├── ReadFailoverPolicy.php
│   ├── DefaultRecoveryPolicy.php
│   └── StabilityWindowPolicy.php
│
├── Coordination/
│   ├── FailoverLease.php
│   ├── LocalFailoverCoordinator.php
│   └── SharedFailoverCoordinator.php
│
├── Promotion/
│   ├── PromotionRequest.php
│   ├── PromotionResult.php
│   ├── PromotionPlanner.php
│   └── PromotionVerifier.php
│
├── Recovery/
│   ├── RecoveryCoordinator.php
│   ├── RecoveryEvidence.php
│   ├── RecoveryDecision.php
│   ├── EndpointRecoveryManager.php
│   ├── ReplicaRecoveryManager.php
│   └── EndpointReintegrationManager.php
│
├── Failback/
│   ├── FailbackCoordinator.php
│   ├── FailbackRequest.php
│   └── FailbackDecision.php
│
├── Integration/
│   ├── RetryFailoverIntegration.php
│   ├── CircuitFailoverIntegration.php
│   ├── RoutingFailoverIntegration.php
│   ├── PoolFailoverIntegration.php
│   ├── ReplicaFailoverIntegration.php
│   ├── TransactionFailoverIntegration.php
│   └── ShardingFailoverIntegration.php
│
├── Runtime/
│   ├── FailoverOperationContext.php
│   └── TopologyRefreshManager.php
│
├── Telemetry/
│   └── FailoverTelemetry.php
│
├── Diagnostics/
│   ├── FailoverDiagnostics.php
│   ├── FailoverExplainResult.php
│   └── RecoveryDiagnostics.php
│
├── Testing/
│   ├── FakeTopologyProvider.php
│   ├── FakeAuthorityProvider.php
│   ├── FakePromotionProvider.php
│   ├── FakeFailoverCoordinator.php
│   ├── FakeRecoveryPolicy.php
│   └── TopologyScenarioBuilder.php
│
└── Exception/
    ├── DatabaseFailoverException.php
    ├── FailoverUnavailableException.php
    ├── NoEligibleFailoverTargetException.php
    ├── AuthorityUnknownException.php
    ├── AuthorityConflictException.php
    ├── StaleTopologyException.php
    ├── FailoverCoordinationException.php
    ├── PromotionFailedException.php
    ├── FencingFailedException.php
    ├── RecoveryFailedException.php
    ├── RecoveryTimeoutException.php
    ├── ReplicaNotReadyException.php
    ├── FailbackRejectedException.php
    └── DistributedRecoveryException.php
```

---

# 252. Dependencias

Flujo principal:

```text
Connection / Query Failure
          ↓
Failure Classification
          ↓
Resilience Evidence
          ↓
Circuit Breaker
          ↓
Failover Evaluation
          ↓
Topology
          ↓
Authority
          ↓
Candidate Resolution
          ↓
Routing
          ↓
Connection Manager
```

Integraciones:

```text
Failover
├── Retry
├── Circuit Breaker
├── Connection Manager
├── Connection Pool
├── Read/Write Routing
├── Replica System
├── Replica Lag Awareness
├── Load Balancer
├── Transactions
├── Sharding
├── Cache
├── Telemetry
├── Security
└── Runtime
```

---

# 253. Dependencias prohibidas

Failover core no dependerá directamente de:

```text
ORM Entity
Model API
HTTP Controller
Livewire-like frontend runtime
specific Redis implementation
specific cloud provider
specific HA product
specific database vendor
```

Las integraciones concretas vivirán detrás de contratos.

---

# 254. Invariantes arquitectónicas

## DB-FAILOVER-001
Failover no será Retry.

## DB-FAILOVER-002
Failover no será Reconnect.

## DB-FAILOVER-003
Failover no será Load Balancing.

## DB-FAILOVER-004
Failover no será Circuit Breaker.

## DB-FAILOVER-005
Failover no será Health Check.

## DB-FAILOVER-006
Failover no será Recovery.

## DB-FAILOVER-007
Failover no será Failback.

## DB-FAILOVER-008
Connection Failure no implicará Endpoint Failure.

## DB-FAILOVER-009
Endpoint Failure no implicará Logical Database Failure.

## DB-FAILOVER-010
Replica Failure no implicará Writer Failure.

## DB-FAILOVER-011
Shard Failure no implicará Global Database Failure.

## DB-FAILOVER-012
UNKNOWN no será FAILED.

## DB-FAILOVER-013
UNKNOWN no será HEALTHY.

## DB-FAILOVER-014
UNKNOWN no será SAFE.

## DB-FAILOVER-015
Unreachable Writer no significará Non-Authoritative Writer.

## DB-FAILOVER-016
Endpoint Role no será Write Authority.

## DB-FAILOVER-017
Configured Writer no demostrará current authority.

## DB-FAILOVER-018
Write failover requerirá autoridad válida.

## DB-FAILOVER-019
Replica reachable no implicará write eligibility.

## DB-FAILOVER-020
Writer OPEN circuit no autorizará writes en replica.

## DB-FAILOVER-021
Circuit OPEN no será Failover Command.

## DB-FAILOVER-022
Health Failure no será Promotion Command.

## DB-FAILOVER-023
Writer authority deberá ser explícita.

## DB-FAILOVER-024
Authority UNKNOWN impedirá automatic write failover por default.

## DB-FAILOVER-025
Authority conflict impedirá writes por default.

## DB-FAILOVER-026
Split-brain prevention tendrá prioridad sobre write availability.

## DB-FAILOVER-027
Authority Epoch podrá distinguir generaciones de writer.

## DB-FAILOVER-028
Old authority no deberá reutilizarse como current authority.

## DB-FAILOVER-029
Fencing será utilizado cuando esté disponible.

## DB-FAILOVER-030
Fencing failure impedirá promoción segura cuando sea requisito.

## DB-FAILOVER-031
VoltStack no inventará quorum.

## DB-FAILOVER-032
VoltStack no implementará consenso distribuido genérico en Database core.

## DB-FAILOVER-033
External HA managers podrán ser authority providers.

## DB-FAILOVER-034
Application routing failover será distinto de infrastructure failover.

## DB-FAILOVER-035
Promotion no implicará failover completo.

## DB-FAILOVER-036
Promotion deberá ser seguida por authority verification.

## DB-FAILOVER-037
Promotion deberá ser seguida por topology reconciliation.

## DB-FAILOVER-038
Promotion deberá ser seguida por routing reconciliation.

## DB-FAILOVER-039
Promotion deberá ser seguida por connection reconciliation.

## DB-FAILOVER-040
Read failover respetará consistency.

## DB-FAILOVER-041
Read failover respetará freshness.

## DB-FAILOVER-042
Read failover respetará shard ownership.

## DB-FAILOVER-043
Read failover respetará circuit eligibility.

## DB-FAILOVER-044
Read failover no degradará consistency silenciosamente.

## DB-FAILOVER-045
Writer podrá servir como read fallback solo si policy lo permite.

## DB-FAILOVER-046
Write failover requerirá current authoritative writer.

## DB-FAILOVER-047
Write failover no enviará writes a ordinary replica.

## DB-FAILOVER-048
Locking read seguirá writer/transaction rules.

## DB-FAILOVER-049
DDL no hará arbitrary failover.

## DB-FAILOVER-050
Administrative operations podrán ser non-failoverable.

## DB-FAILOVER-051
Transaction activa permanecerá pinned.

## DB-FAILOVER-052
Transaction no migrará entre conexiones.

## DB-FAILOVER-053
Transaction no migrará entre endpoints.

## DB-FAILOVER-054
Transaction no migrará entre authority epochs.

## DB-FAILOVER-055
Savepoint no permitirá server migration.

## DB-FAILOVER-056
Mid-transaction endpoint failure hará fallar la transaction.

## DB-FAILOVER-057
Whole transaction retry será decisión separada.

## DB-FAILOVER-058
Whole transaction retry requerirá replay safety.

## DB-FAILOVER-059
UNKNOWN commit no será blind-retried.

## DB-FAILOVER-060
Failover no cambiará UNKNOWN a FAILED.

## DB-FAILOVER-061
Failover no cambiará UNKNOWN a COMMITTED.

## DB-FAILOVER-062
Failover no reescribirá historia transaccional.

## DB-FAILOVER-063
Previous operation outcome permanecerá independiente de new authority.

## DB-FAILOVER-064
Retry podrá utilizar nuevo target solo cuando sea elegible.

## DB-FAILOVER-065
Retry Budget y Recovery Budget serán independientes.

## DB-FAILOVER-066
Failover respetará request deadline.

## DB-FAILOVER-067
Failover respetará transaction deadline.

## DB-FAILOVER-068
Failover evaluation será cancelable.

## DB-FAILOVER-069
Cancellation no implicará endpoint failure.

## DB-FAILOVER-070
Topology tendrá generation.

## DB-FAILOVER-071
Stale topology deberá detectarse.

## DB-FAILOVER-072
Stale failover plan deberá revalidarse.

## DB-FAILOVER-073
Topology refresh no cambiará transaction binding existente.

## DB-FAILOVER-074
Cada operación utilizará topology snapshot coherente.

## DB-FAILOVER-075
Persistent worker no asumirá boot-time topology permanente.

## DB-FAILOVER-076
Topology podrá refrescarse durante vida del worker.

## DB-FAILOVER-077
Topology state compartido será concurrency-safe.

## DB-FAILOVER-078
Operation failover state será scoped.

## DB-FAILOVER-079
No existirá static mutable current writer sin generation semantics.

## DB-FAILOVER-080
Old pooled connections podrán quedar stale tras failover.

## DB-FAILOVER-081
Stale writer connections no recibirán nuevos writes.

## DB-FAILOVER-082
Pool reconciliation será parte de recovery.

## DB-FAILOVER-083
Failover no implicará cerrar todas las conexiones indiscriminadamente.

## DB-FAILOVER-084
Connection generation podrá relacionarse con topology generation.

## DB-FAILOVER-085
Connection authority context podrá relacionarse con authority epoch.

## DB-FAILOVER-086
DNS change no será authority proof.

## DB-FAILOVER-087
Proxy endpoint podrá abstraer physical failover.

## DB-FAILOVER-088
Application-visible endpoint no será necesariamente physical node.

## DB-FAILOVER-089
Managed database failover será soportable mediante provider.

## DB-FAILOVER-090
Replica recovery requerirá health.

## DB-FAILOVER-091
Replica recovery requerirá replication validation cuando aplique.

## DB-FAILOVER-092
Connected replica no implicará ready replica.

## DB-FAILOVER-093
Catching-up replica no será fresh por definición.

## DB-FAILOVER-094
UNKNOWN lag no será zero lag.

## DB-FAILOVER-095
Recovered endpoint podrá permanecer quarantined.

## DB-FAILOVER-096
Recovery podrá requerir stability window.

## DB-FAILOVER-097
Recovery hysteresis podrá prevenir flapping.

## DB-FAILOVER-098
Circuit recovery y endpoint recovery serán distintos.

## DB-FAILOVER-099
Circuit CLOSED no demostrará replication readiness.

## DB-FAILOVER-100
Endpoint HEALTHY no demostrará write authority.

## DB-FAILOVER-101
Former writer no recuperará authority automáticamente.

## DB-FAILOVER-102
Writer automatic failback estará deshabilitado por default.

## DB-FAILOVER-103
Writer failback requerirá nueva authority transition.

## DB-FAILOVER-104
Replica failback podrá ser automático tras verificación.

## DB-FAILOVER-105
Failback respetará circuit status.

## DB-FAILOVER-106
Failback respetará freshness.

## DB-FAILOVER-107
Failback respetará stability.

## DB-FAILOVER-108
Shard ownership nunca cambiará por simple endpoint failure.

## DB-FAILOVER-109
Cross-shard failover estará prohibido.

## DB-FAILOVER-110
Shard replica failover permanecerá dentro del mismo shard.

## DB-FAILOVER-111
Shard writer failover requerirá authority del mismo shard.

## DB-FAILOVER-112
Shard map generation será respetada.

## DB-FAILOVER-113
Resharding requerirá revalidación de ownership.

## DB-FAILOVER-114
Partial distributed result no será fake success.

## DB-FAILOVER-115
Distributed partial failure conservará evidencia por shard.

## DB-FAILOVER-116
VoltStack no fingirá distributed rollback.

## DB-FAILOVER-117
Compensation será responsabilidad de capa apropiada.

## DB-FAILOVER-118
Failover no cambiará tenant context.

## DB-FAILOVER-119
Tenant isolation será preservado.

## DB-FAILOVER-120
Dedicated tenant DB tendrá failure domain independiente.

## DB-FAILOVER-121
Shared tenant infrastructure podrá compartir failover state.

## DB-FAILOVER-122
Tenant metadata no será authority proof.

## DB-FAILOVER-123
Failover no hará FLUSHALL por default.

## DB-FAILOVER-124
Topology cache deberá reconciliarse tras cambio.

## DB-FAILOVER-125
Result Cache conservará sus propias consistency rules.

## DB-FAILOVER-126
Entity Cache conservará sus propias consistency rules.

## DB-FAILOVER-127
ORM Metadata Cache no se invalidará solo por failover.

## DB-FAILOVER-128
Compiled Query Cache no se invalidará solo por endpoint change.

## DB-FAILOVER-129
Capability change podrá requerir invalidación específica.

## DB-FAILOVER-130
Platform capability será consultada antes de operaciones HA específicas.

## DB-FAILOVER-131
Version no será Capability.

## DB-FAILOVER-132
MySQL y MariaDB podrán tener capacidades diferentes.

## DB-FAILOVER-133
SQLite no fingirá server failover.

## DB-FAILOVER-134
Unsupported failover será explícito.

## DB-FAILOVER-135
External failover capability será distinguible de native capability.

## DB-FAILOVER-136
Failover decision tendrá safety level.

## DB-FAILOVER-137
SAFE requerirá evidencia suficiente.

## DB-FAILOVER-138
CONDITIONALLY_SAFE tendrá condiciones explícitas.

## DB-FAILOVER-139
UNSAFE no será ejecutado automáticamente.

## DB-FAILOVER-140
UNKNOWN no será ejecutado como SAFE.

## DB-FAILOVER-141
Failover tendrá bounded recovery budget.

## DB-FAILOVER-142
Topology refresh tendrá límites.

## DB-FAILOVER-143
Candidate evaluation tendrá límites.

## DB-FAILOVER-144
Recovery no extenderá indefinidamente una request.

## DB-FAILOVER-145
Concurrent failover detection será soportada.

## DB-FAILOVER-146
VoltStack evitará failover storms.

## DB-FAILOVER-147
Promotion coordinada tendrá single-flight semantics cuando sea necesaria.

## DB-FAILOVER-148
Failover coordination lease no será write authority.

## DB-FAILOVER-149
Shared coordination provider será opcional.

## DB-FAILOVER-150
External HA coordination no será duplicada innecesariamente.

## DB-FAILOVER-151
Failure evidence tendrá timestamp.

## DB-FAILOVER-152
Failure evidence tendrá domain.

## DB-FAILOVER-153
Failure evidence podrá tener confidence.

## DB-FAILOVER-154
Conflicting evidence deberá ser representable.

## DB-FAILOVER-155
Conflicting writer evidence producirá authority uncertainty/conflict.

## DB-FAILOVER-156
Network partition no implicará writer death.

## DB-FAILOVER-157
Unreachable no implicará non-authoritative.

## DB-FAILOVER-158
Recovery tendrá estado explícito.

## DB-FAILOVER-159
DEGRADED será distinto de FAILED.

## DB-FAILOVER-160
RECOVERING será distinto de HEALTHY.

## DB-FAILOVER-161
Recovered-with-degraded-capacity será representable.

## DB-FAILOVER-162
Recovery UNKNOWN será preservado.

## DB-FAILOVER-163
Reintegration requerirá validación.

## DB-FAILOVER-164
Reintegration no será equivalente a connection success.

## DB-FAILOVER-165
Reintegration podrá ser gradual.

## DB-FAILOVER-166
Recovered replica podrá iniciar con peso reducido.

## DB-FAILOVER-167
Load Balancer recibirá solo endpoints elegibles.

## DB-FAILOVER-168
Failover no sustituirá Load Balancing.

## DB-FAILOVER-169
Failover no sustituirá Circuit Breaker.

## DB-FAILOVER-170
Failover no sustituirá Retry.

## DB-FAILOVER-171
Failover no sustituirá Health Monitoring.

## DB-FAILOVER-172
Failover no sustituirá Replica Lag Awareness.

## DB-FAILOVER-173
Failover no sustituirá Transaction Recovery.

## DB-FAILOVER-174
Failover no sustituirá application idempotency.

## DB-FAILOVER-175
Failover no garantizará exactly-once.

## DB-FAILOVER-176
Failover no garantizará zero data loss sin infraestructura que lo soporte.

## DB-FAILOVER-177
Failover no garantizará zero downtime por sí mismo.

## DB-FAILOVER-178
Failover no degradará seguridad.

## DB-FAILOVER-179
Failover no expondrá credentials en diagnostics.

## DB-FAILOVER-180
Administrative failover será auditable.

## DB-FAILOVER-181
Manual promotion será auditable.

## DB-FAILOVER-182
Authority changes serán observables.

## DB-FAILOVER-183
Topology changes serán observables.

## DB-FAILOVER-184
Recovery transitions serán observables.

## DB-FAILOVER-185
Telemetry tendrá cardinalidad controlada.

## DB-FAILOVER-186
Raw SQL no será metric label por default.

## DB-FAILOVER-187
User ID no será metric label.

## DB-FAILOVER-188
Request ID no será metric label global.

## DB-FAILOVER-189
Failover diagnostics serán estructurados.

## DB-FAILOVER-190
Failover reason será explicable.

## DB-FAILOVER-191
Candidate rejection reason será explicable.

## DB-FAILOVER-192
Authority source será diagnosticable.

## DB-FAILOVER-193
Topology generation será diagnosticable.

## DB-FAILOVER-194
Authority epoch será diagnosticable cuando exista.

## DB-FAILOVER-195
Recovery state será diagnosticable.

## DB-FAILOVER-196
Failover safety level será diagnosticable.

## DB-FAILOVER-197
Testing podrá simular topology transitions.

## DB-FAILOVER-198
Testing podrá simular authority conflicts.

## DB-FAILOVER-199
Testing podrá simular network partitions.

## DB-FAILOVER-200
Testing podrá simular unknown commit outcomes.

---

# 255. Modelo formal

Sea:

```text
T = set of discovered targets
o = operation
```

Cada target:

```text
t ∈ T
```

posee:

```text
Role(t)
Health(t)
Circuit(t)
Authority(t)
Shard(t)
Capabilities(t)
Freshness(t)
```

---

# 256. Read eligibility

Para una operación de lectura `o`:

```text
ReadEligible(t,o)
=
RoleAllowsRead(t)
∧
HealthEligible(t)
∧
CircuitEligible(t)
∧
ShardCompatible(t,o)
∧
ConsistencyCompatible(t,o)
∧
FreshnessCompatible(t,o)
```

---

# 257. Write eligibility

Para un write:

```text
WriteEligible(t,o)
=
Role(t) = WRITER
∧
Authority(t) = AUTHORITATIVE
∧
HealthAllowsWrite(t)
∧
CircuitAllowsWrite(t)
∧
ShardCompatible(t,o)
∧
CapabilitiesCompatible(t,o)
```

---

# 258. Failover candidate set

```text
Candidates(o)
=
{ t ∈ T | Eligible(t,o) }
```

Si:

```text
Candidates(o) = ∅
```

VoltStack no inventará un target.

---

# 259. Authority transition

Sea:

```text
A_e
```

la autoridad del epoch `e`.

Una transición válida:

```text
A_e = Writer A
        ↓
fencing / external authority transition
        ↓
A_(e+1) = Writer B
```

Nunca:

```text
A_e = Writer A
A_e = Writer B
```

simultáneamente dentro del modelo de autoridad válido.

---

# 260. Transaction invariant

Para una transaction `Tx`:

```text
Binding(Tx)
=
(Connection, Endpoint, AuthorityEpoch)
```

durante su ejecución física.

No se permite:

```text
Binding(Tx)_start ≠ Binding(Tx)_continuation
```

como failover transparente.

---

# 261. Unknown commit invariant

Sea:

```text
Outcome(Tx) = UNKNOWN
```

Entonces:

```text
Failover(Tx)
```

no transforma:

```text
UNKNOWN → ROLLED_BACK
```

ni:

```text
UNKNOWN → COMMITTED
```

---

# 262. Recovery invariant

Un endpoint recuperado `r` solo podrá reintegrarse cuando:

```text
HealthValid(r)
∧
TopologyValid(r)
∧
RoleValid(r)
∧
CircuitRecoveryValid(r)
∧
ReplicationValid(r, policy)
```

---

# 263. Failback invariant

Para un antiguo writer:

```text
Recovered(oldWriter)
≠
Authoritative(oldWriter)
```

Se requiere una nueva transición explícita de autoridad.

---

# 264. Flujo completo

```text
                         DATABASE OPERATION
                                │
                                ▼
                        DATABASE ROUTING
                                │
                                ▼
                         TARGET ENDPOINT
                                │
                                ▼
                            EXECUTION
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
              SUCCESS                       FAILURE
                                               │
                                               ▼
                                  FAILURE CLASSIFICATION
                                               │
                                               ▼
                                     RESILIENCE EVIDENCE
                                               │
                         ┌─────────────────────┼────────────────────┐
                         │                     │                    │
                       RETRY              CIRCUIT BREAKER       FAILOVER
                         │                     │                    │
                         │                     ▼                    │
                         │               CONTAIN FAILURE            │
                         │                                          │
                         └──────────────────────┬───────────────────┘
                                                ▼
                                       TOPOLOGY REFRESH
                                                │
                                                ▼
                                      AUTHORITY VALIDATION
                                                │
                               ┌────────────────┴───────────────┐
                               │                                │
                           UNKNOWN                           VALID
                               │                                │
                               ▼                                ▼
                       FAIL / WAIT / MANUAL             CANDIDATE RESOLUTION
                                                                │
                                                                ▼
                                                      ELIGIBILITY ANALYSIS
                                                                │
                                              ┌─────────────────┴─────────────┐
                                              │                               │
                                          NO TARGET                       TARGET
                                              │                               │
                                              ▼                               ▼
                                            FAIL                        SWITCH ROUTING
                                                                              │
                                                                              ▼
                                                                   POOL RECONCILIATION
                                                                              │
                                                                              ▼
                                                                         PROBING
                                                                              │
                                                                              ▼
                                                                      RESUME TRAFFIC
                                                                              │
                                                                              ▼
                                                                         RECOVERY
                                                                              │
                                                                              ▼
                                                                      REINTEGRATION
```

---

# 265. Regla maestra

> **VoltStack nunca confundirá disponibilidad con autoridad. Un endpoint alternativo solo podrá recibir una operación cuando sea válido para el rol, shard, consistencia, capacidades y contexto requeridos; para escrituras, además deberá existir evidencia suficiente de autoridad vigente. La pérdida de comunicación con un writer nunca será por sí sola prueba de que ese writer dejó de ser autoritativo.**

En forma compacta:

```text
Reachable
≠
Eligible

Healthy
≠
Authoritative

Replica
≠
Backup Writer

Writer Unreachable
≠
Writer Deauthorized

Circuit OPEN
≠
Failover

Retry
≠
Failover

Promotion
≠
Recovery Complete

Recovery
≠
Failback

Recovered Old Writer
≠
Current Writer

New Writer
≠
Previous Transaction Failed

UNKNOWN COMMIT
≠
SAFE RETRY

Connection Failure
≠
Endpoint Failure

Endpoint Failure
≠
Shard Failure

Shard Failure
≠
Global Failure

Failover
≠
Cross-Shard Routing

Failover
≠
Consistency Downgrade
```

Prioridad arquitectónica:

```text
Data Correctness
      >
Single Write Authority
      >
Split-Brain Prevention
      >
Shard Ownership
      >
Transaction Integrity
      >
Consistency Guarantees
      >
Known Outcome Preservation
      >
Failure Containment
      >
Recovery
      >
Availability
      >
Transparent Failover
      >
Convenience
```

---

# 266. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
✓ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
✓ 238_DATABASE_RETRY_POLICY_SYSTEM.md
✓ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
✓ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 267. Siguiente documento

```text
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

El siguiente documento cerrará el bloque de resiliencia definiendo cómo VoltStack Database protegerá aplicación, workers y servidores de base de datos frente a agotamiento de:

```text
connections
connection pool capacity
concurrent queries
memory
result buffers
database cursors
prepared statements
transactions
transaction duration
query execution slots
worker capacity
queue capacity
file descriptors
temporary resources
streaming resources
hydration memory
IdentityMap memory
UnitOfWork memory
batch/chunk resources
distributed query fan-out
retry amplification
```

La arquitectura deberá incluir:

```text
Resource Budget
Resource Limits
Admission Control
Concurrency Limiting
Backpressure
Load Shedding

Connection Budget
Query Budget
Transaction Budget
Memory Budget
Result Budget
Cursor Budget
Hydration Budget

Per-Request Limits
Per-Worker Limits
Per-Database Limits
Per-Tenant Limits
Per-Shard Limits

Soft Limit
Hard Limit
Warning Threshold
Critical Threshold

Resource Lease
Resource Permit
Resource Accounting
Resource Reservation

Queue Limits
Wait Limits
Acquisition Timeout

Bulkhead Isolation
Fairness
Priority
Starvation Prevention

Retry Storm Protection
Failover Storm Protection

Large Dataset Protection
Streaming Protection
Chunk Protection
Bulk Operation Protection

Persistent Runtime Protection
FrankenPHP
RoadRunner
OpenSwoole

Telemetry
Diagnostics
Security
Testing
```

Su principio central será:

> **VoltStack no deberá aceptar trabajo ilimitado esperando que la base de datos o el runtime sobrevivan; cada recurso finito deberá poseer límites, ownership y políticas de admisión explícitas, de modo que ante saturación el sistema degrade o rechace trabajo de forma controlada antes de convertir presión local en una falla generalizada.**