# 321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md

# VoltStack Quantum Database
## Database HTTP Request Lifecycle Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 321 — Database HTTP Request Lifecycle Integration  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md`

---

# 1. Propósito

Este documento define cómo `VoltStack/Quantum/Database` participa dentro del ciclo completo de una petición HTTP de VoltStack.

La integración deberá cubrir:

- recepción de la petición;
- creación del Request Scope;
- resolución de configuración;
- creación de `DatabaseContext`;
- resolución de tenant;
- autenticación;
- autorización;
- adquisición de conexiones;
- Query Engine;
- ORM;
- `EntityManager`;
- `UnitOfWork`;
- `IdentityMap`;
- transacciones;
- generación de respuesta;
- streaming;
- errores;
- cancelación;
- desconexión del cliente;
- `afterCommit`;
- eventos;
- telemetría;
- reutilización de conexiones;
- limpieza del estado;
- workers persistentes;
- FrankenPHP;
- RoadRunner;
- OpenSwoole;
- concurrencia;
- seguridad;
- testing.

La regla fundamental será:

> **Una petición HTTP define una frontera de contexto y aislamiento para Database, pero no constituye automáticamente una transacción Database.**

Por tanto:

```text
HTTP Request
≠
Database Transaction
```

y:

```text
Request Scope
≠
Transaction Scope
```

---

# 2. Objetivo arquitectónico

La integración deberá permitir una experiencia simple:

```php
$user = User::find($id);
```

sin que el desarrollador tenga que administrar manualmente:

```text
Connection
EntityManager
IdentityMap
UnitOfWork
DatabaseContext
TenantContext
QueryContext
```

pero sin convertirlos en estado global.

La simplicidad de la API pública:

```text
Database::...
User::...
Repository::...
```

no deberá implicar:

```text
static mutable state
```

---

# 3. Regla principal de lifecycle

Cada petición deberá ejecutar conceptualmente:

```text
HTTP Request
      │
      ▼
Create Request Scope
      │
      ▼
Resolve Request Context
      │
      ▼
Create DatabaseContext
      │
      ▼
Execute Application
      │
      ▼
Produce Response
      │
      ▼
Finalize Database Scope
      │
      ▼
Reset / Release Resources
```

---

# 4. Arquitectura general

```text
Client
  │
  ▼
HTTP Server
  │
  ▼
VoltStack HttpKernel
  │
  ▼
RequestScopeManager
  │
  ├── RequestContext
  ├── TenantContext
  ├── AuthenticationContext
  ├── AuthorizationContext
  ├── TelemetryContext
  └── DatabaseContext
          │
          ▼
      EntityManager
          │
     ┌────┼─────┐
     ▼    ▼     ▼
    UoW IdentityMap Repository
          │
          ▼
      Query Engine
          │
          ▼
     Execution Engine
          │
          ▼
   ConnectionManager
          │
          ▼
        Driver
          │
          ▼
        DBMS
```

Al finalizar:

```text
Response / Failure
        │
        ▼
Database Finalization
        │
        ├── Transaction resolution
        ├── Cursor cleanup
        ├── EntityManager clear
        ├── UoW clear
        ├── IdentityMap clear
        ├── Connection reset
        ├── Telemetry finalize
        └── Context disposal
                │
                ▼
          Request Scope End
```

---

# 5. Separaciones fundamentales

VoltStack deberá mantener:

```text
HTTP Request
≠
Database Transaction
```

```text
HTTP Response
≠
Database Commit
```

```text
Request Scope
≠
Worker Scope
```

```text
Worker Lifetime
≠
Request Lifetime
```

```text
Connection Lifetime
≠
Request Lifetime
```

```text
Connection Reuse
≠
Connection State Reuse
```

```text
EntityManager Scope
≠
Connection Scope
```

```text
Authentication Context
≠
Database Credentials
```

```text
Authorization
≠
Database Permission
```

```text
Client Disconnect
≠
Database Rollback
```

```text
Response Sent
≠
Application Work Finished
```

---

# 6. HTTP request phases

Se propone el siguiente lifecycle:

```text
RECEIVED
   ↓
SCOPE_CREATED
   ↓
CONTEXT_RESOLVED
   ↓
APPLICATION_RUNNING
   ↓
RESPONSE_CREATED
   ↓
RESPONSE_EMITTING
   ↓
DATABASE_FINALIZING
   ↓
SCOPE_RESETTING
   ↓
COMPLETED
```

Con ramas:

```text
FAILED
CANCELLED
CLIENT_DISCONNECTED
TAINTED
```

---

# 7. Request Scope

Cada request tendrá un:

```text
RequestScope
```

único.

Ejemplo conceptual:

```text
RequestScope
├── RequestId
├── RequestContext
├── TenantContext?
├── AuthenticationContext?
├── AuthorizationContext?
├── DatabaseContext
├── TelemetryContext
├── CancellationContext
└── ResourceRegistry
```

---

# 8. RequestId

Cada request deberá poseer:

```text
RequestId
```

estable durante todo su lifecycle.

Podrá correlacionarse con:

```text
TraceId
CorrelationId
```

pero:

```text
RequestId
≠
TraceId
```

---

# 9. Worker scope

En FrankenPHP:

```text
Worker
  │
  ├── Request A
  ├── Request B
  ├── Request C
  └── ...
```

Por tanto:

```text
Worker Scope
```

es superior en duración a:

```text
Request Scope
```

---

# 10. Regla crítica para persistent runtimes

> **Todo estado Database mutable que pueda variar entre peticiones deberá pertenecer al Request/Operation Scope o a una frontera más pequeña, nunca al worker global.**

---

# 11. Estado compartible

Podrá compartirse entre requests cuando sea inmutable:

```text
Compiled Metadata
Type Registry
Driver Registry
Dialect Registry
Capability Definitions
Compiler Definitions
Mapping Metadata
Configuration Snapshot
```

---

# 12. Estado no compartible

No deberá compartirse directamente:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
TenantContext
QueryContext mutable
Loaded Entities
Open Result Cursor
Current Transaction
Request-specific Connection State
```

---

# 13. DatabaseContext

Cada petición tendrá un:

```text
DatabaseContext
```

propio.

---

# 14. DatabaseContext model

Conceptualmente:

```text
DatabaseContext
├── ContextId
├── RequestId
├── ConnectionIntent
├── TenantContext?
├── ShardContext?
├── ActorContext?
├── TransactionContext?
├── ReadWriteContext
├── ConsistencyRequirement
├── QueryPolicy
├── SecurityContext
├── TelemetryContext
└── ResourceBudget
```

---

# 15. DatabaseContext creation

La creación deberá ocurrir después de tener suficiente información contextual.

Ejemplo:

```text
Request
  ↓
Route Resolution
  ↓
Tenant Resolution
  ↓
Authentication
  ↓
DatabaseContext Resolution
```

El orden exacto podrá variar según aplicación, pero las dependencias deberán ser explícitas.

---

# 16. DatabaseContext ≠ Connection

Crear:

```text
DatabaseContext
```

no deberá abrir automáticamente una conexión.

---

# 17. Lazy connection acquisition

VoltStack deberá favorecer:

```text
lazy connection acquisition
```

---

# 18. Request without Database

Una petición:

```text
GET /health/static
```

podría no utilizar Database.

Por tanto:

```text
Request Start
```

no deberá implicar:

```text
DB Connect
```

---

# 19. Connection acquisition

La conexión se obtendrá cuando una operación la requiera:

```text
Query
Transaction
Schema operation
ORM persistence
```

---

# 20. ConnectionManager

La adquisición deberá pasar por:

```text
ConnectionManager
```

respetando:

- logical database;
- tenant;
- shard;
- read/write intent;
- transaction affinity;
- consistency;
- endpoint health;
- capabilities.

---

# 21. Request ≠ Connection

Una petición puede utilizar:

```text
0
1
N
```

conexiones físicas.

---

# 22. Connection ≠ Request

Una conexión física puede ser reutilizada secuencialmente por múltiples requests.

---

# 23. Reuse rule

Antes de reutilizar:

```text
Connection A
```

deberá demostrarse que se encuentra en baseline seguro.

---

# 24. Connection baseline

El reset deberá considerar al menos:

```text
active transaction
isolation level
read-only state
session variables
temporary tables
open cursors
prepared statements
role
schema/search_path
tenant variables
timezone
application variables
locks
```

según plataforma.

---

# 25. Connection reset failure

Si el reset:

```text
fails
```

o:

```text
outcome = UNKNOWN
```

la conexión deberá descartarse.

---

# 26. Authentication integration

Después de autenticar:

```text
HTTP Request
   ↓
Authentication
   ↓
ActorContext
```

Database podrá recibir una representación segura del actor.

---

# 27. ActorContext

Ejemplo:

```text
ActorContext
├── ActorType
├── ActorId
├── AuthenticationMethod
├── SecurityGeneration?
└── Delegation?
```

---

# 28. Actor ≠ Database User

Normalmente:

```text
Application User
≠
Database Credential
```

---

# 29. Database credentials

La aplicación utilizará credenciales controladas por infraestructura.

La identidad del usuario se aplicará mediante:

```text
Authorization
Data Access Policies
Query Scopes
Audit
Tenant Isolation
```

---

# 30. Authorization integration

La autorización deberá producir restricciones semánticas antes de ejecución cuando corresponda.

Ejemplo:

```text
Request
  ↓
Actor
  ↓
Authorization
  ↓
Data Access Policy
  ↓
Query
```

---

# 31. Authorization ≠ SQL concatenation

Nunca:

```php
$sql .= " WHERE user_id = " . $actor->id;
```

como mecanismo arquitectónico.

Las restricciones deberán integrarse mediante Query Model/AST/scopes seguros.

---

# 32. Tenant resolution

En aplicaciones multitenant:

```text
Request
  ↓
Tenant Resolver
  ↓
TenantContext
  ↓
DatabaseContext
```

---

# 33. Tenant resolution sources

Podrá utilizar:

```text
hostname
subdomain
route
authentication claim
signed header
explicit administrative context
```

según configuración.

---

# 34. Untrusted tenant input

Un header:

```text
X-Tenant-ID
```

no deberá convertirse directamente en autoridad de tenant sin validación.

---

# 35. Tenant context binding

Una vez resuelto:

```text
TenantContext
```

deberá quedar asociado al Request Scope.

---

# 36. Tenant immutability

Cambiar arbitrariamente de tenant a mitad del request estará prohibido por defecto.

---

# 37. Cross-tenant operation

Deberá utilizar una API explícita:

```php
Tenant::runAs($tenant, function () {
    // privileged operation
});
```

con aislamiento y autorización.

---

# 38. Cross-tenant scope

El cambio temporal deberá crear un:

```text
nested explicit operation context
```

en vez de mutar silenciosamente el contexto principal.

---

# 39. EntityManager lifecycle

Cada request que utilice ORM deberá obtener un:

```text
EntityManager
```

scoped.

---

# 40. EntityManager state

El EntityManager contendrá acceso a:

```text
UnitOfWork
IdentityMap
Hydration context
Persistence context
```

pero no será global.

---

# 41. EntityManager lifecycle

```text
CREATE
  ↓
OPEN
  ↓
USE
  ↓
CLEAR
  ↓
CLOSE / DISPOSE
```

Con posibles estados:

```text
TAINTED
CLOSED
```

---

# 42. EntityManager reuse

El mismo EntityManager no deberá sobrevivir entre requests.

---

# 43. IdentityMap

La garantía:

```text
same entity type
+
same identifier
+
same effective database context
=
same managed object
```

aplica únicamente dentro de su scope.

---

# 44. IdentityMap request isolation

Por tanto:

```text
Request A User#10
```

y:

```text
Request B User#10
```

no requieren la misma instancia PHP.

---

# 45. IdentityMap ≠ global cache

Nunca usar IdentityMap como cache entre requests.

---

# 46. UnitOfWork lifecycle

El `UnitOfWork` será request/operation-scoped.

---

# 47. Dirty entities at request end

Al finalizar una petición:

```text
dirty managed entities
```

no deberán persistirse automáticamente sólo porque termina el request.

---

# 48. No implicit flush at request end

Regla:

> **VoltStack no ejecutará `flush()` automáticamente al finalizar cada request salvo que una política explícita de aplicación lo configure.**

Default:

```text
NO IMPLICIT FLUSH
```

---

# 49. Why

Un flush implícito puede:

- persistir cambios accidentales;
- ocultar fronteras transaccionales;
- generar queries inesperadas;
- producir fallos después de crear la respuesta;
- dificultar testing.

---

# 50. Explicit persistence

Preferir:

```php
$user->save();
```

o:

```php
$entityManager->flush();
```

en fronteras claras.

---

# 51. Request transaction

VoltStack podrá ofrecer middleware:

```text
DatabaseTransactionMiddleware
```

pero será opcional.

---

# 52. Transaction-per-request

Una política podría ejecutar:

```text
BEGIN
  Controller
COMMIT
```

pero no será default universal.

---

# 53. Why not universal

No es apropiado para:

- streaming;
- SSE;
- descargas largas;
- WebSockets;
- endpoints read-only;
- llamadas externas lentas;
- operaciones con múltiples DB;
- workflows con commits parciales.

---

# 54. Request transaction policy

Se podrán soportar:

```text
NONE
EXPLICIT
MIDDLEWARE
ATTRIBUTE
ROUTE_POLICY
```

---

# 55. Attribute example

```php
#[Transactional]
public function create(CreateOrderRequest $request): Response
{
    // ...
}
```

---

# 56. Transactional middleware

Conceptualmente:

```text
Request
   ↓
BEGIN
   ↓
Controller
   ↓
Success?
 ┌─┴─┐
Yes  No
 │    │
 ▼    ▼
COMMIT ROLLBACK
```

---

# 57. Response object creation ≠ success

Un controller puede crear:

```text
Response(200)
```

antes de que el commit ocurra.

Por tanto el middleware deberá confirmar la transacción antes de considerar exitosa la operación transaccional.

---

# 58. Commit failure

Ejemplo:

```text
Controller returns 200
       ↓
COMMIT
       ↓
FAIL
```

No deberá enviarse el `200` si todavía puede evitarse.

---

# 59. Recommended sequence

Para respuestas no streaming:

```text
Controller
   ↓
Response Object
   ↓
COMMIT
   ↓
Commit confirmed
   ↓
Emit Response
```

---

# 60. UNKNOWN commit

Si:

```text
COMMIT sent
connection lost
```

entonces:

```text
TransactionOutcome = UNKNOWN
```

---

# 61. HTTP mapping

Un outcome `UNKNOWN` deberá convertirse en un error operacional apropiado.

Nunca asumir:

```text
UNKNOWN = ROLLED_BACK
```

---

# 62. Client retry danger

Un cliente puede reintentar después de recibir:

```text
500
503
connection reset
```

aunque el commit anterior realmente haya ocurrido.

---

# 63. Idempotency

Endpoints mutativos críticos podrán utilizar:

```text
Idempotency-Key
```

o mecanismo equivalente.

---

# 64. HTTP idempotency architecture

```text
Request
  ↓
Idempotency Key
  ↓
BEGIN
  ├── Claim key
  ├── Business mutation
  └── Store outcome
  ↓
COMMIT
```

cuando sea posible.

---

# 65. HTTP retry ≠ transaction retry

Debe distinguirse:

```text
Client Retry
Framework Retry
Transaction Retry
Query Retry
```

---

# 66. Retry multiplication

La arquitectura deberá impedir combinaciones descontroladas.

---

# 67. GET semantics

El método HTTP:

```text
GET
```

podrá contribuir a un:

```text
READ_ONLY intent
```

pero no será la única fuente de verdad.

---

# 68. HTTP method ≠ DB behavior

Un endpoint `POST` puede ser read-only y un código defectuoso en `GET` podría intentar mutar.

Database no deberá inferir toda la política exclusivamente del verbo HTTP.

---

# 69. Read-only route

Se podrá declarar:

```php
#[DatabaseReadOnly]
public function index(): Response
{
}
```

---

# 70. Read-only enforcement

Cuando sea soportado:

```text
ReadOnly Route
→
Read Intent
→
Replica eligibility
→
Optional read-only transaction/session
```

---

# 71. Read replica routing

Una consulta podrá usar réplica sólo cuando:

```text
ReadIntent
∧
ConsistencyAllowsReplica
∧
NoActiveWriteTransaction
∧
ReplicaEligible
```

---

# 72. Locking read

```text
SELECT ... FOR UPDATE
```

deberá dirigirse al writer.

---

# 73. Transaction affinity

Una vez iniciada una transacción:

```text
Transaction
→
Physical Connection
```

deberá mantenerse correctamente fijada.

---

# 74. Sticky connections

Después de una escritura, una petición podrá requerir:

```text
read-your-writes
```

---

# 75. Sticky scope

El sticky state será:

```text
request/database-context scoped
```

no worker-global.

---

# 76. HTTP redirect

Un redirect posterior a una escritura crea una nueva petición.

Por tanto:

```text
Request-local sticky
```

no necesariamente sobrevive.

---

# 77. Cross-request consistency

Si se requiere read-your-writes entre:

```text
POST
→
302
→
GET
```

deberá existir una estrategia explícita.

---

# 78. Possible strategies

Por ejemplo:

```text
writer grace period
consistency token
replication position
signed session hint
application policy
```

según plataforma.

---

# 79. Response lifecycle

Se distinguirá:

```text
Response Created
Response Headers Prepared
Response Started
Response Body Emitting
Response Completed
```

---

# 80. Why important

Antes de:

```text
Response Started
```

el framework todavía puede reemplazar una respuesta por un error.

Después:

```text
HTTP status/headers
```

pueden ser irreversibles.

---

# 81. Streaming responses

Una respuesta streaming:

```text
StreamedResponse
SSE
large export
```

requiere tratamiento especial.

---

# 82. Streaming ≠ normal response

No mantener automáticamente:

```text
Database Transaction
```

durante toda una transmisión larga.

---

# 83. Streaming query

Puede existir:

```text
DB Cursor
→
HTTP Stream
```

pero deberá manejar:

- backpressure;
- disconnect;
- cursor lifecycle;
- connection ownership;
- timeout;
- transaction requirements.

---

# 84. Client disconnect

El cliente puede desconectarse mientras:

```text
Database Query
```

continúa.

---

# 85. Disconnect propagation

Cuando sea seguro y soportado:

```text
Client Disconnect
→
CancellationContext
→
Query Cancellation
```

---

# 86. Cancellation ≠ rollback

Cancelar un query no implica necesariamente:

```text
Transaction rolled back
```

---

# 87. Cancellation outcome

Después de cancelación deberá verificarse el estado de:

```text
Connection
Transaction
```

antes de reutilizarlos.

---

# 88. Streaming disconnect

Si el cliente abandona una descarga:

```text
cursor
```

deberá cerrarse.

---

# 89. Resource ownership

Todo recurso abierto durante el request deberá tener un propietario explícito.

Ejemplos:

```text
Connection
Cursor
Stream
Temporary Table
Transaction
EntityManager
```

---

# 90. ResourceRegistry

El Request Scope podrá mantener:

```text
DatabaseResourceRegistry
```

para cleanup determinista.

---

# 91. Cleanup ordering

Orden conceptual:

```text
Stop application work
      ↓
Resolve active transaction
      ↓
Close cursors/streams
      ↓
Finalize ORM
      ↓
Clear UoW/IdentityMap
      ↓
Reset connections
      ↓
Release/discard connections
      ↓
Finalize telemetry
      ↓
Dispose DatabaseContext
```

---

# 92. Active transaction at request end

Una transacción inesperadamente abierta deberá considerarse:

```text
lifecycle violation
```

---

# 93. Safe default

Si se detecta:

```text
ACTIVE TRANSACTION
```

sin propietario válido al finalizar:

```text
ROLLBACK
```

cuando el estado permita demostrarlo.

---

# 94. Unknown transaction state

Si el estado es incierto:

```text
connection
→
DISCARD
```

---

# 95. No implicit commit

Nunca:

```text
request ended
→
commit whatever is open
```

---

# 96. Exception lifecycle

Si ocurre una excepción:

```text
Application
  ↓
Exception
  ↓
HTTP Exception Handler
```

Database deberá primero preservar información suficiente para resolver recursos correctamente.

---

# 97. Exception ≠ rollback automatically

Sólo se hará rollback de una transacción cuyo scope/policy corresponda.

---

# 98. Nested transaction ownership

Una capa no deberá hacer rollback arbitrario de una transacción perteneciente a otra frontera sin respetar `TransactionContext`.

---

# 99. Transaction ownership

Cada transacción deberá conocer:

```text
Owner
Scope
Depth
Policy
Connection
Outcome
```

---

# 100. Error response

La respuesta de error deberá generarse sin exponer:

- credentials;
- SQL parameters sensibles;
- tenant secrets;
- raw connection strings;
- internal tokens.

---

# 101. Development diagnostics

En desarrollo podrán mostrarse:

```text
query fingerprint
database logical name
platform
safe SQL representation
timing
transaction state
```

según política de redacción.

---

# 102. Production diagnostics

En producción se favorecerá:

```text
ErrorId
TraceId
safe error code
```

---

# 103. afterCommit integration

Callbacks `afterCommit` deberán ejecutarse cuando el commit correspondiente se confirme.

---

# 104. afterCommit ≠ response callback

No confundir:

```text
afterCommit
```

con:

```text
afterResponse
```

---

# 105. Example

```text
COMMIT
  ↓
afterCommit
  ↓
Response
```

y:

```text
Response
  ↓
afterResponse
```

representan fronteras diferentes.

---

# 106. afterCommit failure

Si:

```text
COMMIT = SUCCESS
afterCommit callback = FAILURE
```

el commit no puede deshacerse.

---

# 107. Critical post-commit work

Si el trabajo posterior debe ser durable:

```text
Transactional Outbox
```

será preferible a un callback en memoria.

---

# 108. Events

Eventos Database podrán emitirse durante:

```text
query
connection
transaction
ORM
persistence
```

---

# 109. Request correlation

Todo evento deberá poder correlacionarse con:

```text
RequestId
TraceId
DatabaseContextId
TransactionId?
```

---

# 110. Events ≠ transaction semantics

Un listener no deberá redefinir si una transacción realmente hizo commit.

---

# 111. Telemetry integration

El Request Scope deberá enlazar:

```text
HTTP Span
   ↓
Database Span
   ↓
Query Span
```

---

# 112. Example trace

```text
HTTP GET /users/{id}
│
├── auth
│
├── controller
│   ├── database.query
│   └── database.hydration
│
└── response
```

---

# 113. Database metrics

Podrán asociarse al request:

```text
query_count
query_duration
connection_acquire_time
transaction_count
rows_read
rows_written
hydrated_entities
cache_hits
cache_misses
```

---

# 114. N+1 detection

El Request Scope constituye una frontera natural para correlacionar patrones:

```text
1 query
+
N related queries
```

---

# 115. N+1 evidence

La detección deberá utilizar semántica de queries y relaciones, no sólo strings SQL.

---

# 116. Debug toolbar

En desarrollo:

```text
HTTP Request
   ↓
Database Debug Collector
   ↓
Debug Toolbar
```

podrá mostrar información segura.

---

# 117. Debug toolbar ≠ runtime dependency

Database Core no dependerá del toolbar.

---

# 118. Cache integration

Caches request-locales podrán pertenecer al Request Scope.

---

# 119. Cache categories

No confundir:

```text
IdentityMap
Query Cache
Result Cache
Metadata Cache
Entity Cache
Hydration Cache
HTTP Cache
```

---

# 120. Request-local cache cleanup

Todo cache L0 mutable deberá limpiarse al finalizar el request.

---

# 121. Shared caches

Caches L2 podrán sobrevivir entre requests, pero sus keys deberán incluir el contexto semántico necesario.

---

# 122. Tenant cache safety

Nunca:

```text
cache key = user:10
```

si:

```text
User#10 Tenant A
```

y:

```text
User#10 Tenant B
```

son entidades diferentes.

---

# 123. Validation integration

Request validation deberá ocurrir antes de persistir cuando corresponda.

---

# 124. Validation ≠ Database constraint

Ambas capas son complementarias.

---

# 125. Race condition

Aunque:

```text
Validation says email available
```

otra petición puede insertarlo antes del commit.

Por tanto:

```text
UNIQUE constraint
```

sigue siendo autoridad de integridad.

---

# 126. Constraint mapping

Una violación Database podrá mapearse a un error de aplicación apropiado cuando exista una regla conocida.

---

# 127. Do not leak DB internals

No responder:

```text
SQLSTATE 23505 ...
```

directamente al cliente salvo debugging autorizado.

---

# 128. Concurrent requests

Dos requests simultáneos deberán poseer contextos completamente independientes.

---

# 129. Concurrency model

```text
Worker / Runtime
   │
   ├── Request A
   │    ├── DatabaseContext A
   │    ├── EntityManager A
   │    └── Transaction A
   │
   └── Request B
        ├── DatabaseContext B
        ├── EntityManager B
        └── Transaction B
```

---

# 130. OpenSwoole

En OpenSwoole:

```text
Coroutine A
Coroutine B
```

pueden coexistir dentro del mismo proceso.

---

# 131. Coroutine-local state

Estado mutable deberá ser:

```text
coroutine/request scoped
```

---

# 132. Forbidden global state

Nunca:

```php
Database::$currentConnection
Database::$currentTenant
Model::$entityManager
Transaction::$current
```

como mutable global compartido.

---

# 133. Facades

Las Facades podrán parecer estáticas:

```php
Database::connection();
```

pero resolverán:

```text
Current Scope
→
Scoped Service
```

---

# 134. Facade ≠ static service

La sintaxis estática no deberá implicar lifecycle estático.

---

# 135. Model API

Igualmente:

```php
User::query();
```

deberá resolver:

```text
ModelContextResolver
→
Current DatabaseContext
→
EntityManager / Query Engine
```

---

# 136. Model static state

Prohibido almacenar:

```text
current EntityManager
current tenant
current connection
```

en propiedades estáticas de `User`.

---

# 137. Controller integration

Flujo típico:

```text
HttpKernel
  ↓
Middleware
  ↓
ControllerResolver
  ↓
Controller
  ↓
Repository / Model / Database
```

---

# 138. Parameter injection

Un controller podrá recibir:

```php
public function show(
    UserRepository $users,
    DatabaseContext $databaseContext,
): Response
```

mediante Container scoped.

---

# 139. Entity route binding

Si VoltStack soporta:

```php
public function show(User $user)
```

el resolver deberá utilizar el DatabaseContext del request.

---

# 140. Route model binding

La resolución:

```text
Route parameter
→
Entity identifier
→
Repository
→
Entity
```

deberá respetar:

```text
tenant
authorization
soft-delete policy
shard
```

---

# 141. Binding ≠ authorization

Encontrar una entidad no implica permiso para accederla.

---

# 142. 404 vs 403 policy

La aplicación podrá decidir ocultar existencia de recursos.

Database no decidirá esta política HTTP por sí solo.

---

# 143. Middleware ordering

Ejemplo recomendado:

```text
Request
  ↓
Request Scope
  ↓
Telemetry
  ↓
Tenant Resolution
  ↓
Authentication
  ↓
Authorization Context
  ↓
Database Policy Context
  ↓
Controller
```

---

# 144. Lazy DB principle

Aunque `DatabaseContext` exista:

```text
physical connection
```

sólo se adquiere al necesitarse.

---

# 145. Middleware must not connect unnecessarily

Un middleware que sólo construye contexto no deberá provocar queries ocultas por defecto.

---

# 146. Authentication Database usage

El propio sistema de autenticación puede necesitar Database antes de tener ActorContext.

Esto deberá soportarse mediante un contexto bootstrap controlado.

---

# 147. Bootstrap database context

Se podrá utilizar:

```text
PreAuthenticationDatabaseContext
```

con capacidades limitadas.

---

# 148. Transition

Después de autenticación:

```text
PreAuth DB Context
      ↓
Authenticated Request Context
```

sin reutilizar estado inseguro.

---

# 149. Tenant-before-auth vs auth-before-tenant

VoltStack deberá soportar ambos modelos:

```text
Tenant → Authentication
```

o:

```text
Authentication → Tenant
```

según aplicación.

---

# 150. Dependency declaration

El pipeline deberá declarar qué componente necesita:

```text
tenant
actor
database
```

para evitar ciclos ocultos.

---

# 151. Session integration

Si sesiones se almacenan en Database:

```text
Session Middleware
→
Database
```

podrá ejecutarse temprano.

---

# 152. Session DB context

El storage de sesiones no deberá asumir automáticamente el mismo tenant DB que la aplicación.

---

# 153. Infrastructure database

Podrá existir:

```text
system
session
tenant
analytics
```

como logical databases separados.

---

# 154. Request may use multiple logical databases

Ejemplo:

```text
Request
├── system DB
├── tenant DB
└── audit DB
```

sin asumir una transacción distribuida.

---

# 155. Multi-database atomicity

```text
Transaction A
+
Transaction B
```

no serán tratados como una sola transacción ACID.

---

# 156. Saga/outbox

Cuando se requiera coordinación:

```text
Outbox
Saga
Compensation
Workflow
```

podrán utilizarse.

---

# 157. HTTP response and external side effects

Evitar:

```text
BEGIN
DB update
external API call
COMMIT
```

sin diseño explícito.

---

# 158. External API inside transaction

Puede aumentar:

```text
lock duration
deadlocks
timeouts
uncertain outcomes
```

---

# 159. Preferred pattern

Cuando sea posible:

```text
Request
  ↓
DB transaction
  ├── business mutation
  └── outbox
  ↓
COMMIT
  ↓
HTTP Response
  ↓
Async side effect
```

---

# 160. HTTP status semantics

Database integration no deberá asumir:

```text
2xx = DB committed
```

en todos los endpoints.

---

# 161. Response mapping

La capa HTTP decide cómo convertir:

```text
DatabaseException
```

en:

```text
HTTP Response
```

---

# 162. Error taxonomy

Ejemplos:

```text
ValidationConflict
ConcurrencyConflict
ResourceNotFound
AuthorizationFailure
DatabaseUnavailable
Timeout
UnknownTransactionOutcome
```

podrán mapearse semánticamente.

---

# 163. Optimistic locking

Un:

```text
OptimisticLockException
```

podrá convertirse en:

```text
409 Conflict
```

según política HTTP.

---

# 164. Deadlock

Un deadlock puede ser retryable internamente.

No deberá convertirse inmediatamente en error HTTP si una transacción completa puede repetirse con seguridad.

---

# 165. Transaction retry

La repetición deberá ocurrir desde una frontera:

```text
replayable
```

---

# 166. Controller replay danger

Reejecutar automáticamente un controller completo puede repetir:

- emails;
- HTTP calls;
- filesystem writes;
- queue dispatch;
- random values.

Por tanto no será default.

---

# 167. Safe transaction callback

Preferir:

```php
Database::transaction(function () {
    // explicitly replayable database work
});
```

---

# 168. Request timeout

El servidor HTTP puede imponer:

```text
Request Deadline
```

---

# 169. Deadline propagation

Podrá propagarse a:

```text
DatabaseContext
→
QueryContext
→
Query Timeout
```

respetando límites mínimos.

---

# 170. Deadline composition

Si:

```text
HTTP remaining = 2s
Query configured timeout = 10s
```

el query no debería asumir automáticamente 10 segundos disponibles.

---

# 171. Effective timeout

Conceptualmente:

```text
EffectiveQueryDeadline
=
min(
    QueryPolicyDeadline,
    RequestRemainingDeadline,
    TransactionDeadline,
    ResourceBudgetDeadline
)
```

cuando corresponda.

---

# 172. Deadline ≠ guaranteed cancellation

Un DBMS/driver puede no soportar cancelación perfecta.

La capability deberá consultarse.

---

# 173. HTTP client disconnect

No todos los runtimes detectan disconnect de la misma manera.

La integración deberá abstraer:

```text
CancellationSignal
```

---

# 174. Platform capability

La cancelación Database dependerá de:

```text
Driver Capability
Platform Capability
Connection State
Query Phase
```

---

# 175. Response streaming transaction policy

Por defecto:

```text
No long-lived transaction across HTTP streaming
```

---

# 176. Streaming entity hydration

Para grandes datasets:

```text
Cursor
→
Hydrate
→
Serialize
→
Emit
```

deberá evitar crecimiento indefinido de `IdentityMap`.

---

# 177. Streaming ORM strategy

Podrá:

```text
detach processed entities
clear in chunks
use scalar/DTO hydration
```

según semántica.

---

# 178. Partial response failure

Una vez emitidos bytes:

```text
HTTP response
```

no puede reemplazarse limpiamente por un error convencional.

---

# 179. Streaming failure telemetry

El sistema deberá registrar:

```text
bytes emitted
rows processed
failure phase
database state
```

sin exponer información sensible.

---

# 180. HTTP cache interaction

HTTP Cache y Database Result Cache serán independientes.

---

# 181. Invalidation

Un commit Database podrá generar eventos que eventualmente invaliden:

```text
application cache
HTTP cache
```

pero la coordinación deberá ser explícita.

---

# 182. Cache invalidation before commit

Nunca publicar invalidación externa definitiva antes de saber si el cambio Database fue confirmado, salvo protocolo específico.

---

# 183. afterCommit invalidation

Podrá utilizarse:

```text
afterCommit
```

para invalidaciones locales/no críticas.

---

# 184. Durable invalidation

Para invalidaciones críticas distribuidas:

```text
Transactional Outbox
```

puede ser preferible.

---

# 185. Request telemetry finalization

Antes de destruir el Request Scope:

```text
Database telemetry
```

deberá finalizar.

---

# 186. Telemetry failure

Un fallo del exporter no deberá modificar el resultado Database.

---

# 187. Telemetry ≠ transaction participant

Nunca:

```text
telemetry export failure
→
rollback committed business transaction
```

---

# 188. Audit

Auditoría crítica podrá requerir una estrategia más fuerte.

---

# 189. Audit transaction

Cuando sea necesario:

```text
Business Mutation
+
Audit Record
```

podrán persistirse en la misma transacción local.

---

# 190. External audit sink

Si se publica externamente:

```text
Outbox
```

puede utilizarse.

---

# 191. Security lifecycle

El `DatabaseSecurityContext` deberá construirse desde información validada.

---

# 192. No raw headers

Database no deberá consumir directamente:

```text
Authorization header
Cookie
X-Tenant
```

como autoridad.

---

# 193. Context normalization

La capa HTTP convierte datos de transporte en:

```text
validated framework context
```

antes de que Database los utilice.

---

# 194. Credential isolation

Connection strings no deberán incorporarse a:

```text
Request attributes
HTTP logs
responses
trace baggage
```

---

# 195. SQL parameter privacy

Los parámetros sensibles deberán redactarse en:

```text
exceptions
debug
telemetry
logs
profilers
```

según clasificación.

---

# 196. Request body ≠ trusted query

Inputs del usuario deberán convertirse mediante APIs parametrizadas.

---

# 197. Dynamic identifiers

Columnas/tablas dinámicas deberán validarse mediante:

```text
typed identifiers
allowlists
metadata
```

no mediante binding de valores.

---

# 198. Raw SQL

`raw()` deberá seguir siendo un escape hatch explícito.

La existencia de HTTP input nunca deberá convertir raw SQL en seguro.

---

# 199. Long-lived worker architecture

FrankenPHP será el runtime oficial predeterminado.

Arquitectura:

```text
FrankenPHP Worker
      │
      ├── Immutable Framework State
      │
      ├── Shared Compiled Database Metadata
      │
      └── Request Loop
             │
             ├── Scope A
             │    └── reset
             │
             ├── Scope B
             │    └── reset
             │
             └── Scope N
```

---

# 200. FrankenPHP integration

El adapter deberá coordinar:

```text
request start
request end
worker start
worker stop
fatal failure
```

con Database lifecycle.

---

# 201. Worker start

En worker start podrán prepararse:

```text
immutable metadata
registries
compiled configuration
driver definitions
dialects
capability definitions
```

---

# 202. Worker start ≠ connect all DBs

No abrir conexiones a todas las bases durante bootstrap salvo necesidad explícita.

---

# 203. Worker request start

Crear:

```text
RequestScope
DatabaseContext
EntityManager
Request-local telemetry
```

lazy según necesidad.

---

# 204. Worker request end

Ejecutar:

```text
DatabaseStateReset
```

obligatoriamente.

---

# 205. Worker stop

Cerrar:

```text
pooled connections
background resources
telemetry buffers
```

según ownership.

---

# 206. Fatal error

Si no puede demostrarse limpieza:

```text
worker
```

podrá marcarse:

```text
TAINTED
```

---

# 207. Worker quarantine

Un worker tainted no deberá seguir procesando requests si existe riesgo de contaminación de estado.

---

# 208. RoadRunner integration

Aplicará el mismo contrato:

```text
Worker Lifetime
>
Request Lifetime
```

---

# 209. OpenSwoole integration

Añade:

```text
concurrent coroutine scopes
```

por lo que aislamiento deberá ser aún más estricto.

---

# 210. Runtime adapter contract

Se propone:

```php
interface DatabaseHttpRuntimeAdapter
{
    public function beginRequest(
        HttpRequestContext $request
    ): DatabaseRequestScope;

    public function endRequest(
        DatabaseRequestScope $scope,
        RequestOutcome $outcome
    ): void;
}
```

---

# 211. Scope manager

```php
interface DatabaseRequestScopeManager
{
    public function create(
        HttpRequestContext $context
    ): DatabaseRequestScope;

    public function current(): DatabaseRequestScope;

    public function reset(
        DatabaseRequestScope $scope
    ): ResetReport;
}
```

---

# 212. ResetReport

Deberá proporcionar evidencia:

```text
transactionsResolved
cursorsClosed
entityManagerCleared
identityMapCleared
connectionsReset
connectionsDiscarded
temporaryStateRemoved
errors
finalStatus
```

---

# 213. Reset status

Se propone:

```text
CLEAN
CLEAN_WITH_DISCARDED_RESOURCES
TAINTED
FAILED
UNKNOWN
```

---

# 214. UNKNOWN reset

Nunca:

```text
UNKNOWN
→
assume clean
```

---

# 215. Worker continuation rule

Conceptualmente:

```text
ContinueWorker
=
ResetStatus ∈ {
    CLEAN,
    CLEAN_WITH_DISCARDED_RESOURCES
}
```

según política.

---

# 216. Request Outcome

Se propone:

```text
SUCCESS
APPLICATION_FAILURE
DATABASE_FAILURE
CANCELLED
CLIENT_DISCONNECTED
TIMEOUT
UNKNOWN
```

---

# 217. Request Outcome ≠ Transaction Outcome

Una petición puede fallar después de que una transacción haya hecho commit.

---

# 218. Example

```text
DB COMMIT
  ↓
SUCCESS
  ↓
Template rendering
  ↓
FAIL
```

Entonces:

```text
Request = FAILED
Database transaction = COMMITTED
```

---

# 219. Critical invariant

Nunca reportar:

```text
request failed
→
database rolled back
```

sin evidencia.

---

# 220. Opposite example

```text
Controller logic succeeds
  ↓
COMMIT fails
```

Entonces:

```text
application logic = completed
database transaction = failed/unknown
request = failure
```

---

# 221. Response generation timing

La aplicación deberá evitar side effects irreversibles de HTTP antes de resolver commits críticos.

---

# 222. Cookies and sessions

Cambios de sesión/cookies también pueden constituir efectos independientes de Database.

---

# 223. Session commit ≠ DB transaction commit

Si session storage y business DB son distintos:

```text
Session Write
≠
Business DB Commit
```

---

# 224. Request completion model

Formalmente:

```text
RequestCompletion
=
ApplicationCompletion
∧
RequiredDatabaseFinalization
∧
RequiredResponseEmission
∧
RequiredScopeCleanup
```

pero cada componente posee resultados independientes.

---

# 225. Resource safety

Una petición sólo será:

```text
DatabaseScopeClean
```

cuando pueda demostrarse:

```text
NoUnresolvedTransaction
∧
NoOwnedOpenCursor
∧
NoManagedRequestStateLeak
∧
AllReusableConnectionsSanitized
∧
AllUnsafeConnectionsDiscarded
```

---

# 226. Scope contamination

Si un objeto scoped aparece en otro request:

```text
scope leak
```

será un defecto crítico.

---

# 227. Scope identity

Servicios mutables podrán almacenar:

```text
ScopeId
```

para detectar uso fuera de scope en modo debug/testing.

---

# 228. Stale service protection

Ejemplo:

```text
EntityManager from Request A
used in Request B
```

deberá producir:

```text
ScopeViolationException
```

cuando sea detectable.

---

# 229. Deferred callbacks

Closures/callbacks registrados durante el request no deberán conservar servicios scoped más allá de su lifetime.

---

# 230. Async work

Si una operación debe continuar después del request:

```text
Job
```

deberá transportar DTOs/referencias, no el EntityManager actual.

---

# 231. Request → Job boundary

```text
HTTP Request
     │
     ▼
Database Transaction
     │
     ▼
Outbox / afterCommit
     │
     ▼
Queue
     │
     ▼
New Job Scope
```

---

# 232. Request context ≠ job context

Un Job deberá reconstruir su propio:

```text
DatabaseContext
```

como se definió en el documento 320.

---

# 233. Testing architecture

La integración HTTP deberá probarse mediante:

```text
Unit
Integration
HTTP Integration
Persistent Runtime
Concurrency
Failure Injection
Security
Performance
Leak Detection
```

---

# 234. Unit tests

Cubrir:

- lifecycle state machine;
- context resolution;
- transaction policy;
- cleanup planning;
- error mapping;
- deadline composition;
- read/write intent;
- scope ownership.

---

# 235. HTTP integration tests

Deberán ejecutar:

```text
real HttpKernel
+
real Database integration
```

cuando la propiedad dependa de ambos.

---

# 236. Real DB testing

Transacciones, locks, resets y conexiones deberán probarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

independientemente.

---

# 237. Request isolation test

Ejecutar:

```text
Request A
Request B
```

secuencialmente sobre el mismo worker y comprobar ausencia de contaminación.

---

# 238. Tenant isolation test

```text
Request A → Tenant A
Request B → Tenant B
```

deberá comprobar:

```text
no Entity
no Connection session state
no QueryContext
no Cache L0
```

filtrado desde A hacia B.

---

# 239. IdentityMap leak test

Cargar:

```text
User#1
```

en Request A.

Request B deberá obtener una instancia administrada por su propio scope.

---

# 240. Transaction leak test

Dejar intencionalmente una transacción abierta y verificar:

```text
rollback/discard
```

según estado.

---

# 241. Connection reset test

Modificar:

```text
isolation
timezone
role
search_path
session variables
```

y comprobar baseline en la siguiente petición.

---

# 242. Unknown reset test

Simular pérdida de conexión durante reset.

Resultado:

```text
connection discarded
```

---

# 243. Streaming test

Verificar:

```text
large stream
client disconnect
cursor close
connection cleanup
```

---

# 244. Commit-before-response test

En middleware transaccional:

```text
controller returns response
commit fails
```

y comprobar que no se emita falsamente éxito si la respuesta aún no comenzó.

---

# 245. Unknown commit test

Provocar:

```text
COMMIT sent
connection lost
```

y verificar que:

```text
UNKNOWN
```

se preserve.

---

# 246. Client retry test

Simular:

```text
commit success
response lost
client retries
```

para validar idempotency en endpoints críticos.

---

# 247. Concurrent request test

Usar sincronización determinista:

```text
barriers
latches
```

no `sleep()` como mecanismo de correctness.

---

# 248. OpenSwoole test

Ejecutar múltiples coroutines con:

```text
different tenants
different transactions
different EntityManagers
```

y comprobar aislamiento.

---

# 249. FrankenPHP soak test

Ejecutar:

```text
10,000+
requests
```

sobre workers persistentes y medir:

```text
memory growth
connection growth
identity leaks
cursor leaks
transaction leaks
scope leaks
```

---

# 250. Performance test

Medir por separado:

```text
request scope creation
DatabaseContext creation
connection acquisition
query execution
hydration
ORM flush
transaction commit
state reset
connection reset
```

---

# 251. No benchmark ambiguity

No atribuir:

```text
HTTP request latency
```

completa exclusivamente a Database.

---

# 252. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Http
```

---

# 253. Proposed directory structure

```text
src/Quantum/Database/Integration/Http/
├── Contract/
│   ├── DatabaseHttpRuntimeAdapter.php
│   ├── DatabaseRequestScopeManager.php
│   ├── DatabaseHttpContextResolver.php
│   └── DatabaseRequestFinalizer.php
│
├── Scope/
│   ├── DatabaseRequestScope.php
│   ├── DatabaseRequestScopeId.php
│   ├── DatabaseRequestScopeManager.php
│   ├── DatabaseRequestResourceRegistry.php
│   └── DatabaseScopeGuard.php
│
├── Context/
│   ├── HttpDatabaseContextFactory.php
│   ├── HttpDatabaseContextResolver.php
│   ├── RequestDatabaseIntent.php
│   ├── RequestActorContextBridge.php
│   └── RequestTenantContextBridge.php
│
├── Transaction/
│   ├── DatabaseTransactionMiddleware.php
│   ├── HttpTransactionPolicy.php
│   ├── HttpTransactionCoordinator.php
│   └── HttpTransactionOutcomeMapper.php
│
├── Connection/
│   ├── HttpConnectionScope.php
│   ├── HttpConnectionResetCoordinator.php
│   └── HttpConnectionReuseGuard.php
│
├── ORM/
│   ├── HttpEntityManagerFactory.php
│   ├── HttpEntityManagerScope.php
│   └── HttpOrmStateResetter.php
│
├── Routing/
│   ├── HttpReadWriteIntentResolver.php
│   ├── HttpConsistencyResolver.php
│   └── HttpShardContextResolver.php
│
├── Tenant/
│   ├── HttpTenantDatabaseBridge.php
│   └── HttpTenantScopeGuard.php
│
├── Security/
│   ├── HttpDatabaseSecurityContextFactory.php
│   └── HttpDatabaseInputGuard.php
│
├── Streaming/
│   ├── DatabaseHttpStream.php
│   ├── DatabaseStreamResourceGuard.php
│   └── DatabaseStreamCancellationBridge.php
│
├── Cancellation/
│   ├── HttpDatabaseCancellationBridge.php
│   └── HttpDatabaseDeadlineResolver.php
│
├── Runtime/
│   ├── FrankenPhpDatabaseRuntimeAdapter.php
│   ├── RoadRunnerDatabaseRuntimeAdapter.php
│   └── OpenSwooleDatabaseRuntimeAdapter.php
│
├── Telemetry/
│   ├── HttpDatabaseTelemetryBridge.php
│   └── HttpDatabaseDebugCollector.php
│
├── Finalization/
│   ├── DatabaseRequestFinalizer.php
│   ├── DatabaseRequestResetReport.php
│   └── DatabaseWorkerContinuationPolicy.php
│
└── Exception/
    ├── DatabaseRequestLifecycleException.php
    ├── DatabaseScopeViolationException.php
    ├── DatabaseRequestResetException.php
    └── DatabaseRequestTaintedException.php
```

---

# 254. Container integration

Servicios scoped podrán registrarse como:

```text
RequestScoped
```

mediante la arquitectura definida en:

```text
312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md
```

---

# 255. Suggested service scopes

| Servicio | Scope |
|---|---|
| DriverRegistry | Application/Worker |
| DialectRegistry | Application/Worker |
| TypeRegistry | Application/Worker |
| MetadataRegistry | Application/Worker |
| CompilerRegistry | Application/Worker |
| DatabaseContext | Request |
| EntityManager | Request |
| UnitOfWork | Request |
| IdentityMap | Request |
| TransactionContext | Request/Transaction |
| QueryContext | Operation |
| Cursor | Operation |
| Connection Lease | Operation/Transaction |

---

# 256. Configuration

Ejemplo conceptual:

```php
return [
    'database' => [
        'http' => [
            'lazy_connections' => true,

            'transactions' => [
                'default' => 'explicit',
            ],

            'reset' => [
                'strict' => true,
                'discard_on_unknown' => true,
            ],

            'disconnect' => [
                'cancel_queries' => true,
            ],

            'streaming' => [
                'allow_long_transactions' => false,
            ],
        ],
    ],
];
```

---

# 257. Developer experience

El caso normal deberá continuar siendo:

```php
final class UserController
{
    public function show(int $id): Response
    {
        $user = User::findOrFail($id);

        return response()->json($user);
    }
}
```

sin configuración manual de lifecycle.

---

# 258. Explicit transaction

```php
public function transfer(
    TransferRequest $request
): Response {
    Database::transaction(function () use ($request) {
        // atomic DB work
    });

    return response()->noContent();
}
```

---

# 259. Repository style

```php
final class OrderController
{
    public function __construct(
        private OrderRepository $orders,
    ) {}

    public function show(OrderId $id): Response
    {
        $order = $this->orders->get($id);

        return response()->json($order);
    }
}
```

Ambos estilos utilizarán el mismo engine.

---

# 260. Transaction middleware example

```php
#[Transactional]
public function create(
    CreateInvoiceRequest $request
): Response {
    $invoice = Invoice::create(
        $request->validated()
    );

    return response()->created($invoice);
}
```

Conceptualmente:

```text
BEGIN
  Controller
  flush/persistence
COMMIT
emit response
```

---

# 261. Outbox example

```php
Database::transaction(function () use ($data) {
    $order = Order::create($data);

    Outbox::dispatch(
        new OrderCreated(
            orderId: $order->id,
        )
    );
});
```

---

# 262. HTTP idempotency example

```php
#[Idempotent]
#[Transactional]
public function charge(
    ChargeRequest $request
): Response {
    // ...
}
```

La implementación concreta podrá integrarse con un subsistema de idempotencia sin acoplarlo al Driver.

---

# 263. Architectural invariants

## DB-HTTP-001

HTTP Request ≠ Database Transaction.

## DB-HTTP-002

Request Scope ≠ Worker Scope.

## DB-HTTP-003

Connection Lifetime ≠ Request Lifetime.

## DB-HTTP-004

Connection Reuse ≠ State Reuse.

## DB-HTTP-005

Response Created ≠ Response Sent.

## DB-HTTP-006

Response Sent ≠ Database Commit.

## DB-HTTP-007

Client Disconnect ≠ Database Rollback.

## DB-HTTP-008

Request Failure ≠ Transaction Rollback.

## DB-HTTP-009

Request Success ≠ Transaction Commit.

## DB-HTTP-010

EntityManager ≠ global service.

---

# 264. Scope invariants

## DB-HTTP-011

Cada request tendrá un scope independiente.

## DB-HTTP-012

DatabaseContext será scoped.

## DB-HTTP-013

EntityManager será scoped.

## DB-HTTP-014

UnitOfWork será scoped.

## DB-HTTP-015

IdentityMap será scoped.

## DB-HTTP-016

TransactionContext no será worker-global.

## DB-HTTP-017

TenantContext no será worker-global.

## DB-HTTP-018

Mutable QueryContext no será worker-global.

## DB-HTTP-019

Open cursors no sobrevivirán al scope propietario.

## DB-HTTP-020

Scope leaks serán errores arquitectónicos.

---

# 265. Connection invariants

## DB-HTTP-021

Request start no abrirá necesariamente una conexión.

## DB-HTTP-022

Una petición podrá usar cero conexiones.

## DB-HTTP-023

Una petición podrá usar múltiples conexiones.

## DB-HTTP-024

Una conexión reutilizable deberá estar sanitizada.

## DB-HTTP-025

Unknown reset ≠ clean connection.

## DB-HTTP-026

Dirty connection no volverá al pool.

## DB-HTTP-027

Active transaction no sobrevivirá accidentalmente al request.

## DB-HTTP-028

Open cursor deberá cerrarse.

## DB-HTTP-029

Session state deberá resetearse.

## DB-HTTP-030

Connection ownership será explícito.

---

# 266. ORM invariants

## DB-HTTP-031

No habrá implicit flush por request por defecto.

## DB-HTTP-032

Dirty entity ≠ automatically persistent at request end.

## DB-HTTP-033

IdentityMap ≠ cross-request cache.

## DB-HTTP-034

Managed entity no sobrevivirá como managed entre requests.

## DB-HTTP-035

EntityManager de Request A no será válido en Request B.

## DB-HTTP-036

Model static API no almacenará EntityManager estático.

## DB-HTTP-037

Repository resolverá servicios scoped.

## DB-HTTP-038

Route binding respetará DatabaseContext.

## DB-HTTP-039

Entity resolution ≠ authorization.

## DB-HTTP-040

Hydration state será scope-safe.

---

# 267. Transaction invariants

## DB-HTTP-041

Transaction-per-request será opcional.

## DB-HTTP-042

Request end no implicará commit automático.

## DB-HTTP-043

Unexpected active transaction se resolverá conservadoramente.

## DB-HTTP-044

Unknown transaction state no será fabricado como rollback.

## DB-HTTP-045

Commit crítico precederá response emission cuando sea posible.

## DB-HTTP-046

Savepoint ≠ request transaction.

## DB-HTTP-047

afterCommit ≠ afterResponse.

## DB-HTTP-048

afterCommit failure no revertirá commit confirmado.

## DB-HTTP-049

Durable post-commit work utilizará Outbox cuando sea necesario.

## DB-HTTP-050

Multi-database work no implicará distributed ACID.

---

# 268. Security invariants

## DB-HTTP-051

Raw HTTP input no será Database authority.

## DB-HTTP-052

Tenant input deberá validarse.

## DB-HTTP-053

Actor ≠ Database credential.

## DB-HTTP-054

Authentication ≠ authorization.

## DB-HTTP-055

Authorization ≠ DB privilege.

## DB-HTTP-056

SQL values serán parametrizados.

## DB-HTTP-057

Dynamic identifiers serán validados.

## DB-HTTP-058

Credentials no aparecerán en HTTP diagnostics.

## DB-HTTP-059

Sensitive query parameters serán redactados.

## DB-HTTP-060

Cross-tenant operations serán explícitas y autorizadas.

---

# 269. Runtime invariants

## DB-HTTP-061

FrankenPHP workers podrán sobrevivir múltiples requests.

## DB-HTTP-062

Mutable Database state no sobrevivirá requests.

## DB-HTTP-063

Immutable compiled state podrá reutilizarse.

## DB-HTTP-064

Worker start ≠ connect every database.

## DB-HTTP-065

Worker request end ejecutará reset.

## DB-HTTP-066

Failed reset podrá taint worker.

## DB-HTTP-067

Tainted worker podrá retirarse.

## DB-HTTP-068

RoadRunner respetará las mismas fronteras.

## DB-HTTP-069

OpenSwoole tendrá aislamiento por coroutine.

## DB-HTTP-070

Global mutable current connection estará prohibida.

---

# 270. Streaming invariants

## DB-HTTP-071

Streaming response ≠ normal buffered response.

## DB-HTTP-072

Streaming no mantendrá long transaction por defecto.

## DB-HTTP-073

Cursor ownership será explícito.

## DB-HTTP-074

Client disconnect cerrará recursos cuando sea detectable.

## DB-HTTP-075

Cancellation ≠ rollback.

## DB-HTTP-076

Cancelled query podrá taint transaction/connection según driver.

## DB-HTTP-077

Partial response failure será observable.

## DB-HTTP-078

Large ORM streams controlarán IdentityMap growth.

## DB-HTTP-079

Backpressure será respetado.

## DB-HTTP-080

Response completion cerrará recursos owned.

---

# 271. Consistency invariants

## DB-HTTP-081

GET ≠ guaranteed replica-safe.

## DB-HTTP-082

POST ≠ guaranteed writer-required.

## DB-HTTP-083

Locking read utilizará writer.

## DB-HTTP-084

Transaction mantendrá connection affinity.

## DB-HTTP-085

Sticky state será scoped.

## DB-HTTP-086

Redirect crea nueva frontera request.

## DB-HTTP-087

Cross-request read-your-writes requerirá estrategia explícita.

## DB-HTTP-088

Replica lag será considerado.

## DB-HTTP-089

UNKNOWN freshness ≠ fresh.

## DB-HTTP-090

Consistency requirement será explícito cuando importe.

---

# 272. Failure invariants

## DB-HTTP-091

Exception ≠ automatic rollback universal.

## DB-HTTP-092

Commit failure después de controller success producirá request failure.

## DB-HTTP-093

Request failure después de commit no cambiará commit.

## DB-HTTP-094

UNKNOWN commit permanecerá UNKNOWN.

## DB-HTTP-095

HTTP retry podrá duplicar mutación.

## DB-HTTP-096

Critical mutations podrán usar idempotency.

## DB-HTTP-097

Deadlock retry se hará desde frontera segura.

## DB-HTTP-098

Controller completo no será reejecutado automáticamente por defecto.

## DB-HTTP-099

External side effect no será revertido por DB rollback.

## DB-HTTP-100

Failure diagnostics serán seguros.

---

# 273. Telemetry invariants

## DB-HTTP-101

Database telemetry se correlacionará con request.

## DB-HTTP-102

Telemetry ≠ transaction participant.

## DB-HTTP-103

Telemetry failure no cambiará DB outcome.

## DB-HTTP-104

Query count será observable.

## DB-HTTP-105

Connection acquisition será observable.

## DB-HTTP-106

Transaction duration será observable.

## DB-HTTP-107

N+1 podrá correlacionarse por request.

## DB-HTTP-108

Metrics evitarán cardinalidad no acotada.

## DB-HTTP-109

Sensitive values serán redactados.

## DB-HTTP-110

Reset failures serán observables.

---

# 274. Testing invariants

## DB-HTTP-111

Persistent worker isolation será probado.

## DB-HTTP-112

Tenant contamination será probada.

## DB-HTTP-113

IdentityMap contamination será probada.

## DB-HTTP-114

Transaction leak será probado.

## DB-HTTP-115

Connection reset será probado.

## DB-HTTP-116

Unknown reset será probado.

## DB-HTTP-117

Client disconnect será probado cuando runtime lo permita.

## DB-HTTP-118

Streaming cleanup será probado.

## DB-HTTP-119

Unknown commit será probado.

## DB-HTTP-120

Concurrency tests no dependerán de sleeps.

---

# 275. Additional invariants

## DB-HTTP-121

Validation success ≠ DB constraint success.

## DB-HTTP-122

Database constraint ≠ HTTP validation message.

## DB-HTTP-123

Session transaction ≠ business transaction.

## DB-HTTP-124

HTTP Cache ≠ Result Cache.

## DB-HTTP-125

Cache invalidation crítica podrá requerir Outbox.

## DB-HTTP-126

Request deadline podrá limitar query deadline.

## DB-HTTP-127

Timeout ≠ confirmed cancellation.

## DB-HTTP-128

Cancellation support será capability-aware.

## DB-HTTP-129

Request context ≠ Job context.

## DB-HTTP-130

Async work no conservará servicios request-scoped.

---

# 276. Lifecycle invariants

## DB-HTTP-131

Request finalization será determinista.

## DB-HTTP-132

Cleanup será idempotente cuando sea posible.

## DB-HTTP-133

Cleanup failure será visible.

## DB-HTTP-134

Reset UNKNOWN no permitirá reuse ciego.

## DB-HTTP-135

DatabaseResourceRegistry rastreará recursos owned.

## DB-HTTP-136

Resource ownership será transferido explícitamente.

## DB-HTTP-137

Response streaming extenderá sólo los recursos necesarios.

## DB-HTTP-138

Deferred callback no conservará scope inválido.

## DB-HTTP-139

ScopeId podrá utilizarse para detectar stale services.

## DB-HTTP-140

Scope violation fallará explícitamente en lugar de contaminar otro request.

---

# 277. Final invariants

## DB-HTTP-141

Simplicidad de API ≠ estado global.

## DB-HTTP-142

Facade estática ≠ servicio estático.

## DB-HTTP-143

Model API estática ≠ ORM global.

## DB-HTTP-144

Connection pooling ≠ session state pooling.

## DB-HTTP-145

Persistent worker ≠ persistent request context.

## DB-HTTP-146

Application completion ≠ DB completion.

## DB-HTTP-147

DB completion ≠ response completion.

## DB-HTTP-148

Response completion ≠ worker completion.

## DB-HTTP-149

Worker reuse requerirá reset demostrado.

## DB-HTTP-150

VoltStack nunca fabricará aislamiento que no pueda demostrar.

---

# 278. Anti-pattern: global EntityManager

Incorrecto:

```php
final class Database
{
    public static EntityManager $manager;
}
```

En workers persistentes puede filtrar estado entre requests.

---

# 279. Anti-pattern: connection per request always

```text
request start
→
connect DB
```

desperdicia recursos para endpoints que no utilizan Database.

---

# 280. Anti-pattern: commit at request end

```text
if request ends:
    commit()
```

es inseguro.

---

# 281. Anti-pattern: flush everything

```text
request finalizer
→
EntityManager::flush()
```

puede persistir mutaciones accidentales.

---

# 282. Anti-pattern: transaction around streaming

```text
BEGIN
stream 2 GB
COMMIT
```

por defecto es inaceptable.

---

# 283. Anti-pattern: trust tenant header

```php
Database::tenant(
    $request->header('X-Tenant')
);
```

sin resolución/validación.

---

# 284. Anti-pattern: worker-global tenant

```php
CurrentTenant::$tenant = $tenant;
```

es incompatible con concurrencia segura.

---

# 285. Anti-pattern: response before critical commit

```text
emit 200
↓
commit
↓
failure
```

puede informar éxito de una operación no confirmada.

---

# 286. Anti-pattern: rollback after response failure

Si el commit ya ocurrió:

```text
response socket failure
→
rollback
```

no puede deshacerlo.

---

# 287. Anti-pattern: assume failed request means no mutation

El cliente nunca deberá depender de esa suposición para operaciones críticas.

---

# 288. Anti-pattern: HTTP method as sole routing policy

```text
GET → replica
POST → writer
```

es demasiado simplista.

---

# 289. Anti-pattern: reuse dirty connection

Una conexión no deberá regresar al pool sólo porque:

```text
PDO object still works
```

---

# 290. Anti-pattern: global transaction state

```php
Transaction::$current = $transaction;
```

no es seguro para workers/coroutines.

---

# 291. Anti-pattern: async closure with scoped objects

```php
afterResponse(function () use ($entityManager) {
    // ...
});
```

puede utilizar un servicio fuera de scope.

---

# 292. Anti-pattern: external call inside long transaction

```text
BEGIN
update
HTTP request 30 sec
update
COMMIT
```

aumenta riesgos operacionales.

---

# 293. Anti-pattern: hidden DB query in serialization

Serializar una entidad no deberá disparar accidentalmente:

```text
lazy load
→
DB query
```

durante response emission.

---

# 294. Serialization policy

Por defecto:

```text
Serialization
≠
Permission to perform lazy DB I/O
```

---

# 295. Lazy loading during response

Podrá:

```text
ALLOW
WARN
FORBID
```

según configuración.

Para APIs estrictas se recomienda:

```text
FORBID
```

fuera de la fase de aplicación.

---

# 296. Formal request model

Sea:

```text
R = HTTP Request
S(R) = Request Scope
D(R) = DatabaseContext
```

entonces:

```text
∀ R1 ≠ R2:
Mutable(D(R1)) ∩ Mutable(D(R2)) = ∅
```

salvo recursos explícitamente compartidos e inmutables.

---

# 297. Connection reuse model

Sean:

```text
R1
R2
```

requests secuenciales y:

```text
C
```

una conexión reutilizable.

Puede existir:

```text
R1 → C → Reset(C) → R2
```

sólo si:

```text
Reset(C) = PROVEN_CLEAN
```

---

# 298. Transaction model

Sea:

```text
T
```

una transacción iniciada dentro de `R`.

No se cumple:

```text
Lifetime(T) = Lifetime(R)
```

necesariamente.

Debe cumplirse:

```text
Lifetime(T) ⊆ Lifetime(valid operation scope)
```

---

# 299. Worker model

Para un worker:

```text
W = {R1, R2, ..., Rn}
```

deberá cumplirse:

```text
MutableState(Ri)
∩
MutableState(Rj)
=
∅
```

para:

```text
i ≠ j
```

salvo mecanismos explícitamente diseñados como caches compartidos.

---

# 300. Safe response model

Para una operación que requiere commit antes de informar éxito:

```text
CanEmitSuccess
=
ApplicationSucceeded
∧
RequiredCommitOutcome = COMMITTED
```

No:

```text
ApplicationSucceeded
```

por sí solo.

---

# 301. Cleanup model

```text
SafeRequestReuse
=
NoActiveTransaction
∧
NoOpenOwnedCursor
∧
NoScopedOrmStateLeak
∧
ConnectionsSanitizedOrDiscarded
∧
ScopedContextDisposed
```

---

# 302. Unknown state rule

Para cualquier recurso:

```text
Reusable(resource)
=
KnownSafe(resource)
```

Nunca:

```text
Reusable(resource)
=
NotKnownBroken(resource)
```

La ausencia de evidencia de fallo no constituye evidencia de limpieza.

---

# 303. Recommended V1 lifecycle

```text
HTTP Request
      │
      ▼
Create RequestScope
      │
      ▼
Create RequestContext
      │
      ▼
Resolve Tenant/Auth as required
      │
      ▼
Create DatabaseContext
      │
      ▼
Resolve Controller
      │
      ▼
Application Work
      │
      ├── lazy connection
      ├── ORM
      ├── queries
      └── explicit transactions
      │
      ▼
Create Response
      │
      ▼
Resolve Required Transaction Outcome
      │
      ▼
Finalize Database Resources
      │
      ▼
Emit / Complete Response
      │
      ▼
Reset Scoped State
      │
      ▼
Reset/Release Connections
      │
      ▼
Dispose RequestScope
      │
      ▼
Worker Ready
```

El orden preciso de response emission/finalization podrá adaptarse para streaming, pero deberá conservar las garantías definidas.

---

# 304. Recommended V1 components

VoltStack V1 deberá implementar:

```text
DatabaseRequestScope
DatabaseContext per request
Request-scoped EntityManager
Request-scoped UnitOfWork
Request-scoped IdentityMap
lazy connection acquisition
transaction ownership
optional transactional middleware
tenant bridge
authentication actor bridge
authorization bridge
read/write intent resolution
request deadlines
query cancellation bridge
resource registry
connection reset
ORM reset
scope reset
ResetReport
worker continuation policy
FrankenPHP adapter
request telemetry correlation
N+1 request correlation
safe exception mapping
streaming resource handling
scope leak detection
persistent worker tests
```

---

# 305. V2

Podrá incorporar:

```text
cross-request consistency tokens
advanced client idempotency middleware
adaptive query deadlines
database-aware HTTP admission control
automatic worker quarantine policies
advanced streaming ORM
request database budgets
multi-database coordination helpers
RoadRunner runtime adapter
OpenSwoole coroutine adapter
advanced scope leak diagnostics
```

---

# 306. V3

Podrá incorporar:

```text
distributed request consistency tokens
multi-region read routing
adaptive replica selection
request-aware workload scheduling
cross-service database causality metadata
advanced cancellation propagation
database workload QoS
automatic anomaly-driven worker recycling
```

sin alterar las invariantes fundamentales.

---

# 307. Arquitectura final

```text
                       HTTP Client
                            │
                            ▼
                      HTTP Runtime
                            │
                            ▼
                       HttpKernel
                            │
                            ▼
                    RequestScopeManager
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          TenantContext  ActorContext  Telemetry
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                     DatabaseContext
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
         EntityManager   QueryContext   TxContext
              │             │              │
              ▼             ▼              ▼
             UoW        Query Engine   Transaction
              │             │              │
              └─────────────┼──────────────┘
                            ▼
                     Execution Engine
                            │
                            ▼
                    ConnectionManager
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
               Writer                Replica
                 │                     │
                 └──────────┬──────────┘
                            ▼
                           DBMS

Application
     │
     ▼
Response Object
     │
     ▼
Required Commit Resolution
     │
     ▼
Database Finalization
     │
     ├── Cursors
     ├── Transactions
     ├── EntityManager
     ├── UnitOfWork
     ├── IdentityMap
     ├── Connections
     └── Telemetry
     │
     ▼
Response Completion
     │
     ▼
Scope Reset
     │
     ▼
Worker Reuse
```

---

# 308. Regla definitiva del Request Scope

> **Toda petición HTTP de VoltStack deberá disponer de una frontera de contexto explícita dentro de la cual vivirá el estado mutable de Database. El final de esa petición deberá destruir o resetear dicho estado antes de que el runtime pueda reutilizar el worker para otra operación.**

---

# 309. Regla definitiva de transacciones

> **VoltStack no equiparará el ciclo de vida de una petición HTTP con el ciclo de vida de una transacción Database. Las transacciones deberán poseer fronteras, ownership y outcomes explícitos independientemente del resultado HTTP.**

Por tanto:

```text
Request Success
≠
Commit
```

y:

```text
Request Failure
≠
Rollback
```

---

# 310. Regla definitiva de conexiones

> **La reutilización de una conexión física estará permitida únicamente cuando VoltStack pueda demostrar que su estado ha sido restaurado a un baseline seguro. Una conexión cuyo estado sea incierto será descartada y nunca reutilizada por conveniencia de rendimiento.**

Formalmente:

```text
ReusableConnection
=
Healthy
∧
NoActiveTransaction
∧
NoOwnedCursor
∧
SessionStateReset
∧
SecurityStateReset
∧
TenantStateReset
```

---

# 311. Regla definitiva del ORM

> **`EntityManager`, `UnitOfWork`, `IdentityMap` y cualquier estado de entidades administradas pertenecerán al Request/Operation Scope y nunca constituirán estado persistente del worker.**

Así:

```text
Persistent Worker
≠
Persistent ORM Session
```

---

# 312. Regla definitiva de persistent runtimes

> **FrankenPHP podrá mantener vivo el proceso, los registries inmutables, metadata compilada y recursos reutilizables seguros, pero cada petición deberá observar un entorno lógico equivalente al de un proceso limpio respecto al estado mutable de Database.**

Esta será una de las propiedades centrales de VoltStack:

```text
Long-Lived Performance
+
Request Isolation
```

sin sacrificar una por la otra.

---

# 313. Regla definitiva de incertidumbre

> **Cuando el estado final de una transacción, conexión, cancelación o reset no pueda demostrarse, VoltStack conservará explícitamente la incertidumbre y evitará reutilizar recursos o repetir operaciones como si el fallo estuviera confirmado.**

Nunca:

```text
UNKNOWN
→
assume rollback
```

ni:

```text
UNKNOWN
→
assume clean
```

---

# 314. Invariante final

La integración completa deberá conservar permanentemente:

```text
HTTP Request
≠
Database Transaction
```

```text
Request Scope
≠
Worker Scope
```

```text
EntityManager
≠
Worker State
```

```text
Connection Reuse
≠
State Reuse
```

```text
Response Created
≠
Response Sent
```

```text
Response Sent
≠
Commit
```

```text
Request Failure
≠
Rollback
```

```text
Client Disconnect
≠
Cancellation Confirmation
```

```text
Cancellation
≠
Rollback
```

```text
Worker Persistence
≠
Request State Persistence
```

---

# 315. Cierre del bloque 32

Con este documento queda definida la integración de `VoltStack/Quantum/Database` con los principales subsistemas del framework:

```text
311 Framework Integration Architecture
        │
        ├── 312 Container
        ├── 313 Config
        ├── 314 Cache
        ├── 315 Events
        ├── 316 Telemetry
        ├── 317 Validation
        ├── 318 Authentication
        ├── 319 Authorization
        ├── 320 Jobs / Queue
        └── 321 HTTP Request Lifecycle
```

La arquitectura resultante permite:

```text
VoltStack Framework
        │
        ▼
Integration Contracts
        │
        ▼
Quantum Database
        │
        ├── ORM
        ├── Query Engine
        ├── Schema
        ├── Transactions
        ├── Cache
        ├── Security
        ├── Resilience
        └── Runtime
```

sin invertir la dirección de dependencias del núcleo.

---

# 316. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 33
DATABASE GOVERNANCE AND COMPATIBILITY
```

El objetivo será garantizar que la evolución futura de `VoltStack/Quantum/Database` pueda ocurrir sin romper arbitrariamente:

- aplicaciones;
- paquetes;
- plugins;
- drivers;
- dialects;
- custom compilers;
- ORM extensions;
- schemas;
- migrations;
- APIs públicas;
- formatos persistidos.

---

# 317. Siguiente documento

```text
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md
```

Este documento deberá definir:

```text
Public API Compatibility
Semantic Compatibility
Behavioral Compatibility
Database Compatibility
Schema Compatibility
Migration Compatibility
Metadata Compatibility
Extension Compatibility
Driver Compatibility
Serialized Format Compatibility
```

y formalizar especialmente:

```text
Same Method Signature
≠
Same Behavior
```

```text
Deprecated
≠
Immediately Removed
```

```text
Backward Compatible
≠
Bug-for-Bug Compatible
```

```text
Internal API
≠
Public Contract
```

junto con:

- compatibility boundaries;
- public/internal/experimental APIs;
- semantic versioning;
- deprecation lifecycle;
- compatibility matrix;
- extension contracts;
- persistent formats;
- migration compatibility;
- driver/plugin compatibility;
- runtime compatibility;
- testing;
- release gates;
- BC checker;
- upgrade diagnostics;
- governance rules.