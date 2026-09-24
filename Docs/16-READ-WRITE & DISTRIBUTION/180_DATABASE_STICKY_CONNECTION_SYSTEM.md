# 180_DATABASE_STICKY_CONNECTION_SYSTEM.md

# VoltStack Quantum Database
## Database Sticky Connection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 180 — Database Sticky Connection System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md`  
**Siguiente documento:** `181_DATABASE_FAILOVER_SYSTEM.md`

---

# 1. Propósito

`Database Sticky Connection System` define la arquitectura mediante la cual VoltStack preservará determinadas garantías de consistencia después de una escritura, manteniendo temporalmente lecturas posteriores sobre una fuente autoritativa —o exigiendo una replica capaz de demostrar suficiente progreso— cuando exista riesgo de leer datos obsoletos.

Los documentos anteriores establecieron:

```text
176 Read/Write Connection System
    ↓
roles y capacidades de conexión

177 Read/Write Routing System
    ↓
selección semántica de endpoints

178 Replica System
    ↓
topología e identidad de replicas

179 Replica Lag Awareness System
    ↓
evidencia de lag, posición y freshness
```

Este documento añade:

```text
Confirmed Write
      │
      ▼
Write Observation
      │
      ▼
Sticky Requirement
      │
      ▼
Sticky Context
      │
      ▼
Later Read
      │
      ├── Replica proves requirement
      │          ↓
      │       Replica
      │
      └── Cannot prove
                 ↓
          Authoritative Node
```

La regla central será:

> **Sticky routing en VoltStack representará una restricción temporal o causal de routing creada por una escritura confirmada; no será una conexión física retenida indefinidamente ni una variable global que fuerce todas las lecturas al writer.**

Formalmente:

```text
StickyRequirement
≠
PhysicalConnectionPin
≠
TransactionPinning
≠
ConnectionPooling
≠
HTTP Session
≠
ReplicaLag
```

---

# 2. Problema fundamental

Considérese:

```text
T0
Client → INSERT user
         ↓
       Writer
         ↓
       COMMIT
```

La replicación puede tardar:

```text
Writer
   │
   │ replication delay
   ▼
Replica
```

Inmediatamente:

```text
T0 + 10ms

Client → SELECT user
              ↓
           Replica
```

La replica podría responder:

```text
NOT FOUND
```

aunque la escritura haya sido confirmada.

Este problema aparece como:

```text
read-after-write inconsistency
```

---

# 3. Solución ingenua

Un sistema simple podría establecer:

```php
$sticky = true;
```

y después:

```text
all reads
→ writer
```

durante toda la request.

Esto mejora read-after-write, pero introduce problemas:

- reduce uso de replicas;
- mezcla scopes;
- no expresa qué escritura debe observarse;
- no utiliza replication position;
- puede durar demasiado;
- puede durar demasiado poco;
- no funciona correctamente entre requests;
- puede filtrarse entre workers;
- puede confundirse con transaction pinning.

VoltStack utilizará un modelo explícito.

---

# 4. Modelo conceptual

```text
Write
  │
  ▼
Commit confirmed
  │
  ▼
WriteObservation
  │
  ▼
StickyRequirement
  │
  ▼
StickyContext
  │
  ▼
Read Routing
```

---

# 5. Sticky routing ≠ sticky connection física

El nombre del subsistema mantiene el término conocido `Sticky Connection`, pero arquitectónicamente la unidad principal será:

```text
StickyRoutingRequirement
```

No:

```text
Connection object
```

---

# 6. Principio fundamental

VoltStack deberá preferir:

```text
sticky semantic requirement
```

sobre:

```text
sticky physical socket
```

---

# 7. Por qué

Una requirement puede decir:

```text
This read must observe at least write position P.
```

Entonces:

```text
Writer
```

no necesariamente es el único endpoint capaz de satisfacerla.

Una replica suficientemente avanzada también puede hacerlo.

---

# 8. Read-your-writes

El objetivo principal será proporcionar una base para:

```text
Read-Your-Writes Consistency
```

dentro de scopes definidos.

---

# 9. Read-your-writes ≠ strong global consistency

Garantizar:

```text
user observes their own confirmed writes
```

no implica:

```text
all clients globally observe all writes immediately
```

---

# 10. Sticky requirement

Modelo:

```php
interface StickyRequirement
{
    public function kind(): StickyRequirementKind;
}
```

---

# 11. StickyRequirementKind

```php
enum StickyRequirementKind
{
    case REQUIRED_POSITION;
    case AUTHORITATIVE;
    case TIME_WINDOW;
    case COMPOSITE;
}
```

---

# 12. REQUIRED_POSITION

Estrategia preferida cuando la infraestructura puede obtener una posición de replicación.

```php
final readonly class RequiredPositionStickyRequirement
    implements StickyRequirement
{
    public function __construct(
        public ReplicationCoordinate $requiredPosition,
    ) {}
}
```

---

# 13. Semántica

Después de:

```text
WRITE W
```

se obtiene:

```text
ReplicationCoordinate P
```

Entonces una lectura posterior exige:

```text
ObservedPosition(endpoint) >= P
```

cuando la relación de posiciones sea válida.

---

# 14. Replica suficientemente avanzada

Si:

```text
Replica A position < P
Replica B position = P
Replica C position > P
```

entonces:

```text
A → reject
B → eligible
C → eligible
```

---

# 15. Sticky does not always mean writer

Esta es una propiedad fundamental:

```text
StickyRequirement active
≠
Writer must always be used
```

Si una replica demuestra la requirement:

```text
sticky read
→ replica allowed
```

---

# 16. AUTHORITATIVE

Cuando no pueda establecerse una condición de replica equivalente:

```php
final readonly class AuthoritativeStickyRequirement
    implements StickyRequirement
{
}
```

exige:

```text
authoritative routing
```

---

# 17. TIME_WINDOW

Fallback aproximado:

```php
final readonly class TimeWindowStickyRequirement
    implements StickyRequirement
{
    public function __construct(
        public Instant $until,
    ) {}
}
```

---

# 18. Time window limitation

Una ventana:

```text
5 seconds
```

no demuestra que la replica haya alcanzado la escritura.

Por tanto:

```text
TimeElapsed
≠
ReplicationCaughtUp
```

---

# 19. Time-based sticky

Se utilizará como:

```text
operational fallback
```

no como prueba causal fuerte.

---

# 20. COMPOSITE

Podrá combinar condiciones:

```text
required position P
OR
authoritative endpoint
```

o:

```text
required position P
AND
deadline
```

---

# 21. StickyRequirement composition

La composición deberá ser explícita y limitada.

No se permitirá construir árboles arbitrarios sin control de complejidad.

---

# 22. WriteObservation

La activación sticky debe originarse en una escritura confirmada.

Modelo:

```php
final readonly class DatabaseWriteObservation
{
    public function __construct(
        public WriteObservationId $id,
        public DatabaseExecutionDomain $domain,
        public ?ReplicationCoordinate $coordinate,
        public Instant $committedAt,
        public WriteObservationConfidence $confidence,
    ) {}
}
```

---

# 23. Confirmed commit only

Regla crítica:

```text
Statement succeeded
≠
Write confirmed
```

Sticky fuerte solo podrá derivarse de:

```text
COMMIT CONFIRMED
```

o de una operación autocommit cuyo resultado sea inequívocamente confirmado.

---

# 24. UNKNOWN transaction outcome

Si:

```text
COMMIT sent
connection lost
outcome UNKNOWN
```

VoltStack no deberá crear:

```text
RequiredPosition(P)
```

como si la escritura estuviera confirmada.

---

# 25. Conservative handling

El sistema podrá producir:

```text
AuthoritativeReadRequired
```

como medida conservadora si una policy superior lo considera útil.

Pero nunca afirmará que el write ocurrió.

---

# 26. Rollback

Una transaction confirmada como:

```text
ROLLED_BACK
```

no genera sticky write observation.

---

# 27. Savepoint rollback

Escrituras revertidas mediante savepoint tampoco deberán alimentar el sticky state final.

---

# 28. Transaction integration

Durante una transaction:

```text
TransactionConnectionPin
```

tiene precedencia sobre sticky routing.

---

# 29. Inside transaction

```text
Active Transaction
      │
      ▼
Pinned Connection
```

No:

```text
Active Transaction
      │
      ▼
Sticky Router
      │
      ▼
Different Connection
```

---

# 30. Sticky evaluated outside transaction

La sticky requirement normalmente participa en:

```text
subsequent operations
```

fuera de la transaction que produjo el write.

---

# 31. Transaction commit integration

Flujo:

```text
Transaction
    │
    ├── INSERT
    ├── UPDATE
    └── DELETE
          │
          ▼
       COMMIT
          │
     confirmed?
      │       │
     NO      YES
      │       │
      ▼       ▼
 no write   WriteObservation
 sticky          │
                 ▼
          Sticky Activation
```

---

# 32. Multiple writes in one transaction

Una transaction puede contener:

```text
W1
W2
W3
```

No será necesario crear tres sticky requirements si el commit produce una posición que cubre todo el transaction outcome.

---

# 33. Commit position

Preferir:

```text
CommitReplicationCoordinate
```

sobre posiciones de statements individuales.

---

# 34. Multiple autocommit writes

Podrán producir:

```text
P1
P2
P3
```

El sticky state deberá conservar la requirement causal más fuerte necesaria.

---

# 35. Position dominance

Si:

```text
P2 AFTER P1
```

entonces:

```text
Required(P2)
```

puede sustituir:

```text
Required(P1)
```

dentro del mismo replication domain/epoch.

---

# 36. Incomparable positions

Si:

```text
P1 INCOMPARABLE P2
```

no se deberá eliminar ninguna arbitrariamente.

---

# 37. Multiple domains

Una aplicación puede utilizar:

```text
database A
database B
```

El sticky state deberá estar particionado por execution domain.

---

# 38. Sticky key

Conceptualmente:

```text
StickyKey
=
DatabaseExecutionDomain
+
ReplicationDomain
```

---

# 39. No cross-database contamination

Una escritura en:

```text
database A
```

no deberá forzar automáticamente lecturas de:

```text
database B
```

al writer.

---

# 40. No cross-shard contamination

Igualmente:

```text
Shard A sticky state
≠
Shard B sticky state
```

---

# 41. Tenant isolation

Cuando Multitenancy esté instalado:

```text
Tenant A sticky state
∩
Tenant B sticky state
=
∅
```

---

# 42. StickyContext

Modelo scoped:

```php
interface StickyContext
{
    public function requirement(
        StickyKey $key,
    ): ?StickyRequirement;
}
```

---

# 43. Mutable internal context

Internamente:

```php
interface MutableStickyContext extends StickyContext
{
    public function register(
        StickyKey $key,
        StickyRequirement $requirement,
    ): void;

    public function clear(
        StickyKey $key,
    ): void;
}
```

---

# 44. StickyContext responsibilities

Contendrá:

- active requirements;
- write observations;
- expiration;
- scope identity;
- domain identity;
- generation;
- diagnostic reasons.

No contendrá:

- EntityManager;
- entities;
- QueryResults;
- PDO connections;
- credentials;
- repositories.

---

# 45. Sticky scope

La sticky requirement deberá pertenecer a un scope explícito.

---

# 46. StickyScope

```php
enum StickyScope
{
    case OPERATION;
    case REQUEST;
    case JOB;
    case SESSION_TOKEN;
    case EXPLICIT;
}
```

---

# 47. REQUEST

Comportamiento clásico:

```text
write during request
→ later reads in same request use sticky semantics
```

---

# 48. Limitation of request scope

Una redirect produce:

```text
Request A
→ write
→ redirect
→ Request B
```

Si sticky state solo vive en Request A:

```text
Request B
```

puede leer una replica atrasada.

---

# 49. Cross-request sticky

Para soportarlo se requiere propagación explícita.

---

# 50. No implicit PHP session coupling

VoltStack Database core no dependerá directamente de:

```text
HTTP Session
```

---

# 51. Portable token model

Podrá utilizarse un:

```text
StickyConsistencyToken
```

portable.

---

# 52. StickyConsistencyToken

Conceptualmente:

```php
final readonly class StickyConsistencyToken
{
    public function __construct(
        public StickyTokenId $id,
        public DatabaseExecutionDomainReference $domain,
        public StickyRequirementDescriptor $requirement,
        public Instant $expiresAt,
        public StickyTokenIntegrity $integrity,
    ) {}
}
```

---

# 53. Token purpose

Permite transportar:

```text
minimum required consistency
```

entre scopes autorizados.

---

# 54. Token ≠ connection serialization

Nunca contendrá:

```text
PDO object
socket
ConnectionLease
TransactionContext
Savepoint
```

---

# 55. Token security

Un token propagado fuera del proceso deberá estar:

```text
authenticated
integrity protected
bounded
versioned
```

según el integration layer.

---

# 56. Token privacy

No deberá exponer innecesariamente:

- hostnames;
- credentials;
- tenant internals;
- raw topology;
- sensitive database identifiers.

---

# 57. Token replay

Replay de un consistency token puede ser aceptable si únicamente eleva la consistencia de lectura.

Pero deberá evaluarse según integration/security policy.

---

# 58. Token cannot lower consistency

Un cliente no deberá poder modificar:

```text
RequiredPosition(P2)
```

por:

```text
RequiredPosition(P1)
```

para debilitar una garantía establecida por el servidor.

---

# 59. Scope resolver

```php
interface StickyContextResolver
{
    public function current(): StickyContext;
}
```

---

# 60. No static current sticky

Prohibido:

```php
Sticky::$current = true;
```

---

# 61. Runtime scope

Resolver deberá integrarse con:

```text
HTTP Request
Queue Job
CLI Command
Scheduler
Fiber
Coroutine
Test Scope
```

sin asumir HTTP.

---

# 62. Sticky state machine

Conceptualmente:

```text
INACTIVE
   │
   │ confirmed write
   ▼
ACTIVE
   │
   ├── stronger write → ACTIVE(updated)
   │
   ├── replica catches up → SATISFIED
   │
   ├── expiration → EXPIRED
   │
   └── scope end → CLEARED
```

---

# 63. StickyState

```php
enum StickyState
{
    case INACTIVE;
    case ACTIVE;
    case SATISFIED;
    case EXPIRED;
    case CLEARED;
}
```

---

# 64. ACTIVE

Existe una requirement que puede afectar routing.

---

# 65. SATISFIED

Una replica candidata ha demostrado satisfacer la requirement.

Esto no necesariamente destruye inmediatamente el requirement.

---

# 66. Why retain requirement

Otra replica puede estar atrasada.

Por tanto:

```text
Replica A satisfies P
```

no significa:

```text
all replicas satisfy P
```

---

# 67. Sticky requirement vs candidate

La evaluación se hará por endpoint/replica.

---

# 68. StickyEvaluation

```php
final readonly class StickyEvaluation
{
    public function __construct(
        public StickyEvaluationStatus $status,
        public StickyEvaluationReason $reason,
    ) {}
}
```

---

# 69. Evaluation statuses

```php
enum StickyEvaluationStatus
{
    case SATISFIED;
    case UNSATISFIED;
    case UNKNOWN;
}
```

---

# 70. Writer evaluation

El nodo autoritativo que confirmó la escritura normalmente satisface la requirement de read-your-writes mientras continúe siendo la autoridad compatible.

---

# 71. Failover caveat

Después de failover:

```text
old writer
→ no longer authoritative
```

No deberá asumirse automáticamente que:

```text
new writer
```

contiene todas las escrituras previas.

El documento 181 definirá este problema.

---

# 72. Replica evaluation

Para `RequiredPosition(P)`:

```text
ReplicaPosition >= P
→ SATISFIED
```

---

# 73. Behind

```text
ReplicaPosition < P
→ UNSATISFIED
```

---

# 74. Unknown position

```text
UNKNOWN
→ UNKNOWN
```

---

# 75. Incomparable

```text
INCOMPARABLE
→ UNKNOWN
```

para strong sticky requirement.

---

# 76. Time window evaluation

Mientras:

```text
Now < StickyUntil
```

una time-window policy puede exigir writer.

---

# 77. Window expiration

Cuando:

```text
Now >= StickyUntil
```

la time-based requirement expira.

---

# 78. Expiration ≠ replica caught up

Regla crítica:

```text
StickyWindowExpired
≠
ReplicaCaughtUp
```

---

# 79. Time window semantics

Significa:

> La política deja de forzar writer después de este tiempo.

No:

> La replica está demostrablemente actualizada.

---

# 80. Hybrid strategy

Una estrategia más robusta:

```text
RequiredPosition P
+
maximum sticky duration
```

puede hacer:

```text
if replica >= P:
    use replica

else if within sticky deadline:
    use writer

else:
    apply fallback policy
```

---

# 81. Fallback policy

Opciones conceptuales:

```php
enum StickyFallbackPolicy
{
    case AUTHORITATIVE;
    case FAIL;
    case ALLOW_EVENTUAL;
    case CUSTOM;
}
```

---

# 82. AUTHORITATIVE

Fallback seguro típico:

```text
cannot prove replica
→ writer
```

---

# 83. FAIL

Útil cuando:

```text
required consistency
```

es obligatoria y ninguna fuente válida puede satisfacerla.

---

# 84. ALLOW_EVENTUAL

Solo mediante opt-in explícito.

Representa una degradación de consistencia.

---

# 85. No silent degradation

Nunca:

```text
required position unavailable
→ silently use any replica
```

---

# 86. Sticky policy

Modelo:

```php
final readonly class StickyRoutingPolicy
{
    public function __construct(
        public StickyActivationPolicy $activation,
        public StickyFallbackPolicy $fallback,
        public ?Duration $defaultWindow,
        public bool $preferReplicaWhenSatisfied,
    ) {}
}
```

---

# 87. Activation policies

```php
enum StickyActivationPolicy
{
    case DISABLED;
    case AFTER_WRITE;
    case AFTER_TRANSACTION_COMMIT;
    case EXPLICIT;
}
```

---

# 88. AFTER_WRITE

Para autocommit writes confirmados.

---

# 89. AFTER_TRANSACTION_COMMIT

Para transacciones.

No se activa tras cada statement interno.

---

# 90. EXPLICIT

La aplicación puede solicitar:

```php
DB::consistency()
    ->requireAuthoritativeReads();
```

---

# 91. Manual sticky activation

Puede ser útil después de:

- external synchronization;
- imported state;
- security-sensitive workflow;
- domain-specific consistency event.

---

# 92. Manual activation cannot fake write

Una activation manual podrá exigir:

```text
AUTHORITATIVE
```

pero no inventar una replication position no observada.

---

# 93. Read routing integration

Flujo completo:

```text
READ
 │
 ▼
RoutingContext
 │
 ├── Active Transaction?
 │       │
 │      YES
 │       ▼
 │   Pinned Connection
 │
 └── NO
     │
     ▼
 Sticky Requirement?
     │
  ┌──┴───┐
  │      │
 NO     YES
  │      │
  ▼      ▼
Normal  Evaluate Candidate
Route       │
            ├── Writer → valid?
            │
            ├── Replica A → satisfies?
            │
            ├── Replica B → satisfies?
            │
            └── Replica C → satisfies?
                     │
                     ▼
             Eligible Candidates
                     │
                     ▼
                  Router
```

---

# 94. Sticky as routing constraint

El Sticky System producirá:

```text
RoutingConstraint
```

No:

```text
selected connection
```

---

# 95. StickyRoutingConstraint

```php
final readonly class StickyRoutingConstraint
{
    public function __construct(
        public StickyKey $key,
        public StickyRequirement $requirement,
        public StickyFallbackPolicy $fallback,
    ) {}
}
```

---

# 96. Routing constraint composition

Debe combinarse con:

```text
TransactionConstraint
ReadWriteRoleConstraint
TenantConstraint
ShardConstraint
FreshnessConstraint
HealthConstraint
```

---

# 97. Constraint priority

Una prioridad conceptual:

```text
Transaction Pin
      ↓
Execution Domain
      ↓
Write Authority
      ↓
Consistency / Sticky
      ↓
Replica Freshness
      ↓
Health
      ↓
Load Balancing
```

---

# 98. Sticky does not override domain

Nunca:

```text
sticky writer from tenant A
→ query tenant B
```

---

# 99. Sticky does not override shard

Nunca:

```text
sticky shard A
→ route shard B read to shard A writer
```

---

# 100. Sticky does not override transaction

Transaction pinning gana.

---

# 101. Sticky and explicit writer

Si query solicita:

```text
WRITE/AUTHORITATIVE connection
```

sticky no necesita cambiar la decisión.

---

# 102. Sticky and replica-only query

Si una query exige explícitamente replica pero sticky exige authoritative consistency:

deberá existir conflicto explícito.

---

# 103. Constraint conflict

No:

```text
pick whichever is easier
```

---

# 104. StickyConstraintConflictException

Podrá indicar:

```text
Requested:
    REPLICA_ONLY

Required:
    AUTHORITATIVE

Result:
    CONFLICT
```

---

# 105. Explicit eventual read

Una aplicación podría solicitar deliberadamente:

```php
DB::read()
    ->eventual()
    ->get(...);
```

---

# 106. Override policy

Que esto pueda ignorar sticky state dependerá de policy.

Para evitar degradaciones accidentales, por default:

```text
sticky consistency wins
```

---

# 107. Explicit unsafe override

Si se permite:

```text
allowEventualConsistencyOverride()
```

deberá ser muy explícito.

---

# 108. ORM integration

Model API:

```php
$user->save();
```

no deberá manipular sticky state directamente.

---

# 109. Persistence integration

Flujo:

```text
Model API
  ↓
EntityManager
  ↓
UnitOfWork
  ↓
Persistence Engine
  ↓
Transaction
  ↓
Commit
  ↓
WriteObservation
  ↓
Sticky System
```

---

# 110. Flush distinction

```text
flush()
≠
sticky activation
```

si la transaction aún no ha sido confirmada.

---

# 111. Flush outside explicit transaction

Si Persistence Engine crea una transaction interna:

sticky activation ocurre después de:

```text
internal transaction commit confirmed
```

---

# 112. Active Record API

Podrá proporcionar ergonomía:

```php
User::query()
    ->consistentAfterWrite()
    ->find($id);
```

pero internamente producirá requirements semánticas.

---

# 113. No ORM-local sticky flag

Evitar:

```php
Model::$useWriteConnection = true;
```

como estado global.

---

# 114. IdentityMap interaction

Supongamos:

```php
$user->name = 'Alice';
$em->flush();
```

Después:

```php
$em->find(User::class, $id);
```

puede devolver la misma instancia desde IdentityMap.

No hay query.

---

# 115. Sticky not consulted on IdentityMap hit

Correcto, porque no ocurrió routing DB.

---

# 116. Explicit refresh

```php
$em->refresh($user);
```

sí deberá respetar sticky/consistency requirements.

---

# 117. Repository queries

```php
$users->findByEmail(...);
```

deberán pasar por Query/Execution/Routing y por tanto recibir sticky constraints.

---

# 118. Raw queries

Las APIs raw también deberán pasar por routing context cuando utilicen managed connections.

---

# 119. Bypass APIs

Una API que recibe una physical connection explícita puede bypassar routing.

Eso deberá ser evidente y documentado.

---

# 120. Cache interaction

Una sticky read puede ser satisfecha por cache únicamente si:

```text
cache consistency evidence
```

demuestra la requirement correspondiente.

---

# 121. Cache hit ≠ read-your-writes

No:

```text
sticky active
→ cache hit
→ assume valid
```

---

# 122. Cache invalidation integration

Una escritura puede registrar:

```text
deferred invalidation
```

en TransactionContext.

Después del commit:

```text
cache invalidation
+
sticky activation
```

son procesos distintos.

---

# 123. Cache invalidation ≠ sticky

Invalidar cache no hace que replicas estén actualizadas.

---

# 124. Sticky propagation modes

Propuesta:

```php
enum StickyPropagationMode
{
    case LOCAL_SCOPE;
    case EXPLICIT_TOKEN;
    case INTEGRATION_DEFINED;
}
```

---

# 125. LOCAL_SCOPE

Default seguro del core.

---

# 126. EXPLICIT_TOKEN

Permite propagación controlada.

---

# 127. INTEGRATION_DEFINED

HTTP, queues o aplicaciones podrán implementar mecanismos específicos.

---

# 128. HTTP integration

Una integración HTTP podría transportar un token mediante:

- server-side session;
- encrypted cookie;
- framework request context;
- internal header.

Pero Database core no deberá depender de ninguno.

---

# 129. Queue integration

Un job podría recibir:

```text
ConsistencyToken
```

si realmente necesita observar un write previo.

---

# 130. No automatic queue propagation

No todos los jobs necesitan esa consistencia.

Propagar automáticamente puede:

- aumentar writer load;
- retener requirements innecesarias;
- mezclar execution domains.

---

# 131. Remote service propagation

No forma parte del core.

Puede construirse sobre tokens/versioned consistency metadata.

---

# 132. Token expiration

Todo token portable deberá expirar.

---

# 133. Expiration bound

Evitar tokens que fuerzan writer:

```text
forever
```

---

# 134. Position requirement expiration

Incluso una required position podrá tener expiration operacional para evitar estado eterno.

---

# 135. Expiration policy

Al expirar:

```text
EXPIRE_REQUIREMENT
FAIL
FALLBACK_AUTHORITATIVE
```

según semántica.

---

# 136. Requirement lifecycle

Modelo:

```text
CREATED
   ↓
ACTIVE
   ↓
SATISFIABLE
   ↓
EXPIRED/CLEARED
```

`SATISFIABLE` significa que existe evidencia de endpoints que pueden satisfacerla, no que deja de existir.

---

# 137. Requirement strengthening

Una nueva escritura puede fortalecer el requisito.

```text
P1
↓
P2 AFTER P1
↓
require P2
```

---

# 138. Requirement weakening

No deberá ocurrir automáticamente dentro del mismo scope.

---

# 139. Monotonic consistency requirement

Dentro de un mismo domain:

```text
RequiredPosition(t+1)
>=
RequiredPosition(t)
```

cuando las posiciones sean comparables.

---

# 140. Monotonic reads

Esto ayuda a soportar:

```text
Monotonic Read Consistency
```

junto con read-your-writes.

---

# 141. Monotonic reads distinction

Read-your-writes:

```text
read must observe own write
```

Monotonic reads:

```text
later read must not observe state older than earlier read
```

---

# 142. Scope expansion

La primera versión podrá centrarse en write observations.

La arquitectura deberá permitir incorporar:

```text
ReadObservation
```

para monotonic-read guarantees futuras.

---

# 143. ReadObservation

Conceptualmente:

```php
final readonly class DatabaseReadObservation
{
    public function __construct(
        public DatabaseExecutionDomain $domain,
        public ?ReplicationCoordinate $observedPosition,
    ) {}
}
```

---

# 144. No fake read position

Si una query no puede determinar la posición correspondiente:

no deberá inventarse.

---

# 145. Sticky state merging

Cuando dos consistency contexts se combinan:

```text
Context A requires P1
Context B requires P2
```

si:

```text
P2 AFTER P1
```

resultado:

```text
P2
```

---

# 146. Incomparable merge

Si son incomparables:

```text
CompositeRequirement(P1, P2)
```

o conflicto explícito.

Nunca escoger uno arbitrariamente.

---

# 147. StickyContextRegistry

Podrá almacenar requirements por key:

```php
interface StickyContextRegistry
{
    public function get(
        StickyKey $key,
    ): ?StickyRequirement;

    public function register(
        StickyKey $key,
        StickyRequirement $requirement,
    ): void;
}
```

---

# 148. Registry scope

Debe ser:

```text
request/job/coroutine scoped
```

salvo storage portable explícito.

---

# 149. Persistent runtimes

FrankenPHP será especialmente importante.

Nunca:

```text
Request A writes
↓
static sticky = true
↓
Request ends
↓
Request B inherits sticky
```

---

# 150. Request cleanup

Al terminar scope:

```text
local sticky state
→ cleared
```

---

# 151. Portable token exception

Un token explícitamente propagado podrá reestablecer requirements en el nuevo scope.

---

# 152. RoadRunner

Worker reuse exige la misma disciplina.

---

# 153. OpenSwoole

Más crítico todavía:

```text
Coroutine A
Coroutine B
```

pueden ejecutarse simultáneamente.

---

# 154. Coroutine-local storage

Current sticky context deberá ser:

```text
coroutine/fiber scoped
```

---

# 155. No mutable singleton

Prohibido:

```php
final class StickyManager
{
    private bool $sticky = false;
}
```

si la instancia se comparte entre scopes.

---

# 156. StickyManager

El manager deberá ser coordinador stateless/read-mostly:

```php
interface StickyManager
{
    public function registerWrite(
        DatabaseWriteObservation $observation,
        StickyContext $context,
    ): void;

    public function constraintFor(
        DatabaseExecutionDomain $domain,
        StickyContext $context,
    ): ?StickyRoutingConstraint;
}
```

---

# 157. Mutable state ownership

Estado mutable:

```text
StickyContext
```

No:

```text
StickyManager
```

---

# 158. Context generation

Cada context tendrá una generation/identity para detectar stale handles.

---

# 159. Stale context

Un handle obtenido en Request A no deberá reutilizarse en Request B.

---

# 160. Lifecycle cleanup

```text
Scope start
↓
StickyContext created
↓
operations
↓
scope completion
↓
dispose
```

---

# 161. Disposed context

No podrá reactivarse.

---

# 162. Writer fallback

Si ninguna replica satisface requirement:

```text
authoritative endpoint
```

será el fallback normal para read-your-writes.

---

# 163. Writer unavailable

Si writer tampoco está disponible:

no deberá degradarse silenciosamente.

---

# 164. Result

Dependiendo de policy:

```text
FAIL
WAIT
ALLOW_EVENTUAL
```

---

# 165. Fail closed

Para strict consistency:

```text
writer unavailable
+
no qualifying replica
=
consistency unavailable
```

---

# 166. ConsistencyUnavailableException

Debe diferenciarse de:

```text
ConnectionUnavailableException
```

porque puede haber conexiones disponibles que no satisfacen la garantía.

---

# 167. Failover interaction

Durante failover:

```text
new authoritative node
```

debe demostrar compatibilidad con sticky requirements existentes cuando sea posible.

---

# 168. Old position

Una requirement:

```text
epoch 17 / position P
```

puede volverse incomparable con:

```text
epoch 18
```

---

# 169. No silent reset

Failover no deberá hacer:

```text
clear sticky requirements
→ pretend consistency satisfied
```

---

# 170. Failover reconciliation

Documento 181 deberá determinar:

```text
can new authority prove it contains P?
```

---

# 171. Lost writes possibility

Si la promoción ocurre antes de replicar una escritura confirmada:

el nuevo writer podría no contenerla dependiendo de la arquitectura de replicación.

Sticky System deberá conservar:

```text
consistency uncertainty
```

---

# 172. Load balancing

Una vez filtrados candidatos por sticky requirement:

```text
eligible replicas
```

podrán pasar al Load Balancing System.

---

# 173. Example

```text
Sticky requirement:
    >= P

Replica A:
    < P
    reject

Replica B:
    >= P
    eligible

Replica C:
    >= P
    eligible
```

Load balancer recibe:

```text
{B, C}
```

---

# 174. Sticky preference

Policy:

```text
preferReplicaWhenSatisfied = true
```

permite volver a replicas tan pronto como demuestren consistencia.

---

# 175. Writer preference

Otra policy podría mantener writer hasta que termine el scope.

---

# 176. Modes

Propuesta:

```php
enum StickyRoutingMode
{
    case REQUEST_WRITER;
    case POSITION_AWARE;
    case TIME_WINDOW;
    case HYBRID;
}
```

---

# 177. REQUEST_WRITER

Compatibilidad simple:

```text
write occurred in request
→ all later reads in request → writer
```

---

# 178. POSITION_AWARE

Más avanzado:

```text
write position P
→ replica allowed when >= P
```

---

# 179. TIME_WINDOW

```text
write
→ writer for N seconds
```

aproximado.

---

# 180. HYBRID

Puede combinar:

```text
position awareness
+
writer fallback
+
bounded token lifetime
```

---

# 181. Recommended default

Para VoltStack:

```text
POSITION_AWARE when capability exists
↓
REQUEST_WRITER / TIME_WINDOW fallback
```

sin fingir equivalencia entre ambos.

---

# 182. Configuration

Ejemplo conceptual:

```php
'database' => [
    'sticky' => [
        'enabled' => true,

        'mode' => 'hybrid',

        'scope' => 'request',

        'fallback' => 'authoritative',

        'time_window' => '5s',

        'prefer_replica_when_satisfied' => true,
    ],
],
```

---

# 183. Configuration validation

Configuraciones incompatibles deberán fallar temprano.

Ejemplo:

```text
mode = POSITION_AWARE
platform supports no replication positions
fallback = none
```

debe producir diagnóstico.

---

# 184. Capability negotiation

El sistema resolverá:

```text
RequestedStickyMode
+
PlatformCapabilities
+
ReplicaCapabilities
        │
        ▼
EffectiveStickyStrategy
```

---

# 185. Effective strategy

Debe ser observable.

---

# 186. No silent downgrade

Si se solicita:

```text
POSITION_AWARE
```

y no está disponible:

VoltStack no deberá pasar silenciosamente a:

```text
TIME_WINDOW
```

si ello cambia la garantía.

---

# 187. Explicit fallback

La configuración puede declarar:

```text
POSITION_AWARE
fallback:
    REQUEST_WRITER
```

---

# 188. Strategy resolution result

```php
final readonly class StickyStrategyResolution
{
    public function __construct(
        public StickyRoutingMode $requested,
        public StickyRoutingMode $effective,
        public StickyStrategyResolutionStatus $status,
        public array $reasons,
    ) {}
}
```

---

# 189. Resolution statuses

```php
enum StickyStrategyResolutionStatus
{
    case EXACT;
    case FALLBACK;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 190. Telemetry

Eventos:

```text
StickyRequirementActivated
StickyRequirementStrengthened
StickyRequirementSatisfied
StickyRequirementExpired
StickyRequirementCleared
StickyReplicaRejected
StickyWriterFallbackUsed
StickyConsistencyUnavailable
StickyTokenImported
StickyTokenRejected
```

---

# 191. Observational telemetry

Listeners no deberán mutar requirements.

---

# 192. Metrics

Ejemplos:

```text
db.sticky.activations
db.sticky.writer_fallbacks
db.sticky.replica_satisfied
db.sticky.replica_rejected
db.sticky.requirement_expired
db.sticky.consistency_unavailable
db.sticky.token.imported
db.sticky.token.rejected
```

---

# 193. Writer load metric

Podrá medirse:

```text
db.sticky.writer_read_ratio
```

para determinar cuánto tráfico de lectura permanece en writer por sticky semantics.

---

# 194. Position-aware benefit

Métrica:

```text
db.sticky.replica_reentry
```

puede indicar cuántas lecturas volvieron a replicas antes de terminar el sticky scope.

---

# 195. Cardinality

No usar:

```text
user_id
tenant_id
raw_position
token_id
```

como labels por default.

---

# 196. Diagnostics API

Conceptualmente:

```php
DB::sticky()->inspect();
```

---

# 197. Example diagnostics

```text
DATABASE STICKY CONTEXT

Scope:
    REQUEST

Domain:
    default

State:
    ACTIVE

Strategy:
    POSITION_AWARE

Requirement:
    REQUIRED_POSITION

Replication Domain:
    cluster-1

Epoch:
    17

Position:
    [redacted]

Activated By:
    TRANSACTION_COMMIT

Fallback:
    AUTHORITATIVE

Prefer Replica When Satisfied:
    YES
```

---

# 198. Explain routing

```php
DB::sticky()->explain($query);
```

Resultado:

```text
READ ROUTING EXPLANATION

Sticky Requirement:
    REQUIRED_POSITION

Replica A:
    position BEFORE required
    → REJECTED

Replica B:
    position UNKNOWN
    → REJECTED

Replica C:
    position AFTER required
    → ELIGIBLE

Writer:
    authoritative
    → ELIGIBLE

Final Candidate Set:
    Replica C
    Writer

Routing Policy:
    prefer eligible replica

Selected:
    Replica C
```

---

# 199. Time-window diagnostics

```text
Strategy:
    TIME_WINDOW

Activated:
    10:30:00.000

Expires:
    10:30:05.000

Remaining:
    2.1s

Meaning:
    Writer routing policy active.

Important:
    Expiration does not prove replica synchronization.
```

---

# 200. Error hierarchy

```text
DatabaseStickyConnectionException
├── StickyContextNotFoundException
├── StickyContextDisposedException
├── StickyRequirementException
├── StickyRequirementConflictException
├── StickyConstraintConflictException
├── StickyStrategyUnsupportedException
├── StickyStrategyResolutionException
├── StickyPositionUnavailableException
├── StickyPositionIncomparableException
├── StickyConsistencyUnavailableException
├── StickyTokenException
├── StickyTokenExpiredException
├── StickyTokenIntegrityException
├── StickyTokenDomainMismatchException
├── StickyScopeMismatchException
├── StickyResourceLimitException
└── StickyInvariantViolationException
```

---

# 201. Testing architecture

Suite:

```text
StickyRequirementTests
StickyContextTests
StickyManagerTests
StickyWriteObservationTests
StickyTransactionIntegrationTests
StickyPositionTests
StickyTimeWindowTests
StickyRoutingTests
StickyReplicaEligibilityTests
StickyTokenTests
StickyScopeTests
StickyTenantIsolationTests
StickyShardIsolationTests
StickyFailoverIntegrationTests
StickyPersistentRuntimeTests
StickyCoroutineTests
StickyTelemetryTests
StickyDiagnosticsTests
StickySecurityTests
```

---

# 202. Same-request test

```text
WRITE confirmed
→ sticky activated
→ later READ
```

deberá respetar requirement.

---

# 203. Rollback test

```text
WRITE
→ ROLLBACK
```

no deberá activar sticky state.

---

# 204. Unknown commit test

```text
COMMIT sent
→ connection lost
→ UNKNOWN
```

no deberá producir falsa confirmed write observation.

---

# 205. Position test

```text
required P
replica BEFORE P
```

→ rejected.

---

# 206. Position equality test

```text
replica EQUAL P
```

→ satisfies.

---

# 207. Position ahead test

```text
replica AFTER P
```

→ satisfies.

---

# 208. Unknown position test

```text
replica position UNKNOWN
```

→ no strong satisfaction.

---

# 209. Incomparable test

Different replication epoch:

```text
INCOMPARABLE
```

→ no strong satisfaction.

---

# 210. Writer fallback test

Ninguna replica satisface P:

```text
writer
```

deberá permanecer eligible bajo authoritative fallback.

---

# 211. Time-window test

Dentro de window:

```text
writer required
```

según policy.

---

# 212. Expiration test

Después del window:

la sticky time policy expira sin marcar replicas como caught-up.

---

# 213. Multiple writes test

```text
P1
P2 AFTER P1
```

deberá conservar:

```text
P2
```

---

# 214. Incomparable writes test

```text
P1 INCOMPARABLE P2
```

no deberá eliminar arbitrariamente uno.

---

# 215. Tenant isolation test

Sticky de Tenant A no afectará Tenant B.

---

# 216. Shard isolation test

Sticky de Shard A no afectará Shard B.

---

# 217. Transaction pinning test

Active transaction deberá usar pinned connection independientemente de sticky routing.

---

# 218. ORM flush test

`flush()` antes del commit no deberá activar confirmed sticky requirement.

---

# 219. IdentityMap test

IdentityMap hit no deberá ejecutar sticky routing.

---

# 220. Explicit refresh test

Refresh sí deberá respetar sticky constraints.

---

# 221. Token integrity test

Token modificado deberá rechazarse.

---

# 222. Token expiration test

Token expirado no deberá restaurar requirement como activo.

---

# 223. Token domain test

Token de database A no deberá aplicarse a database B.

---

# 224. Worker reuse test

Request B no heredará sticky state de Request A.

---

# 225. Coroutine test

Coroutine B no heredará sticky state de Coroutine A.

---

# 226. Resource test

Requirements/tokens/history deberán permanecer bounded.

---

# 227. Proposed directory structure

```text
src/Quantum/Database/Routing/Sticky/
│
├── StickyManager.php
├── StickyRoutingMode.php
├── StickyState.php
├── StickyScope.php
├── StickyKey.php
│
├── Requirement/
│   ├── StickyRequirement.php
│   ├── StickyRequirementKind.php
│   ├── RequiredPositionStickyRequirement.php
│   ├── AuthoritativeStickyRequirement.php
│   ├── TimeWindowStickyRequirement.php
│   └── CompositeStickyRequirement.php
│
├── Context/
│   ├── StickyContext.php
│   ├── MutableStickyContext.php
│   ├── DefaultStickyContext.php
│   ├── StickyContextResolver.php
│   ├── StickyContextRegistry.php
│   ├── StickyContextId.php
│   └── StickyContextGeneration.php
│
├── Observation/
│   ├── DatabaseWriteObservation.php
│   ├── WriteObservationId.php
│   └── WriteObservationConfidence.php
│
├── Evaluation/
│   ├── StickyEvaluator.php
│   ├── StickyEvaluation.php
│   ├── StickyEvaluationStatus.php
│   └── StickyEvaluationReason.php
│
├── Routing/
│   ├── StickyRoutingConstraint.php
│   ├── StickyRoutingConstraintProvider.php
│   └── StickyFallbackPolicy.php
│
├── Strategy/
│   ├── StickyRoutingPolicy.php
│   ├── StickyActivationPolicy.php
│   ├── StickyStrategyResolver.php
│   ├── StickyStrategyResolution.php
│   └── StickyStrategyResolutionStatus.php
│
├── Token/
│   ├── StickyConsistencyToken.php
│   ├── StickyTokenId.php
│   ├── StickyTokenCodec.php
│   ├── StickyTokenSigner.php
│   └── StickyTokenValidator.php
│
├── Propagation/
│   ├── StickyPropagationMode.php
│   ├── StickyContextImporter.php
│   └── StickyContextExporter.php
│
├── Diagnostics/
│   ├── StickyInspector.php
│   ├── StickyExplainer.php
│   └── StickyDiagnosticReport.php
│
├── Telemetry/
│   ├── StickyTelemetry.php
│   ├── StickyRequirementActivated.php
│   ├── StickyRequirementStrengthened.php
│   ├── StickyReplicaRejected.php
│   └── StickyWriterFallbackUsed.php
│
└── Exception/
    ├── DatabaseStickyConnectionException.php
    ├── StickyContextNotFoundException.php
    ├── StickyContextDisposedException.php
    ├── StickyRequirementConflictException.php
    ├── StickyConstraintConflictException.php
    ├── StickyStrategyUnsupportedException.php
    ├── StickyConsistencyUnavailableException.php
    ├── StickyTokenException.php
    ├── StickyTokenExpiredException.php
    ├── StickyTokenIntegrityException.php
    └── StickyInvariantViolationException.php
```

---

# 228. Architectural invariants

## DB-STK-001
Sticky routing será una semántica de consistencia, no una physical connection retenida.

## DB-STK-002
StickyRequirement será distinta de ConnectionLease.

## DB-STK-003
StickyRequirement será distinta de Transaction pinning.

## DB-STK-004
StickyRequirement será distinta de connection pooling.

## DB-STK-005
StickyRequirement será distinta de HTTP session.

## DB-STK-006
StickyRequirement será distinta de ReplicaLag.

## DB-STK-007
Sticky state se activará únicamente mediante evidencia válida o solicitud explícita.

## DB-STK-008
Statement success no equivaldrá a confirmed write.

## DB-STK-009
Transaction commit confirmado podrá generar WriteObservation.

## DB-STK-010
Rollback no generará confirmed WriteObservation.

## DB-STK-011
UNKNOWN commit outcome no generará falsa confirmed WriteObservation.

## DB-STK-012
Savepoint-rolled-back writes no alimentarán sticky state final.

## DB-STK-013
Transaction pinning tendrá precedencia sobre sticky routing.

## DB-STK-014
Sticky System no moverá active transaction.

## DB-STK-015
Sticky System no será TransactionManager.

## DB-STK-016
Sticky System no será Replica System.

## DB-STK-017
Sticky System no será Lag Awareness System.

## DB-STK-018
Sticky System no será Load Balancer.

## DB-STK-019
Sticky System no será Failover System.

## DB-STK-020
Required replication position será preferida cuando exista evidencia adecuada.

## DB-STK-021
Time-window sticky será reconocida como aproximación.

## DB-STK-022
Time-window expiration no probará replica catch-up.

## DB-STK-023
Sticky active no significará writer-only necesariamente.

## DB-STK-024
Replica podrá satisfacer sticky requirement si demuestra progreso suficiente.

## DB-STK-025
Replica BEFORE required position será rechazada.

## DB-STK-026
Replica EQUAL required position podrá ser aceptada.

## DB-STK-027
Replica AFTER required position podrá ser aceptada.

## DB-STK-028
Replica UNKNOWN position no satisfará strong requirement.

## DB-STK-029
Replica INCOMPARABLE position no satisfará strong requirement.

## DB-STK-030
Cross-epoch comparison no se asumirá válida.

## DB-STK-031
Cross-domain comparison no se asumirá válida.

## DB-STK-032
Sticky requirements estarán particionadas por execution domain.

## DB-STK-033
Sticky requirements estarán aisladas por tenant cuando corresponda.

## DB-STK-034
Sticky requirements estarán aisladas por shard.

## DB-STK-035
Write en database A no afectará database B automáticamente.

## DB-STK-036
Write en shard A no afectará shard B automáticamente.

## DB-STK-037
StickyContext será scope-local por default.

## DB-STK-038
StickyManager no almacenará current mutable sticky state global.

## DB-STK-039
No habrá static sticky flag.

## DB-STK-040
FrankenPHP request reuse no filtrará sticky state.

## DB-STK-041
RoadRunner worker reuse no filtrará sticky state.

## DB-STK-042
OpenSwoole coroutines tendrán sticky state aislado.

## DB-STK-043
Fiber/coroutine scope deberá ser respetado.

## DB-STK-044
Scope completion limpiará local sticky state.

## DB-STK-045
Disposed StickyContext no podrá reutilizarse.

## DB-STK-046
Stale StickyContext handle será detectable.

## DB-STK-047
Cross-request propagation será explícita.

## DB-STK-048
Database core no dependerá de HTTP Session.

## DB-STK-049
Portable token no serializará connection objects.

## DB-STK-050
Portable token no serializará TransactionContext.

## DB-STK-051
Portable token no serializará savepoints.

## DB-STK-052
Portable token estará versionado.

## DB-STK-053
Portable token tendrá expiration.

## DB-STK-054
Portable token tendrá integrity protection cuando cruce trust boundaries.

## DB-STK-055
Portable token no expondrá credentials.

## DB-STK-056
Portable token no expondrá topology secrets innecesarios.

## DB-STK-057
Token manipulado será rechazado.

## DB-STK-058
Token de domain incorrecto será rechazado.

## DB-STK-059
Token no podrá debilitar consistency requirement.

## DB-STK-060
Requirement strengthening será monotónico cuando positions sean comparables.

## DB-STK-061
Newer required position podrá reemplazar una anterior dominada.

## DB-STK-062
Incomparable requirements no serán descartadas arbitrariamente.

## DB-STK-063
Composite requirements serán bounded.

## DB-STK-064
Sticky requirement no contendrá entities.

## DB-STK-065
Sticky requirement no contendrá QueryResults.

## DB-STK-066
StickyContext no será service container.

## DB-STK-067
StickyContext no contendrá repositories.

## DB-STK-068
StickyContext no contendrá credentials.

## DB-STK-069
Sticky activation después de transaction ocurrirá tras commit confirmado.

## DB-STK-070
Flush no equivaldrá a sticky activation.

## DB-STK-071
ORM save no manipulará global sticky flags.

## DB-STK-072
Active Record API convergerá en el mismo Sticky System.

## DB-STK-073
Repository queries respetarán sticky constraints.

## DB-STK-074
Managed raw queries respetarán routing constraints.

## DB-STK-075
Physical-connection bypass será explícito.

## DB-STK-076
IdentityMap hit no ejecutará sticky routing.

## DB-STK-077
Explicit ORM refresh sí respetará sticky consistency.

## DB-STK-078
Cache invalidation será distinta de sticky consistency.

## DB-STK-079
Cache hit no satisfará sticky requirement sin evidencia adecuada.

## DB-STK-080
Cache freshness será distinta de replica freshness.

## DB-STK-081
Sticky System producirá routing constraints, no endpoint selection.

## DB-STK-082
ReadWriteRouter consumirá StickyRoutingConstraint.

## DB-STK-083
Sticky constraint se combinará con execution-domain constraints.

## DB-STK-084
Sticky constraint no superará transaction pin.

## DB-STK-085
Sticky constraint no superará tenant isolation.

## DB-STK-086
Sticky constraint no superará shard isolation.

## DB-STK-087
Sticky constraint conflict será explícito.

## DB-STK-088
REPLICA_ONLY incompatible con AUTHORITATIVE requirement no se resolverá silenciosamente.

## DB-STK-089
Consistency degradation requerirá opt-in explícito.

## DB-STK-090
Strict consistency fallará cerrado.

## DB-STK-091
Writer fallback será explícito.

## DB-STK-092
Writer fallback no significará replica failure.

## DB-STK-093
Writer unavailable será distinto de consistency unavailable.

## DB-STK-094
No qualifying endpoint producirá consistency failure bajo strict policy.

## DB-STK-095
Sticky mode será configurable.

## DB-STK-096
Requested sticky strategy será distinta de effective strategy.

## DB-STK-097
Capability negotiation será explícita.

## DB-STK-098
Unsupported position-aware mode no degradará silenciosamente.

## DB-STK-099
Fallback strategy deberá declararse.

## DB-STK-100
Effective strategy será diagnosticable.

## DB-STK-101
REQUEST_WRITER será estrategia válida de compatibilidad.

## DB-STK-102
POSITION_AWARE será estrategia causal preferida cuando sea soportada.

## DB-STK-103
TIME_WINDOW no se presentará como causal proof.

## DB-STK-104
HYBRID podrá combinar position evidence y fallback.

## DB-STK-105
Replica eligibility se evaluará antes de load balancing.

## DB-STK-106
Load balancer no podrá reintroducir replica sticky-ineligible.

## DB-STK-107
PreferReplicaWhenSatisfied será una policy, no invariant.

## DB-STK-108
Writer-until-scope-end podrá ser otra policy.

## DB-STK-109
Sticky requirement podrá sobrevivir aunque una replica concreta ya la satisfaga.

## DB-STK-110
Una replica satisfaciendo P no probará que todas satisfacen P.

## DB-STK-111
Sticky evaluation será por candidate.

## DB-STK-112
Sticky evaluation producirá SATISFIED, UNSATISFIED o UNKNOWN.

## DB-STK-113
UNKNOWN no será convertido en SATISFIED.

## DB-STK-114
Sticky expiration será distinta de requirement satisfaction.

## DB-STK-115
Time-window expiration no cambiará replica state.

## DB-STK-116
Time-window expiration no cambiará replica lag.

## DB-STK-117
Write observations tendrán execution domain.

## DB-STK-118
Write observations tendrán commit timestamp.

## DB-STK-119
Write observations tendrán confidence.

## DB-STK-120
Raw replication positions podrán ser redacted en diagnostics.

## DB-STK-121
Multiple writes en transaction podrán consolidarse en commit coordinate.

## DB-STK-122
Autocommit writes podrán fortalecer current requirement.

## DB-STK-123
Requirement weakening automático estará prohibido.

## DB-STK-124
Read-your-writes no implicará global strong consistency.

## DB-STK-125
Sticky routing podrá contribuir a monotonic consistency.

## DB-STK-126
Monotonic reads serán distintas de read-your-writes.

## DB-STK-127
Future ReadObservation deberá conservar evidence semantics.

## DB-STK-128
Read position no se inventará cuando no pueda observarse.

## DB-STK-129
Failover no borrará sticky uncertainty silenciosamente.

## DB-STK-130
Failover podrá invalidar position comparability.

## DB-STK-131
New authority deberá evaluarse contra existing requirement.

## DB-STK-132
Old epoch requirement no se asumirá satisfecha por new epoch.

## DB-STK-133
Possible lost write después de failover permanecerá uncertainty.

## DB-STK-134
Sticky telemetry será observational.

## DB-STK-135
Telemetry listeners no mutarán StickyContext.

## DB-STK-136
Telemetry no expondrá raw tokens.

## DB-STK-137
Telemetry no expondrá raw replication positions por default.

## DB-STK-138
Metrics evitarán user/tenant/token high-cardinality labels.

## DB-STK-139
Diagnostics explicarán activation reason.

## DB-STK-140
Diagnostics explicarán requirement.

## DB-STK-141
Diagnostics explicarán fallback.

## DB-STK-142
Diagnostics explicarán candidate rejection.

## DB-STK-143
Diagnostics distinguirán time approximation de position proof.

## DB-STK-144
Sticky context memory será bounded.

## DB-STK-145
Requirement count por scope será bounded.

## DB-STK-146
Token size será bounded.

## DB-STK-147
Composite requirement depth será bounded.

## DB-STK-148
No se retendrá full query history.

## DB-STK-149
No se retendrán application object graphs.

## DB-STK-150
No se retendrán physical connections por sticky state.

## DB-STK-151
Sticky routing no reservará pool connection indefinidamente.

## DB-STK-152
Connection reuse seguirá siendo responsabilidad del Connection System.

## DB-STK-153
Sticky System será independiente del Driver concreto.

## DB-STK-154
Platform-specific position comparison permanecerá en adapters/capabilities.

## DB-STK-155
MySQL/MariaDB/PostgreSQL semantics no se mezclarán en core.

## DB-STK-156
SQLite sin replication no fingirá position-aware sticky.

## DB-STK-157
Managed providers podrán aportar opaque replication coordinates.

## DB-STK-158
Opaque coordinates no serán comparadas genéricamente.

## DB-STK-159
Consistency token import será validado antes de activar state.

## DB-STK-160
Imported state no podrá escapar del authorized execution domain.

## DB-STK-161
Cross-tenant token será rechazado.

## DB-STK-162
Cross-shard token será rechazado.

## DB-STK-163
Cancellation limpiará cualquier state temporal asociado.

## DB-STK-164
Scope teardown verificará ausencia de leaked mutable state.

## DB-STK-165
Leak detection nunca auto-commitirá transacciones.

## DB-STK-166
Sticky cleanup no modificará database state.

## DB-STK-167
Sticky cleanup no realizará hidden queries.

## DB-STK-168
Sticky activation no realizará hidden application queries.

## DB-STK-169
Replica verification podrá usar Lag Awareness System.

## DB-STK-170
Sticky System no duplicará Replica Lag System.

## DB-STK-171
Sticky System no duplicará Replication Position Comparator.

## DB-STK-172
Sticky System no duplicará ReadWrite Router.

## DB-STK-173
Sticky state deberá ser determinista para mismas observations/policy.

## DB-STK-174
Correctness tendrá prioridad sobre replica utilization.

## DB-STK-175
VoltStack nunca degradará silenciosamente una requirement de read-after-write únicamente para mantener tráfico sobre replicas.

---

# 229. Modelo formal

Sea:

```text
W = confirmed write
P(W) = replication coordinate produced by W
S = current sticky state
R = candidate replica
```

Tras un write confirmado:

```text
S' = Strengthen(S, P(W))
```

si existe posición comparable.

Para una lectura:

```text
StickyEligible(R, S)
=
BaseEligible(R)
∧
RequirementSatisfied(R, S)
```

Para position-aware sticky:

```text
RequirementSatisfied(R, P)
=
Compare(
    Position(R),
    P
) ∈ {EQUAL, AFTER}
```

Si:

```text
Compare = BEFORE
```

entonces:

```text
UNSATISFIED
```

Si:

```text
Compare ∈ {UNKNOWN, INCOMPARABLE, CONCURRENT}
```

entonces:

```text
UNKNOWN
```

y una strict policy no deberá aceptar la replica.

---

# 230. Modelo de activación

```text
                    DATABASE WRITE
                          │
                          ▼
                 Statement Execution
                          │
                          ▼
                  Transaction Scope
                          │
                          ▼
                       COMMIT
                    /          \
              confirmed        unknown/fail
                  │                 │
                  ▼                 ▼
        Write Observation       No confirmed
                  │              write evidence
                  ▼
       Sticky Strategy Resolver
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
     Position  Writer      Time
      Aware    Sticky     Window
        │         │          │
        └─────────┼──────────┘
                  ▼
             StickyContext
```

---

# 231. Modelo de routing

```text
                         READ
                           │
                           ▼
                    Routing Context
                           │
                           ▼
                 Active Transaction?
                     /           \
                   YES            NO
                    │              │
                    ▼              ▼
              Pinned Conn.   Sticky Context?
                                /       \
                              NO         YES
                               │          │
                               ▼          ▼
                         Normal Route   Requirement
                                          │
                                          ▼
                              Candidate Evaluation
                               /       |        \
                              /        |         \
                           Writer   Replica A  Replica B
                              │        │          │
                              ▼        ▼          ▼
                           valid?   satisfies?  satisfies?
                              \        |          /
                               \       |         /
                                ▼      ▼        ▼
                               Eligible Candidates
                                        │
                                        ▼
                                   Load Balancer
                                        │
                                        ▼
                                  Connection Lease
```

---

# 232. Modelo de propagación

```text
Request A
   │
   ├── WRITE
   │
   └── COMMIT
         │
         ▼
 Sticky Requirement P
         │
         ├──────── local scope ────────┐
         │                             │
         ▼                             ▼
 later read                     Export token
                                        │
                                        ▼
                                    Request B
                                        │
                                        ▼
                                 Validate token
                                        │
                                        ▼
                                Import requirement P
                                        │
                                        ▼
                                   read routing
```

La propagación será:

```text
explicit
validated
bounded
domain-aware
```

---

# 233. Arquitectura resultante del bloque

Con los documentos:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

VoltStack obtiene:

```text
Read/Write Connections
        │
        ▼
Replication Topology
        │
        ▼
Replica State
        │
        ▼
Replica Progress / Lag
        │
        ▼
Write Observation
        │
        ▼
Sticky Consistency Requirement
        │
        ▼
Read Routing Constraints
        │
        ▼
Eligible Endpoints
```

La evolución conceptual es:

```text
Basic framework:

write happened?
→ use writer
```

hacia:

```text
VoltStack:

confirmed write
      +
execution domain
      +
replication coordinate
      +
consistency scope
      +
replica progress
      +
routing policy
      │
      ▼
Can this endpoint satisfy the required observation?
```

---

# 234. Regla maestra final

> **Sticky Connection en VoltStack no significará “seguir usando la misma conexión”, sino preservar una requirement de consistencia derivada de una escritura confirmada. El sistema podrá mantener las lecturas sobre el nodo autoritativo mientras sea necesario y permitirá regresar a replicas tan pronto como puedan demostrar que satisfacen la requirement, sin degradar silenciosamente la garantía ni compartir estado mutable entre scopes.**

En forma compacta:

```text
Confirmed Write
      │
      ▼
Consistency Evidence
      │
      ▼
Sticky Requirement
      │
      ▼
Candidate Verification
      │
      ├── Replica satisfies → replica allowed
      │
      └── Cannot satisfy    → authoritative fallback
```

Nunca:

```text
write
↓
global $sticky = true
↓
all future reads use writer
↓
hope worker state gets reset
```

---

# 235. Siguiente documento

```text
181_DATABASE_FAILOVER_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack responderá cuando:

```text
Current Authoritative Node
        │
        ▼
      FAILURE
        │
        ▼
Failover Detection
        │
        ▼
Candidate Authority Evaluation
        │
        ▼
Promotion / Provider Transition
        │
        ▼
New Topology Generation
        │
        ▼
New Authoritative Node
```

incluyendo:

```text
Failure Detection
Authority Evidence
Promotion State
Failover Epoch
Topology Transition
Connection Invalidations
Transaction Safety
UNKNOWN Transaction Outcomes
Replica Promotion
Write Fencing
Split-Brain Protection
Sticky Requirement Reconciliation
Replication Position Compatibility
Recovery
Failback
Telemetry
Persistent Runtime Safety
```

manteniendo como regla:

```text
Connection Failure
≠
Permission To Fail Over
```

y especialmente:

```text
New Writer Available
≠
New Writer Contains Every Previously Confirmed Write
```

La arquitectura de failover deberá preservar esa incertidumbre en lugar de ocultarla.