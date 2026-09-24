# 166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md

# VoltStack Quantum Database
## Database Transaction Context System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 166 — Database Transaction Context System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md`  
**Siguiente documento:** `167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Context System` define el modelo de estado contextual que representa una transacción activa, su identidad, ownership, conexión fijada, configuración efectiva, estado, deadlines, nesting, callbacks y demás recursos estrictamente asociados a su ejecución.

El `TransactionContext` será la fuente canónica de estado mutable de una transacción dentro de un scope de ejecución.

Regla central:

> **Toda información mutable necesaria para ejecutar una transacción deberá pertenecer al `TransactionContext` asociado al scope de ejecución; nunca deberá residir como estado global, estático o mutable compartido dentro del `TransactionManager`.**

Por tanto:

```text
TransactionManager
=
Shared Coordinator

TransactionContext
=
Scoped Mutable Transaction State
```

y:

```text
TransactionContext
≠ TransactionManager
≠ Connection
≠ UnitOfWork
≠ EntityManager
≠ HTTP Request
≠ Coroutine
```

---

# 2. Relación con Transaction Manager

El documento anterior estableció:

```text
Application
    ↓
TransactionManager
    ↓
TransactionContext
    ↓
Connection Lease
    ↓
Pinned Connection
    ↓
Driver
    ↓
Database
```

La separación fundamental será:

```text
TransactionManager
    = comportamiento

TransactionContext
    = estado
```

El manager podrá ser compartido entre requests/workers.

El contexto nunca.

---

# 3. Problema que resuelve

En runtimes tradicionales PHP-FPM, el proceso normalmente termina o se recicla con frecuencia.

En runtimes persistentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

un proceso puede atender:

```text
Request A
Request B
Request C
...
```

Si el estado transaccional se conserva accidentalmente en servicios compartidos:

```text
Request A
    ↓
BEGIN
    ↓
shared service stores transaction

Request B
    ↓
sees transaction from A
```

se produce una violación crítica de aislamiento.

El `TransactionContext System` elimina esta categoría de errores mediante:

```text
Runtime Scope
    ↓
Context Storage
    ↓
Transaction Context
```

---

# 4. Principio de diseño

VoltStack adoptará:

```text
Shared Immutable/Stateless Services
+
Scoped Mutable Context
```

Por ejemplo:

```text
Worker
│
├── TransactionManager
├── ConnectionManager
├── PlatformRegistry
├── TypeRegistry
│
├── Request Scope A
│   └── TransactionContext A
│
└── Request Scope B
    └── TransactionContext B
```

---

# 5. Responsabilidades

El `TransactionContext` almacenará o referenciará:

1. Transaction ID;
2. Transaction Operation ID;
3. runtime scope;
4. transaction owner;
5. effective transaction definition;
6. state;
7. outcome;
8. rollback-only status;
9. rollback-only cause;
10. connection lease;
11. pinned connection identity;
12. nesting level;
13. participation stack;
14. savepoint stack;
15. callback registry;
16. attempt number;
17. deadline;
18. cancellation state;
19. start time;
20. lifecycle timestamps;
21. transaction-local resources;
22. diagnostic metadata;
23. completion status.

---

# 6. No responsabilidades

El contexto no deberá:

- ejecutar `BEGIN`;
- ejecutar `COMMIT`;
- ejecutar `ROLLBACK`;
- generar SQL;
- ejecutar queries;
- hidratar entidades;
- persistir entidades;
- resolver repositories;
- administrar IdentityMap;
- administrar UnitOfWork;
- decidir retry;
- implementar isolation semantics;
- emitir directamente SQL de savepoints;
- resolver routing de conexión.

El contexto describe estado.

Otros componentes actúan sobre ese estado.

---

# 7. Modelo conceptual

```text
TransactionContext
│
├── Identity
│   ├── TransactionId
│   └── TransactionOperationId
│
├── Ownership
│   ├── RuntimeScopeId
│   └── TransactionOwner
│
├── Definition
│   └── EffectiveTransactionDefinition
│
├── Lifecycle
│   ├── TransactionState
│   ├── TransactionOutcome
│   ├── CompletionStatus
│   └── Timestamps
│
├── Connection
│   ├── ConnectionLease
│   └── PinnedConnection
│
├── Control
│   ├── RollbackOnly
│   ├── Deadline
│   └── Cancellation
│
├── Nesting
│   ├── ParticipationStack
│   └── SavepointStack
│
├── Callbacks
│
└── Transaction Local Resources
```

---

# 8. Contrato conceptual

```php
interface TransactionContext
{
    public function transactionId(): TransactionId;

    public function operationId(): TransactionOperationId;

    public function owner(): TransactionOwner;

    public function definition(): EffectiveTransactionDefinition;

    public function state(): TransactionState;

    public function outcome(): ?TransactionOutcome;

    public function connectionLease(): ConnectionLease;

    public function rollbackOnly(): bool;

    public function nestingLevel(): int;

    public function attempt(): int;

    public function deadline(): ?TransactionDeadline;

    public function cancellation(): ?CancellationToken;

    public function callbacks(): TransactionCallbackRegistry;

    public function resources(): TransactionResourceRegistry;
}
```

---

# 9. Context mutable interno

Aunque la interfaz pública deberá ser predominantemente read-only, internamente será necesario modificar el contexto.

Por ello se recomienda separar:

```text
TransactionContext
```

de:

```text
MutableTransactionContext
```

Ejemplo:

```php
interface MutableTransactionContext extends TransactionContext
{
    public function transitionTo(TransactionState $state): void;

    public function setOutcome(TransactionOutcome $outcome): void;

    public function markRollbackOnly(
        RollbackOnlyReason $reason,
    ): void;
}
```

---

# 10. Mutation authority

No cualquier componente podrá modificar el contexto.

Las mutaciones deberán realizarse mediante componentes autorizados:

```text
TransactionManager
TransactionStateMachine
TransactionCompletionCoordinator
NestedTransactionCoordinator
SavepointManager
```

No:

```text
Repository
Entity
Controller
Query Builder
Hydrator
```

---

# 11. Transaction identity

Cada transacción física tendrá:

```text
TransactionId
```

único dentro del sistema.

Ejemplo conceptual:

```text
tx_01JTXK8H3...
```

No deberá derivarse de:

- connection password;
- user ID;
- tenant secrets;
- SQL;
- database credentials.

---

# 12. TransactionOperationId

Cuando exista retry:

```text
Operation op-100
├── tx-501 attempt 1
├── tx-502 attempt 2
└── tx-503 attempt 3
```

todos podrán compartir:

```text
TransactionOperationId = op-100
```

mientras cada intento tendrá su propio:

```text
TransactionId
```

---

# 13. Identity formula

```text
TransactionExecutionIdentity
=
TransactionOperationId
+
TransactionId
+
AttemptNumber
```

---

# 14. Runtime scope

Cada contexto pertenecerá a:

```text
RuntimeScope
```

Ejemplos:

```text
HTTP Request
Queue Job
Console Command
Scheduler Operation
Coroutine
Test Scope
```

---

# 15. RuntimeScopeId

Contrato conceptual:

```php
final readonly class RuntimeScopeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

No deberá depender de una implementación HTTP concreta.

---

# 16. TransactionOwner

El owner representa quién posee el lifecycle físico.

```php
final readonly class TransactionOwner
{
    public function __construct(
        public RuntimeScopeId $scope,
        public TransactionOwnerType $type,
    ) {}
}
```

---

# 17. Owner vs participant

Distinguir:

```text
Transaction Owner
```

de:

```text
Transaction Participant
```

Ejemplo:

```text
Outer transaction()
    ↓
owns physical transaction

Inner transaction(REQUIRED)
    ↓
participates
```

El inner scope no obtiene ownership físico.

---

# 18. Ownership invariant

Formalmente:

```text
PhysicalCompletionAuthority(T)
=
Owner(T)
```

Un participant no puede realizar arbitrariamente:

```text
COMMIT
ROLLBACK
```

de la transacción física.

---

# 19. Connection lease

El contexto deberá conservar:

```text
ConnectionLease
```

durante toda la vida de la transacción.

```text
TransactionContext
      ↓
ConnectionLease
      ↓
Connection
```

---

# 20. Pinned connection

Una vez activa la transacción:

```text
PinnedConnection(T)
=
constant
```

hasta completion.

---

# 21. Connection identity

Además de la referencia al lease podrá conservarse metadata segura:

```php
final readonly class TransactionConnectionIdentity
{
    public function __construct(
        public ConnectionName $connection,
        public ConnectionInstanceId $instance,
        public DatabaseEndpointId $endpoint,
    ) {}
}
```

útil para diagnostics.

---

# 22. No connection credentials

Nunca almacenar en diagnostic context:

```text
password
DSN completo con credenciales
tokens
cert private keys
```

---

# 23. EffectiveTransactionDefinition

El contexto almacenará la configuración efectiva, no solo la solicitada.

Ejemplo:

```php
final readonly class EffectiveTransactionDefinition
{
    public function __construct(
        public TransactionIsolation $isolation,
        public bool $readOnly,
        public TransactionPropagation $propagation,
        public ?TransactionTimeout $timeout,
        public NestedTransactionStrategy $nestedStrategy,
    ) {}
}
```

---

# 24. Requested vs effective

Podrá conservarse opcionalmente para diagnostics:

```text
Requested Definition
        ↓
Capability Resolution
        ↓
Effective Definition
```

pero la ejecución utilizará exclusivamente:

```text
EffectiveTransactionDefinition
```

---

# 25. Transaction state

El contexto mantendrá:

```text
TransactionState
```

como estado actual.

Modelo general:

```text
NEW
 ↓
STARTING
 ↓
ACTIVE
 ├───────────────┐
 ↓               ↓
COMMITTING   ROLLING_BACK
 ↓               ↓
COMMITTED    ROLLED_BACK

Failure paths:
FAILED
UNKNOWN
```

---

# 26. State ownership

El contexto almacena el estado.

Pero:

```text
TransactionStateMachine
```

decide si la transición es válida.

---

# 27. Forbidden mutation

No:

```php
$context->state = TransactionState::COMMITTED;
```

Sí:

```php
$stateMachine->transition(
    $context,
    TransactionState::COMMITTING,
);
```

---

# 28. Transaction outcome

Distinguir:

```text
TransactionState
```

de:

```text
TransactionOutcome
```

Ejemplo:

```text
state:
    COMMITTED

outcome:
    COMMITTED
```

Pero en failure:

```text
state:
    UNKNOWN

outcome:
    UNKNOWN
```

---

# 29. State ≠ outcome

`State` responde:

> ¿Dónde está el lifecycle?

`Outcome` responde:

> ¿Qué sabemos que ocurrió en la base de datos?

---

# 30. Completion status

También deberá distinguirse:

```text
Database Outcome
```

de:

```text
Local Completion Status
```

Ejemplo:

```text
Database:
    COMMITTED

afterCommit:
    FAILED

cleanup:
    SUCCESS
```

Por tanto:

```text
TransactionOutcome = COMMITTED
CompletionStatus = COMPLETED_WITH_CALLBACK_FAILURE
```

---

# 31. Rollback-only

El contexto contendrá:

```text
rollbackOnly = true|false
```

más:

```text
RollbackOnlyReason
```

---

# 32. Rollback-only monotonicity

Una vez:

```text
rollbackOnly = true
```

no podrá volver a:

```text
false
```

durante esa transacción.

Formalmente:

```text
RollbackOnly(T,t₁) = true
⇒
RollbackOnly(T,t₂) = true

∀ t₂ > t₁
```

---

# 33. Rollback-only reasons

Ejemplos:

```text
INNER_TRANSACTION_FAILURE
DATABASE_TRANSACTION_ABORTED
DEADLOCK
TIMEOUT
CANCELLATION
USER_REQUEST
SAVEPOINT_FAILURE
PERSISTENCE_FAILURE
```

---

# 34. Primary rollback cause

El contexto deberá conservar al menos:

```text
first rollback-only cause
```

porque suele representar el origen más útil.

Causas posteriores podrán agregarse de manera bounded.

---

# 35. Rollback-only does not rollback

Importante:

```text
markRollbackOnly()
≠
rollback()
```

Solo modifica la posibilidad de completion.

---

# 36. Transaction timing

El contexto podrá mantener:

```text
createdAt
startedAt
completionStartedAt
completedAt
```

para diagnostics.

---

# 37. Monotonic timing

Para duración:

```text
MonotonicClock
```

deberá utilizarse cuando sea posible.

---

# 38. Duration

```text
TransactionDuration
=
CompletedMonotonicTime
-
StartedMonotonicTime
```

---

# 39. Deadline

Si existe timeout:

```text
started
+
timeout
=
deadline
```

---

# 40. Deadline representation

Preferiblemente:

```php
final readonly class TransactionDeadline
{
    public function __construct(
        public MonotonicInstant $deadline,
    ) {}
}
```

La representación de wall-clock podrá existir únicamente para observabilidad.

---

# 41. Deadline checks

El contexto deberá permitir:

```php
$context->deadline()?->isExpired($clock);
```

sin ejecutar ninguna acción por sí mismo.

---

# 42. Deadline expiration

El contexto no hará rollback.

En cambio:

```text
TransactionManager
or
Execution Coordinator
```

detectará expiration y:

```text
mark rollback-only
```

o cancelará la operación conforme a política.

---

# 43. Cancellation

El contexto podrá contener:

```text
CancellationToken
```

scoped a la operación.

---

# 44. Cancellation token ≠ transaction state

Un token cancelado no significa automáticamente:

```text
ROLLED_BACK
```

Solo expresa:

```text
cancellation requested
```

---

# 45. Cancellation before commit

Puede conducir a:

```text
rollback-only
```

---

# 46. Cancellation after commit dispatch

No puede cambiar mágicamente:

```text
Database Outcome
```

y podría terminar en:

```text
UNKNOWN
```

---

# 47. Nesting model

El contexto deberá representar scopes lógicos anidados.

Ejemplo:

```text
Physical Transaction
│
├── Participant A
│
├── Participant B
│   └── Participant C
│
└── Participant D
```

---

# 48. Nesting level

```text
root = 0
first nested = 1
second nested = 2
```

o una convención equivalente consistente.

---

# 49. Nested level ≠ savepoint count

```text
NestingLevel
≠
SavepointCount
```

porque una estrategia `JOIN_EXISTING` puede incrementar nesting sin crear savepoint.

---

# 50. Participation stack

Se recomienda:

```text
TransactionParticipationStack
```

---

# 51. Participant record

```php
final readonly class TransactionParticipant
{
    public function __construct(
        public TransactionParticipantId $id,
        public int $depth,
        public TransactionPropagation $propagation,
        public NestedTransactionStrategy $strategy,
        public bool $ownsPhysicalBoundary,
        public ?SavepointId $savepoint,
    ) {}
}
```

---

# 52. Root participant

El root tendrá:

```text
ownsPhysicalBoundary = true
```

Los scopes joined:

```text
ownsPhysicalBoundary = false
```

---

# 53. Stack model

```text
[
    RootParticipant,
    NestedParticipantA,
    NestedParticipantB
]
```

Top:

```text
NestedParticipantB
```

---

# 54. Stack operations

```text
push participant
peek participant
pop participant
```

deberán ser:

```text
O(1)
```

---

# 55. Stack integrity

No podrá cerrarse:

```text
Participant A
```

mientras:

```text
Participant B
```

más interno siga activo.

---

# 56. LIFO invariant

```text
ParticipantCompletionOrder
=
LIFO
```

salvo que una futura estrategia explícita defina otra cosa.

---

# 57. Savepoint stack

Cuando se utilicen savepoints:

```text
TransactionContext
    ↓
SavepointStack
```

---

# 58. Savepoint record

```php
final readonly class TransactionSavepoint
{
    public function __construct(
        public SavepointId $id,
        public string $physicalName,
        public int $depth,
        public SavepointState $state,
    ) {}
}
```

---

# 59. Savepoint physical names

No deberán provenir directamente de input de usuario.

Se generarán internamente.

Ejemplo:

```text
vs_sp_0001
vs_sp_0002
```

---

# 60. Savepoint stack semantics

```text
SAVEPOINT A
    ↓
SAVEPOINT B
    ↓
ROLLBACK TO B
    ↓
RELEASE B
```

deberá actualizar el stack coherentemente.

---

# 61. Savepoint state

Estados conceptuales:

```text
CREATED
ROLLED_BACK_TO
RELEASED
INVALID
UNKNOWN
```

---

# 62. Savepoint uncertainty

Si la conexión falla durante una operación de savepoint:

```text
Transaction health
```

podrá volverse incierta.

El contexto no deberá inventar un stack limpio.

---

# 63. Context health

Además del lifecycle podrá existir:

```text
TransactionHealth
```

por ejemplo:

```php
enum TransactionHealth
{
    case HEALTHY;
    case DEGRADED;
    case ROLLBACK_REQUIRED;
    case UNCERTAIN;
}
```

---

# 64. Health ≠ state

Ejemplo:

```text
state:
    ACTIVE

health:
    ROLLBACK_REQUIRED
```

significa que la transacción todavía está físicamente activa pero no puede completarse mediante commit.

---

# 65. Attempt information

El contexto deberá conocer:

```text
attemptNumber
```

para retry telemetry.

Ejemplo:

```text
attempt = 2
```

---

# 66. Retry state does not live across contexts

Cada attempt crea:

```text
new TransactionContext
```

No se reinicia el contexto anterior.

---

# 67. Operation-level state

Información compartida entre attempts deberá vivir en:

```text
TransactionOperationContext
```

o `RetryCoordinator`, no en un contexto de transaction attempt ya finalizado.

---

# 68. Callback Registry

Cada contexto tendrá:

```text
TransactionCallbackRegistry
```

---

# 69. Callback categories

```text
beforeCommit
afterCommit
afterRollback
afterCompletion
```

---

# 70. Callback registry scope

El registry existe únicamente durante la vida de:

```text
TransactionContext
```

y será destruido en cleanup.

---

# 71. Callback registration metadata

Podrá almacenar:

```php
final readonly class TransactionCallbackRegistration
{
    public function __construct(
        public TransactionCallbackId $id,
        public TransactionCallbackPhase $phase,
        public int $priority,
        public int $sequence,
        public callable $callback,
    ) {}
}
```

---

# 72. Callback determinism

Orden:

```text
priority
+
registration sequence
```

deberá producir ejecución determinista.

---

# 73. Callback memory safety

El registry deberá evitar crecimiento ilimitado.

Políticas:

```text
max callbacks
max callback metadata size
```

---

# 74. Transaction-local resources

Algunas extensiones necesitan asociar datos temporales a la transacción.

Para ello:

```text
TransactionResourceRegistry
```

---

# 75. Resource Registry

Contrato conceptual:

```php
interface TransactionResourceRegistry
{
    public function has(TransactionResourceKey $key): bool;

    public function get(TransactionResourceKey $key): mixed;

    public function put(
        TransactionResourceKey $key,
        mixed $resource,
    ): void;

    public function remove(TransactionResourceKey $key): void;
}
```

---

# 76. Resource keys

Las keys deberán ser:

```text
typed/namespaced
```

no strings globales arbitrarios.

Ejemplo:

```text
orm.persistence_consistency
events.outbox
cache.deferred_invalidation
```

---

# 77. Resource Registry is not service container

Nunca deberá convertirse en:

```text
transaction-scoped dependency injection container
```

Su uso estará restringido a estado contextual pequeño y explícito.

---

# 78. Resource ownership

Cada recurso deberá declarar, si aplica:

```text
cleanup policy
completion hook
resource limit
```

---

# 79. Resource cleanup

Al finalizar:

```text
TransactionResourceRegistry
→ clear
```

después de que los coordinadores correspondientes hayan terminado.

---

# 80. ORM integration resource

El ORM podría registrar una referencia contextual abstracta como:

```text
PersistenceTransactionState
```

pero el contexto base no deberá importar clases concretas del ORM.

---

# 81. Deferred cache invalidation

Cache integration podrá asociar:

```text
PendingCacheInvalidations
```

y ejecutarlas:

```text
afterCommit
```

---

# 82. Transactional event outbox

Event integration podrá asociar:

```text
PendingTransactionalEvents
```

pero:

```text
TransactionContext
```

no conocerá su estructura concreta.

---

# 83. Context attributes

Podrá existir:

```text
TransactionAttributeBag
```

para metadata pequeña.

Ejemplo:

```text
operation.name
feature.name
diagnostic.category
```

---

# 84. Attribute restrictions

No almacenar:

```text
raw request body
full entities
credentials
large result sets
```

---

# 85. Typed attributes

Preferir:

```php
TransactionAttributeKey<T>
```

conceptualmente, en lugar de strings sin contrato.

---

# 86. Context activation

Un contexto recién creado deberá activarse mediante:

```text
TransactionContextResolver
```

o un `TransactionContextStorage`.

---

# 87. Context storage abstraction

```php
interface TransactionContextStorage
{
    public function current(): ?TransactionContext;

    public function push(TransactionContext $context): void;

    public function pop(TransactionContext $context): void;
}
```

---

# 88. Why stack

Una simple variable:

```text
currentContext
```

es insuficiente para soportar correctamente:

```text
suspension
REQUIRES_NEW
nested scopes
context restoration
```

Un stack contextual es más general.

---

# 89. Context stack

Ejemplo futuro:

```text
Runtime Scope
│
└── Context Stack
    ├── tx-A suspended
    └── tx-B active
```

Cuando `tx-B` termina:

```text
pop tx-B
resume tx-A
```

---

# 90. Active vs suspended context

El storage deberá distinguir:

```text
ACTIVE
SUSPENDED
```

si se implementa `REQUIRES_NEW`.

---

# 91. Suspension

Suspender significa:

```text
remove from current execution routing
```

pero no:

```text
commit
rollback
release connection
```

---

# 92. Suspended connection

La conexión de la transacción suspendida permanece:

```text
leased
+
pinned
```

mientras la transacción exista.

---

# 93. Resource implications of REQUIRES_NEW

Ejemplo:

```text
Outer tx
    connection #1

Inner REQUIRES_NEW
    connection #2
```

por lo que el sistema deberá considerar:

```text
pool capacity
deadlock risks
resource budgets
```

---

# 94. Context restoration

Restoration deberá ser estrictamente LIFO:

```text
suspend A
activate B
complete B
restore A
```

---

# 95. Restoration failure

Si no puede restaurarse el contexto esperado:

```text
TransactionContextCorruptionException
```

y el scope deberá considerarse comprometido.

---

# 96. Context token

Para operaciones push/pop seguras podrá utilizarse:

```php
final readonly class TransactionContextToken
{
    public function __construct(
        public RuntimeScopeId $scope,
        public TransactionId $transaction,
        public int $stackGeneration,
    ) {}
}
```

---

# 97. Token validation

`pop()` deberá verificar que el token corresponde al contexto actualmente activo.

Esto evita:

```text
out-of-order pop
```

---

# 98. Scoped execution helper

Podrá existir:

```php
$storage->runWith(
    $context,
    fn () => ...
);
```

que garantice:

```text
push
try
finally
pop
```

---

# 99. Exception-safe context restoration

Patrón obligatorio:

```php
$token = $storage->push($context);

try {
    return $callback();
} finally {
    $storage->pop($token);
}
```

---

# 100. ContextResolver

Otros subsistemas no necesitan acceso al storage completo.

Podrán depender de:

```php
interface TransactionContextResolver
{
    public function current(): ?TransactionContext;
}
```

---

# 101. Least privilege

Ejemplo:

```text
QueryExecutor
→ TransactionContextResolver
```

No:

```text
QueryExecutor
→ MutableTransactionContextStorage
```

---

# 102. Query execution integration

Cuando Query Executor ejecuta:

```text
SELECT
INSERT
UPDATE
DELETE
```

consulta:

```text
Current TransactionContext?
```

---

# 103. Query connection resolution

```text
if transaction context ACTIVE:
    use context.pinnedConnection
else:
    use normal ConnectionManager routing
```

---

# 104. Context state validation before query

Una query no deberá ejecutarse normalmente si:

```text
state = COMMITTED
ROLLED_BACK
UNKNOWN
FAILED
```

---

# 105. Query during COMMITTING

Por default:

```text
COMMITTING
→ new application queries forbidden
```

salvo hooks explícitamente autorizados antes del dispatch.

---

# 106. Query during ROLLING_BACK

También:

```text
new application query
→ forbidden
```

---

# 107. Query context propagation

El Query Context podrá recibir:

```text
TransactionId
TransactionOperationId
TransactionIsolation
ReadOnly
```

como metadata derivada.

---

# 108. Read-only enforcement

Si:

```text
context.definition.readOnly = true
```

el Query Executor/semantic policy podrá rechazar operaciones mutativas antes del driver cuando sea posible.

---

# 109. Database remains final enforcement

Framework read-only checks:

```text
additional guard
```

No sustituyen capacidades/read-only semantics de la plataforma.

---

# 110. Transaction-local query metadata

No deberá almacenarse en el contexto:

```text
every executed AST
every result
every row
```

Esto provocaría memory growth.

---

# 111. Bounded diagnostics

Si se conservan breadcrumbs:

```text
last N operations
```

deberán ser:

```text
bounded
sanitized
optional
```

---

# 112. Diagnostic snapshot

Para observabilidad se podrá crear:

```text
TransactionContextSnapshot
```

inmutable.

---

# 113. Snapshot example

```php
final readonly class TransactionContextSnapshot
{
    public function __construct(
        public TransactionId $transactionId,
        public TransactionOperationId $operationId,
        public TransactionState $state,
        public ?TransactionOutcome $outcome,
        public TransactionHealth $health,
        public bool $rollbackOnly,
        public int $nestingLevel,
        public int $attempt,
        public string $connectionName,
        public int $callbackCount,
        public int $savepointCount,
        public Duration $elapsed,
    ) {}
}
```

---

# 114. Snapshot ≠ live context

Un diagnostic snapshot:

```text
cannot mutate transaction
cannot commit
cannot rollback
cannot access raw connection
```

---

# 115. Snapshot security

No deberá incluir:

```text
connection password
SQL parameter values
entity contents
auth tokens
```

---

# 116. Leak detection

Al finalizar un runtime scope:

```text
TransactionContextStorage
```

deberá poder determinar:

```text
stack empty?
```

---

# 117. Leak condition

```text
ScopeClosing
∧ ContextStack ≠ Empty
=
TransactionContextLeak
```

---

# 118. Leak cleanup

Pipeline:

```text
scope closing
    ↓
active context found
    ↓
report leak
    ↓
attempt safe rollback
    ↓
dispose unsafe connection if necessary
    ↓
clear transaction resources
    ↓
clear stack
```

---

# 119. No leak commit

Nunca:

```text
scope shutdown
→ auto commit
```

---

# 120. Multiple leaked contexts

Si existen contextos suspendidos/anidados:

```text
ContextStack
├── tx-A
└── tx-B
```

cleanup deberá seguir orden:

```text
top → bottom
```

con diagnostics completos.

---

# 121. Context corruption

Distinguir:

```text
Transaction Leak
```

de:

```text
Transaction Context Corruption
```

Leak:

> un contexto legítimo no fue cerrado.

Corruption:

> stack/ownership/identity ya no cumple invariantes.

---

# 122. Corruption examples

```text
wrong pop order
missing root
duplicate active context
connection mismatch
owner mismatch
negative nesting depth
savepoint stack impossible
terminal context still active
```

---

# 123. Corruption response

Por default:

```text
fail closed
mark scope unhealthy
quarantine affected connections
clear unsafe context state
emit critical diagnostic
```

---

# 124. Context lifecycle

Modelo:

```text
CREATED
   ↓
BOUND
   ↓
ACTIVE
   ↓
COMPLETING
   ↓
TERMINAL
   ↓
UNBOUND
   ↓
DISPOSED
```

Este lifecycle contextual es distinto del:

```text
TransactionState
```

de base de datos.

---

# 125. Why separate lifecycle

Puede existir:

```text
TransactionState = COMMITTED
ContextLifecycle = COMPLETING
```

mientras aún se ejecutan:

```text
afterCommit
afterCompletion
cleanup
```

---

# 126. ContextLifecycle enum

```php
enum TransactionContextLifecycle
{
    case CREATED;
    case BOUND;
    case ACTIVE;
    case COMPLETING;
    case TERMINAL;
    case UNBOUND;
    case DISPOSED;
}
```

---

# 127. Context disposal

Después de `DISPOSED`:

```text
callbacks inaccessible for registration
resources cleared
connection lease released/disposed
context cannot be reactivated
```

---

# 128. Context reuse

Prohibido:

```text
disposed context
→ begin another transaction
```

Cada transacción tendrá contexto nuevo.

---

# 129. Immutable identity

Durante toda la vida:

```text
TransactionId
OperationId
Owner
Attempt
```

serán inmutables.

---

# 130. Effective definition immutability

Después de comenzar físicamente:

```text
isolation
readOnly
propagation
```

no podrán cambiar arbitrariamente.

---

# 131. Isolation mutation

Si una plataforma permite cambios especiales, deberán modelarse mediante contratos explícitos.

Nunca:

```php
$context->definition->isolation = ...;
```

---

# 132. Tenant/database context

Una transacción podrá estar ligada a un:

```text
DatabaseExecutionDomain
```

que incluya, según configuración:

```text
connection
database
tenant
shard
role
```

---

# 133. Tenant context rule

Si una transacción comenzó para:

```text
Tenant A
```

no podrá cambiar silenciosamente a:

```text
Tenant B
```

sobre el mismo contexto.

---

# 134. Transaction domain

Modelo conceptual:

```php
final readonly class TransactionExecutionDomain
{
    public function __construct(
        public ConnectionName $connection,
        public ?TenantContextId $tenant,
        public ?ShardId $shard,
        public DatabaseRole $role,
    ) {}
}
```

El core podrá depender de contratos genéricos para evitar dependencia obligatoria de Multitenancy.

---

# 135. Domain immutability

```text
ExecutionDomain(T)
=
constant
```

durante la transacción.

---

# 136. Cross-database operations

Si una operación intenta usar una conexión incompatible:

```text
TransactionConnectionMismatchException
```

salvo futura arquitectura distribuida explícita.

---

# 137. Transaction context ≠ distributed transaction

El contexto actual representa:

```text
one logical local database transaction boundary
```

No debe fingir:

```text
XA
2PC
distributed atomic commit
```

---

# 138. Event correlation

Eventos de database podrán recibir:

```text
TransactionCorrelation
```

derivada del contexto.

---

# 139. TransactionCorrelation

```php
final readonly class TransactionCorrelation
{
    public function __construct(
        public TransactionOperationId $operation,
        public TransactionId $transaction,
        public int $attempt,
    ) {}
}
```

---

# 140. Correlation object safety

Deberá ser inmutable y seguro para telemetry/events.

---

# 141. Context propagation to async work

Una transacción local no deberá propagarse automáticamente a:

```text
new process
queue job
remote service
detached coroutine
```

---

# 142. Queue dispatch inside transaction

Si se agenda trabajo:

```text
Transaction A
    ↓
dispatch Job
```

el job futuro tendrá:

```text
new runtime scope
new transaction context
```

---

# 143. No transaction serialization

Nunca serializar:

```text
TransactionContext
```

para enviarlo a queue/cache/session.

---

# 144. No connection serialization

Asimismo:

```text
ConnectionLease
PinnedConnection
SavepointStack
```

no son serializables como estado distribuido.

---

# 145. Fiber/coroutine awareness

El runtime context abstraction deberá distinguir ejecuciones concurrentes dentro del mismo worker.

---

# 146. OpenSwoole problem

Incorrecto:

```php
static $currentTransaction;
```

porque:

```text
Coroutine A
Coroutine B
```

comparten proceso.

---

# 147. OpenSwoole solution

```text
Coroutine A
→ RuntimeContext A
→ TransactionContext A

Coroutine B
→ RuntimeContext B
→ TransactionContext B
```

---

# 148. Fiber support

El diseño deberá ser compatible conceptualmente con:

```text
PHP Fibers
```

sin asumir thread-local storage clásico.

---

# 149. FrankenPHP model

```text
FrankenPHP Worker
│
├── Shared Container
│   └── TransactionManager
│
├── Request Scope #1
│   └── ContextStorage #1
│       └── tx-1
│
└── Request Scope #2
    └── ContextStorage #2
        └── tx-2
```

---

# 150. RoadRunner model

```text
Worker
│
├── Job/Request A
│   └── fresh transaction context storage
│
└── Job/Request B
    └── fresh transaction context storage
```

---

# 151. Runtime reset

Al finalizar cualquier scope:

```text
TransactionContextStorage
TransactionResourceRegistry
CallbackRegistry
ParticipationStack
SavepointStack
```

deberán quedar vacíos.

---

# 152. Reset ≠ rollback

Importante:

```text
reset memory state
```

no sustituye:

```text
database rollback
```

Primero deberá intentarse una completion segura.

---

# 153. Context creation factory

Se recomienda:

```text
TransactionContextFactory
```

---

# 154. Factory contract

```php
interface TransactionContextFactory
{
    public function create(
        TransactionOperationId $operationId,
        TransactionOwner $owner,
        EffectiveTransactionDefinition $definition,
        ConnectionLease $lease,
        TransactionAttempt $attempt,
    ): MutableTransactionContext;
}
```

---

# 155. Why factory

Centraliza:

- IDs;
- clocks;
- initial state;
- resource registry;
- callback registry;
- diagnostic defaults;
- policy limits.

---

# 156. Initial state

Un contexto recién creado:

```text
TransactionState = NEW
ContextLifecycle = CREATED
Outcome = null
RollbackOnly = false
NestingLevel = 0
Savepoints = empty
Callbacks = empty
```

---

# 157. Binding

Después de asociarlo al runtime:

```text
ContextLifecycle
CREATED → BOUND
```

---

# 158. Activation

Después de `BEGIN` confirmado:

```text
TransactionState
STARTING → ACTIVE

ContextLifecycle
BOUND → ACTIVE
```

---

# 159. Completion

Cuando comienza commit/rollback:

```text
ContextLifecycle
ACTIVE → COMPLETING
```

---

# 160. Terminal state

Después de determinar outcome:

```text
ContextLifecycle
COMPLETING → TERMINAL
```

aunque callbacks/cleanup todavía puedan estar ejecutándose según implementación; alternativamente `TERMINAL` podrá representar que el outcome ya está fijado.

---

# 161. Unbinding

Después de completion:

```text
TERMINAL → UNBOUND
```

---

# 162. Disposal

Finalmente:

```text
UNBOUND → DISPOSED
```

---

# 163. Context state validation

Cada operación podrá comprobar simultáneamente:

```text
TransactionState
+
ContextLifecycle
```

Ejemplo:

```text
TransactionState = ACTIVE
ContextLifecycle = DISPOSED
```

es una combinación imposible.

---

# 164. Valid state matrix

Ejemplo conceptual:

| Context Lifecycle | Transaction State esperado |
|---|---|
| CREATED | NEW |
| BOUND | NEW / STARTING |
| ACTIVE | ACTIVE |
| COMPLETING | COMMITTING / ROLLING_BACK |
| TERMINAL | COMMITTED / ROLLED_BACK / FAILED / UNKNOWN |
| UNBOUND | terminal |
| DISPOSED | terminal |

---

# 165. Invalid combination

```text
ContextLifecycle = ACTIVE
TransactionState = COMMITTED
```

deberá activar:

```text
TransactionInvariantViolationException
```

---

# 166. Concurrency safety

Un `TransactionContext` no será thread-safe/coroutine-shared por diseño.

En cambio:

> deberá pertenecer a una única ejecución lógica.

---

# 167. Single execution ownership

```text
TransactionContext
→ one active runtime execution lane
```

salvo futuras APIs explícitas.

---

# 168. Parallel queries in one transaction

Si VoltStack soporta en el futuro:

```text
parallel DB operations
```

sobre una misma transacción, deberá comprobar primero que:

```text
driver
connection
protocol
transaction semantics
```

permiten concurrencia.

Por default:

```text
serialized access
```

será más seguro.

---

# 169. Context lock ≠ solution

Agregar un mutex al contexto no convierte automáticamente una conexión en segura para uso concurrente.

La capability deberá venir del driver/runtime.

---

# 170. Reentrancy

El contexto deberá detectar operaciones de lifecycle reentrantes peligrosas.

Ejemplo:

```text
beforeCommit callback
    ↓
commit same transaction again
```

debe rechazarse.

---

# 171. Reentrant commit

```text
state = COMMITTING
commit()
→ InvalidTransactionStateException
```

---

# 172. Reentrant rollback

Durante:

```text
ROLLING_BACK
```

un segundo rollback será rechazado o tratado por una operación interna idempotente distinta.

---

# 173. Context generation

Cada binding podrá utilizar:

```text
ContextGeneration
```

para detectar stale handles.

---

# 174. Stale transaction handle

Caso:

```text
tx handle retained
scope ended
new request starts
old handle commit()
```

deberá fallar.

---

# 175. Handle validation

Se verificará:

```text
handle.transactionId
==
currentContext.transactionId

AND

handle.scope
==
currentRuntimeScope
```

cuando la operación requiera contexto actual.

---

# 176. Detached transaction handles

Por default:

```text
transaction handle
```

no deberá ser usable después de salir del scope propietario.

---

# 177. Transaction Context Inspector

Podrá existir:

```php
$inspector->snapshot($context);
```

---

# 178. Diagnostic output

```text
TRANSACTION CONTEXT

Transaction:
    tx-892

Operation:
    op-117

Attempt:
    1

Runtime Scope:
    request-202

Owner:
    request-202

Lifecycle:
    ACTIVE

State:
    ACTIVE

Health:
    HEALTHY

Isolation:
    READ_COMMITTED

Read Only:
    false

Rollback Only:
    false

Connection:
    primary#7

Nesting:
    1

Savepoints:
    0

Callbacks:
    3

Deadline:
    4.8s remaining

Elapsed:
    203ms
```

---

# 179. Unknown outcome diagnostic

```text
TRANSACTION CONTEXT

Transaction:
    tx-893

State:
    UNKNOWN

Outcome:
    UNKNOWN

Last Phase:
    COMMIT

Commit Dispatch:
    DISPATCHED

Connection:
    quarantined

Retry:
    prohibited

Persistence Consistency:
    uncertain
```

---

# 180. Error hierarchy

```text
DatabaseTransactionContextException
│
├── TransactionContextNotFoundException
├── TransactionContextAlreadyBoundException
├── TransactionContextOwnershipException
├── TransactionContextStateException
├── TransactionContextCorruptionException
├── TransactionContextLeakException
├── TransactionContextDisposedException
├── TransactionContextStackException
├── TransactionContextRestoreException
├── TransactionConnectionMismatchException
├── TransactionExecutionDomainMismatchException
├── TransactionResourceLimitException
├── TransactionParticipantException
├── TransactionSavepointContextException
└── TransactionContextInvariantViolationException
```

---

# 181. Testing strategy

El sistema requerirá:

```text
ContextUnitTests
ContextStorageTests
ContextLifecycleTests
ContextOwnershipTests
ContextStackTests
ContextNestingTests
ContextSavepointTests
ContextResourceTests
ContextLeakTests
ContextRuntimeIsolationTests
ContextConcurrencyTests
ContextDiagnosticsTests
```

---

# 182. Creation tests

Verificar:

```text
unique TransactionId
correct OperationId
attempt assigned
owner assigned
initial state NEW
rollbackOnly false
empty callbacks
empty savepoints
```

---

# 183. Activation tests

```text
create
bind
start
activate
```

deberá producir combinaciones válidas de lifecycle/state.

---

# 184. Invalid transition tests

Probar:

```text
DISPOSED → ACTIVE
COMMITTED → ACTIVE
ROLLED_BACK → COMMITTING
UNKNOWN → ACTIVE
```

y rechazar todas.

---

# 185. Ownership tests

```text
scope A creates context
scope B attempts mutation
```

deberá fallar.

---

# 186. Stack tests

Validar:

```text
push A
push B
pop B
pop A
```

y rechazar:

```text
push A
push B
pop A
```

---

# 187. Suspension tests

Cuando se implemente:

```text
push A
suspend A
push B
complete B
pop B
restore A
```

deberá recuperar exactamente el contexto original.

---

# 188. Connection tests

Verificar:

```text
same lease throughout transaction
same connection instance
no release while active
```

---

# 189. Rollback-only tests

Una vez marcado:

```text
true
```

nunca deberá revertirse.

---

# 190. Callback tests

Verificar:

```text
registration
ordering
limits
cleanup
late registration
```

---

# 191. Resource registry tests

Verificar:

```text
typed keys
duplicate policy
limits
cleanup
scope isolation
```

---

# 192. Leak tests

Crear:

```text
active context
```

y cerrar scope.

Esperado:

```text
leak detected
rollback requested
cleanup
stack empty
```

---

# 193. Context corruption tests

Simular:

```text
wrong stack order
owner mismatch
connection mismatch
terminal active context
```

y comprobar fail-closed.

---

# 194. Worker isolation test

```text
Request A
    tx-A

Request B
    tx-B
```

Nunca:

```text
B.current() == tx-A
```

---

# 195. Coroutine isolation test

```text
Coroutine A:
    current = tx-A

Coroutine B:
    current = tx-B
```

intercalando suspensiones/resumes.

---

# 196. Stale handle test

```text
Request A:
    handle tx-A
    completes

Request B:
    old tx-A handle.commit()
```

deberá fallar.

---

# 197. Memory tests

Miles de transacciones secuenciales no deberán producir crecimiento continuo por:

```text
callbacks
resources
savepoint records
context snapshots
```

---

# 198. Performance model

Operaciones principales:

```text
current()          O(1)
push()             O(1)
pop()              O(1)
peek participant   O(1)
push participant   O(1)
push savepoint     O(1)
resource lookup    O(1) expected
```

---

# 199. Memory complexity

Por transacción:

```text
O(
    nesting depth
    +
    savepoints
    +
    callbacks
    +
    registered resources
)
```

Todos deberán estar bounded por políticas cuando sea necesario.

---

# 200. Proposed directory structure

```text
src/Quantum/Database/Transaction/
│
├── Context/
│   ├── TransactionContext.php
│   ├── MutableTransactionContext.php
│   ├── DefaultTransactionContext.php
│   ├── TransactionContextFactory.php
│   ├── TransactionContextLifecycle.php
│   ├── TransactionContextSnapshot.php
│   └── TransactionContextInspector.php
│
├── Context/Storage/
│   ├── TransactionContextResolver.php
│   ├── TransactionContextStorage.php
│   ├── ScopedTransactionContextStorage.php
│   ├── TransactionContextToken.php
│   └── TransactionContextStack.php
│
├── Context/Ownership/
│   ├── TransactionOwner.php
│   ├── TransactionOwnerType.php
│   ├── RuntimeScopeId.php
│   └── TransactionOwnershipValidator.php
│
├── Context/Domain/
│   ├── TransactionExecutionDomain.php
│   └── TransactionConnectionIdentity.php
│
├── Context/Nesting/
│   ├── TransactionParticipant.php
│   ├── TransactionParticipantId.php
│   └── TransactionParticipationStack.php
│
├── Context/Savepoint/
│   ├── TransactionSavepoint.php
│   ├── SavepointId.php
│   ├── SavepointState.php
│   └── TransactionSavepointStack.php
│
├── Context/Resource/
│   ├── TransactionResourceRegistry.php
│   ├── TransactionResourceKey.php
│   └── DefaultTransactionResourceRegistry.php
│
├── Context/Timing/
│   ├── TransactionDeadline.php
│   ├── TransactionTiming.php
│   └── TransactionClock.php
│
├── Context/Health/
│   ├── TransactionHealth.php
│   └── RollbackOnlyReason.php
│
└── Context/Exception/
    └── ...
```

---

# 201. Dependency rules

Permitido:

```text
TransactionContext
→ Transaction contracts
→ Connection lease contracts
→ Runtime scope contracts
→ Cancellation contracts
→ immutable transaction definitions
```

No permitido:

```text
TransactionContext
→ PDO
→ ORM Entity
→ UnitOfWork implementation
→ Repository
→ QueryBuilder
→ HTTP Request concrete class
→ FrankenPHP-specific globals
→ RoadRunner-specific globals
→ OpenSwoole globals
```

---

# 202. Architectural invariants

## DB-TXC-001
Cada transacción física tendrá un `TransactionContext`.

## DB-TXC-002
Cada context pertenecerá a un runtime scope.

## DB-TXC-003
El contexto será la fuente canónica del estado mutable transaccional.

## DB-TXC-004
El TransactionManager compartido no almacenará current transaction mutable.

## DB-TXC-005
El contexto no ejecutará SQL.

## DB-TXC-006
El contexto no ejecutará queries.

## DB-TXC-007
El contexto no administrará ORM state.

## DB-TXC-008
TransactionId será inmutable.

## DB-TXC-009
TransactionOperationId será inmutable.

## DB-TXC-010
Attempt number será inmutable.

## DB-TXC-011
Transaction owner será inmutable.

## DB-TXC-012
Runtime scope identity será inmutable.

## DB-TXC-013
Effective definition será inmutable una vez iniciada la transacción.

## DB-TXC-014
Execution domain será inmutable durante la transacción.

## DB-TXC-015
Pinned connection será constante durante ACTIVE.

## DB-TXC-016
Connection lease pertenecerá al contexto hasta completion.

## DB-TXC-017
Connection credentials no estarán en diagnostic context.

## DB-TXC-018
State y outcome serán conceptos distintos.

## DB-TXC-019
Database outcome y completion status serán distintos.

## DB-TXC-020
State transitions serán validadas por StateMachine.

## DB-TXC-021
No habrá state mutation arbitraria.

## DB-TXC-022
Rollback-only será monotónico.

## DB-TXC-023
Rollback-only no ejecutará rollback automáticamente.

## DB-TXC-024
Rollback-only conservará una causa.

## DB-TXC-025
Causas adicionales estarán bounded.

## DB-TXC-026
UNKNOWN será representado explícitamente.

## DB-TXC-027
UNKNOWN no será convertido a COMMITTED sin evidencia.

## DB-TXC-028
UNKNOWN no será convertido a ROLLED_BACK sin evidencia.

## DB-TXC-029
Context lifecycle será distinto de transaction lifecycle.

## DB-TXC-030
Un contexto terminal podrá seguir temporalmente en completion local.

## DB-TXC-031
Un contexto disposed nunca será reutilizado.

## DB-TXC-032
Un contexto disposed nunca será reactivado.

## DB-TXC-033
Un transaction handle stale será rechazado.

## DB-TXC-034
Cross-scope handle usage será rechazado.

## DB-TXC-035
Cross-owner commit será rechazado.

## DB-TXC-036
Cross-owner rollback será rechazado.

## DB-TXC-037
Owner y participant serán conceptos distintos.

## DB-TXC-038
Solo owner tendrá physical completion authority.

## DB-TXC-039
Nested participant joined no tendrá physical commit authority.

## DB-TXC-040
Nesting level no equivaldrá a savepoint count.

## DB-TXC-041
Participation stack será LIFO.

## DB-TXC-042
Out-of-order participant completion será rechazado.

## DB-TXC-043
Savepoint stack será transaction-scoped.

## DB-TXC-044
Savepoint physical names serán generados internamente.

## DB-TXC-045
User input no controlará directamente savepoint names.

## DB-TXC-046
Savepoint uncertainty no será convertida en certeza.

## DB-TXC-047
TransactionHealth será distinto de TransactionState.

## DB-TXC-048
ACTIVE puede coexistir con ROLLBACK_REQUIRED.

## DB-TXC-049
Attempt retry creará nuevo contexto.

## DB-TXC-050
Un contexto no será reiniciado para retry.

## DB-TXC-051
Operation-level retry state vivirá fuera del attempt context.

## DB-TXC-052
Callback registry será context-scoped.

## DB-TXC-053
Callbacks serán eliminados en cleanup.

## DB-TXC-054
Callback ordering será determinista.

## DB-TXC-055
Callback registration será bounded.

## DB-TXC-056
Transaction resources serán context-scoped.

## DB-TXC-057
Resource Registry no será service container.

## DB-TXC-058
Resource keys serán namespaced/typed.

## DB-TXC-059
Transaction resources serán limpiados al finalizar.

## DB-TXC-060
Large entities no se almacenarán como diagnostic attributes.

## DB-TXC-061
Raw result sets no se almacenarán en context.

## DB-TXC-062
Query history, si existe, será bounded.

## DB-TXC-063
Context storage será runtime-scoped.

## DB-TXC-064
Context lookup será O(1) esperado.

## DB-TXC-065
Context push será O(1).

## DB-TXC-066
Context pop será O(1).

## DB-TXC-067
Context stack soportará futuras suspension semantics.

## DB-TXC-068
Suspension no implicará commit.

## DB-TXC-069
Suspension no implicará rollback.

## DB-TXC-070
Suspension no liberará la pinned connection.

## DB-TXC-071
Context restoration será LIFO.

## DB-TXC-072
Wrong restoration order será corruption.

## DB-TXC-073
Context tokens impedirán stale/out-of-order operations.

## DB-TXC-074
Context activation será exception-safe.

## DB-TXC-075
Context deactivation será exception-safe.

## DB-TXC-076
QueryExecutor dependerá de resolver read-only cuando sea suficiente.

## DB-TXC-077
QueryExecutor no recibirá mutable storage innecesariamente.

## DB-TXC-078
Queries dentro de transacción usarán pinned connection.

## DB-TXC-079
Queries terminales serán rechazadas.

## DB-TXC-080
Application queries durante COMMITTING serán rechazadas.

## DB-TXC-081
Application queries durante ROLLING_BACK serán rechazadas.

## DB-TXC-082
Read-only transaction metadata estará disponible al executor.

## DB-TXC-083
Framework read-only enforcement no sustituirá DB enforcement.

## DB-TXC-084
Transaction context no será serializable a queues.

## DB-TXC-085
Connection lease no será serializable.

## DB-TXC-086
Pinned connection no será serializable.

## DB-TXC-087
Savepoint stack no será serializable.

## DB-TXC-088
Transaction context no se propagará automáticamente a procesos remotos.

## DB-TXC-089
Queue jobs recibirán contextos nuevos.

## DB-TXC-090
Detached async work no heredará automáticamente una transacción.

## DB-TXC-091
Tenant/domain changes incompatibles serán rechazados.

## DB-TXC-092
Shard changes incompatibles serán rechazados.

## DB-TXC-093
Database connection changes incompatibles serán rechazados.

## DB-TXC-094
Transaction Context no fingirá distributed transaction semantics.

## DB-TXC-095
Transaction correlation será segura para telemetry.

## DB-TXC-096
Sensitive payloads no estarán en correlation metadata.

## DB-TXC-097
Diagnostic snapshots serán inmutables.

## DB-TXC-098
Diagnostic snapshots no podrán mutar la transacción.

## DB-TXC-099
Diagnostic snapshots no expondrán raw connection.

## DB-TXC-100
Scope shutdown verificará context stack vacío.

## DB-TXC-101
Context restante al shutdown será leak.

## DB-TXC-102
Leak nunca provocará auto-commit.

## DB-TXC-103
Leak provocará safe rollback attempt.

## DB-TXC-104
Unsafe leaked connection será quarantined/discarded.

## DB-TXC-105
Leak cleanup seguirá orden top-down.

## DB-TXC-106
Context corruption será distinta de leak.

## DB-TXC-107
Context corruption utilizará fail-closed.

## DB-TXC-108
Corruption podrá marcar runtime scope unhealthy.

## DB-TXC-109
Terminal context activo será invariant violation.

## DB-TXC-110
Negative nesting depth será imposible.

## DB-TXC-111
Impossible savepoint stack será corruption.

## DB-TXC-112
Connection mismatch será invariant violation.

## DB-TXC-113
Owner mismatch será invariant violation.

## DB-TXC-114
Context factory centralizará initial state.

## DB-TXC-115
Initial rollback-only será false.

## DB-TXC-116
Initial savepoint stack estará vacío.

## DB-TXC-117
Initial callback registry estará vacío.

## DB-TXC-118
Initial transaction state será NEW.

## DB-TXC-119
Initial context lifecycle será CREATED.

## DB-TXC-120
Binding precederá activación.

## DB-TXC-121
BEGIN confirmado precederá transaction ACTIVE.

## DB-TXC-122
Completion precederá disposal.

## DB-TXC-123
Unbinding precederá disposal.

## DB-TXC-124
Lifecycle/state combinations serán validadas.

## DB-TXC-125
FrankenPHP requests no compartirán context.

## DB-TXC-126
RoadRunner jobs no compartirán context.

## DB-TXC-127
OpenSwoole coroutines no compartirán context.

## DB-TXC-128
PHP Fiber execution deberá respetar runtime context isolation.

## DB-TXC-129
Static current transaction estará prohibido.

## DB-TXC-130
Global mutable transaction state estará prohibido.

## DB-TXC-131
Shared service state no referenciará un context activo permanentemente.

## DB-TXC-132
Context cleanup deberá ser determinista.

## DB-TXC-133
Context cleanup deberá ser idempotente a nivel interno cuando sea seguro.

## DB-TXC-134
Reset memory state no sustituirá database rollback.

## DB-TXC-135
Database completion deberá intentarse antes de eliminar estado local.

## DB-TXC-136
Cancellation token no equivaldrá a rollback.

## DB-TXC-137
Deadline expiration no equivaldrá a rollback confirmado.

## DB-TXC-138
Timeout durante commit podrá producir UNKNOWN.

## DB-TXC-139
Monotonic clock se preferirá para duración.

## DB-TXC-140
Wall-clock timestamps serán observacionales.

## DB-TXC-141
Transaction duration será bounded únicamente por política explícita.

## DB-TXC-142
Parallel transaction connection usage no se asumirá seguro.

## DB-TXC-143
Context mutex no sustituirá driver concurrency capability.

## DB-TXC-144
Lifecycle reentrancy peligrosa será rechazada.

## DB-TXC-145
Reentrant commit será rechazado.

## DB-TXC-146
Reentrant rollback será rechazado.

## DB-TXC-147
Context generation podrá detectar handles stale.

## DB-TXC-148
Context inspector no tendrá mutation authority.

## DB-TXC-149
El contexto podrá probarse independientemente del ORM.

## DB-TXC-150
Ante ambigüedad, VoltStack conservará explícitamente la incertidumbre.

---

# 203. Anti-patterns

## 203.1 Current transaction en singleton

```php
class TransactionManager
{
    private ?TransactionContext $current;
}
```

**Rechazado para runtimes persistentes.**

---

## 203.2 Context en variable estática

```php
TransactionContext::$current = $context;
```

**Prohibido.**

---

## 203.3 Context como service container

```php
$context->set('mailer', $mailer);
$context->set('repository', $repository);
```

**Rechazado.**

---

## 203.4 Context ejecutando queries

```php
$context->execute($query);
```

**Rechazado.**

---

## 203.5 Context limpiando ORM

```php
$context->identityMap()->clear();
```

**Rechazado.**

---

## 203.6 Serializar contexto

```php
queue()->push(serialize($transactionContext));
```

**Prohibido.**

---

## 203.7 Cambiar conexión durante transaction

```text
tx begins on primary#1
query later uses primary#4
```

**Prohibido.**

---

## 203.8 Reset rollback-only

```php
$context->setRollbackOnly(false);
```

**Prohibido.**

---

## 203.9 Guardar todas las queries

```php
$context->queries[] = $fullQuery;
```

sin límite.

**Rechazado.**

---

## 203.10 Reutilizar context para retry

```text
tx context
→ failure
→ reset()
→ retry
```

**Rechazado.**

---

# 204. Ejemplo — request normal

```text
HTTP Request
     ↓
RuntimeScope created
     ↓
TransactionContextStorage empty
     ↓
DB::transaction()
     ↓
ContextFactory
     ↓
tx-100 context CREATED
     ↓
bind
     ↓
BEGIN
     ↓
ACTIVE
     ↓
queries use pinned connection
     ↓
COMMIT
     ↓
COMMITTED
     ↓
callbacks
     ↓
unbound
     ↓
disposed
     ↓
RuntimeScope closes
     ↓
ContextStorage empty
```

---

# 205. Ejemplo — nested REQUIRED

```text
Context tx-100

Participation Stack
┌──────────────────────────┐
│ nested service B depth 2 │
├──────────────────────────┤
│ nested service A depth 1 │
├──────────────────────────┤
│ root owner depth 0       │
└──────────────────────────┘
```

Existe:

```text
1 physical transaction
1 TransactionContext
3 logical participants
```

---

# 206. Ejemplo — nested savepoint

```text
TransactionContext tx-101
│
├── Physical Transaction
│
├── Participant Root
│
├── Participant Nested A
│   └── Savepoint vs_sp_0001
│
└── Participant Nested B
    └── Savepoint vs_sp_0002
```

---

# 207. Ejemplo — future REQUIRES_NEW

```text
Runtime Context Stack

┌───────────────────────┐
│ tx-B ACTIVE           │
│ connection primary#12 │
├───────────────────────┤
│ tx-A SUSPENDED        │
│ connection primary#8  │
└───────────────────────┘
```

Después de `tx-B`:

```text
┌───────────────────────┐
│ tx-A ACTIVE           │
│ connection primary#8  │
└───────────────────────┘
```

---

# 208. Ejemplo — context leak

```text
Request #90
    ↓
BEGIN tx-55
    ↓
application exits unexpectedly
    ↓
scope shutdown
    ↓
ContextStorage not empty
```

VoltStack:

```text
detect tx-55
    ↓
report leak
    ↓
attempt rollback
    ↓
determine connection disposition
    ↓
clear callbacks/resources
    ↓
remove context
```

Nunca:

```text
commit tx-55
```

---

# 209. Ejemplo — OpenSwoole isolation

```text
Worker
│
├── Coroutine 101
│   └── RuntimeContext
│       └── tx-A
│
└── Coroutine 102
    └── RuntimeContext
        └── tx-B
```

Aunque ambos utilicen:

```text
same TransactionManager instance
```

sus contextos permanecen completamente separados.

---

# 210. Master context formula

```text
TransactionContext(T)
=
Identity(T)
+
Ownership(T)
+
EffectiveDefinition(T)
+
Lifecycle(T)
+
Outcome(T)
+
Health(T)
+
ConnectionLease(T)
+
ControlState(T)
+
ParticipationStack(T)
+
SavepointStack(T)
+
Callbacks(T)
+
Resources(T)
+
Timing(T)
```

---

# 211. Current context formula

```text
CurrentTransactionContext
=
TransactionContextResolver(
    CurrentRuntimeExecutionScope
)
```

Nunca:

```text
CurrentTransactionContext
=
GlobalStatic
```

---

# 212. Context isolation formula

Para dos scopes concurrentes:

```text
Scope(A) ≠ Scope(B)
```

debe implicar:

```text
TransactionContext(A)
≠
TransactionContext(B)
```

y:

```text
MutableState(A)
∩
MutableState(B)
=
∅
```

salvo referencias explícitamente inmutables/compartibles.

---

# 213. Connection affinity formula

Para toda operación participante `o`:

```text
o ∈ Transaction(T)
⇒
Connection(o)
=
PinnedConnection(T)
```

---

# 214. Rollback-only formula

```text
RollbackOnly(T)
=
true
```

implica:

```text
CommitAllowed(T)
=
false
```

pero no implica todavía:

```text
Outcome(T)
=
ROLLED_BACK
```

---

# 215. Completion formula

```text
TransactionContextDisposed
⇒
TransactionOutcomeKnown
∨
TransactionOutcomeUnknownExplicitly
```

Nunca deberá destruirse un contexto dejando la incertidumbre sin representar en diagnostics/resultados.

---

# 216. Arquitectura final

```text
                      RUNTIME EXECUTION SCOPE
                               │
                               ▼
                    TransactionContextStorage
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
       TransactionContextResolver     Context Stack
                 │                           │
                 └─────────────┬─────────────┘
                               ▼
                    TransactionContext
                               │
      ┌─────────────┬──────────┼──────────┬─────────────┐
      │             │          │          │             │
      ▼             ▼          ▼          ▼             ▼
   Identity      Lifecycle   Ownership  Connection    Control
      │             │          │          │             │
      │             │          │          │      ┌──────┴──────┐
      │             │          │          │      ▼             ▼
      │             │          │          │ RollbackOnly    Deadline
      │             │          │          │
      │             │          │          ▼
      │             │          │   ConnectionLease
      │             │          │          │
      │             │          │          ▼
      │             │          │   Pinned Connection
      │             │          │
      └─────────────┴──────────┴──────────┘
                               │
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
           Participation    Savepoints    Callbacks
               Stack          Stack        Registry
                 │             │             │
                 └─────────────┼─────────────┘
                               ▼
                    Transaction Resources
```

---

# 217. Decisiones arquitectónicas finales

VoltStack adoptará un:

```text
Runtime-Scoped Transaction Context
```

como única fuente canónica del estado mutable de una transacción.

El `TransactionManager` será:

```text
shared + stateless regarding current transaction
```

mientras:

```text
TransactionContext
```

será:

```text
scoped + mutable + non-shareable
```

La resolución será:

```text
Runtime Scope
→ TransactionContextStorage
→ TransactionContextResolver
→ Current TransactionContext
```

La conexión permanecerá:

```text
leased + pinned
```

durante toda la transacción.

La arquitectura distinguirá explícitamente:

```text
TransactionState
≠ TransactionOutcome
≠ TransactionHealth
≠ TransactionContextLifecycle
```

Asimismo:

```text
Owner
≠ Participant
```

y:

```text
Nesting
≠ Savepoint
```

El contexto soportará una pila para preparar VoltStack para:

```text
JOIN_EXISTING
SAVEPOINT
SUSPEND/RESUME
REQUIRES_NEW
```

sin rediseñar posteriormente el sistema contextual.

Para runtimes persistentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

se establece como invariante absoluta:

> **Ningún estado transaccional mutable podrá sobrevivir accidentalmente de un request, job o coroutine a otro.**

Finalmente:

> **El Transaction Context representa lo que VoltStack sabe acerca de una transacción concreta dentro de una ejecución concreta; cuando ese conocimiento sea incompleto, el contexto conservará explícitamente la incertidumbre en lugar de fabricar un estado limpio.**

---

# 218. Relación con documentos siguientes

```text
164 Transaction Architecture
        ↓
165 Transaction Manager
        ↓
166 Transaction Context
        ↓
167 Transaction Isolation
        ↓
168 Nested Transactions
        ↓
169 Savepoints
        ↓
170 Transaction Retry
        ↓
171 Deadlock Handling
        ↓
172 Optimistic Locking
        ↓
173 Pessimistic Locking
        ↓
174 Concurrency Control
        ↓
175 Transaction Events
```

El `TransactionContext` definido aquí será utilizado directamente por todos estos sistemas.

---

# 219. Siguiente documento

```text
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md
```

El siguiente documento deberá definir:

- modelo canónico de isolation levels;
- `READ_UNCOMMITTED`;
- `READ_COMMITTED`;
- `REPEATABLE_READ`;
- `SERIALIZABLE`;
- snapshot isolation;
- vendor-specific isolation;
- requested vs effective isolation;
- Platform capabilities;
- default isolation;
- isolation negotiation;
- no silent downgrade;
- isolation validation;
- transaction definition integration;
- connection configuration;
- transaction start ordering;
- dirty reads;
- non-repeatable reads;
- phantom reads;
- lost updates;
- write skew;
- serialization failures;
- retry interaction;
- read-only transactions;
- ORM implications;
- query planner implications;
- read/write routing;
- replica restrictions;
- persistent runtime safety;
- diagnostics;
- telemetry;
- cross-platform compatibility;
- testing;
- invariantes arquitectónicas.