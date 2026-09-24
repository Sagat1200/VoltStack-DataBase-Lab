# 255_DATABASE_STATE_RESET_SYSTEM.md

# VoltStack Quantum Database
## Database State Reset System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 255 — Database State Reset System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `254_DATABASE_STATE_ISOLATION_SYSTEM.md`  
**Siguiente documento:** `256_DATABASE_CONNECTION_REUSE_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database State Reset System** de `VoltStack/Quantum/Database`.

Su responsabilidad es establecer cómo VoltStack devuelve el estado reutilizable de Database a un **baseline conocido, verificable y seguro** cuando termina una ejecución, operación, transacción o lease.

La regla central es:

> **Un recurso sólo podrá reutilizarse cuando VoltStack pueda demostrar que regresó a un baseline válido. Si el resultado del reset es desconocido, el recurso no será considerado limpio.**

Formalmente:

```text
Reusable(R)
⇒
ResetVerified(R)
```

y:

```text
Reset(R) = UNKNOWN
⇒
Reusable(R) = false
```

---

# 2. Relación con State Isolation

El documento anterior estableció:

```text
Isolation
    ↓
prevents incorrect state sharing
```

Este documento establece:

```text
Reset
    ↓
restores reusable state
```

Ambos mecanismos son complementarios.

```text
Persistent Runtime Safety
        │
        ├── State Isolation
        │      ↓
        │   ownership boundaries
        │
        └── State Reset
               ↓
            safe reuse
```

---

# 3. Reset ≠ Isolation

Nunca deberá asumirse:

```text
reset everything later
=
safe isolation
```

Una arquitectura basada únicamente en limpiar globals después de cada request sería frágil.

VoltStack utilizará:

```text
Isolation First
+
Reset Before Reuse
```

---

# 4. Reset ≠ Destruction

También:

```text
Reset
≠
Destroy
```

Reset intenta recuperar un recurso para reutilización.

Destroy elimina el recurso.

Ejemplo:

```text
Physical Connection
     │
     ├── reset succeeds
     │       ↓
     │    return to pool
     │
     └── reset uncertain/fails
             ↓
          discard
```

---

# 5. Reset ≠ Rollback

Regla crítica:

```text
Reset
≠
Transaction Rollback
```

Un rollback puede formar parte del reset de una conexión, pero no resuelve todo el estado.

Podrían permanecer:

```text
session variables
temporary tables
roles
timezone
search_path
SQL modes
advisory locks
prepared statements
driver buffers
```

---

# 6. Reset ≠ Object Graph Rewind

Igualmente:

```text
Database Rollback
≠
ORM Object Graph Rewind
```

Después de rollback, objetos PHP pueden conservar valores modificados.

Reset del ORM debe decidir qué hacer con ellos.

---

# 7. Objetivos

El sistema deberá proporcionar:

1. clasificación del estado reseteable;
2. definición de baselines;
3. planes de reset;
4. orden determinista;
5. reset de execution state;
6. reset de ORM;
7. reset de transactions;
8. reset de routing;
9. reset de tenant/shard context;
10. reset de query state;
11. reset de telemetry;
12. reset de connection session;
13. reset de resources;
14. verificación posterior;
15. quarantine;
16. discard;
17. worker recycle;
18. capabilities por plataforma;
19. observabilidad;
20. testing.

---

# 8. No objetivos

Este sistema no será responsable de:

```text
query compilation
SQL optimization
ORM mapping
connection pooling policy
worker scheduling
business transaction recovery
```

aunque se integrará con ellos.

---

# 9. Principio de baseline

Todo estado reutilizable deberá tener un baseline conocido.

Formalmente:

```text
CurrentState(R)
   ↓
ResetPlan
   ↓
Baseline(R)
```

---

# 10. Baseline

Un baseline representa:

> El conjunto de condiciones que deben cumplirse para considerar un recurso apto para una nueva ejecución.

No significa necesariamente:

```text
factory-new object
```

Puede ser:

```text
known reusable state
```

---

# 11. Baseline types

```php
enum ResetBaselineType
{
    case APPLICATION;
    case WORKER;
    case EXECUTION;
    case CONNECTION;
    case CUSTOM;
}
```

---

# 12. Baseline descriptor

Conceptualmente:

```php
final readonly class ResetBaseline
{
    public function __construct(
        public ResetBaselineId $id,
        public ResetBaselineType $type,
        public array $requirements,
        public BaselineGeneration $generation,
    ) {}
}
```

---

# 13. Resettable state classification

No todo estado debe resetearse.

Clasificación:

```php
enum ResetStrategy
{
    case NONE;
    case CLEAR;
    case RESTORE_BASELINE;
    case ROLLBACK;
    case CLOSE;
    case RELEASE;
    case DISCARD;
    case RECREATE;
    case CUSTOM;
}
```

---

# 14. Immutable state

Estado verdaderamente inmutable:

```text
metadata
type definitions
dialect definitions
compiler rules
```

normalmente:

```text
ResetStrategy::NONE
```

---

# 15. Scoped state

Estado perteneciente exclusivamente al scope:

```text
EntityManager
IdentityMap
UnitOfWork
DatabaseContext
```

normalmente deberá:

```text
close / clear / release
```

y no reutilizarse entre executions salvo diseño explícito.

---

# 16. Worker reusable state

Ejemplos:

```text
connection pool
metadata cache
compiled query cache
telemetry exporter
```

pueden sobrevivir.

Pero cualquier fragmento contextual interno deberá limpiarse.

---

# 17. Physical resource state

Ejemplos:

```text
physical DB connection
driver handle
prepared statement handle
```

requieren reglas especiales antes de reuse.

---

# 18. Reset scope

Se define:

```text
ResetScope
```

como el conjunto de componentes cuyo estado deberá finalizarse/restaurarse.

---

# 19. Reset reasons

```php
enum ResetReason
{
    case EXECUTION_COMPLETED;
    case EXECUTION_FAILED;
    case EXECUTION_CANCELLED;
    case TRANSACTION_COMPLETED;
    case TRANSACTION_FAILED;
    case RESOURCE_RELEASED;
    case WORKER_RECYCLE;
    case TEST_TEARDOWN;
    case MANUAL;
}
```

---

# 20. Reset context

```php
final readonly class DatabaseResetContext
{
    public function __construct(
        public ResetReason $reason,
        public DatabaseScopeId $scopeId,
        public ScopeGeneration $generation,
        public ResetPolicy $policy,
    ) {}
}
```

---

# 21. Reset lifecycle

Modelo:

```text
DISCOVER
   ↓
PLAN
   ↓
QUIESCE
   ↓
RESET
   ↓
VERIFY
   ↓
REUSE / QUARANTINE / DISCARD
```

---

# 22. Discovery

El sistema identifica qué estado necesita reset.

---

# 23. Planning

Construye:

```text
DatabaseResetPlan
```

con pasos ordenados.

---

# 24. Quiesce

Antes del reset deberá impedirse nueva actividad sobre el recurso.

Ejemplo:

```text
Scope
ACTIVE
  ↓
CLOSING
```

Nuevas queries:

```text
rejected
```

---

# 25. Reset execution

Cada componente ejecuta su estrategia.

---

# 26. Verification

No basta con llamar:

```php
$component->reset();
```

Debe existir evidencia del resultado.

---

# 27. Final decision

Después de verificar:

```text
CLEAN
DIRTY
UNKNOWN
TAINTED
```

---

# 28. Reset states

```php
enum ResetStatus
{
    case NOT_REQUIRED;
    case PENDING;
    case RUNNING;
    case CLEAN;
    case PARTIAL;
    case FAILED;
    case UNKNOWN;
}
```

---

# 29. UNKNOWN ≠ CLEAN

Invariante:

```text
UNKNOWN
≠
CLEAN
```

---

# 30. PARTIAL ≠ CLEAN

Igualmente:

```text
PARTIAL
≠
CLEAN
```

salvo que la parte no limpiada haya sido demostrada irrelevante para reuse.

---

# 31. Reset plan

Conceptualmente:

```php
final readonly class DatabaseResetPlan
{
    /**
     * @param list<ResetStep> $steps
     */
    public function __construct(
        public ResetPlanId $id,
        public array $steps,
        public ResetPolicy $policy,
    ) {}
}
```

---

# 32. Reset step

```php
interface ResetStep
{
    public function execute(
        DatabaseResetContext $context
    ): ResetStepResult;
}
```

---

# 33. Reset order matters

El reset deberá ser ordenado.

Ejemplo incorrecto:

```text
release connection
    ↓
rollback transaction
```

La conexión ya podría estar siendo usada por otro request.

---

# 34. Orden general

Modelo recomendado:

```text
Stop new DB operations
        ↓
Cancel/wait active operations
        ↓
Close streams/cursors
        ↓
Finalize lazy/chunk/import/export resources
        ↓
Resolve transaction
        ↓
Release ORM state
        ↓
Reset routing/context state
        ↓
Reset connection session
        ↓
Release physical leases
        ↓
Clear diagnostics/local telemetry
        ↓
Verify
```

---

# 35. Dependency graph

Reset ordering deberá modelarse como DAG cuando sea necesario.

Ejemplo:

```text
Cursor
  ↓
Statement
  ↓
Transaction
  ↓
Connection Lease
  ↓
Physical Connection
```

---

# 36. Reverse ownership order

Como regla general:

> Los recursos hijos deberán finalizarse antes que sus owners.

---

# 37. Execution state machine

```text
ACTIVE
   ↓
CLOSING
   ↓
RESETTING
   ↓
CLOSED
```

Alternativas:

```text
RESETTING
   ↓
TAINTED
```

---

# 38. No new work during reset

Una vez:

```text
CLOSING
```

el scope no aceptará:

```text
new query
new transaction
new cursor
new lazy traversal
new persistence operation
```

---

# 39. In-flight operations

El sistema deberá decidir:

```text
WAIT
CANCEL
FAIL_RESET
```

según política.

---

# 40. Reset timeout

El reset no podrá bloquear indefinidamente.

```php
final readonly class ResetBudget
{
    public function __construct(
        public Duration $timeout,
    ) {}
}
```

---

# 41. Reset timeout outcome

Si el reset excede el budget:

```text
UNKNOWN
```

o:

```text
FAILED
```

según evidencia.

El recurso afectado no será reutilizado.

---

# 42. DatabaseContext reset

El `DatabaseContext` no debería mutarse para convertir:

```text
Context A
```

en:

```text
Context B
```

Preferencia:

```text
close A
create B
```

---

# 43. Context reset

El reset eliminará la referencia current del runtime-local storage.

---

# 44. Context after close

Acceder a:

```text
DatabaseContext A
```

como current context después de finalizar A deberá fallar.

---

# 45. Tenant context reset

Deberá eliminar:

```text
current tenant
tenant routing state
tenant-specific overrides
```

del execution storage.

---

# 46. No default previous tenant

Después del reset:

```text
tenant = none/unresolved
```

no:

```text
tenant = previous tenant
```

---

# 47. Shard context reset

Misma regla:

```text
shard = unresolved
```

hasta que una nueva ejecución lo determine.

---

# 48. Security context reset

Database security context deberá eliminar:

```text
actor
authorization scope
privilege elevation
data access filters
```

---

# 49. Elevated privilege reset

Especialmente:

```text
elevated DB privilege
```

nunca deberá sobrevivir al scope.

---

# 50. Routing state reset

Limpiar:

```text
sticky writer
selected replica
endpoint affinity
minimum replication position
temporary failover decision
```

según lifetime.

---

# 51. Topology state distinction

No deberá eliminarse necesariamente:

```text
worker-global topology snapshot
```

si es shared y válido.

---

# 52. Request routing ≠ topology metadata

```text
RoutingDecision
≠
TopologyKnowledge
```

---

# 53. Sticky writer reset

Al terminar request:

```text
sticky writer = false/unset
```

para el siguiente request.

---

# 54. EntityManager reset

El EntityManager deberá finalizarse explícitamente.

---

# 55. Preferred EntityManager policy

Para runtimes persistentes:

```text
one EntityManager
per execution scope
```

---

# 56. EntityManager close

Al finalizar:

```text
OPEN
 ↓
CLOSING
 ↓
CLOSED
```

---

# 57. EntityManager reuse

Default:

```text
EntityManager reuse across executions = false
```

---

# 58. Why not recycle EntityManager

Porque contiene/coordina:

```text
IdentityMap
UnitOfWork
managed state
lifecycle state
```

y recrearlo es más seguro que intentar reconstruir su semántica completa.

---

# 59. IdentityMap reset

Al cerrar EntityManager:

```text
IdentityMap.clear()
```

---

# 60. IdentityMap clear semantics

Eliminará:

```text
entity identity references
reserved hydration identities
managed instance references
```

---

# 61. IdentityMap clear ≠ entity destruction

Las entidades PHP pueden seguir existiendo si application code mantiene referencias.

Pasarán a estar conceptualmente:

```text
DETACHED
```

---

# 62. UnitOfWork reset

Deberá eliminar:

```text
new entities
dirty entities
removed entities
change sets
snapshots
scheduled relationship operations
persistence plan state
```

---

# 63. Pending UoW changes

No deberán descartarse silenciosamente durante una ejecución normal.

Pero al finalizar un scope ya fallido/cancelado:

```text
discard scoped UoW
```

será parte de cleanup.

---

# 64. Warning in development

Si un scope termina exitosamente con cambios UoW pendientes no flushados, tooling podrá advertir:

```text
PendingUnitOfWorkAtScopeClose
```

---

# 65. Flush ≠ reset

`flush()` no limpia automáticamente todo el UoW.

---

# 66. Clear ≠ commit

`clear()` tampoco confirma DB transaction.

---

# 67. Transaction reset

Antes de reutilizar connection state:

```text
no active transaction
```

deberá ser una precondición.

---

# 68. Active transaction at scope end

Si existe:

```text
ACTIVE transaction
```

por default VoltStack intentará:

```text
ROLLBACK
```

si posee la transaction.

---

# 69. Transaction ownership

Si la transacción pertenece a un owner externo explícito, el lifecycle deberá respetar ese ownership.

En un execution root normal no debería sobrevivir al scope.

---

# 70. Rollback success

Si rollback es confirmado:

```text
transaction state = ROLLED_BACK
```

---

# 71. Rollback failure

Si falla antes de conocerse resultado:

```text
UNKNOWN
```

---

# 72. Unknown transaction state

La conexión física deberá:

```text
QUARANTINE
or
DISCARD
```

---

# 73. Commit UNKNOWN

Si previamente existió:

```text
COMMIT sent
+
connection lost
```

reset no podrá convertirlo en:

```text
ROLLED_BACK
```

---

# 74. Reset does not rewrite history

Regla:

> El State Reset System jamás fabricará un resultado transaccional conocido para facilitar reutilización.

---

# 75. Savepoint reset

Savepoint stack se descarta junto con TransactionContext.

---

# 76. Transaction events

Queues de:

```text
afterCommit
afterRollback
```

deberán finalizar según outcome real.

---

# 77. Deferred event reset

No deberán sobrevivir a otra transaction.

---

# 78. Retry state reset

Limpiar:

```text
attempt count
last retryable error
backoff state
deadlock evidence
```

al terminar la operación/transaction.

---

# 79. Query execution state reset

Cada operación deberá liberar:

```text
parameter bindings
temporary execution buffers
execution context
timeout handles
cancellation registrations
```

---

# 80. Query Builder

Un Query Builder inmutable no requiere reset.

Uno mutable no deberá convertirse en worker-global.

---

# 81. Query compiler

Compiler deberá ser stateless/immutable.

Por tanto:

```text
reset = none
```

para su definición compartida.

---

# 82. Compilation context

El contexto temporal de compilación deberá descartarse al finalizar la operación.

---

# 83. Planner context

Igualmente.

---

# 84. Optimizer context

Igualmente.

---

# 85. Hydration reset

Una hydration session deberá liberar:

```text
row state
assembly registries
join deduplication state
reservation state
temporary references
```

---

# 86. Hydration plan cache

No deberá limpiarse por request si contiene sólo planes inmutables correctamente generacionados.

---

# 87. Relationship loader reset

Limpiar:

```text
pending batches
relationship observations
temporary coverage assembly
```

---

# 88. Lazy Collection reset

Una active lazy iteration deberá:

```text
close source
close cursor
release lease if owned
discard iteration state
```

---

# 89. Lazy definition

Una definición inmutable no necesita reset.

---

# 90. Chunk reset

Al finalizar traversal:

```text
release current chunk
release continuation state
release operation-local checkpoint candidate
close owned resources
```

---

# 91. Durable checkpoint

Un checkpoint ya persistido externamente no se borra simplemente por reset.

---

# 92. Import reset

Deberá cerrar:

```text
input streams
temporary buffers
batch state
error collectors
owned transaction
owned DB resources
```

---

# 93. Export reset

Deberá cerrar:

```text
DB cursor
output resource if owned
temporary encoding buffers
operation state
```

---

# 94. Ownership matters for external streams

Si el caller proporcionó el output stream:

```text
ExportSystem
```

no deberá cerrarlo automáticamente salvo contrato explícito.

---

# 95. Cursor reset

Todo `ResultCursor` activo deberá cerrarse antes de liberar su connection lease.

---

# 96. Cursor close failure

Si no puede demostrarse que el driver quedó en estado reutilizable:

```text
connection = quarantine/discard
```

---

# 97. StreamingResult reset

Misma regla.

---

# 98. Prepared statement reset

Distinguir:

```text
compiled statement description
```

de:

```text
live driver prepared statement
```

---

# 99. Live prepared handles

Deberán cerrarse/liberarse cuando su lifetime termine si el driver lo requiere.

---

# 100. Statement cache

Un statement cache connection-local podrá sobrevivir sólo si:

```text
driver/platform supports it
+
session state remains compatible
+
policy permits it
```

---

# 101. Connection reset

Es la parte más delicada del sistema.

---

# 102. Connection baseline

Una conexión reusable deberá satisfacer, como mínimo:

```text
no active transaction
no active cursor
no active stream
no invalid driver state
known session configuration
known authorization role
known schema/search path
known timezone policy
no leaked temporary state
no leaked advisory locks
```

según plataforma.

---

# 103. Connection reset plan

```text
ConnectionLease release
        ↓
detect active resources
        ↓
close cursors/statements
        ↓
resolve transaction
        ↓
restore session state
        ↓
release locks/temp state
        ↓
driver reset
        ↓
ping/validate if policy requires
        ↓
verify
        ↓
pool/discard
```

---

# 104. Connection state categories

```php
enum ConnectionResetState
{
    case CLEAN;
    case DIRTY_KNOWN;
    case DIRTY_UNKNOWN;
    case BROKEN;
}
```

---

# 105. CLEAN

Puede regresar al pool.

---

# 106. DIRTY_KNOWN

Existe contaminación conocida y existe un reset plan seguro.

---

# 107. DIRTY_UNKNOWN

No se conoce completamente el estado.

Default:

```text
discard
```

---

# 108. BROKEN

Conexión no operativa.

```text
discard
```

---

# 109. Session state registry

VoltStack podrá registrar cambios conocidos de session state.

Ejemplo:

```text
SET TIME ZONE
SET ROLE
SET search_path
SET SESSION ...
```

---

# 110. Known mutation tracking

Si VoltStack ejecutó la mutación, podrá conocer qué restaurar.

---

# 111. Raw SQL problem

Raw SQL podría modificar session state sin informar al framework.

Ejemplo:

```sql
SET ROLE admin;
```

---

# 112. Raw session mutation policy

VoltStack deberá definir:

```text
ALLOW_TRACKED_ONLY
ALLOW_AND_FORCE_FULL_RESET
MARK_DIRTY
FORBID
```

según contexto.

---

# 113. Escape hatch consequences

> Utilizar raw SQL para modificar estado de sesión puede reducir la capacidad de VoltStack para demostrar que una conexión es reusable.

---

# 114. Session variables

Deberán restaurarse al baseline conocido.

---

# 115. Timezone

Si VoltStack define timezone de sesión:

```text
reset → configured baseline timezone
```

no necesariamente server default.

---

# 116. SQL mode

MySQL/MariaDB pueden requerir baseline específico.

---

# 117. Search path

PostgreSQL puede requerir restaurar:

```text
search_path
```

---

# 118. Role

PostgreSQL:

```text
SET ROLE
```

o cambios equivalentes deberán revertirse.

---

# 119. Temporary tables

Las tablas temporales pueden sobrevivir durante la sesión.

Por tanto requieren estrategia.

---

# 120. Temporary state policy

Opciones:

```text
TRACK_AND_DROP
FULL_SESSION_RESET
DISCARD_CONNECTION
```

---

# 121. Advisory locks

Locks session-level deberán liberarse.

---

# 122. Unknown advisory lock state

Si no puede verificarse:

```text
discard connection
```

cuando represente riesgo.

---

# 123. User variables

MySQL/MariaDB session variables deberán contemplarse.

---

# 124. PostgreSQL session reset

Cuando capabilities lo permitan, podrán utilizarse primitivas de plataforma equivalentes a reset global de session state.

Pero:

```text
Platform Reset Command
≠
Universal Guarantee
```

VoltStack deberá conocer qué cubre realmente.

---

# 125. MySQL/MariaDB reset

Deberá usar capacidades reales del driver/plataforma y no asumir que cerrar un statement limpia todo el estado de sesión.

---

# 126. SQLite reset

SQLite tiene modelo diferente.

Particularmente:

```text
connection
transaction
PRAGMA state
temporary objects
in-memory DB lifetime
```

---

# 127. SQLite :memory:

Para:

```text
:memory:
```

recrear conexión puede destruir toda la base.

Por tanto:

```text
DISCARD
```

no siempre es semánticamente equivalente a reset.

---

# 128. SQLite test policy

El Test Environment System podrá definir un baseline específico para SQLite in-memory.

---

# 129. Driver capabilities

Se usarán capabilities, no vendor conditionals dispersos.

Ejemplos conceptuales:

```php
interface ConnectionResetCapabilities
{
    public function supportsSessionReset(): bool;

    public function supportsTransactionStateInspection(): bool;

    public function supportsRoleReset(): bool;

    public function supportsTemporaryStateReset(): bool;

    public function supportsAdvisoryLockReset(): bool;
}
```

---

# 130. Version ≠ capability

No:

```php
if ($version >= ...)
```

como arquitectura principal.

Sí:

```php
$capabilities->supportsSessionReset();
```

---

# 131. MySQL ≠ MariaDB

VoltStack mantendrá ambas plataformas como first-class independientes.

---

# 132. Reset compiler

Si una plataforma requiere comandos SQL para reset:

```text
Reset Planner
     ↓
Platform Reset Compiler
     ↓
Query/Execution Engine
```

El State Reset System no concatenará SQL directamente.

---

# 133. ConnectionResetPlan

```php
final readonly class ConnectionResetPlan
{
    /**
     * @param list<ConnectionResetOperation> $operations
     */
    public function __construct(
        public array $operations,
        public ResetVerificationPolicy $verification,
    ) {}
}
```

---

# 134. Reset operation examples

```text
ROLLBACK_ACTIVE_TRANSACTION
CLOSE_OPEN_CURSOR
RESET_ROLE
RESET_TIMEZONE
RESET_SEARCH_PATH
RESET_SESSION_VARIABLES
DROP_TEMPORARY_STATE
RELEASE_ADVISORY_LOCKS
RESET_DRIVER_SESSION
VALIDATE_CONNECTION
```

---

# 135. Connection reset verification

Después del reset deberán verificarse aquellas propiedades necesarias para reuse.

---

# 136. Verification ≠ full introspection

No siempre será posible consultar todo el estado del DBMS.

Por eso la seguridad puede depender de:

```text
known mutation tracking
+
platform reset guarantees
+
driver guarantees
+
verification
```

---

# 137. Proof model

```text
Reusable
=
KnownInitialBaseline
∧
TrackedMutations
∧
SuccessfulReset
∧
RequiredVerification
∧
NoUnknownOutcome
```

---

# 138. Full reset

Cuando la plataforma ofrece una operación fuerte de reset, VoltStack podrá preferirla.

---

# 139. Incremental reset

Puede ser más rápido:

```text
restore only changed state
```

pero exige tracking confiable.

---

# 140. Hybrid reset

Modelo recomendado:

```text
tracked incremental cleanup
+
platform reset where useful
+
verification
```

---

# 141. Reset performance

Reset demasiado costoso puede eliminar beneficios del pooling.

Por ello deberá medirse.

---

# 142. Performance never overrides correctness

```text
fast unsafe reuse
```

no será una opción válida.

---

# 143. Connection reuse threshold

Si reset cuesta más que recrear conexión:

```text
discard + reconnect
```

puede ser preferible.

---

# 144. Cost model

Conceptualmente:

```text
Cost(reset)
vs
Cost(reconnect)
```

podrá informar políticas.

Pero no disminuir garantías.

---

# 145. Reset idempotency

Los pasos deberían ser idempotentes cuando sea posible.

---

# 146. Idempotent example

```text
close already closed cursor
```

puede devolver:

```text
already clean
```

---

# 147. Non-idempotent reset

Si una operación no es segura de repetir, deberá registrarlo.

---

# 148. Reset retry

No se reintentará ciegamente.

---

# 149. Reset retry policy

Sólo si:

```text
operation retry-safe
+
outcome known
```

---

# 150. Unknown reset outcome

No:

```text
retry until success
```

si el efecto previo es incierto.

Preferir:

```text
discard resource
```

---

# 151. Reset failures

Jerarquía:

```text
DatabaseStateResetException
├── ResetPlanningException
├── ResetExecutionException
├── ResetVerificationException
├── ResetTimeoutException
├── ResetDependencyException
├── EntityManagerResetException
├── TransactionResetException
├── ConnectionResetException
├── UnknownConnectionStateException
├── ResourceCloseException
└── WorkerResetException
```

---

# 152. Failure containment

Reset failure deberá intentar limitar daño.

Ejemplo:

```text
connection reset failed
     ↓
discard connection
     ↓
worker remains healthy
```

---

# 153. Worker taint

Si el failure afecta estado global desconocido:

```text
worker = TAINTED
```

---

# 154. Worker reset

No todo worker state debe limpiarse entre requests.

---

# 155. Worker persistent state

Puede mantenerse:

```text
compiled metadata
type registry
dialect registry
bounded caches
connection pool
telemetry exporter
```

---

# 156. Worker request state

Debe desaparecer:

```text
current DatabaseContext
current tenant
current EntityManager
current transaction
current profiler session
current cancellation token
```

---

# 157. Worker reset boundary

Después de cada execution:

```text
Execution Finalization
      ↓
Database State Reset
      ↓
Isolation Verification
      ↓
Worker Reusable
```

---

# 158. Worker reusable states

```php
enum WorkerDatabaseState
{
    case READY;
    case BUSY;
    case RESETTING;
    case TAINTED;
    case RECYCLE_REQUIRED;
}
```

---

# 159. Worker reset failure

Si no puede garantizarse baseline:

```text
RECYCLE_REQUIRED
```

---

# 160. Worker recycle

El runtime adapter deberá poder solicitar:

```text
graceful worker recycle
```

cuando sea soportado.

---

# 161. FrankenPHP

El reset deberá funcionar correctamente bajo workers persistentes de FrankenPHP.

---

# 162. FrankenPHP principle

VoltStack no deberá depender de:

```text
PHP process termination
```

para limpiar estado Database.

---

# 163. RoadRunner

Cada request/job procesado por worker deberá ejecutar finalization/reset aunque falle la aplicación.

---

# 164. OpenSwoole

La limpieza deberá ser coroutine-aware.

---

# 165. Coroutine reset

No deberá limpiar accidentalmente estado perteneciente a otra coroutine activa.

---

# 166. Scope-targeted reset

Por ello:

```text
reset(scope A)
```

no significa:

```text
clear all runtime locals
```

si B está ejecutándose concurrentemente.

---

# 167. Fiber reset

Misma regla para Fibers.

---

# 168. Runtime adapter

Arquitectura:

```text
Database Core
    │
    ├── Reset Contracts
    │
    └── Scope Lifecycle
            ↓
Runtime Adapter
├── Traditional PHP
├── FrankenPHP
├── RoadRunner
└── OpenSwoole
```

---

# 169. Reset registry

Se propone:

```text
DatabaseResetRegistry
```

para registrar componentes que participan en reset.

---

# 170. Reset participant

```php
interface DatabaseResetParticipant
{
    public function reset(
        DatabaseResetContext $context
    ): ResetResult;
}
```

---

# 171. Ordered participant

```php
interface OrderedResetParticipant extends DatabaseResetParticipant
{
    public function dependencies(): array;
}
```

---

# 172. Registration

Ejemplos:

```text
TransactionResetParticipant
CursorResetParticipant
OrmResetParticipant
RoutingResetParticipant
TelemetryResetParticipant
ConnectionResetParticipant
```

---

# 173. Dependency resolution

No depender únicamente de:

```text
priority = 100
```

para relaciones críticas.

Preferir dependencias semánticas:

```text
CursorReset
before
ConnectionRelease
```

---

# 174. Cyclic reset dependency

Deberá detectarse durante bootstrap/testing.

---

# 175. Reset registry freeze

Después de bootstrap:

```text
registry.freeze()
```

en runtimes persistentes.

---

# 176. Extension safety

Extensiones Database podrán registrar reset participants.

Pero deberán declarar:

```text
state lifetime
dependencies
failure severity
verification strategy
```

---

# 177. Plugin requirement

Un plugin que mantenga execution state pero no implemente cleanup correcto deberá considerarse incompatible con persistent runtime.

---

# 178. Persistent-runtime capability

Extensiones podrán declarar:

```text
PERSISTENT_RUNTIME_SAFE
PERSISTENT_RUNTIME_UNSAFE
UNKNOWN
```

---

# 179. UNKNOWN extension

En strict mode:

```text
UNKNOWN
→ reject or warn strongly
```

---

# 180. Reset and caches

No hacer:

```text
flush all caches after every request
```

---

# 181. Cache reset by lifetime

L0:

```text
clear
```

L1:

```text
retain if safe
```

L2:

```text
unaffected
```

salvo invalidación semántica.

---

# 182. Reset ≠ Cache invalidation

Son problemas distintos.

```text
Reset
→ lifecycle cleanliness

Invalidation
→ cached data correctness
```

---

# 183. Entity cache

No debería contener managed objects; por tanto no requiere limpiar todas sus entradas al cerrar EntityManager.

---

# 184. Metadata cache

No se limpia por request.

---

# 185. Query compiled cache

No se limpia por request salvo generación/version invalidada.

---

# 186. Telemetry reset

Debe finalizar:

```text
query profiler session
N+1 observation state
request counters
trace-local buffers
debug information
```

---

# 187. Exporter lifetime

Telemetry exporter puede ser worker/application shared.

No se destruye por request.

---

# 188. Profiler reset

Las queries de A no aparecerán en B.

---

# 189. Debug toolbar reset

Los datos de toolbar serán execution-bound.

---

# 190. Audit reset

Correlation state será limpiado.

Los registros ya persistidos no se eliminan.

---

# 191. Event reset

Limpiar:

```text
current event stack
deferred scoped events
temporary listener state
```

---

# 192. Event listeners

Listeners singleton deberán ser stateless o worker-safe.

---

# 193. Cancellation reset

Eliminar:

```text
CancellationToken
cancellation registrations
timeout callbacks
```

del scope.

---

# 194. Timer cleanup

Timers vinculados a query/operation deberán cancelarse.

---

# 195. Deadline reset

Deadline del request anterior no podrá heredarse.

---

# 196. Resource permits

Permits de Resource Governance deberán liberarse.

---

# 197. Permit leak

Si:

```text
permit acquired
```

pero no liberado:

```text
resource accounting corruption
```

puede ocurrir.

---

# 198. Reset must release owned permits

Esto deberá verificarse.

---

# 199. Memory reset

Reset deberá eliminar referencias scoped que impidan GC.

---

# 200. Reset ≠ immediate memory return

PHP puede no devolver memoria inmediatamente al sistema operativo.

---

# 201. Memory baseline

Lo importante es evitar:

```text
unbounded retained references
```

---

# 202. Leak detection

Después del reset:

```text
ScopedStrongReferences
```

deberían caer a cero para componentes controlados.

---

# 203. Weak references

Debug tooling podrá usar `WeakReference` cuando sea útil para detectar objetos scoped retenidos.

---

# 204. Garbage collection

El sistema no deberá depender exclusivamente de:

```php
gc_collect_cycles();
```

para correctness.

---

# 205. GC ≠ Reset

GC libera memoria inalcanzable.

Reset establece semántica de lifecycle.

---

# 206. Reset diagnostics

Se generará:

```text
DatabaseResetReport
```

---

# 207. Reset report

Conceptualmente:

```php
final readonly class DatabaseResetReport
{
    public function __construct(
        public ResetStatus $status,
        public array $steps,
        public array $warnings,
        public array $quarantinedResources,
        public bool $workerRecycleRequired,
    ) {}
}
```

---

# 208. Debug output

Ejemplo:

```text
Database Reset
────────────────────────────
Scope: dbscope-481
Reason: EXECUTION_COMPLETED

Cursor cleanup       CLEAN
Transaction          CLEAN
EntityManager        CLEAN
IdentityMap          CLEAN
UnitOfWork           CLEAN
Routing state        CLEAN
Telemetry state      CLEAN
Connection session   CLEAN
Lease release        CLEAN

Final:
CLEAN

Worker reusable:
YES
```

---

# 209. Failure output

```text
Database Reset
────────────────────────────
Connection session:
UNKNOWN

Reason:
driver disconnected during reset

Action:
connection discarded

Worker:
CLEAN

Final:
CLEAN_WITH_RESOURCE_DISCARD
```

---

# 210. Worker failure output

```text
Database Reset
────────────────────────────
Runtime-local state:
UNKNOWN

Action:
worker recycle required

Worker reusable:
NO
```

---

# 211. Telemetry

Eventos:

```text
database.reset.started
database.reset.completed
database.reset.failed
database.reset.timeout
database.reset.resource_discarded
database.reset.resource_quarantined
database.reset.worker_recycle_requested
```

---

# 212. Metrics

Ejemplos:

```text
database_reset_total
database_reset_failures_total
database_reset_duration
database_connection_reset_total
database_connection_discard_total
database_worker_recycle_total
```

---

# 213. Cardinality

No usar IDs únicos como metric labels.

---

# 214. Tracing

Reset podrá aparecer como span:

```text
database.reset
```

con subspans agregados.

---

# 215. Security

Logs de reset no deberán incluir:

```text
password
DSN secret
raw credential
sensitive query parameters
PII
```

---

# 216. Testing architecture

El State Reset System requerirá pruebas unitarias, integración y soak.

---

# 217. Basic reset test

```text
create scope
mutate scoped state
close scope
verify baseline
```

---

# 218. EntityManager test

Después del reset:

```text
EntityManager A = CLOSED
```

y no puede ejecutar operaciones.

---

# 219. IdentityMap test

Después del reset:

```text
IdentityMap A = empty/unusable
```

---

# 220. UnitOfWork test

No quedan scheduled operations.

---

# 221. Tenant reset test

```text
Scope A → tenant ACME
reset
Scope B → no tenant until resolution
```

---

# 222. Sticky routing test

```text
Scope A → write → sticky
reset
Scope B → sticky false
```

---

# 223. Transaction rollback test

Scope termina con transaction activa.

Verificar rollback.

---

# 224. Rollback failure test

Simular pérdida de conexión.

Esperar:

```text
connection discarded
```

---

# 225. Cursor leak test

Scope termina con cursor abierto.

Reset deberá cerrarlo antes de release.

---

# 226. Cursor close failure test

Resultado:

```text
connection quarantine/discard
```

---

# 227. Session role test

A:

```text
role = privileged
```

Reset.

B:

```text
baseline role
```

---

# 228. Timezone test

A modifica timezone.

B recibe baseline.

---

# 229. Search path test

PostgreSQL:

```text
A changes search_path
reset
B baseline search_path
```

---

# 230. Temporary state test

Crear temporary table/state.

Reset.

Verificar ausencia o descarte de conexión.

---

# 231. Advisory lock test

Adquirir session lock.

Reset.

Verificar liberación o descarte.

---

# 232. Raw SQL dirty state test

Modificar estado mediante escape hatch.

Comprobar política:

```text
full reset
or
discard
```

---

# 233. Cancellation test

A cancela operación.

Reset.

B no hereda cancellation.

---

# 234. Deadline test

A deadline expirado.

B recibe deadline propio.

---

# 235. Telemetry reset test

Profile A no aparece en B.

---

# 236. N+1 reset test

Observations A no aparecen en B.

---

# 237. Event reset test

Deferred events de A no se ejecutan en transaction B.

---

# 238. Permit reset test

Resource permits vuelven al accounting correcto.

---

# 239. Runtime-local reset test

`currentScope()` después de close:

```text
none
```

---

# 240. Coroutine test

Dos coroutines:

```text
A
B
```

resetear A no deberá borrar state B.

---

# 241. Fiber test

Misma garantía.

---

# 242. Long worker test

Ejecutar decenas o cientos de miles de scopes.

Verificar:

```text
no monotonic state growth
no stale context
no stale EntityManager
no leaked transaction
no leaked connection lease
```

---

# 243. Fault injection

Inyectar failures en cada reset step.

---

# 244. Reset step matrix

Para cada step:

```text
success
known failure
timeout
unknown outcome
exception
process interruption
```

---

# 245. Property-based lifecycle tests

Generar secuencias:

```text
begin
query
cursor
commit
rollback
lazy
cancel
close
```

y comprobar invariantes.

---

# 246. Reset idempotency tests

Cuando un step se declare idempotente:

```text
reset(reset(state))
=
reset(state)
```

---

# 247. Pool integration test

Una conexión sólo regresa al pool si:

```text
ConnectionResetResult = CLEAN
```

---

# 248. Strict test mode

VoltStack Testing podrá activar:

```text
fail on leaked cursor
fail on leaked transaction
fail on pending UoW
fail on unknown reset state
fail on scoped reference leak
```

---

# 249. Production policy

En producción se privilegiará containment:

```text
uncertain resource
→ discard

uncertain worker state
→ recycle
```

---

# 250. Development policy

Además podrá:

```text
throw
capture stack trace
show Debug Toolbar warning
```

---

# 251. Reset API

API interna conceptual:

```php
$result = $resetManager->reset(
    scope: $scope,
    reason: ResetReason::EXECUTION_COMPLETED,
);
```

---

# 252. Reset Manager

```php
interface DatabaseStateResetManager
{
    public function reset(
        DatabaseExecutionScope $scope,
        ResetReason $reason,
    ): DatabaseResetReport;
}
```

---

# 253. Reset Manager responsibilities

Debe:

```text
build plan
order participants
execute steps
collect evidence
verify baseline
decide containment
produce report
```

---

# 254. Reset Manager must not

No deberá:

```text
generate arbitrary SQL itself
commit business transactions
hide unknown outcomes
swallow critical failures
```

---

# 255. Directory Structure

Propuesta:

```text
src/Quantum/Database/Runtime/Reset/
│
├── Contract/
│   ├── DatabaseStateResetManager.php
│   ├── DatabaseResetParticipant.php
│   ├── OrderedResetParticipant.php
│   ├── ResetVerifier.php
│   └── ResetPolicy.php
│
├── Model/
│   ├── DatabaseResetContext.php
│   ├── DatabaseResetPlan.php
│   ├── ResetStep.php
│   ├── ResetStepResult.php
│   ├── ResetResult.php
│   ├── ResetStatus.php
│   ├── ResetReason.php
│   ├── ResetStrategy.php
│   ├── ResetBaseline.php
│   └── DatabaseResetReport.php
│
├── Manager/
│   └── DefaultDatabaseStateResetManager.php
│
├── Registry/
│   └── DatabaseResetRegistry.php
│
├── Planner/
│   ├── DatabaseResetPlanner.php
│   └── ResetDependencyResolver.php
│
├── Participant/
│   ├── CursorResetParticipant.php
│   ├── StreamingResetParticipant.php
│   ├── LazyResetParticipant.php
│   ├── TransactionResetParticipant.php
│   ├── OrmResetParticipant.php
│   ├── RoutingResetParticipant.php
│   ├── ContextResetParticipant.php
│   ├── TelemetryResetParticipant.php
│   ├── ResourcePermitResetParticipant.php
│   └── ConnectionResetParticipant.php
│
├── Connection/
│   ├── ConnectionResetPlanner.php
│   ├── ConnectionResetPlan.php
│   ├── ConnectionResetOperation.php
│   ├── ConnectionResetVerifier.php
│   ├── ConnectionResetCapabilities.php
│   └── ConnectionResetState.php
│
├── Verification/
│   ├── DefaultResetVerifier.php
│   ├── ScopeBaselineVerifier.php
│   └── WorkerBaselineVerifier.php
│
├── Diagnostics/
│   ├── ResetDiagnostics.php
│   └── ResetReportRenderer.php
│
└── Exception/
    ├── DatabaseStateResetException.php
    ├── ResetPlanningException.php
    ├── ResetExecutionException.php
    ├── ResetVerificationException.php
    ├── ResetTimeoutException.php
    ├── EntityManagerResetException.php
    ├── TransactionResetException.php
    ├── ConnectionResetException.php
    ├── UnknownConnectionStateException.php
    ├── ResourceCloseException.php
    └── WorkerResetException.php
```

---

# 256. Architectural Invariants

## DB-RESET-001

Isolation será el mecanismo primario; reset será complementario.

## DB-RESET-002

Todo recurso reutilizable tendrá baseline explícito.

## DB-RESET-003

Reusable implica ResetVerified.

## DB-RESET-004

UNKNOWN no equivaldrá a CLEAN.

## DB-RESET-005

PARTIAL no equivaldrá automáticamente a CLEAN.

## DB-RESET-006

Reset no equivaldrá a destruction.

## DB-RESET-007

Reset no equivaldrá a rollback.

## DB-RESET-008

Rollback no equivaldrá a object graph rewind.

## DB-RESET-009

Reset tendrá lifecycle explícito.

## DB-RESET-010

Reset tendrá budget.

## DB-RESET-011

No se aceptará nuevo trabajo durante reset.

## DB-RESET-012

Recursos hijos cerrarán antes que sus owners.

## DB-RESET-013

Reset ordering será determinista.

## DB-RESET-014

Critical ordering no dependerá sólo de prioridades numéricas.

## DB-RESET-015

DatabaseContext viejo no se reciclará como contexto nuevo.

## DB-RESET-016

Current context se eliminará del runtime-local storage.

## DB-RESET-017

Tenant state no sobrevivirá execution boundary.

## DB-RESET-018

Shard state no sobrevivirá execution boundary.

## DB-RESET-019

Security state no sobrevivirá execution boundary.

## DB-RESET-020

Elevated privilege no sobrevivirá execution boundary.

## DB-RESET-021

Sticky writer state no sobrevivirá execution boundary.

## DB-RESET-022

Replica affinity scoped no sobrevivirá execution boundary.

## DB-RESET-023

EntityManager será cerrado al finalizar scope.

## DB-RESET-024

EntityManager no será reutilizado entre executions por default.

## DB-RESET-025

IdentityMap será limpiada.

## DB-RESET-026

Managed objects sobrevivientes quedarán detached.

## DB-RESET-027

UnitOfWork será limpiado.

## DB-RESET-028

Pending changes no se transferirán a otro scope.

## DB-RESET-029

Flush no equivaldrá a reset.

## DB-RESET-030

Clear no equivaldrá a commit.

## DB-RESET-031

Connection reusable no tendrá active transaction.

## DB-RESET-032

Owned active transaction al cierre deberá resolverse.

## DB-RESET-033

Unknown transaction outcome no será reescrito.

## DB-RESET-034

Unknown transaction connection no volverá al pool.

## DB-RESET-035

Savepoint state no cruzará transaction boundary.

## DB-RESET-036

Deferred transaction events no cruzarán transactions.

## DB-RESET-037

Retry state será limpiado.

## DB-RESET-038

Query bindings serán operation-local.

## DB-RESET-039

Compilation context será descartado.

## DB-RESET-040

Planner context será descartado.

## DB-RESET-041

Optimizer execution context será descartado.

## DB-RESET-042

Hydration session será descartada.

## DB-RESET-043

Hydration plan cache seguro podrá sobrevivir.

## DB-RESET-044

Relationship loading pending state será limpiado.

## DB-RESET-045

Active Lazy source será cerrado.

## DB-RESET-046

Active Chunk traversal será finalizado.

## DB-RESET-047

Durable checkpoint no se borrará por reset local.

## DB-RESET-048

Import resources owned serán cerrados.

## DB-RESET-049

Export resources owned serán cerrados.

## DB-RESET-050

Caller-owned external resource respetará ownership.

## DB-RESET-051

Cursor cerrará antes que lease release.

## DB-RESET-052

Streaming result cerrará antes que lease release.

## DB-RESET-053

Cursor close uncertainty podrá invalidar connection reuse.

## DB-RESET-054

Live prepared handles respetarán driver lifecycle.

## DB-RESET-055

Compiled query no requerirá request reset si es immutable.

## DB-RESET-056

Connection baseline será explícito.

## DB-RESET-057

Connection session state deberá ser conocido antes de reuse.

## DB-RESET-058

DIRTY_UNKNOWN connection no regresará al pool.

## DB-RESET-059

BROKEN connection no regresará al pool.

## DB-RESET-060

Raw session mutation reducirá reuse guarantees.

## DB-RESET-061

Raw SQL no podrá ocultar session-state risk.

## DB-RESET-062

Timezone será restaurada al baseline configurado.

## DB-RESET-063

Role será restaurado.

## DB-RESET-064

Search path será restaurado cuando aplique.

## DB-RESET-065

SQL mode será restaurado cuando aplique.

## DB-RESET-066

Temporary state deberá limpiarse o causar discard.

## DB-RESET-067

Advisory locks deberán liberarse o causar discard.

## DB-RESET-068

Platform reset utilizará capabilities.

## DB-RESET-069

Version no equivaldrá a capability.

## DB-RESET-070

MySQL y MariaDB serán plataformas separadas.

## DB-RESET-071

Reset System no concatenará vendor SQL directamente.

## DB-RESET-072

Reset SQL pasará por infraestructura apropiada.

## DB-RESET-073

SQLite tendrá políticas específicas.

## DB-RESET-074

`:memory:` connection discard tendrá semántica especial.

## DB-RESET-075

Reset verification será explícito.

## DB-RESET-076

Calling reset no equivaldrá a verified reset.

## DB-RESET-077

Known mutation tracking podrá formar parte de la prueba.

## DB-RESET-078

Platform guarantees podrán formar parte de la prueba.

## DB-RESET-079

Driver guarantees podrán formar parte de la prueba.

## DB-RESET-080

Unknown outcome impedirá reuse.

## DB-RESET-081

Reset performance no reducirá correctness.

## DB-RESET-082

Reconnect podrá preferirse a reset si es más seguro/barato.

## DB-RESET-083

Reset retry no será ciego.

## DB-RESET-084

Retry requerirá retry-safe operation y outcome conocido.

## DB-RESET-085

Reset failure deberá contenerse.

## DB-RESET-086

Connection failure contenido no implicará worker failure.

## DB-RESET-087

Unknown worker-global state podrá requerir recycle.

## DB-RESET-088

Worker state tendrá lifecycle explícito.

## DB-RESET-089

Worker persistent caches seguros podrán sobrevivir.

## DB-RESET-090

Current EntityManager no sobrevivirá.

## DB-RESET-091

Current transaction no sobrevivirá.

## DB-RESET-092

Current tenant no sobrevivirá.

## DB-RESET-093

Current profiler session no sobrevivirá.

## DB-RESET-094

Persistent runtime no dependerá de process termination.

## DB-RESET-095

FrankenPHP deberá ejecutar reset por execution.

## DB-RESET-096

RoadRunner deberá ejecutar reset por execution/job.

## DB-RESET-097

OpenSwoole reset será coroutine-aware.

## DB-RESET-098

Reset de scope A no borrará state B.

## DB-RESET-099

Fiber reset será context-aware.

## DB-RESET-100

Runtime adapters implementarán lifecycle específico fuera del core.

## DB-RESET-101

Reset participants podrán ser extensibles.

## DB-RESET-102

Reset participant declarará dependencies.

## DB-RESET-103

Dependency cycles serán rechazados.

## DB-RESET-104

Reset registry podrá congelarse después de bootstrap.

## DB-RESET-105

Persistent-runtime extension deberá declarar safety.

## DB-RESET-106

UNKNOWN extension safety no se asumirá segura.

## DB-RESET-107

Reset no equivaldrá a cache invalidation.

## DB-RESET-108

L0 cache será limpiado.

## DB-RESET-109

L1 safe cache podrá sobrevivir.

## DB-RESET-110

L2 cache no se vaciará por request.

## DB-RESET-111

Entity cache no almacenará managed objects.

## DB-RESET-112

Metadata cache no se vaciará por request.

## DB-RESET-113

Compiled query cache no se vaciará innecesariamente.

## DB-RESET-114

Profiler state será limpiado.

## DB-RESET-115

N+1 observation state será limpiado.

## DB-RESET-116

Debug toolbar data será execution-scoped.

## DB-RESET-117

Audit correlation state será limpiado.

## DB-RESET-118

Persisted audit records no se eliminarán por reset.

## DB-RESET-119

Deferred event state será limpiado.

## DB-RESET-120

Cancellation registrations serán eliminadas.

## DB-RESET-121

Query timers serán cancelados.

## DB-RESET-122

Deadlines no cruzarán executions.

## DB-RESET-123

Resource permits owned serán liberados.

## DB-RESET-124

Reset eliminará scoped strong references controladas.

## DB-RESET-125

GC no sustituirá reset.

## DB-RESET-126

Reset no garantizará devolución inmediata de memoria al OS.

## DB-RESET-127

Reset producirá evidencia diagnóstica.

## DB-RESET-128

Reset telemetry no expondrá secrets.

## DB-RESET-129

Metrics evitarán high-cardinality IDs.

## DB-RESET-130

Reset tendrá unit tests.

## DB-RESET-131

Reset tendrá integration tests.

## DB-RESET-132

Reset tendrá fault-injection tests.

## DB-RESET-133

Reset tendrá persistent-worker soak tests.

## DB-RESET-134

Connection session reset tendrá pruebas por plataforma.

## DB-RESET-135

Transaction residue tendrá pruebas.

## DB-RESET-136

Temporary state tendrá pruebas.

## DB-RESET-137

Advisory lock cleanup tendrá pruebas.

## DB-RESET-138

Raw SQL dirty-state tendrá pruebas.

## DB-RESET-139

Coroutine isolation durante reset tendrá pruebas.

## DB-RESET-140

Fiber isolation durante reset tendrá pruebas.

## DB-RESET-141

Pool aceptará sólo conexiones verificadas.

## DB-RESET-142

Strict testing podrá fallar ante leaked cursor.

## DB-RESET-143

Strict testing podrá fallar ante leaked transaction.

## DB-RESET-144

Strict testing podrá fallar ante pending UoW.

## DB-RESET-145

Strict testing podrá fallar ante unknown reset state.

## DB-RESET-146

Production preferirá discard ante incertidumbre.

## DB-RESET-147

Production preferirá worker recycle ante state contamination no contenida.

## DB-RESET-148

Reset Manager no hará business commit implícito.

## DB-RESET-149

Reset Manager no ocultará unknown outcomes.

## DB-RESET-150

Reset Manager no generará vendor SQL arbitrario.

## DB-RESET-151

State Reset será determinista cuando las condiciones sean equivalentes.

## DB-RESET-152

Reset baseline podrá estar generation-aware.

## DB-RESET-153

Baseline configuration change invalidará assumptions antiguas.

## DB-RESET-154

Connection reuse dependerá del reset verification.

## DB-RESET-155

Worker reuse dependerá del reset/isolation verification.

## DB-RESET-156

Process reuse nunca será prueba suficiente de limpieza.

## DB-RESET-157

Resource destruction será preferible a unsafe reuse.

## DB-RESET-158

Correctness tendrá prioridad sobre pooling.

## DB-RESET-159

Correctness tendrá prioridad sobre worker longevity.

## DB-RESET-160

Un recurso incierto será tratado conservadoramente.

---

# 257. Modelo formal

Sea:

```text
R
```

un recurso reutilizable.

Sea:

```text
B(R)
```

su baseline esperado.

Después de:

```text
Reset(R)
```

se requiere demostrar:

```text
State(R) ≡ B(R)
```

respecto a todas las propiedades relevantes para reuse.

---

# 258. Reuse theorem

```text
Reusable(R)
=
ResetSucceeded(R)
∧
VerificationSucceeded(R)
∧
NoUnknownRelevantState(R)
```

---

# 259. Failure theorem

Si:

```text
RelevantState(R) = UNKNOWN
```

entonces:

```text
Reusable(R) = false
```

---

# 260. Connection theorem

Para una conexión `C`:

```text
Reusable(C)
=
Connected(C)
∧
NoActiveTransaction(C)
∧
NoActiveResultResource(C)
∧
SessionBaselineKnown(C)
∧
NoUnknownOutcome(C)
```

---

# 261. Worker theorem

Para worker `W`:

```text
Reusable(W)
=
NoActiveExecutionState(W)
∧
RuntimeLocalBaseline(W)
∧
NoUncontainedIsolationViolation(W)
∧
NoCriticalResetFailure(W)
```

---

# 262. Reset hierarchy

```text
Application
    │
    ▼
Worker
    │
    ▼
Execution
    │
    ├── ORM
    ├── Transaction
    ├── Operations
    ├── Telemetry
    └── Resources
          │
          ▼
     Connections
```

El reset ocurre principalmente:

```text
bottom-up
```

para recursos dependientes.

---

# 263. Anti-patterns

## Anti-pattern 1 — Reset all globals

```php
GlobalState::clearEverything();
```

sin ownership ni clasificación.

Prohibido como arquitectura principal.

## Anti-pattern 2 — Assume rollback means clean

```text
ROLLBACK
→ connection clean
```

Incorrecto.

## Anti-pattern 3 — Assume request finished means resources closed

Incorrecto en runtimes persistentes.

## Anti-pattern 4 — Return dirty connection to pool

Crítico.

## Anti-pattern 5 — Swallow reset exception

```php
try {
    $connection->reset();
} catch (...) {
}
```

y después devolverla al pool.

Prohibido.

## Anti-pattern 6 — Clear EntityManager and reuse blindly

No será política default.

## Anti-pattern 7 — Flush all caches

Costoso y semánticamente incorrecto.

## Anti-pattern 8 — Reset another coroutine

Crítico.

## Anti-pattern 9 — Treat UNKNOWN as CLEAN

Prohibido.

## Anti-pattern 10 — Depend on PHP shutdown

Incompatible con persistent runtime.

---

# 264. Arquitectura final

```text
                 EXECUTION
                     │
                     ▼
                  ACTIVE
                     │
              execution ends
                     │
                     ▼
                  CLOSING
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
 Close operation resources   Stop new work
        │
        ▼
 Resolve transaction
        │
        ▼
 Close ORM state
        │
        ▼
 Reset scoped context
        │
        ▼
 Reset connection session
        │
        ▼
 Verify baseline
        │
    ┌───┴───────────────┐
    │                   │
    ▼                   ▼
  CLEAN              UNKNOWN/FAILED
    │                   │
    ▼                   ▼
  REUSE          QUARANTINE/DISCARD
                            │
                            ▼
                  recycle worker if needed
```

---

# 265. Modelo recomendado para VoltStack V1

Para V1 deberá preferirse una política conservadora:

```text
EntityManager
→ recreate per execution

IdentityMap
→ recreate per execution

UnitOfWork
→ recreate per execution

DatabaseContext
→ recreate per execution

TransactionContext
→ recreate per transaction

Query/Operation Context
→ recreate per operation

ConnectionLease
→ recreate per acquisition

Physical Connection
→ reusable only after verified reset

Metadata/Types/Compiler Definitions
→ immutable shared

Caches
→ shared only under explicit safety contracts
```

Esto reduce drásticamente la superficie de contaminación.

---

# 266. Principio de recreate vs reset

Cuando un objeto sea:

```text
cheap to recreate
+
context-heavy
+
mutable
```

VoltStack deberá preferir:

```text
RECREATE
```

sobre:

```text
RESET AND REUSE
```

---

# 267. Principio de reuse

Cuando un recurso sea:

```text
expensive to recreate
+
resettable
+
verifiable
```

como una conexión física:

```text
RESET
+
VERIFY
+
REUSE
```

puede ser apropiado.

---

# 268. Matriz recomendada

| Recurso | Estrategia V1 |
|---|---|
| DatabaseContext | Recreate |
| EntityManager | Recreate |
| IdentityMap | Recreate |
| UnitOfWork | Recreate |
| TransactionContext | Recreate |
| QueryExecutionContext | Recreate |
| HydrationSession | Recreate |
| ResultCursor | Close |
| StreamingResult | Close |
| LazyIteration | Close |
| ChunkTraversal | Close |
| ConnectionLease | Release |
| PhysicalConnection | Reset + Verify + Reuse |
| Metadata | Retain |
| TypeRegistry | Retain |
| Compiler | Retain |
| Dialect | Retain |
| Safe Worker Cache | Retain |
| Unsafe/Unknown Resource | Discard |

---

# 269. Regla final

> **VoltStack no intentará mantener vivo todo lo posible. Mantendrá vivo únicamente aquello cuya reutilización sea demostrablemente segura y beneficiosa.**

Por tanto:

```text
Persistent
≠
Everything Reusable
```

```text
Reset Called
≠
Reset Verified
```

```text
Rollback
≠
Clean Connection
```

```text
Clear ORM
≠
Commit
```

```text
Request Finished
≠
State Gone
```

```text
Connection Alive
≠
Connection Reusable
```

```text
UNKNOWN
≠
CLEAN
```

y:

```text
Cheap Contextual Object
→ Recreate

Expensive Reusable Resource
→ Reset + Verify

Uncertain Resource
→ Discard

Uncontained Worker State
→ Recycle
```

---

# 270. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
✓ 254_DATABASE_STATE_ISOLATION_SYSTEM.md
✓ 255_DATABASE_STATE_RESET_SYSTEM.md
│
├── 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
├── 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 271. Siguiente documento

```text
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

El siguiente documento profundizará específicamente en el recurso más importante que VoltStack sí buscará reutilizar entre ejecuciones:

```text
Physical Database Connection
```

Se definirán:

```text
Connection Reuse Architecture
Physical Connection vs Logical Connection
Connection Lease
Connection Ownership
Pool Interaction
Connection Acquisition
Connection Release
Reuse Eligibility
Session Baselines
Reset-before-Reuse
Connection Validation
Connection Health
Idle Connections
Maximum Lifetime
Connection Generation
Transaction Affinity
Prepared Statement Reuse
Session State Tracking
Tenant Safety
Shard Safety
Replica/Writer Affinity
Credential Rotation
Topology Changes
Failover Interaction
Unknown State Handling
Connection Quarantine
Connection Discard
Connection Reauthentication
Persistent Worker Integration
Concurrency Rules
Resource Governance
Telemetry
Testing
```

con la regla fundamental:

> **VoltStack reutilizará una conexión física únicamente cuando pueda demostrar que está viva, pertenece al dominio correcto, no conserva una transacción ni recursos activos, su estado de sesión corresponde al baseline requerido y no existe ningún resultado relevante desconocido.**