# 256_DATABASE_CONNECTION_REUSE_SYSTEM.md

# VoltStack Quantum Database
## Database Connection Reuse System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 256 — Database Connection Reuse System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `255_DATABASE_STATE_RESET_SYSTEM.md`  
**Siguiente documento:** `257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Database Connection Reuse System** de VoltStack.

Su responsabilidad es determinar cuándo una conexión física previamente utilizada puede ser entregada de forma segura a una nueva operación, request, job, Fiber, coroutine o execution scope.

La regla fundamental es:

> **Una conexión física sólo podrá reutilizarse cuando VoltStack pueda demostrar que está viva, pertenece al dominio correcto, no conserva trabajo activo, su estado de sesión satisface el baseline requerido y no existe estado relevante desconocido.**

Formalmente:

```text
Reusable(C)
=
Alive(C)
∧ DomainCompatible(C)
∧ SessionBaselineValid(C)
∧ NoActiveTransaction(C)
∧ NoActiveResources(C)
∧ ResetVerified(C)
∧ NoUnknownRelevantState(C)
```

Por tanto:

```text
ConnectionAlive
≠
ConnectionReusable
```

y:

```text
UNKNOWN
≠
Reusable
```

---

# 2. Contexto arquitectónico

El sistema se encuentra entre:

```text
Application / ORM / Query Engine
              │
              ▼
      Logical Connection
              │
              ▼
       Connection Lease
              │
              ▼
    Connection Reuse System
              │
              ▼
       Connection Pool
              │
              ▼
    Physical Connection
              │
              ▼
            Driver
              │
              ▼
            DBMS
```

Su función no es ejecutar SQL.

Su función es gobernar:

```text
acquisition
ownership
reuse
reset
validation
quarantine
discard
lifetime
```

de conexiones físicas.

---

# 3. Relación con documentos anteriores

El sistema depende especialmente de:

```text
11_DATABASE_CONNECTION_SYSTEM
12_DATABASE_CONNECTION_MANAGER
14_DATABASE_CONNECTION_POOLING_SYSTEM
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
164_DATABASE_TRANSACTION_ARCHITECTURE
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM
178_DATABASE_REPLICA_SYSTEM
181_DATABASE_FAILOVER_SYSTEM
182_DATABASE_LOAD_BALANCING_SYSTEM
235_DATABASE_RESILIENCE_ARCHITECTURE
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM
251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE
252_DATABASE_REQUEST_SCOPE_SYSTEM
253_DATABASE_DATABASE_CONTEXT_SYSTEM
254_DATABASE_STATE_ISOLATION_SYSTEM
255_DATABASE_STATE_RESET_SYSTEM
```

Este documento especializa esas reglas para persistent runtimes.

---

# 4. Problema

En PHP tradicional es frecuente:

```text
request
   ↓
open connection
   ↓
queries
   ↓
process ends
```

El proceso termina y el sistema operativo/runtime libera recursos.

En VoltStack con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el worker puede permanecer activo:

```text
Worker
 │
 ├── Request A
 ├── Request B
 ├── Request C
 ├── Job D
 └── Request E
```

Una conexión puede sobrevivir entre esas ejecuciones.

Esto mejora rendimiento, pero crea riesgos.

---

# 5. Riesgos del reuse

Una conexión puede conservar:

```text
active transaction
session variables
temporary tables
advisory locks
prepared statements
open cursors
driver buffers
selected schema
search_path
timezone
SQL mode
role
authorization state
tenant-specific state
replication/session state
```

Por ello:

```text
same socket
≠
same safe semantic state
```

---

# 6. Objetivos

El sistema deberá proporcionar:

1. modelo de conexión física;
2. separación physical/logical connection;
3. leases;
4. ownership;
5. reuse eligibility;
6. reset-before-reuse;
7. validation;
8. health state;
9. session baselines;
10. generation tracking;
11. idle lifetime;
12. maximum lifetime;
13. credential generation;
14. topology generation;
15. tenant/shard safety;
16. transaction affinity;
17. prepared statement reuse;
18. replica/writer affinity;
19. failover interaction;
20. quarantine;
21. discard;
22. concurrency safety;
23. resource governance;
24. telemetry;
25. testing.

---

# 7. No objetivos

Este sistema no será:

```text
SQL compiler
query executor
transaction manager
load balancer
tenant resolver
driver
```

aunque interactúe con ellos.

---

# 8. Physical Connection

Una `PhysicalConnection` representa el recurso real mantenido contra el DBMS.

Conceptualmente:

```php
interface PhysicalConnection
{
    public function id(): PhysicalConnectionId;

    public function state(): PhysicalConnectionState;

    public function generation(): ConnectionGeneration;
}
```

---

# 9. Logical Connection

Una logical connection representa el acceso semántico solicitado por una operación.

Ejemplo:

```text
database = main
intent = READ
tenant = ACME
consistency = READ_YOUR_WRITES
```

No necesariamente representa un socket.

---

# 10. Physical ≠ Logical

Regla:

```text
LogicalConnection
≠
PhysicalConnection
```

Una logical connection puede adquirir diferentes physical connections en momentos diferentes.

---

# 11. Connection Lease

La aplicación no deberá apropiarse permanentemente de una physical connection.

Recibirá:

```text
ConnectionLease
```

---

# 12. Lease

Un lease representa:

> Derecho temporal y exclusivo de utilizar una conexión física bajo un contexto determinado.

Conceptualmente:

```php
final class ConnectionLease
{
    public function connection(): PhysicalConnection;

    public function release(): void;
}
```

---

# 13. Lease ≠ Connection

```text
Lease
≠
PhysicalConnection
```

El lease contiene ownership temporal.

---

# 14. Ownership

Mientras exista un lease exclusivo:

```text
PhysicalConnection
    │
    ▼
Owner = Scope/Operation X
```

ninguna operación incompatible podrá usarla.

---

# 15. Exclusive ownership

Por default:

```text
one physical connection
→ one active exclusive lease
```

salvo capabilities futuras que demuestren multiplexación segura.

---

# 16. Connection state machine

Se propone:

```text
CREATED
   ↓
CONNECTING
   ↓
READY
   ↓
LEASED
   ↓
IN_USE
   ↓
RELEASING
   ↓
RESETTING
   ↓
VALIDATING
   ↓
IDLE
```

Alternativas:

```text
RESETTING
   ├── QUARANTINED
   ├── BROKEN
   └── CLOSED
```

---

# 17. Estados

```php
enum PhysicalConnectionState
{
    case CREATED;
    case CONNECTING;
    case READY;
    case LEASED;
    case IN_USE;
    case RELEASING;
    case RESETTING;
    case VALIDATING;
    case IDLE;
    case QUARANTINED;
    case BROKEN;
    case CLOSING;
    case CLOSED;
}
```

---

# 18. Reuse lifecycle

```text
Acquire
   ↓
Lease
   ↓
Use
   ↓
Release
   ↓
Inspect
   ↓
Reset
   ↓
Verify
   ↓
Reusable?
   │
   ├── YES → Pool
   │
   └── NO
        ├── Quarantine
        └── Discard
```

---

# 19. Connection reuse decision

La decisión deberá ser explícita.

```php
enum ConnectionReuseDecision
{
    case REUSE;
    case RESET_THEN_REUSE;
    case REVALIDATE;
    case QUARANTINE;
    case DISCARD;
}
```

---

# 20. Reuse Eligibility

Se propone:

```text
ConnectionReuseEligibility
```

que evalúe si una conexión puede volver al pool.

---

# 21. Eligibility dimensions

Como mínimo:

```text
physical health
transaction state
resource state
session state
credential generation
topology generation
database domain
tenant compatibility
shard compatibility
endpoint role
failover generation
connection age
idle duration
reset evidence
unknown-state evidence
```

---

# 22. Eligibility result

```php
final readonly class ConnectionReuseEligibilityResult
{
    public function __construct(
        public bool $reusable,
        public ConnectionReuseDecision $decision,
        public array $reasons,
    ) {}
}
```

---

# 23. Alive ≠ Reusable

Una conexión puede responder:

```text
SELECT 1
```

y aun así conservar:

```text
open transaction
SET ROLE admin
wrong search_path
temporary tables
```

Por tanto:

```text
ping success
≠
reuse eligibility
```

---

# 24. Health ≠ Cleanliness

Distinguir:

```text
ConnectionHealth
```

de:

```text
ConnectionCleanliness
```

---

# 25. Health

Estados posibles:

```php
enum ConnectionHealth
{
    case HEALTHY;
    case DEGRADED;
    case UNHEALTHY;
    case UNKNOWN;
}
```

---

# 26. Cleanliness

```php
enum ConnectionCleanliness
{
    case CLEAN;
    case DIRTY_KNOWN;
    case DIRTY_UNKNOWN;
}
```

Una conexión puede ser:

```text
HEALTHY + DIRTY_KNOWN
```

---

# 27. Reuse rule

Normalmente:

```text
HEALTHY + CLEAN
→ reusable
```

```text
HEALTHY + DIRTY_KNOWN
→ reset required
```

```text
HEALTHY + DIRTY_UNKNOWN
→ discard/quarantine
```

```text
UNHEALTHY
→ discard
```

---

# 28. Baseline

Cada pool deberá conocer el baseline requerido para sus conexiones.

---

# 29. ConnectionBaseline

Conceptualmente:

```php
final readonly class ConnectionBaseline
{
    public function __construct(
        public ConnectionDomain $domain,
        public SessionBaseline $session,
        public CredentialGeneration $credentialGeneration,
        public TopologyGeneration $topologyGeneration,
    ) {}
}
```

---

# 30. Session baseline

Puede contener:

```text
database/catalog
schema
search_path
role
timezone
SQL mode
encoding
collation settings
application name
session variables
isolation defaults
read-only defaults
```

según plataforma.

---

# 31. Baseline fingerprint

VoltStack podrá producir:

```text
ConnectionBaselineFingerprint
```

para comparar configuraciones.

---

# 32. Fingerprint ≠ security secret

No deberá incluir credenciales en texto.

---

# 33. Connection Domain

Una conexión pertenece a un dominio lógico.

Ejemplo:

```text
cluster
database
endpoint group
credential identity
security domain
tenant strategy
shard
```

---

# 34. Domain compatibility

No toda conexión físicamente accesible es compatible con cualquier request.

---

# 35. Example

Una conexión:

```text
database = billing
shard = EU-3
role = replica
```

no deberá entregarse a:

```text
database = billing
shard = US-1
intent = WRITE
```

---

# 36. ConnectionDomain

```php
final readonly class ConnectionDomain
{
    public function __construct(
        public LogicalDatabaseId $database,
        public ?ShardId $shard,
        public EndpointRole $endpointRole,
        public SecurityDomainId $securityDomain,
    ) {}
}
```

---

# 37. Tenant isolation

La relación entre tenant y connection dependerá de la estrategia multitenant.

---

# 38. Shared database tenancy

En:

```text
shared database
shared schema
```

una conexión puede reutilizarse entre tenants si:

```text
tenant identity is NOT session-bound
+
all tenant-scoped session state was reset
```

---

# 39. Database-per-tenant

Si cada tenant usa una DB diferente:

```text
connection domain
```

será tenant-specific.

---

# 40. Schema-per-tenant

Si el tenant determina:

```text
search_path/schema
```

la conexión deberá restaurarse antes de pasar a otro tenant.

---

# 41. Tenant contamination

Nunca:

```text
Request A
tenant = ACME
SET search_path = acme

release

Request B
tenant = BETA

inherits acme search_path
```

---

# 42. Tenant reuse policy

Se podrá definir:

```php
enum TenantConnectionReusePolicy
{
    case SHARED_AFTER_VERIFIED_RESET;
    case SAME_TENANT_ONLY;
    case DEDICATED;
}
```

---

# 43. Shard safety

Una conexión física a shard A:

```text
cannot become
```

una conexión a shard B mediante reset.

Es otro endpoint/domain.

---

# 44. Shard binding

```text
PhysicalConnection
→ ShardEndpoint
```

es parte de su identidad física.

---

# 45. Read/write role

Una conexión a writer y una conexión a replica serán tratadas separadamente.

---

# 46. Endpoint Role

```php
enum EndpointRole
{
    case WRITER;
    case REPLICA;
    case UNKNOWN;
}
```

---

# 47. Role transition

Si una topología cambia:

```text
replica → writer
```

las conexiones existentes deberán reevaluarse.

---

# 48. Topology generation

Se utilizará:

```text
TopologyGeneration
```

para detectar conexiones creadas bajo información obsoleta.

---

# 49. Generation

Ejemplo:

```text
connection topology generation = 41
current topology generation = 43
```

No significa automáticamente discard.

Significa:

```text
revalidation required
```

según policy.

---

# 50. Authority epoch

En failover podrá existir:

```text
AuthorityEpoch
```

que identifica la generación de autoridad del writer.

---

# 51. Old writer connection

Después de failover:

```text
old writer connection
```

no podrá seguir considerándose writer sólo porque el socket continúe vivo.

---

# 52. Writer verification

Cuando la autoridad cambie:

```text
old writer leases
→ invalidate/revalidate
```

según garantías disponibles.

---

# 53. Failover rule

```text
SocketAlive
≠
EndpointStillAuthoritative
```

---

# 54. Credential generation

Las credenciales pueden rotarse.

VoltStack deberá modelar:

```text
CredentialGeneration
```

---

# 55. Credential rotation

Supongamos:

```text
Generation 10
→ password A

Generation 11
→ password B
```

Conexiones abiertas bajo generación 10 pueden seguir funcionando.

Pero la política puede decidir:

```text
allow until retirement
```

o:

```text
drain and reconnect
```

---

# 56. Credentials ≠ connection state

Las credenciales no deberán almacenarse dentro de diagnósticos del reuse system.

---

# 57. Credential retirement

Un pool podrá marcar:

```text
generation 10 = RETIRING
```

y evitar nuevos leases sobre conexiones antiguas.

---

# 58. Reauthentication

Si el driver/DBMS soporta reauthentication segura, podrá modelarse como capability.

No se asumirá universalmente.

---

# 59. Connection generation

Cada conexión tendrá un identificador/generación interna.

```php
final readonly class ConnectionGeneration
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 60. Why generation

Evita que referencias antiguas confundan una conexión recreada con la anterior.

---

# 61. Lease generation binding

Un lease estará ligado a:

```text
PhysicalConnectionId
+
ConnectionGeneration
```

---

# 62. Stale lease

Un lease viejo contra una generación reemplazada deberá fallar.

---

# 63. ABA problem

Esto evita conceptualmente:

```text
Connection #7
closed

Connection #7
recreated

old lease thinks #7 is same resource
```

---

# 64. Lease token

Podrá utilizarse:

```text
LeaseToken
```

único por adquisición.

---

# 65. Release validation

Al liberar:

```text
lease token
```

deberá coincidir con el owner actual.

---

# 66. Double release

```text
release()
release()
```

no deberá devolver dos veces la conexión al pool.

---

# 67. Use-after-release

Después de:

```php
$lease->release();
```

usar la conexión mediante ese lease deberá fallar.

---

# 68. Lease state

```php
enum ConnectionLeaseState
{
    case ACTIVE;
    case RELEASING;
    case RELEASED;
    case INVALIDATED;
}
```

---

# 69. Lease acquisition

Pipeline:

```text
Connection Request
      ↓
Domain Resolution
      ↓
Routing
      ↓
Pool Selection
      ↓
Candidate Selection
      ↓
Eligibility Check
      ↓
Validation if needed
      ↓
Lease Creation
      ↓
Ownership Binding
```

---

# 70. Candidate selection

El pool no deberá entregar cualquier idle connection.

Debe cumplir:

```text
domain
role
health
generation
baseline
resource policy
```

---

# 71. Validation timing

No es necesario hacer ping en cada checkout si existe evidencia suficiente.

---

# 72. Validation policies

```php
enum ConnectionValidationPolicy
{
    case ALWAYS;
    case ON_STALE;
    case ON_SUSPECT;
    case PERIODIC;
    case DRIVER_AWARE;
    case CUSTOM;
}
```

---

# 73. Stale validation

Ejemplo:

```text
idle > validation interval
→ validate before lease
```

---

# 74. Validation query

Si se requiere una query, deberá pasar por una ruta segura definida por Driver/Platform.

---

# 75. Ping

Si el driver dispone de ping nativo:

```text
DriverCapability
```

podrá utilizarse.

---

# 76. Validation failure

```text
candidate rejected
→ discard
→ try another candidate
```

dentro de budgets.

---

# 77. Infinite candidate retry

Prohibido.

---

# 78. Acquisition budget

```php
final readonly class ConnectionAcquisitionBudget
{
    public function __construct(
        public Duration $timeout,
        public int $maxAttempts,
    ) {}
}
```

---

# 79. Pool exhaustion

Si no existe conexión disponible:

```text
wait
fail fast
create new
```

según pool/resource policy.

---

# 80. Resource Governance

El número de conexiones deberá estar limitado.

```text
min connections
max connections
max pending acquisitions
max connections per domain
```

---

# 81. Worker × pool amplification

En persistent runtimes:

```text
Workers × ConnectionsPerWorker
```

puede producir demasiadas conexiones.

---

# 82. Example

```text
100 workers
×
20 connections

=
2000 DB connections
```

aunque cada worker use pocas simultáneamente.

---

# 83. Pool sizing

El pool sizing deberá considerar:

```text
worker count
concurrency per worker
DB connection limit
endpoint limits
tenant/shard cardinality
burst profile
```

---

# 84. Local pool ≠ global pool

Un pool dentro de cada worker no conoce automáticamente el total del cluster.

---

# 85. Global governance

Puede requerirse integración futura con:

```text
deployment configuration
resource governance
external proxy/pooler
```

---

# 86. Connection reuse vs external poolers

VoltStack deberá poder coexistir con:

```text
PgBouncer
ProxySQL
cloud DB proxies
managed connection proxies
```

sin asumir semánticas inexistentes.

---

# 87. Logical session caveat

Con transaction-level external pooling, algunas session semantics pueden cambiar.

Por tanto:

```text
ExternalPoolerMode
```

deberá formar parte de capabilities/configuration cuando sea relevante.

---

# 88. Release pipeline

```text
Lease.release()
      ↓
Stop accepting operations
      ↓
Wait/cancel owned operations
      ↓
Inspect active resources
      ↓
Resolve transaction
      ↓
Reset session
      ↓
Verify
      ↓
Update health
      ↓
Return to pool / quarantine / discard
```

---

# 89. Release ≠ immediate pool return

Crítico:

```text
release requested
≠
connection reusable
```

---

# 90. Transaction affinity

Una transaction activa deberá mantener afinidad con una physical connection.

---

# 91. Transaction pinning

```text
TransactionContext
      ↓
ConnectionLease
      ↓
PhysicalConnection
```

durante la transaction.

---

# 92. No mid-transaction switching

Nunca:

```text
BEGIN on C1
UPDATE on C2
COMMIT on C3
```

para una transaction local.

---

# 93. Nested transactions

Savepoints permanecen en la misma physical connection.

---

# 94. Transaction completion

Después de commit/rollback:

```text
transaction affinity ends
```

pero la conexión aún deberá pasar por release/reset.

---

# 95. UNKNOWN transaction outcome

```text
commit sent
+
connection lost
```

implica:

```text
connection not reusable
```

aunque posteriormente parezca reconectable.

---

# 96. Auto-reconnect

No deberá ocurrir silenciosamente dentro del mismo `PhysicalConnection` identity.

---

# 97. Reconnect creates new generation

Si existe reconexión:

```text
old physical connection
→ CLOSED

new physical connection
→ new generation
```

---

# 98. Connection loss during transaction

No se sustituirá automáticamente por otra conexión y se continuará.

---

# 99. Connection loss outside transaction

Una operación retryable podrá reintentarse mediante Retry System con una nueva conexión.

Eso no es connection reuse.

---

# 100. Prepared Statement Reuse

VoltStack podrá reutilizar prepared statements cuando sea seguro.

Pero:

```text
ConnectionReuse
≠
PreparedStatementReuse
```

---

# 101. Statement locality

Un prepared statement puede ser:

```text
connection-local
```

---

# 102. Statement cache

Por ello:

```text
PhysicalConnection
    └── PreparedStatementCache
```

puede existir.

---

# 103. Statement cache key

Debe incluir, cuando corresponda:

```text
compiled query fingerprint
parameter type signature
platform generation
schema/metadata generation
session-sensitive compilation state
```

---

# 104. Prepared statement invalidation

Puede ocurrir por:

```text
schema change
connection reset
server invalidation
platform change
session state change
driver failure
```

---

# 105. Reset interaction

Si el reset del driver invalida prepared statements:

```text
statement cache
→ clear
```

---

# 106. Unknown statement validity

```text
UNKNOWN
→ do not reuse statement
```

---

# 107. Statement cache failure

No deberá convertir una conexión limpia en inutilizable salvo error del driver/session.

Puede simplemente recompilar/reprepare.

---

# 108. Server-side state

VoltStack distinguirá:

```text
client prepared representation
```

de:

```text
server-side prepared handle
```

---

# 109. Idle connections

Una conexión idle no deberá mantenerse indefinidamente por default.

---

# 110. Idle timeout

```text
maxIdleTime
```

define cuánto puede permanecer sin uso.

---

# 111. Maximum lifetime

También:

```text
maxLifetime
```

desde creación.

---

# 112. Idle ≠ lifetime

```text
idle age
≠
connection age
```

---

# 113. Maximum use count

Opcionalmente:

```text
maxLeaseCount
```

puede retirar conexiones después de cierto uso.

---

# 114. Jitter

Para evitar que todas las conexiones expiren simultáneamente:

```text
effective lifetime
=
base lifetime
± bounded jitter
```

---

# 115. Connection retirement

Estados conceptuales:

```text
ACTIVE
DRAINING
RETIRED
```

---

# 116. Draining

Una conexión `DRAINING`:

```text
current lease may finish
new leases prohibited
```

---

# 117. Retirement triggers

Ejemplos:

```text
max lifetime
credential rotation
topology change
endpoint draining
configuration generation change
suspect health
worker shutdown
```

---

# 118. Idle retired connection

Puede cerrarse inmediatamente.

---

# 119. Leased retired connection

Se marca:

```text
close-on-release
```

---

# 120. Connection health observation

Health puede derivarse de:

```text
connect failures
query transport failures
ping failures
server disconnect
protocol errors
timeout patterns
reset failures
```

---

# 121. Query semantic failure ≠ connection failure

Ejemplo:

```text
unique constraint violation
```

no significa que la conexión esté unhealthy.

---

# 122. Syntax error ≠ unhealthy

Tampoco.

---

# 123. Driver error classification

El Driver Error System deberá clasificar:

```text
transport
protocol
authentication
authorization
query semantic
constraint
timeout
deadlock
server shutdown
```

---

# 124. Connection health degradation

Sólo errores relevantes afectarán health.

---

# 125. Timeout ambiguity

Un timeout puede dejar:

```text
query still running
```

según driver/DBMS.

Por tanto no siempre basta con:

```text
timeout → return connection
```

---

# 126. Cancellation verification

Si no puede demostrarse que la operación terminó:

```text
connection → quarantine/discard
```

---

# 127. Quarantine

Quarantine separa una conexión sospechosa del pool reusable.

---

# 128. Quarantine ≠ idle

Una conexión quarantined nunca se entrega a nuevas operaciones.

---

# 129. Quarantine reasons

```text
unknown transaction state
unknown cursor state
failed reset
uncertain cancellation
topology uncertainty
protocol anomaly
session contamination
```

---

# 130. Quarantine policy

En V1, muchas condiciones podrán resolver simplemente:

```text
quarantine
→ close/discard
```

en vez de implementar recuperación compleja.

---

# 131. Why quarantine exists

Permite arquitectura extensible para:

```text
diagnostics
delayed validation
specialized recovery
```

sin mezclar conexiones sospechosas con el pool.

---

# 132. Discard

`DISCARD` significa:

```text
close physical resource
remove from pool
invalidate generation
release accounting permit
```

---

# 133. Close failure

Si cerrar socket falla:

```text
resource considered unusable
```

de todos modos.

No vuelve al pool.

---

# 134. Discard ≠ reset

Discard termina el recurso.

---

# 135. Connection creation

Pipeline:

```text
Resolve configuration
      ↓
Resolve credentials
      ↓
Resolve endpoint
      ↓
Acquire resource permit
      ↓
Driver connect
      ↓
Authenticate
      ↓
Initialize baseline
      ↓
Validate capabilities
      ↓
Mark READY
```

---

# 136. Baseline initialization

Después de conectar podrán establecerse:

```text
timezone
encoding
role
search path
SQL mode
application name
session options
```

---

# 137. Initialization failure

```text
connection → discard
```

No se entrega parcialmente configurada.

---

# 138. Initialization idempotency

Cuando sea posible, baseline initialization deberá ser idempotente.

---

# 139. Configuration generation

Cada conexión puede estar ligada a:

```text
ConnectionConfigurationGeneration
```

---

# 140. Configuration changes

Si cambia:

```text
timezone
session mode
security policy
endpoint config
```

las conexiones antiguas deberán:

```text
rebaseline
or
retire
```

según compatibilidad.

---

# 141. Dynamic config

No deberá mutarse una conexión leased bajo sus pies.

---

# 142. Config update semantics

```text
new generation
→ applies to new acquisitions/reuses
```

con drain controlado.

---

# 143. Read replica reuse

Una replica connection podrá reutilizarse para reads sólo si sigue:

```text
healthy
eligible
fresh enough for requested consistency
```

---

# 144. Reuse ≠ replica eligibility

Una conexión puede estar limpia pero su replica estar demasiado atrasada.

---

# 145. Replica lag

Por tanto:

```text
ConnectionReusable
∧
EndpointEligibleForRead
```

son decisiones diferentes.

---

# 146. Sticky connection

Sticky semantics no significan necesariamente reutilizar el mismo socket.

---

# 147. Sticky writer

Normalmente significa:

```text
route reads to writer
```

no:

```text
pin exact physical connection
```

salvo transaction/session requirement.

---

# 148. Exact connection pinning

Sólo deberá usarse cuando semánticamente necesario.

---

# 149. Session affinity

Algunas operaciones pueden requerir exact connection affinity.

Ejemplos:

```text
temporary tables
session-level locks
session-local state
```

---

# 150. Session affinity risks

Estas características reducen pool flexibility.

---

# 151. Explicit affinity

Se modelará como:

```php
enum ConnectionAffinity
{
    case NONE;
    case TRANSACTION;
    case SESSION;
    case OPERATION;
}
```

---

# 152. Hidden affinity

No deberá surgir accidentalmente.

---

# 153. Temporary table example

Si una operación crea una temp table y espera usarla después:

```text
must declare session affinity
```

o mantenerse dentro del mismo lease.

---

# 154. Release destroys affinity

Después de release no se garantiza recuperar la misma conexión.

---

# 155. Session features and poolers

Cuando external poolers impidan session affinity, capabilities deberán indicarlo.

---

# 156. Concurrency

Una physical connection no se considerará concurrent-safe por default.

---

# 157. No simultaneous operations

Default:

```text
one connection
→ one active execution stream
```

---

# 158. Driver concurrency capability

Si un driver soporta características especiales, deberán exponerse explícitamente.

---

# 159. Multiplexing

VoltStack V1 no deberá depender de multiplexing de múltiples queries independientes sobre una misma conexión.

---

# 160. Fibers

Dos Fibers concurrentes no compartirán automáticamente un mismo lease.

---

# 161. Coroutines

Misma regla.

---

# 162. Lease transfer

No se permitirá transferir ownership entre execution scopes arbitrariamente.

---

# 163. Shared connection object

El pool puede mantener el objeto.

La aplicación sólo obtiene el lease.

---

# 164. Persistent worker

El pool puede ser worker-scoped.

---

# 165. Worker-local pool

```text
Worker 1
  └── Pool A

Worker 2
  └── Pool B
```

son independientes.

---

# 166. Worker shutdown

Al apagar worker:

```text
stop acquisitions
drain leases
close idle connections
wait bounded time
invalidate remaining leases
close resources
```

---

# 167. Graceful shutdown

No deberá cerrar inmediatamente una conexión con query activa si existe tiempo para finalizar correctamente.

---

# 168. Forced shutdown

Al vencer deadline:

```text
cancel/close
```

según runtime.

---

# 169. Worker recycle

Las conexiones no deberán sobrevivir al proceso que las posee.

---

# 170. FrankenPHP

VoltStack podrá reutilizar conexiones entre requests dentro del mismo worker si la configuración/runtime lo permite.

---

# 171. FrankenPHP rule

Cada request deberá:

```text
acquire
use
release
reset
verify
```

sin depender del fin del proceso.

---

# 172. RoadRunner

Mismo principio.

---

# 173. OpenSwoole

La pool architecture deberá ser coroutine-aware.

---

# 174. OpenSwoole concurrency

Un pool worker-level puede atender varias coroutines, pero cada lease deberá tener ownership inequívoco.

---

# 175. Traditional PHP

El mismo sistema deberá funcionar aunque el reuse efectivo sea mínimo.

---

# 176. Runtime-neutral core

La lógica:

```text
eligibility
lease
reset
validation
generation
health
```

permanecerá independiente del runtime.

---

# 177. Runtime adapters

Sólo lifecycle integration será runtime-specific.

---

# 178. Connection reuse policy

```php
final readonly class ConnectionReusePolicy
{
    public function __construct(
        public bool $enabled,
        public Duration $maxIdleTime,
        public Duration $maxLifetime,
        public int $maxLeaseCount,
        public ConnectionValidationPolicy $validation,
    ) {}
}
```

---

# 179. Reuse disabled

Si:

```text
reuse.enabled = false
```

cada release podrá:

```text
close connection
```

---

# 180. Safe fallback

Esto proporciona un modo conservador para debugging o drivers no compatibles.

---

# 181. Per-platform policy

Las políticas podrán variar por:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 182. SQLite special case

SQLite puede no representar una conexión de red.

Sin embargo:

```text
handle reuse
transaction state
PRAGMA state
temporary state
```

siguen importando.

---

# 183. SQLite in-memory

Con:

```text
:memory:
```

la conexión puede representar también lifetime del database.

Por tanto:

```text
close
```

puede destruir los datos.

---

# 184. SQLite in-memory pool

Deberá ser una configuración explícita.

---

# 185. MySQL/MariaDB

Podrán utilizar capacidades específicas de session reset cuando estén realmente disponibles.

---

# 186. PostgreSQL

Podrá utilizar mecanismos de reset/session cleanup adecuados a la plataforma.

---

# 187. Platform abstraction

Nunca:

```php
if ($driver === 'pgsql') {
    // scattered reset logic
}
```

por todo el framework.

Preferir:

```text
ConnectionPlatform
    ↓
ConnectionResetCapabilities
```

---

# 188. Capability model

```php
interface ConnectionReuseCapabilities
{
    public function supportsSafeSessionReset(): bool;

    public function supportsNativePing(): bool;

    public function supportsSessionReauthentication(): bool;

    public function supportsPreparedStatementReuse(): bool;

    public function supportsConcurrentStatements(): bool;
}
```

---

# 189. Reuse strategy

```php
enum ConnectionReuseStrategy
{
    case DISABLED;
    case RESET_ON_RELEASE;
    case RESET_ON_ACQUIRE;
    case RESET_ON_RELEASE_AND_VALIDATE_ON_ACQUIRE;
    case CUSTOM;
}
```

---

# 190. Reset on release

Ventaja:

```text
idle pool contains clean connections
```

---

# 191. Reset on acquire

Ventaja:

```text
avoid resetting connections never reused
```

pero idle pool puede contener dirty state.

---

# 192. Recommended default

VoltStack V1 debería preferir:

```text
reset on release
+
conditional validation on acquire
```

---

# 193. Why

Así:

```text
pool
≈
known-clean idle resources
```

---

# 194. Release failure

Si reset falla:

```text
never insert candidate into clean idle pool
```

---

# 195. Pool partitions

Podrá organizarse por:

```text
ConnectionPoolKey
```

---

# 196. Pool key

Conceptualmente:

```text
logical database
endpoint
role
shard
security domain
credential generation policy
platform
```

---

# 197. Tenant in pool key

Dependerá de tenant isolation strategy.

No siempre deberá incluir tenant.

---

# 198. Cardinality problem

Pool por tenant puede ser inviable con miles de tenants.

---

# 199. Pool explosion

```text
10,000 tenants
×
5 connections
```

sería inaceptable.

Por ello el modelo deberá permitir:

```text
shared pool + verified tenant reset
```

cuando la arquitectura de tenancy lo permita.

---

# 200. Shard explosion

Mismo problema con muchos shards.

---

# 201. Lazy pool creation

Pools/connections podrán crearse sólo cuando exista demanda.

---

# 202. Idle eviction

Dominios poco utilizados deberán liberar conexiones.

---

# 203. Pool pruning

Un mantenimiento periódico podrá retirar:

```text
expired
idle-too-long
retiring
unhealthy
obsolete-generation
```

---

# 204. Maintenance ≠ request cleanup

Pool maintenance es worker-level.

Request reset es execution-level.

---

# 205. Background maintenance

No deberá requerirse para correctness.

Si no corre:

```text
checkout/release path
```

aún deberá preservar seguridad.

---

# 206. Leak detection

El sistema deberá detectar leases no liberados.

---

# 207. Lease leak

Ejemplo:

```text
Request ends
but lease still ACTIVE
```

---

# 208. Scope finalization

El Request Scope System deberá enumerar owned leases.

---

# 209. Leaked lease policy

Al finalizar scope:

```text
force release/reset
```

si es seguro.

Si no:

```text
discard connection
```

---

# 210. Leak warning

Development:

```text
LeakedConnectionLeaseException
```

o warning estricto.

---

# 211. Production

Containment:

```text
recover resource if provably safe
otherwise discard
```

---

# 212. Stack trace

Development tooling podrá registrar acquisition stack para detectar leaks.

---

# 213. Acquisition stack overhead

Deshabilitado o sampleado en producción.

---

# 214. Resource permit lifecycle

```text
permit acquire
   ↓
connection create
   ↓
connection lifetime
   ↓
connection close
   ↓
permit release
```

No por lease.

---

# 215. Lease permit

Opcionalmente podrá existir otro budget para active leases.

---

# 216. Backpressure

Cuando pool está lleno:

```text
pending acquisitions
```

deberán tener límite.

---

# 217. Queue overflow

Resultado:

```text
ConnectionAcquisitionRejectedException
```

en lugar de crecimiento ilimitado.

---

# 218. Fairness

Pool podrá implementar:

```text
FIFO
priority-aware
deadline-aware
```

pero correctness no dependerá de fairness exacta.

---

# 219. Cancellation while waiting

Una adquisición pendiente deberá poder cancelarse.

---

# 220. Timeout while waiting

Al expirar:

```text
remove waiter
```

sin crear lease fantasma.

---

# 221. Race: release vs timeout

Deberá resolverse atómicamente.

No:

```text
waiter times out
+
connection assigned anyway
```

sin owner.

---

# 222. Atomic ownership transition

La transición:

```text
IDLE
→ LEASED
```

deberá ser atómica respecto al pool.

---

# 223. Atomic release transition

Igualmente:

```text
LEASED
→ RELEASING
```

---

# 224. Connection cancellation

Si query cancel requiere usar la misma conexión/protocol, el lease deberá mantenerse hasta resolver cancellation.

---

# 225. Release during cancellation

No deberá regresar al pool prematuramente.

---

# 226. Async drivers

La arquitectura deberá permitir futuros drivers async.

---

# 227. Promise/future completion

Un recurso async pendiente cuenta como:

```text
active resource
```

hasta finalizar/cancelarse.

---

# 228. Reuse and transactions

El pool no interpretará semántica ORM.

Sólo deberá conocer suficiente connection/transaction state para impedir reuse inseguro.

---

# 229. ORM independence

```text
Connection Pool
≠
EntityManager Pool
```

---

# 230. EntityManager pooling

No recomendado para V1.

---

# 231. ConnectionManager responsibility

`ConnectionManager` resuelve acceso lógico.

---

# 232. Pool responsibility

`ConnectionPool` administra recursos físicos.

---

# 233. Reuse Manager responsibility

`ConnectionReuseManager` decide si un recurso físico puede volver a estar disponible.

---

# 234. Separation

```text
ConnectionManager
→ Which logical connection?

Routing
→ Which endpoint?

Pool
→ Which physical resource?

ReuseManager
→ Can this resource be reused?

ResetSystem
→ Can it be restored to baseline?

Driver
→ How to communicate?
```

---

# 235. ConnectionReuseManager

```php
interface ConnectionReuseManager
{
    public function evaluate(
        PhysicalConnection $connection,
        ConnectionReuseContext $context,
    ): ConnectionReuseEligibilityResult;
}
```

---

# 236. Reuse Context

```php
final readonly class ConnectionReuseContext
{
    public function __construct(
        public ConnectionBaseline $requiredBaseline,
        public ConnectionReusePolicy $policy,
        public TopologyGeneration $topologyGeneration,
        public CredentialGeneration $credentialGeneration,
    ) {}
}
```

---

# 237. Connection metadata

Cada conexión podrá registrar:

```text
createdAt
lastLeasedAt
lastReleasedAt
lastValidatedAt
leaseCount
resetCount
failureCount
endpoint
generation
credentialGeneration
topologyGeneration
baselineFingerprint
health
cleanliness
```

---

# 238. Metadata boundedness

No deberá acumular history infinita.

---

# 239. Connection history

Telemetry se encarga del histórico.

La conexión mantiene sólo estado necesario.

---

# 240. Sensitive metadata

No almacenar:

```text
password
raw DSN
access token
private key
```

en metadata observable.

---

# 241. Telemetry events

```text
database.connection.created
database.connection.leased
database.connection.released
database.connection.reused
database.connection.reset
database.connection.validated
database.connection.retired
database.connection.quarantined
database.connection.discarded
database.connection.closed
database.connection.acquisition_timeout
database.connection.pool_exhausted
```

---

# 242. Metrics

```text
database_connections_open
database_connections_idle
database_connections_active
database_connections_quarantined
database_connection_reuse_total
database_connection_create_total
database_connection_discard_total
database_connection_reset_total
database_connection_validation_total
database_connection_acquisition_duration
database_connection_lease_duration
database_connection_age
```

---

# 243. Reuse ratio

```text
ReuseRatio
=
ReusedAcquisitions
/
TotalSuccessfulAcquisitions
```

---

# 244. Reuse ratio interpretation

Mayor no siempre es mejor.

Un ratio alto con contamination bugs sería peor.

---

# 245. Reset cost metric

Medir:

```text
database_connection_reset_duration
```

---

# 246. Connect cost metric

Medir:

```text
database_connection_connect_duration
```

permite comparar:

```text
reset cost
vs
reconnect cost
```

---

# 247. Pool wait metric

```text
database_connection_pool_wait_duration
```

es importante para saturation.

---

# 248. Cardinality

No usar:

```text
connection_id
tenant_id
request_id
```

como labels métricos no acotados.

---

# 249. Tracing

Un acquisition podrá producir span:

```text
database.connection.acquire
```

con atributos bounded.

---

# 250. Debug diagnostics

Ejemplo:

```text
Connection Pool
─────────────────────────────
Domain: primary/writer
Open: 12
Active: 7
Idle: 5
Waiting: 2
Quarantined: 0

Reuse ratio: 91.4%
Avg acquire: 1.8 ms
Avg reset: 0.6 ms
Avg connect: 18.4 ms
```

---

# 251. Connection diagnostics

```text
Connection: conn-42
Generation: 17
State: IDLE
Health: HEALTHY
Cleanliness: CLEAN
Role: WRITER
Age: 14m
Idle: 3.2s
Leases: 281
Credential generation: 8
Topology generation: 102
Baseline: VALID
Reusable: YES
```

---

# 252. Security diagnostics

Nunca mostrar:

```text
password
full credential
secret token
sensitive certificate material
```

---

# 253. Error hierarchy

```text
DatabaseConnectionReuseException
├── ConnectionAcquisitionException
│   ├── ConnectionAcquisitionTimeoutException
│   └── ConnectionAcquisitionRejectedException
├── ConnectionLeaseException
│   ├── InvalidLeaseException
│   ├── StaleLeaseException
│   ├── DoubleReleaseException
│   └── UseAfterReleaseException
├── ConnectionReuseEligibilityException
├── ConnectionValidationException
├── ConnectionBaselineException
├── ConnectionResetException
├── ConnectionQuarantineException
├── ConnectionRetirementException
├── ConnectionPoolExhaustedException
├── ConnectionGenerationException
└── ConnectionDomainMismatchException
```

---

# 254. Failure semantics

Errores deberán distinguir:

```text
ACQUISITION_FAILED
CONNECTION_FAILED
RESET_FAILED
VALIDATION_FAILED
POOL_EXHAUSTED
CANCELLED
TIMED_OUT
UNKNOWN
```

---

# 255. UNKNOWN

Cuando el estado físico no puede determinarse:

```text
connection → not reusable
```

---

# 256. Retry

Acquisition retry podrá intentar otra conexión.

---

# 257. Retry ≠ same connection retry

Una conexión sospechosa no se reutiliza sólo para cumplir un retry.

---

# 258. Circuit Breaker

Si un endpoint está bajo circuit breaker:

```text
no new connection attempts
```

según policy.

---

# 259. Existing idle connections

El breaker podrá decidir si conexiones existentes verificadas siguen siendo utilizables.

Esto deberá ser explícito.

---

# 260. Failover

Al cambiar topología:

```text
mark affected pools/connections stale
```

y reevaluar.

---

# 261. No silent role assumption

Una conexión creada como writer no conserva eternamente esa autoridad.

---

# 262. Health Check integration

El Database Health Check System podrá observar pools y endpoints.

Pero:

```text
health check
≠
per-connection reset verification
```

---

# 263. Testing architecture

Se requieren:

```text
unit tests
integration tests
driver conformance tests
concurrency tests
fault injection
persistent runtime soak tests
```

---

# 264. Basic reuse test

```text
acquire C
release C
acquire again
```

verificar reuse cuando sea elegible.

---

# 265. Dirty transaction test

```text
BEGIN
release lease
```

deberá provocar rollback/reset antes de reuse.

---

# 266. Unknown transaction test

Simular pérdida durante commit.

Esperar:

```text
C discarded
```

---

# 267. Open cursor test

Liberar lease con cursor abierto.

Esperar:

```text
cursor closed
then reset
```

o discard.

---

# 268. Failed cursor cleanup

Esperar:

```text
connection not reused
```

---

# 269. Role contamination test

```text
SET ROLE privileged
release
acquire for next scope
```

El segundo scope recibe baseline role.

---

# 270. Search path contamination test

Especialmente PostgreSQL.

---

# 271. Timezone contamination test

Verificar baseline.

---

# 272. SQL mode contamination test

MySQL/MariaDB.

---

# 273. Temporary table test

Comprobar cleanup/discard policy.

---

# 274. Advisory lock test

Comprobar que no cruza leases.

---

# 275. Tenant test

```text
Lease A → tenant A
release
Lease B → tenant B
```

sin contaminación.

---

# 276. Shard test

Una conexión shard A nunca satisface request shard B.

---

# 277. Writer/replica test

Una replica connection no satisface write intent.

---

# 278. Replica lag test

Reusable connection sobre replica stale puede ser rechazada para read consistency fuerte.

---

# 279. Credential rotation test

Connections antiguas siguen política:

```text
drain
retire
reconnect
```

---

# 280. Topology generation test

Cambiar topology generation y verificar revalidation.

---

# 281. Failover test

Old writer connection no se reutiliza como writer sin reevaluación.

---

# 282. Idle timeout test

Connection demasiado idle:

```text
close or validate
```

según policy.

---

# 283. Maximum lifetime test

Connection expirada:

```text
retire
```

---

# 284. Max lease count test

Al superar threshold:

```text
close-on-release
```

---

# 285. Double release test

No duplicar conexión en pool.

---

# 286. Use-after-release test

Debe fallar.

---

# 287. Stale lease test

Lease generation antigua debe fallar.

---

# 288. Acquisition timeout race test

Timeout concurrente con release no deberá perder/duplicar conexión.

---

# 289. Cancellation test

Waiter cancelado no recibe lease.

---

# 290. Pool exhaustion test

No superar hard maximum.

---

# 291. Backpressure test

Pending acquisition queue permanece acotada.

---

# 292. Concurrency test

Múltiples Fibers/coroutines no obtienen el mismo exclusive connection simultáneamente.

---

# 293. OpenSwoole test

Ownership correcto entre coroutines.

---

# 294. FrankenPHP soak test

Miles/millones de requests reutilizando workers.

Verificar:

```text
no transaction leakage
no tenant leakage
no role leakage
no cursor leakage
bounded connections
```

---

# 295. RoadRunner soak test

Mismas garantías.

---

# 296. Fault injection

Simular:

```text
server restart
network disconnect
reset failure
ping timeout
authentication expiration
query cancellation uncertainty
driver protocol error
```

---

# 297. Driver conformance

Cada driver deberá pasar una suite común de reuse.

---

# 298. Platform-specific tests

Además:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

tendrán suites específicas.

---

# 299. Persistent worker memory test

Verificar que connection metadata/history no crezca indefinidamente.

---

# 300. Architecture directory

Propuesta:

```text
src/Quantum/Database/Connection/Reuse/
│
├── Contract/
│   ├── ConnectionReuseManager.php
│   ├── ConnectionReuseCapabilities.php
│   ├── ConnectionValidator.php
│   └── ConnectionRetirementPolicy.php
│
├── Model/
│   ├── ConnectionReuseContext.php
│   ├── ConnectionReusePolicy.php
│   ├── ConnectionReuseDecision.php
│   ├── ConnectionReuseEligibilityResult.php
│   ├── ConnectionBaseline.php
│   ├── ConnectionBaselineFingerprint.php
│   ├── ConnectionDomain.php
│   ├── ConnectionGeneration.php
│   ├── CredentialGeneration.php
│   ├── TopologyGeneration.php
│   ├── ConnectionHealth.php
│   ├── ConnectionCleanliness.php
│   └── ConnectionAffinity.php
│
├── Lease/
│   ├── ConnectionLease.php
│   ├── ConnectionLeaseState.php
│   ├── LeaseToken.php
│   ├── LeaseRegistry.php
│   └── LeaseLeakDetector.php
│
├── Manager/
│   └── DefaultConnectionReuseManager.php
│
├── Validation/
│   ├── DefaultConnectionValidator.php
│   ├── ConnectionValidationPolicy.php
│   └── ConnectionValidationResult.php
│
├── Retirement/
│   ├── ConnectionRetirementManager.php
│   ├── ConnectionRetirementReason.php
│   └── ConnectionLifetimePolicy.php
│
├── Quarantine/
│   ├── ConnectionQuarantineManager.php
│   └── ConnectionQuarantineReason.php
│
├── Diagnostics/
│   ├── ConnectionReuseDiagnostics.php
│   └── ConnectionPoolDiagnostics.php
│
└── Exception/
    ├── DatabaseConnectionReuseException.php
    ├── ConnectionAcquisitionException.php
    ├── ConnectionAcquisitionTimeoutException.php
    ├── ConnectionAcquisitionRejectedException.php
    ├── ConnectionLeaseException.php
    ├── InvalidLeaseException.php
    ├── StaleLeaseException.php
    ├── DoubleReleaseException.php
    ├── UseAfterReleaseException.php
    ├── ConnectionValidationException.php
    ├── ConnectionBaselineException.php
    ├── ConnectionQuarantineException.php
    ├── ConnectionPoolExhaustedException.php
    └── ConnectionDomainMismatchException.php
```

---

# 301. Relación con Connection Pool

Arquitectura:

```text
             ConnectionManager
                    │
                    ▼
                 Routing
                    │
                    ▼
             ConnectionPool
              ┌─────┴─────┐
              │           │
              ▼           ▼
          idle list     waiters
              │
              ▼
       Reuse Eligibility
              │
              ▼
          Validation
              │
              ▼
             Lease
              │
              ▼
          Application
              │
              ▼
            Release
              │
              ▼
          Reset System
              │
              ▼
       Reuse Verification
          ┌───┴────┐
          │        │
          ▼        ▼
        Pool     Discard
```

---

# 302. Architectural Invariants

## DB-CONN-REUSE-001

Physical Connection no equivaldrá a Logical Connection.

## DB-CONN-REUSE-002

Connection Lease no equivaldrá a Physical Connection.

## DB-CONN-REUSE-003

El lease representará ownership temporal.

## DB-CONN-REUSE-004

Una conexión tendrá un owner exclusivo por default.

## DB-CONN-REUSE-005

Una conexión no será reutilizable sólo por estar viva.

## DB-CONN-REUSE-006

Health no equivaldrá a cleanliness.

## DB-CONN-REUSE-007

UNKNOWN cleanliness impedirá reuse.

## DB-CONN-REUSE-008

Broken connection impedirá reuse.

## DB-CONN-REUSE-009

Reusable implicará baseline válido.

## DB-CONN-REUSE-010

Reusable implicará ausencia de transaction activa.

## DB-CONN-REUSE-011

Reusable implicará ausencia de cursor activo.

## DB-CONN-REUSE-012

Reusable implicará ausencia de stream activo.

## DB-CONN-REUSE-013

Reusable implicará reset verificado cuando haya existido dirty state.

## DB-CONN-REUSE-014

Release no equivaldrá a pool return.

## DB-CONN-REUSE-015

Reset deberá preceder al pool return bajo la política V1.

## DB-CONN-REUSE-016

Validation no sustituirá reset.

## DB-CONN-REUSE-017

Ping success no equivaldrá a safe reuse.

## DB-CONN-REUSE-018

Session baseline será explícito.

## DB-CONN-REUSE-019

Connection domain será explícito.

## DB-CONN-REUSE-020

Una conexión no cruzará logical database domains incompatibles.

## DB-CONN-REUSE-021

Una conexión no cruzará shards.

## DB-CONN-REUSE-022

Replica connection no ejecutará write intent.

## DB-CONN-REUSE-023

Writer authority deberá poder reevaluarse.

## DB-CONN-REUSE-024

Socket alive no equivaldrá a endpoint authoritative.

## DB-CONN-REUSE-025

Topology generation será observable.

## DB-CONN-REUSE-026

Credential generation será observable sin exponer secrets.

## DB-CONN-REUSE-027

Credential rotation podrá retirar conexiones.

## DB-CONN-REUSE-028

Connection generation protegerá contra stale references.

## DB-CONN-REUSE-029

Lease estará ligado a connection generation.

## DB-CONN-REUSE-030

Stale lease deberá fallar.

## DB-CONN-REUSE-031

Double release no duplicará recursos.

## DB-CONN-REUSE-032

Use-after-release deberá fallar.

## DB-CONN-REUSE-033

Ownership transition será atómica.

## DB-CONN-REUSE-034

Pool candidate deberá pasar eligibility.

## DB-CONN-REUSE-035

Validation policy será explícita.

## DB-CONN-REUSE-036

Validation no será obligatoriamente una query.

## DB-CONN-REUSE-037

Validation failure descartará candidato.

## DB-CONN-REUSE-038

Candidate retries serán acotados.

## DB-CONN-REUSE-039

Acquisition tendrá timeout/budget.

## DB-CONN-REUSE-040

Pool tendrá hard maximum.

## DB-CONN-REUSE-041

Pending acquisitions estarán acotadas.

## DB-CONN-REUSE-042

Pool sizing considerará worker amplification.

## DB-CONN-REUSE-043

Worker-local pool no se confundirá con global pool.

## DB-CONN-REUSE-044

External pooler semantics serán explícitas.

## DB-CONN-REUSE-045

Transaction fijará physical connection.

## DB-CONN-REUSE-046

Una local transaction no cambiará de physical connection.

## DB-CONN-REUSE-047

Savepoints permanecerán en la misma physical connection.

## DB-CONN-REUSE-048

Commit/rollback no devolverá inmediatamente conexión al pool.

## DB-CONN-REUSE-049

UNKNOWN transaction outcome impedirá reuse.

## DB-CONN-REUSE-050

Auto-reconnect no preservará ficticiamente physical identity.

## DB-CONN-REUSE-051

Reconnect generará nueva generation.

## DB-CONN-REUSE-052

Connection loss no será ocultado dentro de transaction.

## DB-CONN-REUSE-053

Prepared statement reuse será independiente de connection reuse.

## DB-CONN-REUSE-054

Prepared statement cache respetará connection locality.

## DB-CONN-REUSE-055

Unknown statement validity impedirá statement reuse.

## DB-CONN-REUSE-056

Schema changes podrán invalidar prepared statements.

## DB-CONN-REUSE-057

Reset podrá invalidar statement cache.

## DB-CONN-REUSE-058

Idle time y connection lifetime serán distintos.

## DB-CONN-REUSE-059

Connection lifetime será acotable.

## DB-CONN-REUSE-060

Idle lifetime será acotable.

## DB-CONN-REUSE-061

Connection retirement será explícito.

## DB-CONN-REUSE-062

Draining connection no aceptará nuevos leases.

## DB-CONN-REUSE-063

Credential rotation podrá causar draining.

## DB-CONN-REUSE-064

Topology change podrá causar draining.

## DB-CONN-REUSE-065

Worker shutdown causará draining.

## DB-CONN-REUSE-066

Constraint violation no marcará connection unhealthy por sí sola.

## DB-CONN-REUSE-067

SQL syntax error no marcará connection unhealthy por sí solo.

## DB-CONN-REUSE-068

Transport failures sí podrán afectar health.

## DB-CONN-REUSE-069

Timeout ambiguity será tratada conservadoramente.

## DB-CONN-REUSE-070

Cancellation uncertainty impedirá reuse.

## DB-CONN-REUSE-071

Quarantined connection no será entregada.

## DB-CONN-REUSE-072

Discard cerrará e invalidará physical resource.

## DB-CONN-REUSE-073

Close failure no devolverá conexión al pool.

## DB-CONN-REUSE-074

Connection creation tendrá initialization pipeline.

## DB-CONN-REUSE-075

Initialization failure impedirá lease.

## DB-CONN-REUSE-076

Configuration generation será rastreable.

## DB-CONN-REUSE-077

Dynamic config no mutará una conexión leased arbitrariamente.

## DB-CONN-REUSE-078

Replica reuse y replica eligibility serán decisiones diferentes.

## DB-CONN-REUSE-079

Sticky writer no implicará exact physical pinning.

## DB-CONN-REUSE-080

Exact pinning requerirá necesidad semántica.

## DB-CONN-REUSE-081

Session affinity será explícita.

## DB-CONN-REUSE-082

Release terminará session affinity salvo contrato especial.

## DB-CONN-REUSE-083

Una physical connection no será concurrent-safe por default.

## DB-CONN-REUSE-084

V1 no dependerá de multiplexing.

## DB-CONN-REUSE-085

Fibers no compartirán lease implícitamente.

## DB-CONN-REUSE-086

Coroutines no compartirán lease implícitamente.

## DB-CONN-REUSE-087

Lease no se transferirá entre scopes arbitrariamente.

## DB-CONN-REUSE-088

Pool podrá ser worker-scoped.

## DB-CONN-REUSE-089

Worker shutdown drenará pools.

## DB-CONN-REUSE-090

Worker recycle cerrará physical connections.

## DB-CONN-REUSE-091

FrankenPHP reuse requerirá request reset.

## DB-CONN-REUSE-092

RoadRunner reuse requerirá execution reset.

## DB-CONN-REUSE-093

OpenSwoole pool será coroutine-aware.

## DB-CONN-REUSE-094

Core de reuse será runtime-neutral.

## DB-CONN-REUSE-095

Runtime adapters manejarán lifecycle específico.

## DB-CONN-REUSE-096

Reuse podrá deshabilitarse.

## DB-CONN-REUSE-097

Reuse disabled será fallback seguro.

## DB-CONN-REUSE-098

Policies podrán variar por plataforma.

## DB-CONN-REUSE-099

SQLite tendrá semántica propia.

## DB-CONN-REUSE-100

SQLite `:memory:` no se tratará como una conexión desechable normal.

## DB-CONN-REUSE-101

MySQL y MariaDB serán plataformas distintas.

## DB-CONN-REUSE-102

Platform behavior se resolverá por capabilities.

## DB-CONN-REUSE-103

Version no equivaldrá a capability.

## DB-CONN-REUSE-104

V1 preferirá reset-on-release.

## DB-CONN-REUSE-105

V1 preferirá conditional validation-on-acquire.

## DB-CONN-REUSE-106

Dirty connection nunca entrará al clean idle pool.

## DB-CONN-REUSE-107

Pool key será semántico.

## DB-CONN-REUSE-108

Tenant no se incluirá siempre en pool key.

## DB-CONN-REUSE-109

Tenant pooling evitará cardinality explosion.

## DB-CONN-REUSE-110

Shard pooling evitará cardinality explosion.

## DB-CONN-REUSE-111

Pools podrán crearse lazily.

## DB-CONN-REUSE-112

Idle pools podrán podarse.

## DB-CONN-REUSE-113

Background maintenance no será requerido para correctness.

## DB-CONN-REUSE-114

Lease leaks serán detectables.

## DB-CONN-REUSE-115

Scope finalization conocerá owned leases.

## DB-CONN-REUSE-116

Leaked lease no cruzará execution boundary.

## DB-CONN-REUSE-117

Unsafe leaked resource será descartado.

## DB-CONN-REUSE-118

Acquisition stack será debug-only/sampleable.

## DB-CONN-REUSE-119

Connection permit durará physical lifetime.

## DB-CONN-REUSE-120

Backpressure será explícito.

## DB-CONN-REUSE-121

Wait queues serán bounded.

## DB-CONN-REUSE-122

Cancelled waiter no recibirá lease.

## DB-CONN-REUSE-123

Timed-out waiter no recibirá lease.

## DB-CONN-REUSE-124

Release/timeout races serán atómicas.

## DB-CONN-REUSE-125

Async pending operation impedirá reuse.

## DB-CONN-REUSE-126

Pool no dependerá del ORM.

## DB-CONN-REUSE-127

EntityManager no será pooled por default.

## DB-CONN-REUSE-128

ConnectionManager y ConnectionPool tendrán responsabilidades separadas.

## DB-CONN-REUSE-129

ReuseManager no ejecutará queries de negocio.

## DB-CONN-REUSE-130

Connection metadata será bounded.

## DB-CONN-REUSE-131

Connection metadata no almacenará secrets.

## DB-CONN-REUSE-132

Telemetry histórico no vivirá indefinidamente en connection object.

## DB-CONN-REUSE-133

Metrics evitarán IDs de alta cardinalidad.

## DB-CONN-REUSE-134

Reuse ratio no será objetivo de correctness.

## DB-CONN-REUSE-135

Reset cost será observable.

## DB-CONN-REUSE-136

Connect cost será observable.

## DB-CONN-REUSE-137

Pool wait será observable.

## DB-CONN-REUSE-138

UNKNOWN impedirá reuse.

## DB-CONN-REUSE-139

Retry podrá elegir otra conexión.

## DB-CONN-REUSE-140

Retry no rehabilitará conexión sospechosa.

## DB-CONN-REUSE-141

Circuit breaker podrá impedir nuevas conexiones.

## DB-CONN-REUSE-142

Failover invalidará assumptions de autoridad.

## DB-CONN-REUSE-143

Health check no equivaldrá a reset verification.

## DB-CONN-REUSE-144

Driver conformance incluirá reuse.

## DB-CONN-REUSE-145

Persistent runtime tendrá soak tests.

## DB-CONN-REUSE-146

Tenant contamination tendrá tests.

## DB-CONN-REUSE-147

Transaction contamination tendrá tests.

## DB-CONN-REUSE-148

Session contamination tendrá tests.

## DB-CONN-REUSE-149

Topology/failover tendrá tests.

## DB-CONN-REUSE-150

Credential rotation tendrá tests.

## DB-CONN-REUSE-151

Pool exhaustion tendrá tests.

## DB-CONN-REUSE-152

Concurrency ownership tendrá tests.

## DB-CONN-REUSE-153

Connection reuse nunca reducirá isolation guarantees.

## DB-CONN-REUSE-154

Connection reuse nunca reducirá transaction guarantees.

## DB-CONN-REUSE-155

Connection reuse nunca reducirá tenant isolation.

## DB-CONN-REUSE-156

Connection reuse nunca reducirá security guarantees.

## DB-CONN-REUSE-157

Un recurso incierto será descartable.

## DB-CONN-REUSE-158

Crear una nueva conexión será preferible a unsafe reuse.

## DB-CONN-REUSE-159

Correctness tendrá prioridad sobre reuse ratio.

## DB-CONN-REUSE-160

La reutilización será una optimización, nunca una condición para la semántica correcta de Database.

---

# 303. Modelo formal de adquisición

Sea:

```text
R
```

un request de conexión y:

```text
C = {c₁, c₂, ..., cₙ}
```

el conjunto de candidatos.

La selección válida es:

```text
Eligible(R)
=
{
 c ∈ C |
 DomainCompatible(c,R)
 ∧ Healthy(c)
 ∧ Clean(c)
 ∧ GenerationCompatible(c,R)
 ∧ RoleCompatible(c,R)
}
```

El pool podrá seleccionar:

```text
Select(Eligible(R))
```

según política de balanceo.

---

# 304. Modelo formal de release

Para una conexión `c`:

```text
Release(c)
=
StopWork(c)
→ ResolveResources(c)
→ ResolveTransaction(c)
→ Reset(c)
→ Verify(c)
→ Decide(c)
```

donde:

```text
Decide(c)
=
POOL        if Reusable(c)
DISCARD     if Unusable(c)
QUARANTINE  if RecoveryPolicy(c)
```

---

# 305. Modelo formal de reuse

```text
Reusable(c)
⇔
Alive(c)
∧ Clean(c)
∧ BaselineValid(c)
∧ DomainValid(c)
∧ NoTransaction(c)
∧ NoActiveResources(c)
∧ GenerationValid(c)
∧ ¬UnknownRelevantState(c)
```

---

# 306. Modelo formal de ownership

Para cualquier conexión `c`:

```text
|ActiveExclusiveLeases(c)| ≤ 1
```

en VoltStack V1.

---

# 307. Modelo formal de transaction affinity

Para una transaction local `T`:

```text
∀ op ∈ T:
PhysicalConnection(op)
=
PhysicalConnection(T)
```

---

# 308. Modelo formal de generation safety

Sea:

```text
L.connectionGeneration = g₁
C.generation = g₂
```

Si:

```text
g₁ ≠ g₂
```

entonces:

```text
ValidLease(L,C) = false
```

---

# 309. Modelo formal de pool capacity

```text
OpenConnections(P)
≤
MaxConnections(P)
```

siempre.

No:

```text
eventually
```

sino como hard invariant.

---

# 310. Anti-patterns

## Anti-pattern 1 — Singleton connection

```php
static $pdo;
```

utilizado sin scope/lease/reset.

Prohibido.

## Anti-pattern 2 — Ping means safe

```text
SELECT 1 succeeds
→ return to pool
```

Incorrecto.

## Anti-pattern 3 — Release before rollback

Crítico.

## Anti-pattern 4 — Release with active cursor

Crítico.

## Anti-pattern 5 — Reuse after UNKNOWN commit

Prohibido.

## Anti-pattern 6 — Silent reconnect

Cambiar socket debajo de una transaction manteniendo la misma identity.

Prohibido.

## Anti-pattern 7 — Pool per tenant indiscriminado

Puede producir explosión de conexiones.

## Anti-pattern 8 — Same connection across Fibers

Sin ownership explícito.

Prohibido.

## Anti-pattern 9 — Keep privileged role

Crítico.

## Anti-pattern 10 — Infinite connection lifetime

Sin health/lifetime policy.

No recomendado.

## Anti-pattern 11 — Unbounded wait queue

Riesgo de memory exhaustion.

## Anti-pattern 12 — Retry suspicious connection forever

Incorrecto.

## Anti-pattern 13 — Mix writer and replica pools blindly

Incorrecto.

## Anti-pattern 14 — Assume old writer remains writer

Peligroso después de failover.

## Anti-pattern 15 — Pool EntityManager like connection

Conceptos distintos.

---

# 311. Configuración propuesta

Ejemplo conceptual:

```php
return [

    'connections' => [

        'default' => [

            'reuse' => [
                'enabled' => true,

                'strategy' => 'reset_on_release',

                'validation' => [
                    'policy' => 'on_stale',
                    'interval' => '30s',
                ],

                'lifetime' => [
                    'max_idle' => '5m',
                    'max_lifetime' => '30m',
                    'max_leases' => 10000,
                    'jitter' => 0.10,
                ],

                'pool' => [
                    'min' => 0,
                    'max' => 20,
                    'max_waiters' => 100,
                    'acquire_timeout' => '5s',
                ],

                'safety' => [
                    'discard_on_unknown_state' => true,
                    'discard_on_failed_reset' => true,
                    'detect_leaked_leases' => true,
                ],
            ],
        ],
    ],
];
```

---

# 312. Secure defaults

VoltStack V1 debería utilizar:

```text
reuse enabled where driver/runtime supports it
reset on release
conditional validation on acquire
discard on unknown state
discard on reset failure
bounded pool
bounded waiter queue
lease leak detection
no EntityManager pooling
no implicit cross-Fiber sharing
no implicit session affinity
```

---

# 313. Performance model

Sin reuse:

```text
Request
→ TCP/TLS
→ DB handshake
→ authentication
→ session initialization
→ query
```

Con reuse:

```text
Request
→ lease clean connection
→ query
→ reset
→ pool
```

El beneficio puede ser considerable especialmente con:

```text
high request rate
TLS
remote DB
authentication overhead
persistent workers
```

---

# 314. Cost model

Sea:

```text
C_new
```

el costo de crear conexión.

Sea:

```text
C_reset
```

el costo de reset.

Sea:

```text
C_validate
```

el costo de validación.

Reuse es operacionalmente atractivo cuando:

```text
C_reset + C_validate
<
C_new
```

pero correctness sigue siendo requisito previo.

---

# 315. Correctness gate

Formalmente:

```text
UseReuseOptimization
=
SafeReusePossible
∧ OperationallyBeneficial
```

Nunca:

```text
OperationallyBeneficial
⇒ Safe
```

---

# 316. Arquitectura final

```text
                   DATABASE OPERATION
                          │
                          ▼
                 ConnectionManager
                          │
                          ▼
                      Routing
                          │
                          ▼
                   Pool Selection
                          │
                          ▼
              ┌──── Candidate? ─────┐
              │                     │
             YES                    NO
              │                     │
              ▼                     ▼
      Eligibility Check       Create Connection
              │                     │
              ▼                     ▼
         Validation            Initialize
              │                     │
              └──────────┬──────────┘
                         ▼
                    Create Lease
                         │
                         ▼
                  Bind Ownership
                         │
                         ▼
                       USE
                         │
                         ▼
                     RELEASE
                         │
                         ▼
                 Stop New Work
                         │
                         ▼
               Close Active Resources
                         │
                         ▼
              Resolve Transaction
                         │
                         ▼
                  Reset Session
                         │
                         ▼
                     Verify
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
           CLEAN       UNKNOWN      BROKEN
             │           │           │
             ▼           ▼           ▼
           POOL      QUARANTINE    DISCARD
                         │
                         ▼
                    RECOVER?
                    │       │
                   YES      NO
                    │       │
                    ▼       ▼
                 VERIFY   DISCARD
                    │
                    ▼
                   POOL
```

---

# 317. Decisión recomendada para VoltStack

Para la primera implementación estable:

```text
Physical Connection
        │
        ├── expensive resource
        ├── worker reusable
        └── strictly controlled
```

mientras:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
ConnectionLease
Query Context
```

seguirán siendo scope-bound.

Esto produce:

```text
Worker
│
├── Shared immutable infrastructure
├── Shared bounded Connection Pool
│
│   ├── PhysicalConnection #1
│   ├── PhysicalConnection #2
│   └── PhysicalConnection #3
│
├── Request A
│   ├── DatabaseContext A
│   ├── EntityManager A
│   └── Lease → Connection #2
│
└── Request B
    ├── DatabaseContext B
    ├── EntityManager B
    └── Lease → Connection #1
```

Nunca:

```text
Request A EntityManager
        ↓
Request B
```

pero sí puede existir:

```text
Physical Connection
Request A
   ↓
RESET + VERIFY
   ↓
Request B
```

---

# 318. Principio final

> **VoltStack reutilizará recursos físicos costosos, no contexto semántico mutable.**

Por tanto:

```text
Physical Connection
→ candidate for reuse

Connection Lease
→ never reused as ownership

DatabaseContext
→ recreate

EntityManager
→ recreate

IdentityMap
→ recreate

UnitOfWork
→ recreate
```

Y:

```text
Connection Alive
≠
Connection Clean
```

```text
Connection Clean
≠
Connection Compatible
```

```text
Connection Compatible
≠
Connection Authoritative
```

```text
Release
≠
Reuse
```

```text
Reset
≠
Verification
```

```text
Ping
≠
Safety
```

```text
UNKNOWN
≠
Reusable
```

La cadena correcta será:

```text
USE
 ↓
RELEASE
 ↓
RESET
 ↓
VERIFY
 ↓
ELIGIBILITY
 ↓
REUSE
```

---

# 319. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
✓ 254_DATABASE_STATE_ISOLATION_SYSTEM.md
✓ 255_DATABASE_STATE_RESET_SYSTEM.md
✓ 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
│
├── 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 320. Siguiente documento

```text
257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
```

El siguiente documento definirá cómo `VoltStack/Quantum/Database` se integra con el ciclo completo de vida de un worker persistente:

```text
BOOT
  ↓
INITIALIZE DATABASE INFRASTRUCTURE
  ↓
READY
  ↓
ACCEPT EXECUTION
  ↓
CREATE REQUEST SCOPE
  ↓
EXECUTE
  ↓
FINALIZE
  ↓
RESET
  ↓
VERIFY
  ↓
READY
  ↓
...
  ↓
DRAIN
  ↓
SHUTDOWN
```

incluyendo:

```text
Worker Bootstrap
Worker Database State
Worker Generations
Execution Admission
Request/Job Lifecycle
Concurrent Executions
Database Scope Creation
Shared Infrastructure
Connection Pool Lifetime
Execution Finalization
State Reset
Isolation Verification
Worker Tainting
Worker Health
Worker Recycling
Graceful Drain
Forced Shutdown
Connection Drain
Long-Lived Worker Memory
Leak Detection
Configuration Reload
Credential Rotation
Topology Changes
Runtime Adapter Contracts
FrankenPHP Preparation
RoadRunner Preparation
OpenSwoole Preparation
Telemetry
Diagnostics
Testing
```

bajo la regla:

> **Un worker VoltStack podrá procesar múltiples ejecuciones únicamente mientras su infraestructura compartida permanezca válida y cada ejecución anterior haya liberado o aislado completamente todo estado que no deba cruzar el siguiente boundary.**