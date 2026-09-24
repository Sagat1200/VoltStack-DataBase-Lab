# 178_DATABASE_REPLICA_SYSTEM.md

# VoltStack Quantum Database
## Database Replica System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 178 — Database Replica System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md`  
**Siguiente documento:** `179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md`

---

# 1. Propósito

`Database Replica System` define la arquitectura mediante la cual VoltStack representará, descubrirá, validará y expondrá **replicas de base de datos** como elementos explícitos de una topología de replicación.

Los documentos anteriores establecieron:

```text
176
Read/Write Connection System
→ qué conexiones existen y qué capacidades poseen

177
Read/Write Routing System
→ cuál conexión elegible debe utilizar una operación
```

Este documento añade:

```text
Replication Topology
        │
        ├── Authoritative Node
        ├── Replica Nodes
        ├── Replication Relationships
        ├── Replica State
        ├── Replica Capabilities
        ├── Replication Position
        └── Topology Generation
                 │
                 ▼
          Routing Evidence
```

La regla central será:

> **Una replica en VoltStack será un nodo perteneciente a una topología de replicación con una relación explícita respecto de una fuente autoritativa; no será simplemente una conexión marcada como `READ`, y VoltStack nunca asumirá que una replica está actualizada, disponible o autorizada para una operación únicamente porque esté configurada como reader.**

Formalmente:

```text
Replica
≠
ReadConnection
≠
ReadRole
≠
HealthyEndpoint
≠
FreshEndpoint
≠
AuthoritativeNode
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. modelo canónico de replica;
2. identidad estable de replica;
3. topología de replicación;
4. nodo autoritativo;
5. relaciones de replicación;
6. estado de replica;
7. lifecycle de replica;
8. capabilities;
9. discovery;
10. configuración estática;
11. descubrimiento dinámico;
12. evidencia observada;
13. topology snapshots;
14. topology generations;
15. replication position;
16. replica eligibility;
17. health integration;
18. read/write routing integration;
19. consistency integration;
20. transaction integration;
21. failover integration futura;
22. lag-awareness integration;
23. multi-region readiness;
24. extensibilidad por plataforma/proveedor;
25. telemetry;
26. diagnostics;
27. persistent-runtime safety;
28. testing.

---

# 3. No objetivos

Este documento no define completamente:

```text
cómo medir lag
cómo seleccionar la replica más fresca
cómo hacer sticky routing
cómo promover una replica
cómo ejecutar failover
cómo balancear carga
cómo seleccionar shards
cómo implementar distributed consensus
```

Estos temas pertenecen a:

```text
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 4. Problema fundamental

Una configuración tradicional podría declarar:

```php
'read' => [
    ['host' => 'db-r1'],
    ['host' => 'db-r2'],
],
```

y asumir:

```text
read host = replica
```

Esta equivalencia es insuficiente.

Un endpoint `READ` podría ser:

- el mismo primary;
- una replica;
- un proxy;
- un cluster endpoint;
- un servidor analytics;
- un nodo read-only independiente;
- un servicio gestionado que oculta su topología.

Por tanto:

```text
ConnectionRole::READ
```

solo significa:

> Este endpoint puede satisfacer determinadas operaciones de lectura.

No significa:

> Este nodo replica datos desde el writer.

---

# 5. Read Connection ≠ Replica

Regla:

```text
ReadConnection
≠
Replica
```

Una replica describe:

```text
topological replication semantics
```

Una read connection describe:

```text
connection capability/routing semantics
```

---

# 6. Replica definition

Modelo conceptual:

```php
final readonly class DatabaseReplica
{
    public function __construct(
        public ReplicaId $id,
        public EndpointId $endpoint,
        public ReplicationSourceId $source,
        public ReplicaCapabilities $capabilities,
        public ReplicaMetadata $metadata,
    ) {}
}
```

---

# 7. ReplicaId

Identidad estable:

```php
final readonly class ReplicaId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
replica.eu-west-1.a
replica.mx-north-1.a
analytics.replica.01
```

---

# 8. ReplicaId ≠ EndpointId

Una replica tiene identidad topológica.

Una conexión tiene identidad operacional.

```text
ReplicaId
≠
EndpointId
```

---

# 9. Why separate identities

Esto permite escenarios como:

```text
Replica logical identity
        │
        ▼
Managed provider endpoint
        │
        ▼
Physical node changes
```

sin perder la identidad conceptual de la replica.

---

# 10. ReplicationNode

Modelo general:

```php
interface ReplicationNode
{
    public function id(): ReplicationNodeId;

    public function endpoint(): EndpointId;
}
```

---

# 11. Node roles

La topología podrá distinguir:

```php
enum ReplicationNodeRole
{
    case AUTHORITATIVE;
    case REPLICA;
    case RELAY;
    case UNKNOWN;
}
```

---

# 12. AUTHORITATIVE

Representa un nodo/fuente reconocida como autoridad de escritura dentro de una determinada topología.

No implica necesariamente:

```text
one permanent physical machine
```

---

# 13. REPLICA

Nodo que recibe cambios desde otra fuente de replicación.

---

# 14. RELAY

Arquitecturas futuras pueden tener:

```text
Primary
   ↓
Relay Replica
   ↓
Replica
```

VoltStack no deberá asumir siempre:

```text
Replica → direct primary replication
```

---

# 15. UNKNOWN

Cuando no exista evidencia suficiente:

```text
UNKNOWN
```

deberá preservarse.

---

# 16. ReplicationTopology

Modelo:

```php
final readonly class ReplicationTopology
{
    public function __construct(
        public ReplicationTopologyId $id,
        public TopologyGeneration $generation,
        public ReplicationNodeSet $nodes,
        public ReplicationEdgeSet $edges,
        public ReplicationTopologyMetadata $metadata,
    ) {}
}
```

---

# 17. Graph model

La topología se modelará como grafo:

```text
G = (N, E)
```

donde:

```text
N = replication nodes
E = replication relationships
```

---

# 18. Simple topology

```text
             ┌── Replica A
Primary ─────┼── Replica B
             └── Replica C
```

---

# 19. Cascading topology

```text
Primary
   │
   ├── Replica A
   │      │
   │      └── Replica C
   │
   └── Replica B
```

---

# 20. Multi-region topology

```text
Authoritative
     │
     ├── Region MX Replica
     ├── Region US Replica
     └── Region EU Replica
```

---

# 21. ReplicationEdge

Relación:

```php
final readonly class ReplicationEdge
{
    public function __construct(
        public ReplicationNodeId $source,
        public ReplicationNodeId $target,
        public ReplicationChannelId $channel,
        public ReplicationMode $mode,
    ) {}
}
```

---

# 22. Replication direction

```text
Source
  │
  ▼
Target
```

deberá ser explícita.

---

# 23. No inferred source

VoltStack no deberá inferir:

```text
all replicas → configured writer
```

si la topología real declara otra cosa.

---

# 24. ReplicationMode

Modelo conceptual:

```php
enum ReplicationMode
{
    case ASYNCHRONOUS;
    case SEMI_SYNCHRONOUS;
    case SYNCHRONOUS;
    case MANAGED;
    case UNKNOWN;
}
```

---

# 25. Mode semantics

Estos nombres describen semántica de alto nivel.

No deberán fingir equivalencia exacta entre tecnologías diferentes.

---

# 26. MANAGED

Un proveedor puede ocultar detalles de replicación.

VoltStack podrá conocer:

```text
replica relationship exists
```

sin conocer el protocolo exacto.

---

# 27. UNKNOWN

Nunca deberá normalizarse automáticamente como:

```text
ASYNCHRONOUS
```

---

# 28. Topology source

La topología puede provenir de:

```text
STATIC_CONFIGURATION
PLATFORM_DISCOVERY
PROVIDER_DISCOVERY
RUNTIME_DISCOVERY
HYBRID
```

---

# 29. TopologySource

```php
enum ReplicationTopologySource
{
    case STATIC_CONFIGURATION;
    case PLATFORM_DISCOVERY;
    case PROVIDER_DISCOVERY;
    case RUNTIME_DISCOVERY;
    case HYBRID;
}
```

---

# 30. Static configuration

Ejemplo:

```php
'replication' => [
    'source' => 'writer',

    'replicas' => [
        'replica-1',
        'replica-2',
    ],
],
```

---

# 31. Static configuration limitation

Configuración estática expresa:

```text
declared topology
```

No necesariamente:

```text
current observed topology
```

---

# 32. Declared vs observed

VoltStack distinguirá:

```text
DeclaredTopology
ObservedTopology
EffectiveTopology
```

---

# 33. DeclaredTopology

Representa intención/configuración.

---

# 34. ObservedTopology

Representa evidencia obtenida en runtime.

---

# 35. EffectiveTopology

Resultado validado utilizado por los consumidores.

---

# 36. Topology reconciliation

Conceptualmente:

```text
Declared Topology
        +
Observed Evidence
        │
        ▼
Topology Reconciliation
        │
        ▼
Effective Topology Snapshot
```

---

# 37. Reconciliation ≠ mutation

El reconciler no deberá modificar arbitrariamente la configuración original.

Produce un nuevo snapshot.

---

# 38. Immutable snapshots

Cada:

```text
ReplicationTopologySnapshot
```

será immutable.

---

# 39. Topology generation

Cada cambio relevante producirá:

```text
TopologyGeneration
```

---

# 40. Why generation

Permite correlacionar:

```text
routing decision
health evidence
replica state
failover
lag measurement
```

con la topología observada.

---

# 41. Snapshot model

```php
final readonly class ReplicationTopologySnapshot
{
    public function __construct(
        public ReplicationTopologyId $id,
        public TopologyGeneration $generation,
        public ReplicationNodeSnapshotSet $nodes,
        public ReplicationEdgeSet $edges,
        public Instant $capturedAt,
        public TopologyConfidence $confidence,
    ) {}
}
```

---

# 42. Topology confidence

```php
enum TopologyConfidence
{
    case VERIFIED;
    case DISCOVERED;
    case CONFIGURED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 43. Configured ≠ verified

Regla:

```text
CONFIGURED
≠
VERIFIED
```

---

# 44. Replica lifecycle

Una replica podrá atravesar:

```text
DISCOVERED
↓
INITIALIZING
↓
SYNCING
↓
READY
↓
DEGRADED
↓
UNAVAILABLE
```

con otros estados terminales/transitorios.

---

# 45. ReplicaState

Propuesta:

```php
enum ReplicaState
{
    case DISCOVERED;
    case INITIALIZING;
    case SYNCING;
    case READY;
    case DEGRADED;
    case PAUSED;
    case DISCONNECTED;
    case FAILED;
    case REMOVED;
    case UNKNOWN;
}
```

---

# 46. State ≠ health

Muy importante:

```text
ReplicaState
≠
EndpointHealth
```

Una replica puede estar:

```text
READY
```

pero con endpoint temporalmente:

```text
DEGRADED
```

---

# 47. State ≠ freshness

También:

```text
ReplicaState::READY
≠
ReplicaIsFullyCurrent
```

---

# 48. READY semantics

`READY` significa:

> La replica está considerada operacional dentro de su lifecycle de replicación.

No:

> Está exactamente en la misma posición que la fuente.

---

# 49. SYNCING

Una replica nueva puede estar recibiendo un snapshot inicial.

No deberá recibir tráfico normal salvo policy explícita.

---

# 50. PAUSED

Replication puede estar pausada aunque el DB endpoint siga respondiendo.

Por tanto:

```text
health = healthy
replication = paused
```

es posible.

---

# 51. DISCONNECTED

La conexión hacia la fuente de replicación está interrumpida.

El endpoint podría continuar sirviendo datos viejos.

---

# 52. FAILED

Indica fallo de replicación confirmado.

---

# 53. REMOVED

La replica ya no forma parte de la topología efectiva.

---

# 54. UNKNOWN

Ausencia de evidencia.

Nunca:

```text
UNKNOWN → READY
```

por conveniencia.

---

# 55. ReplicaCapabilities

Modelo:

```php
final readonly class ReplicaCapabilities
{
    public function __construct(
        public bool $canServeReads,
        public bool $supportsReplicationPosition,
        public bool $supportsLagObservation,
        public bool $supportsReadOnlyVerification,
        public bool $supportsPromotionObservation,
    ) {}
}
```

---

# 56. Capability model

Idealmente evolucionará hacia:

```text
CapabilitySet
```

en lugar de múltiples vendor conditionals.

---

# 57. canServeReads

Que un nodo sea replica no significa automáticamente que:

```text
application reads allowed
```

---

# 58. Examples

Puede existir:

```text
backup replica
```

que no debe recibir tráfico.

---

# 59. ReplicaPurpose

Modelo:

```php
enum ReplicaPurpose
{
    case APPLICATION_READ;
    case ANALYTICS;
    case BACKUP;
    case DISASTER_RECOVERY;
    case REPORTING;
    case INTERNAL;
}
```

---

# 60. Multiple purposes

Una replica podrá tener un conjunto de purposes.

---

# 61. Purpose as routing constraint

Una replica:

```text
BACKUP
```

no deberá entrar automáticamente en el pool de:

```text
APPLICATION_READ
```

---

# 62. ReplicaEligibility

El sistema producirá:

```php
final readonly class ReplicaEligibility
{
    public function __construct(
        public ReplicaId $replica,
        public ReplicaEligibilityStatus $status,
        public array $reasons,
    ) {}
}
```

---

# 63. Eligibility statuses

```php
enum ReplicaEligibilityStatus
{
    case ELIGIBLE;
    case INELIGIBLE;
    case DEGRADED;
    case UNKNOWN;
}
```

---

# 64. Eligibility ≠ selection

```text
ReplicaEligibility
≠
RoutingDecision
```

Replica System dice:

```text
can this replica participate?
```

Router decide:

```text
should this operation use it?
```

---

# 65. Base eligibility

Una replica puede ser elegible si:

```text
state = READY
AND
canServeReads = true
AND
purpose includes APPLICATION_READ
AND
endpoint available
AND
topology relationship valid
```

antes de considerar freshness/lag.

---

# 66. Lag-aware eligibility

Documento 179 añadirá:

```text
lag constraints
```

sobre esta base.

---

# 67. Eligibility reason codes

Ejemplos:

```text
REPLICA_READY
REPLICA_SYNCING
REPLICATION_PAUSED
REPLICATION_FAILED
NOT_APPLICATION_READER
ENDPOINT_UNAVAILABLE
TOPOLOGY_UNKNOWN
SOURCE_UNKNOWN
LAG_UNKNOWN
LAG_EXCEEDED
```

---

# 68. Replication position

VoltStack deberá disponer de una abstracción canónica:

```php
interface ReplicationPosition
{
    public function type(): ReplicationPositionType;
}
```

---

# 69. Position ≠ integer

No deberá asumir:

```text
position = numeric offset
```

porque diferentes sistemas utilizan representaciones distintas.

---

# 70. Opaque position

Preferir:

```php
final readonly class OpaqueReplicationPosition implements ReplicationPosition
{
    public function __construct(
        public ReplicationPositionType $type,
        public string $value,
    ) {}
}
```

---

# 71. Position security

El valor podrá necesitar sanitización en logs/diagnostics.

---

# 72. Position comparison

No deberá hacerse:

```php
$positionA > $positionB;
```

genéricamente.

---

# 73. ReplicationPositionComparator

```php
interface ReplicationPositionComparator
{
    public function compare(
        ReplicationPosition $a,
        ReplicationPosition $b,
    ): ReplicationPositionComparison;
}
```

---

# 74. Comparison results

```php
enum ReplicationPositionComparison
{
    case BEFORE;
    case EQUAL;
    case AFTER;
    case CONCURRENT;
    case INCOMPARABLE;
    case UNKNOWN;
}
```

---

# 75. Why CONCURRENT

Arquitecturas futuras pueden utilizar posiciones que no tengan un orden total simple.

---

# 76. Why INCOMPARABLE

Dos posiciones pueden pertenecer a:

```text
different replication domains
different epochs
different sources
different position formats
```

---

# 77. ReplicationEpoch

Podrá existir:

```php
final readonly class ReplicationEpoch
{
    public function __construct(
        public string $value,
    ) {}
}
```

para distinguir generaciones de autoridad.

---

# 78. Position identity

Conceptualmente:

```text
ReplicationCoordinate
=
ReplicationDomain
+
Epoch
+
Position
```

---

# 79. ReplicationCoordinate

```php
final readonly class ReplicationCoordinate
{
    public function __construct(
        public ReplicationDomainId $domain,
        public ReplicationEpoch $epoch,
        public ReplicationPosition $position,
    ) {}
}
```

---

# 80. Cross-epoch comparison

No deberá asumirse comparable.

---

# 81. Write observation integration

Después de una escritura confirmada, un sistema futuro podrá capturar:

```text
ReplicationCoordinate
```

del writer.

Posteriormente una replica podrá demostrar:

```text
ReplicaPosition >= RequiredPosition
```

si la plataforma soporta comparación válida.

---

# 82. Session consistency

Esto permitirá eventualmente:

```text
write
→ capture coordinate P
→ read
→ choose replica whose coordinate covers P
```

en lugar de usar siempre sticky writer.

---

# 83. Fallback

Si la plataforma no soporta posiciones:

```text
sticky writer
```

podrá continuar siendo una estrategia válida.

---

# 84. Position ≠ lag

Regla:

```text
ReplicationPosition
≠
ReplicaLag
```

---

# 85. Lag

Lag expresa una medida de atraso.

Position expresa ubicación/progreso en el stream de replicación.

---

# 86. Position freshness

Una replica puede tener:

```text
lag estimate = low
```

pero no demostrar una posición requerida específica.

---

# 87. Replica observation

Modelo:

```php
final readonly class ReplicaObservation
{
    public function __construct(
        public ReplicaId $replica,
        public ReplicaState $state,
        public ?ReplicationCoordinate $coordinate,
        public ReplicaObservationConfidence $confidence,
        public Instant $observedAt,
        public ?Instant $validUntil,
    ) {}
}
```

---

# 88. Observation confidence

```php
enum ReplicaObservationConfidence
{
    case VERIFIED;
    case PROVIDER_REPORTED;
    case PLATFORM_REPORTED;
    case INFERRED;
    case CONFIGURED;
    case UNKNOWN;
}
```

---

# 89. Observation ≠ definition

```text
DatabaseReplica
```

es definición/topología.

```text
ReplicaObservation
```

es evidencia dinámica.

---

# 90. No mutable replica singleton

No:

```php
$replica->state = READY;
```

sobre un singleton compartido mutable.

Preferir nuevos snapshots.

---

# 91. ReplicaSnapshot

```php
final readonly class ReplicaSnapshot
{
    public function __construct(
        public DatabaseReplica $definition,
        public ReplicaObservation $observation,
        public ReplicaEligibility $eligibility,
    ) {}
}
```

---

# 92. Discovery architecture

```text
Configuration
      │
      ├─────────────┐
      ▼             ▼
Static Discovery   Runtime Discovery
      │             │
      └──────┬──────┘
             ▼
     Topology Reconciler
             │
             ▼
      Topology Validator
             │
             ▼
     Immutable Snapshot
             │
             ▼
      Replica Registry
```

---

# 93. ReplicaDiscoveryProvider

```php
interface ReplicaDiscoveryProvider
{
    public function discover(
        ReplicaDiscoveryContext $context,
    ): ReplicationTopologyObservation;
}
```

---

# 94. Discovery provider types

Podrán existir:

```text
StaticReplicaDiscoveryProvider
PlatformReplicaDiscoveryProvider
ProviderApiReplicaDiscoveryProvider
ProxyReplicaDiscoveryProvider
CustomReplicaDiscoveryProvider
```

---

# 95. Discovery not in query hot path

Regla:

> El Query Executor no deberá iniciar topology discovery remoto cada vez que ejecuta un SELECT.

---

# 96. Discovery lifecycle

Puede ejecutarse:

```text
bootstrap
periodically
on explicit refresh
after topology event
after failure
```

según runtime/policy.

---

# 97. Refresh ≠ mutation

Refresh produce:

```text
new topology snapshot
```

---

# 98. Discovery failure

Si discovery falla:

```text
current known snapshot
```

podrá conservarse temporalmente bajo política de freshness.

No deberá marcarse automáticamente como actual.

---

# 99. Stale topology

Debe ser representable:

```text
TopologyFreshness::STALE
```

---

# 100. TopologyFreshness

```php
enum TopologyFreshness
{
    case CURRENT;
    case AGING;
    case STALE;
    case EXPIRED;
    case UNKNOWN;
}
```

---

# 101. Expired topology

Una policy estricta podrá rechazar routing basado en una topología expirada.

---

# 102. Static topology

En deployments simples, una configuración estática puede permanecer válida indefinidamente si esa es la política explícita.

---

# 103. Platform adapters

El core deberá definir abstracciones.

Los adapters concretos resolverán particularidades de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
managed providers
proxies
```

---

# 104. MySQL/MariaDB separation

Aunque compartan conceptos, VoltStack no deberá asumir que:

```text
MySQL replication behavior
=
MariaDB replication behavior
```

en todos los casos.

---

# 105. PostgreSQL

Sus conceptos de replication position deberán adaptarse a la abstracción:

```text
ReplicationCoordinate
```

sin filtrarse al router genérico.

---

# 106. SQLite

SQLite local normalmente no tendrá:

```text
replication topology
```

dentro del core.

---

# 107. SQLite replica capability

En un deployment normal:

```text
ReplicaSystem = not applicable
```

No:

```text
single DB = replica of itself
```

---

# 108. Managed providers

Servicios administrados pueden exponer:

```text
writer endpoint
reader endpoint
cluster endpoint
```

sin revelar nodos físicos.

VoltStack deberá permitir nodos lógicos.

---

# 109. Logical replica

Una replica puede representar:

```text
logical managed replica endpoint
```

aunque detrás existan varios servidores.

---

# 110. Provider integration boundary

Proveedor:

```text
discovers/reports topology
```

Core:

```text
normalizes semantic topology
```

---

# 111. No provider conditionals

Evitar:

```php
if ($provider === 'foo-cloud') {
    // ...
}
```

en el Replica System core.

---

# 112. Capability adapters

Preferir:

```text
ReplicaDiscoveryCapability
ReplicationPositionCapability
ReplicaLagCapability
ReplicaPromotionCapability
```

---

# 113. Replica registry

Un registry podrá exponer snapshots:

```php
interface ReplicaRegistry
{
    public function topology(
        ReplicationTopologyId $id,
    ): ReplicationTopologySnapshot;
}
```

---

# 114. Registry state

El registry podrá mantener:

```text
atomic reference to latest immutable snapshot
```

en persistent runtimes.

---

# 115. Atomic publication

Nuevo snapshot:

```text
build
→ validate
→ freeze
→ publish atomically
```

---

# 116. No partial topology publication

Nunca:

```text
add replica A
publish
add edge
publish
remove old primary
publish
```

si representa una única actualización lógica.

---

# 117. Readers see complete generation

Cada operación deberá observar:

```text
one coherent topology generation
```

---

# 118. Mid-routing topology change

Si la topología cambia durante routing:

```text
current RoutingContext
```

continúa con su snapshot salvo que una policy explícita ordene reroute antes de adquisición.

---

# 119. Transaction topology stability

Una transaction activa no migrará porque:

```text
topology generation changed
```

---

# 120. Transaction pinned node promoted/demoted

Si la conexión sigue viva:

VoltStack no deberá inventar nuevas garantías.

El Transaction System continúa ligado a su physical connection y responderá a errores reales.

---

# 121. Writer demotion

Si el nodo deja de aceptar writes durante una transaction:

```text
transaction failure
```

No transparent failover.

---

# 122. Outside transaction

Una nueva operación podrá usar la nueva topología.

---

# 123. Replica and transaction

Una transaction read-only puede ejecutarse sobre una replica si:

```text
replica eligible
AND
connection supports transaction
AND
requested consistency satisfied
AND
routing policy permits
```

---

# 124. Write transaction

No deberá utilizar una replica read-only.

---

# 125. Replica promotion

Una replica puede convertirse en writer durante failover.

Pero:

```text
ReplicaPromotion
```

no se implementa aquí.

---

# 126. Role transition

El Replica System podrá observar:

```text
REPLICA → AUTHORITATIVE
```

como cambio de topología.

---

# 127. Identity preservation

El nodo puede conservar una identidad operacional, pero su:

```text
ReplicationNodeRole
```

cambia en nueva generación.

---

# 128. Split-brain risk

Si discovery reporta múltiples nodos como authoritative de forma incompatible:

```text
TopologyConflict
```

deberá representarse.

---

# 129. No arbitrary winner

VoltStack no deberá hacer:

```text
pick first authoritative node
```

ante conflicto crítico.

---

# 130. TopologyConflict

```php
final readonly class TopologyConflict
{
    public function __construct(
        public TopologyConflictType $type,
        public array $nodes,
        public TopologyConflictSeverity $severity,
    ) {}
}
```

---

# 131. Conflict types

```text
MULTIPLE_AUTHORITATIVE_NODES
MISSING_AUTHORITATIVE_NODE
REPLICATION_CYCLE
DANGLING_SOURCE
DUPLICATE_NODE_IDENTITY
INCOMPATIBLE_EPOCH
CONTRADICTORY_ROLE_EVIDENCE
```

---

# 132. Replication cycle

Un ciclo puede ser válido en algunas tecnologías avanzadas, pero no deberá asumirse válido.

---

# 133. Capability-driven cycles

Topology validator podrá aceptar ciclos solo cuando:

```text
topology mode
+
platform capability
```

los soporte explícitamente.

---

# 134. Missing source

Una replica con:

```text
unknown source
```

podrá existir como observación parcial.

Pero no deberá fingirse una relación.

---

# 135. Partial topology

VoltStack deberá soportar:

```text
PARTIAL
```

en lugar de exigir datos inventados.

---

# 136. Replica metadata

Podrá contener:

```text
region
zone
purpose
provider reference
priority
tags
```

---

# 137. Metadata restrictions

No incluir:

```text
credentials
raw provider secrets
large arbitrary payloads
```

---

# 138. Region

Region será metadata/topology dimension.

No garantía automática de:

```text
lower latency
```

---

# 139. Zone

Puede utilizarse posteriormente para resiliencia/load balancing.

---

# 140. Replica priority

Puede expresar preferencia administrativa.

No deberá reemplazar:

```text
health
consistency
eligibility
```

---

# 141. Routing integration

Flujo:

```text
ReplicaRegistry
      │
      ▼
TopologySnapshot
      │
      ▼
ReplicaEligibility
      │
      ▼
Read/Write Router
      │
      ▼
Eligible Replica Candidates
      │
      ▼
Selector / Load Balancer
```

---

# 142. Replica System does not route

Regla:

```text
ReplicaSystem
≠
ReadWriteRoutingSystem
```

---

# 143. Routing evidence adapter

Podrá existir:

```php
interface ReplicaRoutingEvidenceProvider
{
    public function evidence(
        ReplicationTopologySnapshot $topology,
    ): RoutingEvidenceSnapshot;
}
```

---

# 144. Evidence transformation

Replica-specific concepts podrán normalizarse para router:

```text
replica state
purpose
health
freshness
lag
authority
```

---

# 145. Read candidate construction

Una replica podrá convertirse en:

```text
ConnectionCandidate
```

solo si posee un endpoint de conexión válido.

---

# 146. Replica without application endpoint

Puede existir en topology pero no en routing.

Ejemplo:

```text
backup replica
```

---

# 147. Connection endpoint without replica

También puede existir:

```text
read endpoint
```

que no corresponde a una replica conocida.

---

# 148. Many-to-one mapping

Un endpoint gestionado podría representar varias replicas internas.

---

# 149. One-to-many mapping

Una identidad topológica podría exponer varios endpoints especializados.

El modelo no deberá forzar relación 1:1 universal.

---

# 150. Mapping abstraction

Por tanto, podrá existir:

```text
ReplicaEndpointBinding
```

en lugar de codificar siempre:

```text
ReplicaId → EndpointId
```

como identidad equivalente.

---

# 151. ReplicaEndpointBinding

```php
final readonly class ReplicaEndpointBinding
{
    public function __construct(
        public ReplicaId $replica,
        public EndpointId $endpoint,
        public ReplicaEndpointPurpose $purpose,
    ) {}
}
```

---

# 152. Read-only verification

Cuando la plataforma lo permita:

```text
replica expected read-only
+
observed read-only state
```

aumenta confianza.

---

# 153. Verification failure

Si una replica configurada como read-only acepta writes:

eso no la convierte automáticamente en writer.

Debe generar:

```text
topology/capability anomaly
```

---

# 154. Authority is explicit

Write authority procede de:

```text
effective topology
+
validated role evidence
+
routing policy
```

No de:

```text
can execute UPDATE
```

únicamente.

---

# 155. Health integration

Replica System consumirá health evidence de componentes especializados.

---

# 156. Health dimensions

Podrán distinguirse:

```text
endpoint connectivity health
replication health
replication freshness
provider health
```

---

# 157. Aggregate health

No deberá colapsarse prematuramente todo en:

```text
healthy = true/false
```

---

# 158. ReplicaHealthProfile

```php
final readonly class ReplicaHealthProfile
{
    public function __construct(
        public EndpointHealth $endpoint,
        public ReplicationHealth $replication,
        public ReplicaFreshnessState $freshness,
    ) {}
}
```

---

# 159. ReplicationHealth

```php
enum ReplicationHealth
{
    case HEALTHY;
    case DEGRADED;
    case BROKEN;
    case PAUSED;
    case UNKNOWN;
}
```

---

# 160. Freshness separated

El detalle completo pertenece al documento 179.

---

# 161. Persistent runtime model

En FrankenPHP/RoadRunner/OpenSwoole:

```text
immutable topology snapshots
```

podrán compartirse.

---

# 162. Mutable observation collection

Debe estar controlada y concurrency-safe.

---

# 163. No request-owned topology mutation

Una request no deberá hacer:

```php
$registry->replica('r1')->state = FAILED;
```

---

# 164. Observation publication

Debe pasar por:

```text
collector
→ validator
→ reconciler
→ immutable snapshot
→ atomic publication
```

---

# 165. Scope state

Request-local:

```text
selected topology generation
routing evidence snapshot
replica selection
```

---

# 166. Shared state

Worker-level:

```text
latest immutable topology snapshot
compiled discovery providers
compiled capability adapters
```

---

# 167. Concurrent refresh

Dos refresh simultáneos deberán evitar:

```text
older snapshot overwrites newer snapshot
```

---

# 168. Compare-and-publish

Puede utilizarse:

```text
generation/version check
```

antes de publicar.

---

# 169. Snapshot retention

Solo deberá retenerse un número bounded de generaciones si se necesita debugging.

---

# 170. No unbounded history

Persistent worker no deberá acumular:

```text
all topology snapshots forever
```

---

# 171. Discovery scheduling

La frecuencia dependerá de:

```text
provider
topology volatility
deployment mode
failure events
policy
```

---

# 172. No universal polling frequency

Core no deberá asumir:

```text
refresh every 5 seconds
```

para todos.

---

# 173. Event-driven discovery

Proveedores podrán soportar:

```text
topology change notification
```

---

# 174. Polling fallback

Otros usarán:

```text
periodic refresh
```

---

# 175. Resource governance

Discovery deberá tener:

```text
timeouts
cancellation
rate limits
bounded retries
bounded payload
```

---

# 176. Discovery failure isolation

Una falla del discovery provider no deberá bloquear indefinidamente query execution.

---

# 177. Security model

Replica discovery puede manejar información sensible de infraestructura.

---

# 178. Secret separation

Discovery credentials:

```text
ProviderCredentialReference
```

no deberán almacenarse dentro de topology snapshots.

---

# 179. Untrusted metadata

Provider metadata deberá normalizarse/validarse antes de incorporarse.

---

# 180. No dynamic class loading

Datos de topology no podrán contener:

```text
PHP class to instantiate
```

---

# 181. No unserialize

Prohibido utilizar:

```php
unserialize($providerPayload);
```

sobre datos externos.

---

# 182. Endpoint injection

Un provider comprometido no deberá poder introducir arbitrariamente un endpoint fuera del execution domain autorizado sin validación.

---

# 183. Allowlisted domains

Podrán existir políticas:

```text
allowed clusters
allowed regions
allowed network domains
```

en integraciones empresariales.

---

# 184. Tenant isolation

Cuando Multitenancy esté instalado:

```text
tenant A topology
```

no deberá mezclarse con:

```text
tenant B topology
```

---

# 185. Topology key

Conceptualmente:

```text
ReplicationTopologyKey
=
DatabaseExecutionDomain
+
ReplicationTopologyId
```

---

# 186. Shard isolation

Cada shard podrá tener su propia:

```text
ReplicationTopology
```

---

# 187. Shard first

```text
Shard resolution
↓
Shard topology
↓
Replica eligibility
↓
Read routing
```

---

# 188. Telemetry

Eventos:

```text
ReplicaDiscovered
ReplicaStateChanged
ReplicaRemoved
ReplicaEligibilityChanged
TopologyObserved
TopologyChanged
TopologyConflictDetected
TopologyDiscoveryFailed
```

---

# 189. Events observational

Eventos normales no deberán mutar la topology en medio de dispatch.

---

# 190. Topology changes through coordinator

Cambios:

```text
observation
→ reconciliation
→ validation
→ publication
```

---

# 191. Metrics

Ejemplos:

```text
db.replica.count
db.replica.ready
db.replica.degraded
db.replica.failed
db.replica.discovery.duration
db.replica.discovery.failures
db.replica.topology.generation
db.replica.topology.conflicts
```

---

# 192. Lag metrics

Se definirán detalladamente en 179.

---

# 193. Cardinality

Evitar labels como:

```text
tenant id
full endpoint
replica id
database name
```

cuando puedan ser de alta cardinalidad.

---

# 194. Tracing

Ejemplo:

```text
db.replication.topology_source = provider
db.replication.topology_confidence = verified
db.replication.replica_count = 3
```

---

# 195. Diagnostics API

Conceptualmente:

```php
DB::replicas('default')->inspect();
```

---

# 196. Example diagnostics

```text
REPLICATION TOPOLOGY

Connection:
    default

Generation:
    42

Source:
    PROVIDER_DISCOVERY

Confidence:
    VERIFIED

Authoritative:
    writer-1

Replicas:
    replica-1
        state: READY
        purpose: APPLICATION_READ
        position: available
        eligible: yes

    replica-2
        state: SYNCING
        purpose: APPLICATION_READ
        eligible: no

    replica-3
        state: READY
        purpose: BACKUP
        eligible: no
```

---

# 197. Topology explain

```php
DB::replicas('default')
    ->explain('replica-2');
```

---

# 198. Explain result

```text
Replica:
    replica-2

State:
    SYNCING

Endpoint:
    available

Replication:
    active

Application Read Eligibility:
    NO

Reason:
    INITIAL_SYNCHRONIZATION_IN_PROGRESS

Lag:
    not evaluated by this layer
```

---

# 199. Conflict diagnostics

```text
TOPOLOGY CONFLICT

Type:
    MULTIPLE_AUTHORITATIVE_NODES

Severity:
    CRITICAL

Nodes:
    writer-1
    replica-2

Routing impact:
    WRITE routing suspended

Read impact:
    policy dependent
```

---

# 200. Error hierarchy

```text
DatabaseReplicaException
├── ReplicaNotFoundException
├── ReplicaStateException
├── ReplicaCapabilityException
├── ReplicaEligibilityException
├── ReplicaDiscoveryException
├── ReplicaDiscoveryTimeoutException
├── ReplicaTopologyException
├── ReplicaTopologyConflictException
├── ReplicaTopologyExpiredException
├── ReplicaTopologyValidationException
├── ReplicaSourceUnknownException
├── ReplicaPositionException
├── ReplicaPositionIncomparableException
├── ReplicaObservationException
├── ReplicaEndpointBindingException
├── ReplicaDomainMismatchException
└── ReplicaInvariantViolationException
```

---

# 201. Testing architecture

Suite:

```text
ReplicaIdentityTests
ReplicaDefinitionTests
ReplicationTopologyTests
ReplicationGraphTests
ReplicaStateTests
ReplicaCapabilityTests
ReplicaPurposeTests
ReplicaEligibilityTests
ReplicaDiscoveryTests
TopologyReconciliationTests
TopologyValidationTests
TopologyGenerationTests
ReplicaObservationTests
ReplicationPositionTests
ReplicaEndpointBindingTests
ReplicaRoutingIntegrationTests
ReplicaTransactionIntegrationTests
ReplicaPersistentRuntimeTests
ReplicaConcurrencyTests
ReplicaSecurityTests
ReplicaTelemetryTests
ReplicaDiagnosticsTests
```

---

# 202. Identity test

Verificar:

```text
ReplicaId
≠
EndpointId
```

y que ambas identidades permanezcan estables según sus propios contratos.

---

# 203. Read role test

Configurar:

```text
READ endpoint
```

sin replication metadata.

Resultado:

```text
ReadConnection = yes
Replica = no
```

---

# 204. Replica purpose test

Replica:

```text
READY
+
BACKUP
```

no deberá ser application-read eligible.

---

# 205. Syncing test

```text
SYNCING
```

deberá ser ineligible por default.

---

# 206. Paused replication test

Endpoint:

```text
HEALTHY
```

Replication:

```text
PAUSED
```

La replica no deberá considerarse fresh/normal simplemente por responder.

---

# 207. Unknown state test

```text
UNKNOWN
```

no se convertirá en READY.

---

# 208. Topology generation test

Cambiar:

```text
replica-2 READY → FAILED
```

deberá producir una nueva generation.

---

# 209. Atomic publication test

Readers concurrentes deberán observar:

```text
generation N
```

o:

```text
generation N+1
```

pero nunca un estado parcial entre ambas.

---

# 210. Stale discovery test

Discovery falla.

Snapshot anterior permanece disponible pero:

```text
freshness = STALE
```

cuando corresponda.

---

# 211. Conflict test

Dos authoritative nodes incompatibles deberán producir:

```text
TopologyConflict
```

No selección arbitraria.

---

# 212. Position comparison test

Posiciones de diferentes epochs:

```text
INCOMPARABLE
```

salvo adapter con semántica explícita.

---

# 213. Routing integration test

Solo replicas:

```text
eligible
+
application read purpose
```

deberán entrar al candidate set normal.

---

# 214. Transaction test

Una write transaction nunca deberá ser trasladada a replica.

---

# 215. Worker test

Miles de topology refreshes no deberán producir:

```text
unbounded snapshot retention
state leakage
partial publication
```

---

# 216. Coroutine test

Concurrent routing operations deberán poder observar snapshots coherentes sin mutarlos.

---

# 217. Proposed directory structure

```text
src/Quantum/Database/Replication/
│
├── ReplicaId.php
├── DatabaseReplica.php
├── ReplicaPurpose.php
├── ReplicaState.php
├── ReplicaCapabilities.php
├── ReplicaMetadata.php
│
├── Topology/
│   ├── ReplicationTopologyId.php
│   ├── ReplicationTopology.php
│   ├── ReplicationTopologySnapshot.php
│   ├── ReplicationTopologySource.php
│   ├── ReplicationNode.php
│   ├── ReplicationNodeId.php
│   ├── ReplicationNodeRole.php
│   ├── ReplicationNodeSet.php
│   ├── ReplicationEdge.php
│   ├── ReplicationEdgeSet.php
│   ├── ReplicationMode.php
│   ├── ReplicationChannelId.php
│   ├── TopologyGeneration.php
│   ├── TopologyConfidence.php
│   ├── TopologyFreshness.php
│   └── TopologyConflict.php
│
├── Position/
│   ├── ReplicationPosition.php
│   ├── ReplicationPositionType.php
│   ├── OpaqueReplicationPosition.php
│   ├── ReplicationPositionComparator.php
│   ├── ReplicationPositionComparison.php
│   ├── ReplicationEpoch.php
│   └── ReplicationCoordinate.php
│
├── Observation/
│   ├── ReplicaObservation.php
│   ├── ReplicaObservationConfidence.php
│   ├── ReplicaSnapshot.php
│   ├── ReplicationTopologyObservation.php
│   └── ReplicaObservationCollector.php
│
├── Discovery/
│   ├── ReplicaDiscoveryProvider.php
│   ├── ReplicaDiscoveryContext.php
│   ├── StaticReplicaDiscoveryProvider.php
│   ├── PlatformReplicaDiscoveryProvider.php
│   └── ReplicaDiscoveryCoordinator.php
│
├── Reconciliation/
│   ├── ReplicationTopologyReconciler.php
│   ├── ReplicationTopologyValidator.php
│   └── TopologyConflictDetector.php
│
├── Eligibility/
│   ├── ReplicaEligibility.php
│   ├── ReplicaEligibilityStatus.php
│   ├── ReplicaEligibilityEvaluator.php
│   └── ReplicaEligibilityReason.php
│
├── Endpoint/
│   ├── ReplicaEndpointBinding.php
│   ├── ReplicaEndpointPurpose.php
│   └── ReplicaEndpointBindingRegistry.php
│
├── Health/
│   ├── ReplicaHealthProfile.php
│   └── ReplicationHealth.php
│
├── Registry/
│   ├── ReplicaRegistry.php
│   ├── DefaultReplicaRegistry.php
│   └── ReplicationTopologyPublisher.php
│
├── Routing/
│   └── ReplicaRoutingEvidenceProvider.php
│
├── Diagnostics/
│   ├── ReplicaInspector.php
│   ├── ReplicaExplainer.php
│   └── ReplicationTopologyReport.php
│
├── Telemetry/
│   ├── ReplicaTelemetry.php
│   ├── ReplicaDiscovered.php
│   ├── ReplicaStateChanged.php
│   ├── TopologyChanged.php
│   └── TopologyConflictDetected.php
│
└── Exception/
    ├── DatabaseReplicaException.php
    ├── ReplicaNotFoundException.php
    ├── ReplicaStateException.php
    ├── ReplicaCapabilityException.php
    ├── ReplicaEligibilityException.php
    ├── ReplicaDiscoveryException.php
    ├── ReplicaTopologyException.php
    ├── ReplicaTopologyConflictException.php
    ├── ReplicaTopologyExpiredException.php
    ├── ReplicaPositionException.php
    ├── ReplicaDomainMismatchException.php
    └── ReplicaInvariantViolationException.php
```

---

# 218. Architectural invariants

## DB-REP-001
Replica será distinta de ReadConnection.

## DB-REP-002
Replica será distinta de ConnectionRole::READ.

## DB-REP-003
Replica será distinta de EndpointHealth.

## DB-REP-004
Replica será distinta de freshness.

## DB-REP-005
Replica será distinta de authoritative node.

## DB-REP-006
ReplicaId será distinta de EndpointId.

## DB-REP-007
Replication topology será explícita.

## DB-REP-008
Replication direction será explícita.

## DB-REP-009
Replica source no se inferirá siempre del configured writer.

## DB-REP-010
Topology será modelada como grafo.

## DB-REP-011
Cascading replication será representable.

## DB-REP-012
Managed topology será representable.

## DB-REP-013
Unknown topology details permanecerán UNKNOWN.

## DB-REP-014
ReplicationMode UNKNOWN no será ASYNCHRONOUS implícitamente.

## DB-REP-015
Declared topology será distinta de observed topology.

## DB-REP-016
Observed topology será distinta de effective topology.

## DB-REP-017
Topology reconciliation producirá nuevos snapshots.

## DB-REP-018
Topology snapshots serán immutable.

## DB-REP-019
Cada cambio relevante producirá nueva topology generation.

## DB-REP-020
Configured topology no implicará verified topology.

## DB-REP-021
ReplicaState será distinta de health.

## DB-REP-022
ReplicaState será distinta de freshness.

## DB-REP-023
READY no significará fully caught up.

## DB-REP-024
SYNCING será ineligible por default para application reads.

## DB-REP-025
PAUSED replication podrá coexistir con healthy endpoint.

## DB-REP-026
DISCONNECTED replica podrá seguir respondiendo con datos viejos.

## DB-REP-027
UNKNOWN state no será convertido en READY.

## DB-REP-028
Replica capabilities serán explícitas.

## DB-REP-029
Replica no implicará canServeReads.

## DB-REP-030
ReplicaPurpose será explícito.

## DB-REP-031
BACKUP replica no será application reader por default.

## DB-REP-032
Eligibility será distinta de routing selection.

## DB-REP-033
Replica System determinará participation eligibility.

## DB-REP-034
ReadWrite Router determinará actual selection.

## DB-REP-035
Lag constraints serán una capa adicional de eligibility.

## DB-REP-036
ReplicationPosition será abstracta.

## DB-REP-037
ReplicationPosition no se asumirá integer.

## DB-REP-038
Position comparison será platform/capability-driven.

## DB-REP-039
Positions de diferentes domains no serán comparables por default.

## DB-REP-040
Positions de diferentes epochs no serán comparables por default.

## DB-REP-041
Comparison podrá producir INCOMPARABLE.

## DB-REP-042
Comparison podrá producir UNKNOWN.

## DB-REP-043
ReplicationPosition será distinta de ReplicaLag.

## DB-REP-044
Low lag no probará required replication position.

## DB-REP-045
ReplicationCoordinate incluirá domain/epoch cuando sea necesario.

## DB-REP-046
Write observation podrá almacenar replication coordinate.

## DB-REP-047
Session consistency podrá usar position evidence cuando esté disponible.

## DB-REP-048
Sticky writer seguirá siendo fallback cuando position evidence no exista.

## DB-REP-049
ReplicaObservation será distinta de DatabaseReplica definition.

## DB-REP-050
Replica observations serán timestamped.

## DB-REP-051
Observation confidence será explícita.

## DB-REP-052
Configured observation será distinta de verified observation.

## DB-REP-053
Replica definitions compartidas no contendrán mutable state.

## DB-REP-054
Mutable replica singleton estará prohibido.

## DB-REP-055
Discovery será distinto de query execution.

## DB-REP-056
Query hot path no hará topology discovery remoto implícito.

## DB-REP-057
Discovery podrá ser static, platform, provider o runtime.

## DB-REP-058
Discovery failure no convertirá stale data en fresh data.

## DB-REP-059
Topology freshness será explícita.

## DB-REP-060
Expired topology podrá ser rechazada por policy.

## DB-REP-061
Static topology podrá tener policy de freshness propia.

## DB-REP-062
Platform adapters ocultarán vendor-specific position semantics.

## DB-REP-063
Core no tendrá vendor conditionals dispersos.

## DB-REP-064
MySQL y MariaDB no serán considerados idénticos por arquitectura.

## DB-REP-065
SQLite no será modelado como replica de sí mismo.

## DB-REP-066
Managed provider endpoints podrán ser nodos lógicos.

## DB-REP-067
Provider discovery se normalizará antes de uso.

## DB-REP-068
ReplicaRegistry publicará snapshots coherentes.

## DB-REP-069
Snapshot publication será atómica.

## DB-REP-070
Partial topology update no será visible.

## DB-REP-071
Una operación observará una topology generation coherente.

## DB-REP-072
Topology change no migrará active transaction.

## DB-REP-073
Write transaction no utilizará read-only replica.

## DB-REP-074
Read-only transaction podrá usar replica solo si satisface requisitos.

## DB-REP-075
Replica promotion será distinta de replica observation.

## DB-REP-076
Promotion real se tratará en Failover System.

## DB-REP-077
Role transition producirá nueva topology generation.

## DB-REP-078
Multiple incompatible authoritative nodes producirán conflict.

## DB-REP-079
Topology conflict no elegirá winner arbitrariamente.

## DB-REP-080
Missing authoritative node será representable.

## DB-REP-081
Dangling replication source será detectable.

## DB-REP-082
Replication cycles no serán asumidos válidos.

## DB-REP-083
Cycles requerirán capability/topology support explícito.

## DB-REP-084
Partial topology será representable.

## DB-REP-085
Unknown source no será inventado.

## DB-REP-086
Replica metadata no contendrá credentials.

## DB-REP-087
Region será metadata, no latency guarantee.

## DB-REP-088
Replica priority no superará consistency requirements.

## DB-REP-089
Replica priority no superará health constraints.

## DB-REP-090
Replica System no ejecutará routing final.

## DB-REP-091
Routing evidence podrá derivarse de replica snapshots.

## DB-REP-092
Replica sin application endpoint podrá existir.

## DB-REP-093
Read endpoint sin known replica podrá existir.

## DB-REP-094
Replica-to-endpoint mapping no se asumirá universalmente 1:1.

## DB-REP-095
Managed endpoint podrá representar múltiples physical replicas.

## DB-REP-096
Read-only verification será evidencia, no authority absoluta.

## DB-REP-097
Replica que acepta writes no se convertirá automáticamente en writer.

## DB-REP-098
Write authority será explícita.

## DB-REP-099
Endpoint health será distinto de replication health.

## DB-REP-100
Replication health será distinto de freshness.

## DB-REP-101
Aggregate boolean health no deberá destruir información relevante.

## DB-REP-102
Immutable topology snapshots podrán compartirse entre workers/scopes.

## DB-REP-103
Mutable observation collection será concurrency-safe.

## DB-REP-104
Request code no mutará shared topology directamente.

## DB-REP-105
Observation publication seguirá pipeline validado.

## DB-REP-106
Concurrent refresh no permitirá stale overwrite.

## DB-REP-107
Snapshot history será bounded.

## DB-REP-108
Persistent workers no acumularán topology generations indefinidamente.

## DB-REP-109
Discovery frequency será configurable/capability-driven.

## DB-REP-110
Core no impondrá polling frequency universal.

## DB-REP-111
Event-driven discovery será soportable.

## DB-REP-112
Polling discovery será soportable.

## DB-REP-113
Discovery tendrá timeout.

## DB-REP-114
Discovery tendrá cancellation.

## DB-REP-115
Discovery retries serán bounded.

## DB-REP-116
Discovery payload será bounded.

## DB-REP-117
Discovery failure no bloqueará indefinidamente query execution.

## DB-REP-118
Provider credentials no estarán en topology snapshot.

## DB-REP-119
Provider payload será validado.

## DB-REP-120
Provider payload no podrá causar dynamic class loading.

## DB-REP-121
Provider payload no se deserializará inseguramente.

## DB-REP-122
Endpoint injection será validada contra execution domain.

## DB-REP-123
Tenant topologies estarán aisladas.

## DB-REP-124
Shard topologies estarán aisladas.

## DB-REP-125
Shard se resolverá antes de replica selection.

## DB-REP-126
Replica telemetry será observational.

## DB-REP-127
Telemetry listeners no mutarán topology.

## DB-REP-128
Topology changes pasarán por coordinator/reconciler.

## DB-REP-129
Telemetry no expondrá secrets.

## DB-REP-130
Metrics evitarán high-cardinality identifiers por default.

## DB-REP-131
Diagnostics distinguirán topology, state, health y eligibility.

## DB-REP-132
Diagnostics distinguirán configured y observed evidence.

## DB-REP-133
Diagnostics serán bounded.

## DB-REP-134
Routing deberá poder explicar por qué una replica fue excluida.

## DB-REP-135
Replica lag será tratado por subsystem especializado.

## DB-REP-136
Replica System no inferirá freshness sin evidencia.

## DB-REP-137
Replica System no garantizará read-your-writes por sí solo.

## DB-REP-138
Replica System no garantizará transaction isolation.

## DB-REP-139
Replica System no garantizará durability.

## DB-REP-140
Replica System no implementará distributed consensus.

## DB-REP-141
Replica System no implementará transparent transaction failover.

## DB-REP-142
Replica System no transformará async replication en synchronous semantics.

## DB-REP-143
UNKNOWN permanecerá UNKNOWN cuando no exista evidencia.

## DB-REP-144
Correctness tendrá prioridad sobre replica utilization.

## DB-REP-145
Topology conflicts críticos podrán suspender write routing.

## DB-REP-146
Read routing durante topology conflict dependerá de policy explícita.

## DB-REP-147
Current topology snapshot tendrá identidad/generation.

## DB-REP-148
RoutingDecision podrá registrar topology generation utilizada.

## DB-REP-149
New operations podrán observar nuevas generations.

## DB-REP-150
Existing operation no cambiará silenciosamente de generation.

## DB-REP-151
Replica registry lookup no realizará hidden network I/O.

## DB-REP-152
Hot-path replica eligibility será evaluable desde snapshots.

## DB-REP-153
Expensive discovery estará fuera del hot path.

## DB-REP-154
Replica eligibility result será immutable.

## DB-REP-155
Replica observation result será immutable.

## DB-REP-156
Topology conflict result será immutable.

## DB-REP-157
Replication edge será directional.

## DB-REP-158
Replication source identity será explícita.

## DB-REP-159
Relay replicas serán representables.

## DB-REP-160
Cascading replication será representable.

## DB-REP-161
Multi-region metadata será representable.

## DB-REP-162
Region no sustituirá replication domain.

## DB-REP-163
Topology generation no sustituirá replication epoch.

## DB-REP-164
Replication epoch no sustituirá transaction identity.

## DB-REP-165
Replication coordinate no sustituirá commit outcome.

## DB-REP-166
Solo un commit confirmado podrá producir write observation confiable.

## DB-REP-167
UNKNOWN commit outcome no producirá falsa replication certainty.

## DB-REP-168
Rollback no avanzará write observation.

## DB-REP-169
AfterCommit integration podrá capturar replication evidence cuando sea soportado.

## DB-REP-170
Replica state transitions deberán ser observables.

## DB-REP-171
Invalid state transitions podrán ser rechazadas.

## DB-REP-172
REMOVED replica no será elegible.

## DB-REP-173
FAILED replica no será elegible por default.

## DB-REP-174
SYNCING replica no será elegible por default.

## DB-REP-175
Eligibility policy nunca podrá convertir topology mismatch en match.

---

# 219. Modelo formal

Sea una topología:

```text
T = (N, E, G)
```

donde:

```text
N = conjunto de nodos
E = relaciones de replicación
G = topology generation
```

Para una replica `r`:

```text
Replica(r)
⇔
Role(r) = REPLICA
∧
∃ edge(source, r)
```

salvo topologías parciales donde la fuente pueda ser desconocida explícitamente.

La elegibilidad base:

```text
BaseEligible(r) =
State(r) ∈ AllowedStates
∧
Purpose(r) contains APPLICATION_READ
∧
CanServeReads(r)
∧
EndpointEligible(r)
∧
DomainMatches(r)
∧
TopologyValid(r)
```

El documento 179 extenderá:

```text
ReadEligible(r, requirement)
=
BaseEligible(r)
∧
LagPolicySatisfied(r, requirement)
∧
FreshnessSatisfied(r, requirement)
```

---

# 220. Arquitectura de snapshots

```text
        Discovery Sources
        /      |       \
       /       |        \
 Config    Platform    Provider
       \       |        /
        \      |       /
         ▼     ▼      ▼
       Observation Set
              │
              ▼
          Reconciler
              │
              ▼
          Validator
              │
       ┌──────┴──────┐
       │             │
    Conflict?       Valid
       │             │
       ▼             ▼
Diagnostics      Immutable
                 Topology
                 Snapshot
                    │
                    ▼
               Atomic Publish
                    │
              ┌─────┴─────┐
              ▼           ▼
            Router      Telemetry
```

---

# 221. Arquitectura de lectura con replica

```text
READ operation
      │
      ▼
ConnectionRequirement
      │
      ▼
ReadWrite Router
      │
      ▼
Replica Registry Snapshot
      │
      ▼
Base Replica Eligibility
      │
      ▼
Lag/Freshness Evaluation
      │
      ▼
Eligible Replicas
      │
      ▼
Load Balancer
      │
      ▼
Selected Replica Endpoint
      │
      ▼
Connection Manager
      │
      ▼
Connection Lease
```

---

# 222. Arquitectura de consistencia futura

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
Replication Coordinate P
  │
  ▼
Session Write Observation
  │
  ▼
Later READ
  │
  ▼
Required Coordinate P
  │
  ▼
Replica A position < P ──► reject
Replica B position = P ──► eligible
Replica C position > P ──► eligible
```

cuando la plataforma pueda demostrar esas relaciones.

Si no:

```text
required position unavailable
→ sticky authoritative read
```

según policy.

---

# 223. Integración arquitectónica

```text
                 Database System
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
 Read/Write       Replication      Transaction
 Connections        System          System
        │              │              │
        └──────┬───────┴──────┬───────┘
               │              │
               ▼              ▼
         Routing Context   Constraints
               │              │
               └──────┬───────┘
                      ▼
             Read/Write Router
                      │
                      ▼
              Eligible Endpoint
                      │
                      ▼
              Connection Manager
```

---

# 224. Regla maestra final

> **VoltStack modelará las replicas como miembros explícitos de una topología de replicación y mantendrá separadas su identidad, estado, salud, propósito, posición y elegibilidad. Una replica no será considerada apta para lectura únicamente porque exista o responda; deberá satisfacer las garantías requeridas por la operación y aportar evidencia suficiente para las decisiones que dependan de su estado de replicación.**

En forma compacta:

```text
Replica Definition
        +
Topology
        +
Observation
        +
Health
        +
Purpose
        +
Freshness Evidence
        │
        ▼
Replica Eligibility
        │
        ▼
Routing Candidate
```

y nunca:

```text
Configured as "read"
        ↓
Assume replica
        ↓
Assume fresh
        ↓
Route query
```

---

# 225. Resultado arquitectónico

Con los documentos:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
```

VoltStack dispone ya de tres capas diferenciadas:

```text
Connection System
    │
    │ What connections/capabilities exist?
    ▼
Read/Write Routing
    │
    │ Which eligible connection should be used?
    ▼
Replica System
    │
    │ Which nodes actually participate in replication,
    │ and what is known about them?
    ▼
Replication Evidence
```

Esto prepara la arquitectura para introducir una variable crítica:

```text
TIME / PROGRESS
```

porque conocer que un nodo es una replica todavía no responde:

> **¿Qué tan atrás se encuentra respecto de la fuente autoritativa y puede satisfacer la consistencia requerida por esta lectura?**

---

# 226. Siguiente documento

```text
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
```

El siguiente documento diseñará:

```text
Replica Lag Awareness
├── Lag Observation
├── Lag Measurement
├── Lag Evidence
├── Lag Confidence
├── Measurement Freshness
├── Replication Progress
├── Time-Based Lag
├── Position-Based Lag
├── Unknown Lag
├── Lag Thresholds
├── Consistency Budgets
├── Replica Freshness Classification
├── Lag-Aware Eligibility
├── Lag-Aware Routing
└── Telemetry
```

manteniendo la regla:

```text
Replica exists
≠
Replica is healthy
≠
Replica is caught up
≠
Replica satisfies this read
```

y permitiendo evolucionar desde:

```text
READ → any replica
```

hacia:

```text
READ
→ Consistency Requirement
→ Replica Freshness Requirement
→ Lag/Position Evidence
→ Eligible Replica Set
→ Routing
```

sin acoplar el Query Engine a detalles específicos de MySQL, MariaDB, PostgreSQL o proveedores administrados.