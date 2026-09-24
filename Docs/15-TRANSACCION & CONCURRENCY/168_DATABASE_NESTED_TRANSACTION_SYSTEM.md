# 168_DATABASE_NESTED_TRANSACTION_SYSTEM.md

# VoltStack Quantum Database
## Database Nested Transaction System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 168 — Database Nested Transaction System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md`  
**Siguiente documento:** `169_DATABASE_SAVEPOINT_SYSTEM.md`

---

# 1. Propósito

`Database Nested Transaction System` define cómo VoltStack representará y administrará operaciones transaccionales iniciadas mientras ya existe una transacción activa.

El problema fundamental es que:

```text
Nested Transaction
≠
Necessarily Another Physical Database Transaction
```

Cuando código como:

```php
DB::transaction(function () {
    serviceA();

    DB::transaction(function () {
        serviceB();
    });
});
```

es ejecutado, existen varias estrategias posibles:

- participar en la transacción existente;
- crear un savepoint;
- suspender la transacción actual e iniciar otra;
- ejecutar sin transacción;
- rechazar la operación;
- exigir que exista una transacción.

Por ello VoltStack deberá separar claramente:

```text
Logical Transaction Scope
```

de:

```text
Physical Database Transaction
```

Regla central:

> **Una transacción anidada en VoltStack será primero un scope transaccional lógico; su relación con una transacción física existente será determinada explícitamente por la política de propagación y las capacidades de la plataforma.**

---

# 2. Objetivos

El sistema deberá proporcionar:

1. nested transaction scopes;
2. transaction propagation;
3. logical transaction hierarchy;
4. physical transaction ownership;
5. join-existing semantics;
6. savepoint-backed nesting;
7. independent transaction scopes;
8. transaction suspension;
9. transaction resumption;
10. rollback-only propagation;
11. commit ownership;
12. rollback ownership;
13. nesting depth control;
14. isolation compatibility;
15. read-only compatibility;
16. timeout compatibility;
17. connection affinity;
18. context stack management;
19. failure propagation;
20. retry-boundary awareness;
21. ORM/UoW integration;
22. persistent-runtime isolation;
23. diagnostics;
24. telemetry;
25. platform capability validation.

---

# 3. No objetivos

Este sistema no implementará directamente:

- SQL savepoint syntax;
- isolation algorithms;
- deadlock detection;
- retry policies;
- optimistic locking;
- pessimistic locking;
- distributed transactions;
- XA transactions;
- two-phase commit;
- ORM flush;
- query execution;
- connection pooling.

Estos sistemas podrán integrarse con Nested Transactions, pero permanecerán separados.

---

# 4. Problema conceptual

Considérese:

```php
DB::transaction(function () {
    createOrder();

    DB::transaction(function () {
        reserveInventory();
    });
});
```

La segunda llamada puede significar:

```text
A. Join existing transaction

B. Create SAVEPOINT

C. Start independent physical transaction

D. Suspend outer transaction

E. Reject nested transaction
```

La API no deberá decidir esto accidentalmente.

---

# 5. Modelo principal

VoltStack utilizará:

```text
Transaction Scope
        │
        ▼
Propagation Resolver
        │
        ├── JOIN_EXISTING
        ├── CREATE_PHYSICAL
        ├── CREATE_SAVEPOINT
        ├── SUSPEND_AND_CREATE
        ├── EXECUTE_NON_TRANSACTIONALLY
        └── REJECT
        │
        ▼
Nested Transaction Context
```

---

# 6. Logical vs Physical Transaction

Distinción fundamental:

```text
LogicalTransaction
≠
PhysicalTransaction
```

Una sola transacción física podrá contener:

```text
Physical Transaction TX-1
│
├── Logical Scope A
│
├── Logical Scope B
│
└── Logical Scope C
```

---

# 7. Ejemplo JOIN_EXISTING

```text
BEGIN                 ← Physical TX-1

Scope A
│
├── operation
│
└── Scope B
    └── operation

COMMIT                ← Physical TX-1
```

Scope B:

```text
does not BEGIN
does not COMMIT
does not own physical transaction
```

---

# 8. Ejemplo SAVEPOINT

```text
BEGIN
│
├── Scope A
│
├── SAVEPOINT sp_1
│   │
│   └── Scope B
│
├── RELEASE SAVEPOINT sp_1
│
└── COMMIT
```

Aquí B sigue perteneciendo a la misma transacción física.

---

# 9. Ejemplo REQUIRES_NEW

Conceptualmente:

```text
Physical TX-1
    ↓
SUSPEND
    ↓
Physical TX-2
    ↓
COMMIT / ROLLBACK TX-2
    ↓
RESUME TX-1
```

Esto requiere una estrategia de conexión compatible.

---

# 10. Transaction Scope

Cada llamada transaccional generará un scope lógico.

```php
final readonly class TransactionScope
{
    public function __construct(
        public TransactionScopeId $id,
        public int $depth,
        public TransactionPropagation $propagation,
        public TransactionParticipation $participation,
        public TransactionScopeOwnership $ownership,
    ) {}
}
```

---

# 11. TransactionScopeId

Cada scope tendrá identidad independiente:

```text
scope-1
scope-2
scope-3
```

aunque compartan:

```text
physical-tx-17
```

---

# 12. PhysicalTransactionId

La transacción física tendrá otra identidad:

```php
final readonly class PhysicalTransactionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Por tanto:

```text
ScopeIdentity
≠
PhysicalTransactionIdentity
```

---

# 13. Ejemplo de identidad

```text
Physical Transaction:
    tx-900

Scopes:
    scope-1
    scope-2
    scope-3
```

Todos pueden compartir:

```text
tx-900
```

sin convertirse en el mismo scope lógico.

---

# 14. Transaction Propagation

VoltStack deberá soportar inicialmente:

```php
enum TransactionPropagation
{
    case REQUIRED;
    case REQUIRES_NEW;
    case SUPPORTS;
    case MANDATORY;
    case NOT_SUPPORTED;
    case NEVER;
    case NESTED;
}
```

---

# 15. REQUIRED

Semántica:

```text
Existing transaction?
│
├── YES → JOIN_EXISTING
│
└── NO  → CREATE_PHYSICAL
```

Será el comportamiento recomendado por default.

---

# 16. REQUIRED example

```php
DB::transaction(
    fn () => $service->execute(),
    propagation: TransactionPropagation::REQUIRED,
);
```

Sin transacción:

```text
BEGIN
service
COMMIT
```

Con transacción:

```text
join current
service
return
```

---

# 17. REQUIRES_NEW

Semántica:

```text
Existing transaction?
│
├── YES
│   ├── SUSPEND existing
│   ├── CREATE new physical transaction
│   └── RESUME existing afterwards
│
└── NO
    └── CREATE physical transaction
```

---

# 18. REQUIRES_NEW requires physical independence

Regla:

> `REQUIRES_NEW` no deberá simularse mediante savepoint.

Porque:

```text
SAVEPOINT
≠
Independent Physical Transaction
```

---

# 19. SUPPORTS

Semántica:

```text
Existing transaction?
│
├── YES → JOIN_EXISTING
│
└── NO  → EXECUTE_NON_TRANSACTIONALLY
```

---

# 20. MANDATORY

Semántica:

```text
Existing transaction?
│
├── YES → JOIN_EXISTING
│
└── NO  → ERROR
```

Excepción:

```text
TransactionRequiredException
```

---

# 21. NOT_SUPPORTED

Semántica:

```text
Existing transaction?
│
├── YES
│   ├── SUSPEND
│   ├── execute without transaction
│   └── RESUME
│
└── NO
    └── execute normally
```

---

# 22. NEVER

Semántica:

```text
Existing transaction?
│
├── YES → ERROR
│
└── NO  → execute without transaction
```

---

# 23. NESTED

Semántica recomendada:

```text
Existing transaction?
│
├── YES → CREATE_SAVEPOINT
│
└── NO  → CREATE_PHYSICAL
```

si la plataforma soporta savepoints.

---

# 24. NESTED fallback

VoltStack no deberá convertir silenciosamente:

```text
NESTED
```

en:

```text
REQUIRED
```

si savepoints no están soportados.

Por default:

```text
UnsupportedNestedTransactionException
```

---

# 25. Propagation decision table

| Propagation | Sin TX | Con TX |
|---|---|---|
| REQUIRED | Nueva TX | Join |
| REQUIRES_NEW | Nueva TX | Suspend + nueva TX |
| SUPPORTS | Sin TX | Join |
| MANDATORY | Error | Join |
| NOT_SUPPORTED | Sin TX | Suspend + sin TX |
| NEVER | Sin TX | Error |
| NESTED | Nueva TX | Savepoint |

---

# 26. TransactionParticipation

```php
enum TransactionParticipation
{
    case PHYSICAL_OWNER;
    case JOINED;
    case SAVEPOINT_PARTICIPANT;
    case INDEPENDENT_OWNER;
    case NON_TRANSACTIONAL;
}
```

---

# 27. Ownership

Un scope deberá saber qué puede controlar.

```php
enum TransactionScopeOwnership
{
    case OWNS_PHYSICAL_TRANSACTION;
    case OWNS_SAVEPOINT;
    case PARTICIPATES;
    case NONE;
}
```

---

# 28. Commit ownership

Regla crítica:

> **Solo el owner de una transacción física podrá realizar su commit físico.**

---

# 29. Joined inner commit

Ejemplo:

```text
Outer Scope:
    owns TX-1

Inner Scope:
    joined TX-1
```

Cuando Inner termina correctamente:

```text
Inner complete
```

NO:

```text
COMMIT TX-1
```

---

# 30. Outer commit

Únicamente:

```text
Outer successful completion
        ↓
COMMIT TX-1
```

---

# 31. Rollback ownership

Un joined scope tampoco podrá ejecutar arbitrariamente:

```text
ROLLBACK
```

sobre la transacción física completa.

En su lugar deberá marcar:

```text
rollback-only
```

cuando su failure invalide la transacción compartida.

---

# 32. Rollback-only

Estado:

```php
enum RollbackOnlyState
{
    case CLEAN;
    case MARKED;
}
```

---

# 33. Rollback-only semantics

Ejemplo:

```text
Outer TX
│
└── Inner REQUIRED
      ↓
    failure
      ↓
    rollback-only
```

Aunque Outer capture la excepción:

```php
DB::transaction(function () {
    try {
        serviceB();
    } catch (\Throwable $e) {
        // ignored
    }
});
```

la transacción puede seguir marcada:

```text
ROLLBACK_ONLY
```

---

# 34. Commit while rollback-only

Cuando el owner intente:

```text
COMMIT
```

VoltStack deberá:

```text
ROLLBACK
```

y lanzar:

```text
UnexpectedRollbackException
```

o equivalente.

---

# 35. Why rollback-only exists

Sin esta regla:

```text
inner failed
outer swallowed exception
outer commits
```

podría producir una transacción parcialmente inválida.

---

# 36. Rollback-only reason

Se almacenará información estructurada:

```php
final readonly class RollbackOnlyReason
{
    public function __construct(
        public TransactionScopeId $sourceScope,
        public RollbackReason $reason,
        public ?ThrowableReference $cause,
    ) {}
}
```

---

# 37. Multiple rollback reasons

Una transacción podrá acumular:

```text
RollbackOnlyReason[]
```

con límites de memoria.

---

# 38. NESTED + savepoint failure

Con:

```text
NESTED
```

un inner failure podrá:

```text
ROLLBACK TO SAVEPOINT
```

sin necesariamente marcar toda la transacción:

```text
rollback-only
```

si la recuperación es segura.

---

# 39. Savepoint isolation

```text
SAVEPOINT
```

permite rollback parcial de cambios DB.

No crea:

```text
new physical transaction
```

ni:

```text
new isolation level
```

---

# 40. Savepoint caveat

Rollback a savepoint no implica necesariamente que:

```text
PHP object graph
UnitOfWork
IdentityMap
```

regresen automáticamente al estado previo.

Regla:

```text
DatabaseRollbackToSavepoint
≠
AutomaticORMStateRewind
```

---

# 41. ORM implications

Esto es especialmente importante para:

```text
EntityManager
UnitOfWork
IdentityMap
Snapshots
ChangeSets
```

---

# 42. ORM savepoint checkpoint

VoltStack podrá introducir posteriormente:

```text
ORM Transaction Checkpoint
```

para coordinar:

```text
DB savepoint
+
UoW checkpoint
```

pero no deberá asumirse automáticamente.

---

# 43. Default ORM safety

Si un nested rollback deja el ORM en un estado cuya consistencia no puede probarse:

```text
EntityManager
→ TAINTED
```

podrá ser la política segura.

---

# 44. TransactionContext stack

Cada ejecución tendrá un stack scoped:

```text
TransactionContextStack
```

Ejemplo:

```text
TOP
│
├── Scope C
├── Scope B
└── Scope A
BOTTOM
```

---

# 45. Stack entry

```php
final readonly class TransactionScopeFrame
{
    public function __construct(
        public TransactionScope $scope,
        public TransactionContext $context,
        public ?SuspendedTransaction $suspended,
        public ?SavepointHandle $savepoint,
    ) {}
}
```

---

# 46. Stack is execution-local

Nunca:

```php
static array $transactionStack;
```

El stack deberá ser:

```text
Request-local
Job-local
Coroutine-local
Operation-local
```

---

# 47. Persistent runtime

Especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

deberá cumplirse:

```text
Operation A Transaction Stack
≠
Operation B Transaction Stack
```

---

# 48. Coroutine safety

En OpenSwoole:

```text
Coroutine A
    TX-1

Coroutine B
    TX-2
```

no deberán observar el mismo current transaction.

---

# 49. Current transaction resolution

```php
interface TransactionContextAccessor
{
    public function current(): ?TransactionContext;

    public function currentScope(): ?TransactionScope;
}
```

La implementación será scope-aware.

---

# 50. Transaction suspension

`REQUIRES_NEW` y `NOT_SUPPORTED` requieren:

```text
Transaction Suspension
```

---

# 51. SuspendedTransaction

```php
final readonly class SuspendedTransaction
{
    public function __construct(
        public TransactionContext $context,
        public ConnectionLease $connectionLease,
        public TransactionScopeStackSnapshot $scopeSnapshot,
    ) {}
}
```

---

# 52. Suspension ≠ rollback

```text
SUSPEND
```

no significa:

```text
ROLLBACK
```

La transacción outer continúa existiendo físicamente.

---

# 53. Suspension ≠ commit

Tampoco significa:

```text
COMMIT
```

---

# 54. Suspended connection

Una transacción física activa deberá conservar:

```text
connection affinity
```

Por ello su conexión no podrá regresar al pool como conexión libre.

---

# 55. REQUIRES_NEW connection requirement

Si TX-1 está activa sobre Connection A:

```text
TX-1 → Connection A
```

entonces TX-2 normalmente necesitará:

```text
TX-2 → Connection B
```

---

# 56. Pool exhaustion

Esto implica:

```text
REQUIRES_NEW
+
small connection pool
+
deep nesting
→
pool exhaustion risk
```

---

# 57. Resource governance

VoltStack deberá imponer límites sobre:

```text
nested depth
suspended transactions
connection leases
savepoint count
```

---

# 58. Maximum nesting depth

Configuración conceptual:

```php
'transactions' => [
    'max_nesting_depth' => 32,
],
```

---

# 59. Excess depth

Resultado:

```text
TransactionNestingDepthExceededException
```

antes de crear nuevos recursos.

---

# 60. Suspension depth

Podrá existir:

```text
max_suspended_transactions
```

separado del nesting depth.

---

# 61. Connection availability

`REQUIRES_NEW` deberá validar:

```text
connection can be independently leased
```

No deberá reutilizar accidentalmente la misma conexión con la outer TX activa.

---

# 62. No fake REQUIRES_NEW

Prohibido:

```text
REQUIRES_NEW
→ SAVEPOINT
```

como fallback silencioso.

---

# 63. Same physical connection restriction

En general:

```text
one connection
→ one active physical transaction
```

para el modelo portable base.

Vendor-specific exceptions no alterarán esta regla central sin una extensión explícita.

---

# 64. Resume

Después de completar TX-2:

```text
TX-2 terminal
    ↓
release Connection B
    ↓
restore TX-1 scope
    ↓
resume Connection A
```

---

# 65. Resume failure

Si la transacción suspendida no puede restaurarse correctamente:

```text
SuspendedTransactionResumeException
```

y el estado deberá considerarse potencialmente:

```text
TAINTED
```

o:

```text
UNKNOWN
```

según evidencia.

---

# 66. Resume ordering

Siempre:

```text
complete inner physical transaction
        ↓
cleanup inner context
        ↓
release inner resources
        ↓
resume outer context
```

---

# 67. Exception safety

Resume deberá ejecutarse mediante un mecanismo equivalente a:

```php
try {
    executeInner();
} finally {
    resumeOuter();
}
```

aunque el inner falle.

---

# 68. Suspension stack

Nested `REQUIRES_NEW` puede producir:

```text
TX-A suspended
    ↓
TX-B suspended
    ↓
TX-C active
```

Por ello suspension deberá usar stack, no un único slot.

---

# 69. Propagation Resolver

Componente:

```php
interface TransactionPropagationResolver
{
    public function resolve(
        TransactionPropagation $propagation,
        ?TransactionContext $current,
        TransactionDefinition $requested,
    ): TransactionPropagationDecision;
}
```

---

# 70. Propagation decision

```php
final readonly class TransactionPropagationDecision
{
    public function __construct(
        public TransactionParticipation $participation,
        public TransactionScopeOwnership $ownership,
        public TransactionPhysicalAction $action,
    ) {}
}
```

---

# 71. Physical actions

```php
enum TransactionPhysicalAction
{
    case BEGIN;
    case JOIN;
    case CREATE_SAVEPOINT;
    case SUSPEND_AND_BEGIN;
    case SUSPEND_AND_EXECUTE_NON_TRANSACTIONALLY;
    case EXECUTE_NON_TRANSACTIONALLY;
    case REJECT;
}
```

---

# 72. Resolution pipeline

```text
Transaction Request
        ↓
Current Context?
        ↓
Propagation Resolver
        ↓
Compatibility Validator
        ↓
Resource Validator
        ↓
Platform Capability Validator
        ↓
Physical Action Plan
        ↓
Scope Creation
        ↓
Execution
```

---

# 73. Compatibility validation

Cuando un scope participa en una transacción existente deberán compararse:

```text
isolation
read mode
timeout
connection
tenant/database context
shard
consistency policy
```

---

# 74. Isolation compatibility

Del documento 167:

```text
Outer Effective Isolation
+
Inner Requested Isolation
+
Inner Resolution Policy
→
Compatibility
```

---

# 75. Example isolation conflict

```text
Outer:
    READ_COMMITTED

Inner:
    SERIALIZABLE
    EXACT

Propagation:
    REQUIRED
```

Resultado:

```text
NestedTransactionIsolationConflictException
```

---

# 76. Isolation AT_LEAST

```text
Outer:
    SERIALIZABLE

Inner:
    READ_COMMITTED
    AT_LEAST
```

podrá ser compatible.

---

# 77. Isolation comparison is semantic

Nunca:

```php
$outer->level >= $inner->level
```

La comparación deberá utilizar:

```text
IsolationGuaranteeSet
```

---

# 78. Read-only compatibility

Caso:

```text
Outer:
    READ_ONLY

Inner:
    READ_WRITE

Propagation:
    REQUIRED
```

Debe fallar.

---

# 79. Read-only conflict

Excepción:

```text
NestedTransactionReadModeConflictException
```

---

# 80. Read-write outer + read-only inner

```text
Outer:
    READ_WRITE

Inner:
    READ_ONLY
```

puede participar bajo una política que interprete `READ_ONLY` inner como:

```text
scope-level restriction
```

sin modificar la transacción física.

---

# 81. Scope read-only guard

En ese caso:

```text
Physical TX:
    READ_WRITE

Inner logical scope:
    READ_ONLY
```

El Query Engine podrá impedir writes durante el inner scope.

---

# 82. Physical vs logical read-only

Distinguir:

```text
PhysicalTransactionReadMode
```

de:

```text
LogicalScopeReadRestriction
```

---

# 83. Timeout compatibility

Caso:

```text
Outer:
    remaining timeout = 20s

Inner:
    timeout = 60s
```

El inner no puede extender la vida física del outer.

---

# 84. Effective nested timeout

Para joined scopes:

```text
EffectiveInnerDeadline
=
min(
    OuterDeadline,
    InnerRequestedDeadline
)
```

---

# 85. Shorter timeout

Un inner podrá imponer:

```text
5s
```

sobre un outer con:

```text
30s
```

como scope deadline.

---

# 86. Inner timeout does not necessarily rollback physical transaction

Si el inner joined excede su deadline:

```text
scope failure
```

podrá marcar:

```text
rollback-only
```

según política.

---

# 87. REQUIRES_NEW timeout

Al crear una nueva transacción física:

```text
inner timeout
```

será independiente del timeout del outer, salvo límites globales de operación.

---

# 88. Tenant compatibility

Una joined transaction no deberá cambiar:

```text
tenant
database
shard
```

arbitrariamente.

---

# 89. Tenant conflict

```text
Outer:
    Tenant A

Inner REQUIRED:
    Tenant B
```

Resultado:

```text
NestedTransactionContextConflictException
```

por default.

---

# 90. REQUIRES_NEW tenant

Una transacción independiente podría utilizar otro contexto si la arquitectura de multitenancy lo permite explícitamente.

Pero deberá usar:

```text
independent connection
independent TransactionContext
```

---

# 91. Shard compatibility

```text
JoinedPhysicalTransaction
```

no podrá saltar de:

```text
Shard A
```

a:

```text
Shard B
```

sin distributed transaction support.

---

# 92. Distributed transactions

VoltStack no fingirá:

```text
TX across DB-A and DB-B
```

como una única transacción ACID.

---

# 93. Connection compatibility

JOIN requiere:

```text
same physical transactional connection
```

---

# 94. Connection resolver

El nested scope deberá recibir la conexión desde el contexto activo.

No volver a ejecutar arbitrariamente:

```text
ConnectionManager::resolveDefault()
```

---

# 95. Connection affinity invariant

```text
JoinedScopeConnection
=
ParentPhysicalTransactionConnection
```

---

# 96. Query routing

Mientras exista una joined transaction:

```text
Query
→ active transaction connection
```

No:

```text
Query
→ arbitrary read/write router
```

---

# 97. Savepoint strategy

`NESTED` utilizará:

```text
SavepointSystem
```

definido formalmente en:

```text
169_DATABASE_SAVEPOINT_SYSTEM.md
```

Nested Transaction System decidirá:

```text
savepoint required
```

pero no generará directamente SQL como:

```sql
SAVEPOINT foo;
```

---

# 98. Savepoint abstraction

```php
interface SavepointManager
{
    public function create(
        TransactionContext $context,
    ): SavepointHandle;

    public function rollback(
        SavepointHandle $savepoint,
    ): void;

    public function release(
        SavepointHandle $savepoint,
    ): void;
}
```

---

# 99. Savepoint ownership

Cada `NESTED` scope será owner de su propio:

```text
SavepointHandle
```

---

# 100. Nested savepoints

Ejemplo:

```text
BEGIN
    SAVEPOINT sp_1
        SAVEPOINT sp_2
            work
        RELEASE sp_2
    RELEASE sp_1
COMMIT
```

---

# 101. Savepoint naming

Los nombres serán generados internamente.

Nunca derivar directamente de:

```text
user input
entity name
request parameter
```

---

# 102. Logical completion states

Cada scope tendrá lifecycle propio:

```php
enum TransactionScopeState
{
    case CREATED;
    case ACTIVE;
    case COMPLETING;
    case COMPLETED;
    case FAILED;
    case ROLLED_BACK;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 103. Physical transaction lifecycle remains separate

```text
Scope COMPLETED
```

no implica:

```text
Physical TX COMMITTED
```

---

# 104. Example

```text
Inner joined scope:
    COMPLETED

Physical TX:
    ACTIVE
```

es perfectamente válido.

---

# 105. Outer completion

Cuando el physical owner termina:

```text
Scope:
    COMPLETED

Physical TX:
    COMMITTED
```

---

# 106. Failure propagation

La política deberá diferenciar:

```text
joined scope failure
savepoint scope failure
independent scope failure
non-transactional scope failure
```

---

# 107. Joined failure

Default:

```text
Inner joined failure
    ↓
mark physical transaction rollback-only
    ↓
propagate exception
```

---

# 108. Savepoint failure

Default:

```text
Inner NESTED failure
    ↓
ROLLBACK TO SAVEPOINT
    ↓
scope failed/rolled back
    ↓
outer may continue
```

si:

```text
DB state recovered
+
ORM/context state safe
```

---

# 109. Independent failure

```text
REQUIRES_NEW
```

inner failure:

```text
rollback inner TX
resume outer TX
propagate exception
```

La outer no será marcada rollback-only automáticamente.

---

# 110. Exception caught by outer

La aplicación podrá decidir:

```php
try {
    requiresNewOperation();
} catch (BusinessException $e) {
    // outer may continue
}
```

si la inner fue físicamente independiente.

---

# 111. Inner commit + outer rollback

Con `REQUIRES_NEW`:

```text
Outer TX-1
    ↓
Inner TX-2
    COMMIT
    ↓
Outer TX-1
    ROLLBACK
```

Los cambios de TX-2:

```text
remain committed
```

---

# 112. Important semantic difference

Por ello:

```text
REQUIRES_NEW
≠
NESTED
```

---

# 113. NESTED commit + outer rollback

Con savepoint:

```text
Outer TX
    ↓
Nested savepoint
    RELEASE
    ↓
Outer ROLLBACK
```

Todo se revierte.

---

# 114. Savepoint release is not commit

Regla:

```text
RELEASE SAVEPOINT
≠
COMMIT
```

---

# 115. Callback architecture

Todas las propagaciones deberán utilizar el mismo execution pipeline.

```php
$result = $transactionManager->execute(
    $definition,
    $callback,
);
```

---

# 116. Execution pipeline

```text
Receive Definition
        ↓
Inspect Current Context
        ↓
Resolve Propagation
        ↓
Validate Compatibility
        ↓
Prepare Resources
        ↓
Push Scope Frame
        ↓
Execute Callback
        ↓
Handle Success/Failure
        ↓
Pop Scope Frame
        ↓
Restore Suspended Context
        ↓
Return/Throw
```

---

# 117. Push before callback

El nuevo scope deberá estar visible antes de ejecutar:

```text
callback
```

para que nested calls detecten correctamente el contexto.

---

# 118. Pop in finally

El frame deberá eliminarse aunque:

```text
callback throws
commit fails
rollback fails
savepoint fails
```

conservando diagnostics suficientes.

---

# 119. Cleanup failure

Si callback y cleanup fallan simultáneamente:

```text
PrimaryFailure
+
CleanupFailure
```

deberán preservarse.

No sobrescribir silenciosamente el error original.

---

# 120. Composite failure

Podrá existir:

```php
final class TransactionCompletionException extends DatabaseException
{
    public function primaryFailure(): Throwable;

    public function cleanupFailures(): array;
}
```

---

# 121. Retry boundaries

Retry System deberá distinguir:

```text
logical nested scope
```

de:

```text
physical transaction retry boundary
```

---

# 122. Joined scope cannot independently retry transaction

Si Inner `REQUIRED` participa en TX-1:

```text
retrying only inner callback
```

puede ser incorrecto porque:

```text
TX-1 state already changed
```

---

# 123. Retry owner

Por default:

> Solo el owner del retryable physical transaction boundary podrá reiniciar la transacción completa.

---

# 124. Savepoint retry

Un sistema futuro podrá permitir:

```text
rollback to savepoint
+
retry nested operation
```

solo cuando:

- error sea savepoint-recoverable;
- platform lo permita;
- side effects sean seguros;
- ORM state pueda restaurarse;
- policy lo autorice.

No será asumido por default.

---

# 125. REQUIRES_NEW retry

Inner independent TX podrá tener su propio retry policy.

---

# 126. Outer retry implications

Si `REQUIRES_NEW` commit ocurrió y luego outer debe reintentarse:

```text
inner side effect already committed
```

Esto puede romper idempotencia.

---

# 127. Retry diagnostics

VoltStack deberá poder advertir:

```text
REQUIRES_NEW committed inside retryable outer transaction
```

cuando sea relevante.

---

# 128. ORM integration

El ORM deberá integrarse con scopes sin convertirse en Transaction Manager.

---

# 129. EntityManager scope

Dos estrategias conceptuales:

```text
JOIN_EXISTING
→ same EntityManager persistence context
```

y:

```text
REQUIRES_NEW
→ potentially isolated persistence scope
```

---

# 130. REQUIRES_NEW ORM hazard

Compartir exactamente el mismo:

```text
UnitOfWork
IdentityMap
```

entre dos transacciones físicas concurrentemente suspendidas puede producir inconsistencias.

---

# 131. Recommended policy

Para `REQUIRES_NEW`, se recomienda:

```text
Independent TransactionContext
+
Independent PersistenceContext
```

cuando ORM persistence participe.

---

# 132. Entity identity across contexts

Esto significa que:

```text
Entity A in outer IdentityMap
```

y:

```text
Entity A in inner IdentityMap
```

pueden ser objetos distintos.

La garantía IdentityMap es:

```text
same entity identity
+
same PersistenceContext
→
same instance
```

no global.

---

# 133. Cross-context entity transfer

No deberá transferirse automáticamente una entidad managed del outer al inner.

Preferir:

```text
pass identifier
→ inner repository reloads entity
```

---

# 134. Detached semantics

Si una entidad cruza boundaries podrá necesitar:

```text
DETACHED
```

o explicit reattachment policy.

---

# 135. Flush behavior

`flush()` continúa significando:

```text
synchronize UoW
```

No:

```text
commit nested transaction
```

---

# 136. Inner joined flush

```text
Inner REQUIRED
    ↓
flush()
```

puede enviar SQL dentro de la misma physical TX.

No realiza commit.

---

# 137. Outer rollback after inner flush

```text
Inner flush
Outer rollback
```

deberá revertir físicamente esas escrituras si pertenecen a la misma TX.

---

# 138. Inner REQUIRES_NEW flush

```text
Inner flush
Inner commit
```

sí puede hacer persistentes los cambios independientemente del outer.

---

# 139. Automatic flush

Nested Transaction System no deberá introducir:

```text
implicit flush on scope exit
```

por default.

---

# 140. Event integration

Podrán existir eventos:

```text
TransactionScopeStarted
TransactionScopeJoined
TransactionScopeSuspended
TransactionScopeResumed
TransactionScopeCompleted
TransactionScopeFailed
TransactionMarkedRollbackOnly
```

---

# 141. Transaction events vs scope events

Distinguir:

```text
PhysicalTransactionCommitted
```

de:

```text
TransactionScopeCompleted
```

---

# 142. afterCommit semantics

Un callback:

```php
DB::afterCommit(...)
```

dentro de un joined scope deberá asociarse al:

```text
physical transaction
```

no al final del inner scope.

---

# 143. Joined afterCommit

```text
Inner scope ends
    ↓
NO callback yet

Outer physical TX commits
    ↓
afterCommit callback
```

---

# 144. NESTED afterCommit

Release de savepoint tampoco deberá disparar:

```text
physical afterCommit
```

---

# 145. REQUIRES_NEW afterCommit

Aquí sí:

```text
inner physical commit
→ inner afterCommit
```

aunque outer siga activa.

---

# 146. afterRollback

Deberá distinguir:

```text
scope rollback
savepoint rollback
physical transaction rollback
```

---

# 147. Event scope metadata

Cada evento podrá contener:

```text
ScopeId
PhysicalTransactionId
Depth
Propagation
Participation
Outcome
```

---

# 148. Telemetry

Métricas recomendadas:

```text
db.transaction.scope.started
db.transaction.scope.completed
db.transaction.scope.failed
db.transaction.nested.depth
db.transaction.propagation
db.transaction.rollback_only
db.transaction.suspended
db.transaction.requires_new
db.transaction.savepoint_nested
```

---

# 149. Telemetry cardinality

No usar:

```text
ScopeId
TransactionId
UserId
EntityId
```

como metric labels.

---

# 150. Trace hierarchy

Tracing podrá representar:

```text
Physical Transaction Span
│
├── Logical Scope A
│
│   └── Query
│
└── Logical Scope B
    └── Query
```

---

# 151. REQUIRES_NEW tracing

```text
Outer TX Span
│
├── suspended
│
└── Inner TX Span
     └── ...
```

con relación explícita.

---

# 152. Diagnostics

Ejemplo:

```text
TRANSACTION SCOPE

Scope:
    scope-7

Depth:
    2

Propagation:
    REQUIRED

Participation:
    JOINED

Physical Transaction:
    tx-31

Isolation:
    SERIALIZABLE

Read Mode:
    READ_WRITE

Rollback Only:
    false
```

---

# 153. Rollback-only diagnostic

```text
TRANSACTION MARKED ROLLBACK-ONLY

Physical Transaction:
    tx-31

Source Scope:
    scope-8

Propagation:
    REQUIRED

Reason:
    inner scope failed

Outer State:
    ACTIVE

Commit Allowed:
    no
```

---

# 154. Suspension diagnostic

```text
TRANSACTION SUSPENDED

Outer:
    tx-100

Outer Connection:
    connection-4

Inner:
    tx-101

Propagation:
    REQUIRES_NEW

Inner Connection:
    connection-8
```

---

# 155. Explain API

Podrá existir:

```php
DB::transactions()->explainCurrent();
```

Resultado conceptual:

```text
Depth: 3

scope-1
  REQUIRED
  physical owner
  tx-20

scope-2
  REQUIRED
  joined tx-20

scope-3
  NESTED
  savepoint participant
  sp-2
```

---

# 156. Error hierarchy

```text
DatabaseNestedTransactionException
│
├── TransactionRequiredException
├── ExistingTransactionNotAllowedException
├── UnsupportedNestedTransactionException
├── TransactionNestingDepthExceededException
├── TransactionPropagationException
├── NestedTransactionCompatibilityException
│   ├── NestedTransactionIsolationConflictException
│   ├── NestedTransactionReadModeConflictException
│   ├── NestedTransactionTimeoutConflictException
│   └── NestedTransactionContextConflictException
├── TransactionSuspensionException
├── SuspendedTransactionResumeException
├── TransactionRollbackOnlyException
├── UnexpectedRollbackException
├── TransactionScopeStateException
├── TransactionScopeOwnershipException
├── TransactionScopeStackException
├── NestedTransactionResourceException
└── NestedTransactionInvariantViolationException
```

---

# 157. TransactionRequiredException

Se produce cuando:

```text
MANDATORY
+
no active transaction
```

---

# 158. ExistingTransactionNotAllowedException

Se produce cuando:

```text
NEVER
+
active transaction
```

---

# 159. UnsupportedNestedTransactionException

Ejemplo:

```text
NESTED
+
platform without savepoint capability
```

---

# 160. Ownership exception

Se produce si un joined scope intenta:

```text
physical commit
```

o:

```text
physical rollback
```

sin ser owner.

---

# 161. Scope stack exception

Detectará:

```text
pop wrong scope
resume wrong transaction
corrupted nesting order
```

---

# 162. Resource exception

Podrá representar:

```text
no independent connection available
max suspended transactions exceeded
savepoint resources exhausted
```

---

# 163. Security

Nested transactions no deberán permitir que datos externos controlen directamente:

```text
savepoint SQL
connection identifier
physical transaction identifier
```

---

# 164. No user-provided savepoint SQL

Prohibido:

```php
DB::transaction(
    savepoint: $_GET['savepoint_sql']
);
```

---

# 165. No arbitrary connection switching

Un joined scope no podrá cambiar de conexión mediante input de usuario y seguir pretendiendo pertenecer a la misma transacción.

---

# 166. Context integrity

El `TransactionContext` deberá ser creado y administrado por infraestructura confiable.

No aceptar:

```php
new TransactionContext(
    transactionId: $_POST['tx']
);
```

como mecanismo para recuperar una transacción física.

---

# 167. Transaction contexts are not transferable tokens

```text
TransactionId
```

no será un token para reanudar una transacción desde otra request.

---

# 168. HTTP boundary

Una transacción activa no deberá sobrevivir implícitamente:

```text
Request A
→ Request B
```

---

# 169. Async boundary

Tampoco:

```text
Transaction Context
→ serialized into queue job
```

para continuar físicamente la misma transacción.

---

# 170. Queue jobs

Un job deberá iniciar su propia transacción.

Podrá recibir:

```text
business correlation ID
entity identifiers
```

pero no un active physical transaction handle.

---

# 171. Resource governance

Configuración propuesta:

```php
'transactions' => [
    'propagation' => [
        'default' => 'required',
    ],

    'nesting' => [
        'max_depth' => 32,
        'max_savepoints' => 32,
        'max_suspended' => 8,
    ],
];
```

---

# 172. Default propagation

VoltStack utilizará:

```text
REQUIRED
```

como default general.

Esto favorece:

```text
service composition
```

sin crear nuevas transacciones físicas innecesariamente.

---

# 173. Why REQUIRED default

Permite:

```php
OrderService::create()
```

usar una transacción por sí mismo y también participar correctamente cuando es llamado desde:

```php
CheckoutService::checkout()
```

---

# 174. Example service composition

```php
final class CheckoutService
{
    public function checkout(): void
    {
        DB::transaction(function () {
            $this->orders->create();
            $this->payments->record();
        });
    }
}
```

Ambos servicios internos pueden utilizar:

```text
REQUIRED
```

y compartir la misma physical TX.

---

# 175. Independent audit example

Un caso potencial de:

```text
REQUIRES_NEW
```

sería un registro que deba persistir independientemente.

```php
DB::transaction(function () {
    try {
        $this->processPayment();
    } catch (\Throwable $e) {
        DB::transaction(
            fn () => $this->auditFailure($e),
            propagation: TransactionPropagation::REQUIRES_NEW,
        );

        throw $e;
    }
});
```

Pero deberá considerarse cuidadosamente:

```text
connection availability
retry semantics
idempotency
outer rollback
```

---

# 176. NOT_SUPPORTED example

```php
DB::transaction(function () {
    updateRecords();

    DB::run(
        callback: fn () => performOperationOutsideTransaction(),
        propagation: TransactionPropagation::NOT_SUPPORTED,
    );
});
```

Conceptualmente:

```text
TX-A suspended
    ↓
non-transactional scope
    ↓
TX-A resumed
```

---

# 177. MANDATORY example

Un servicio que solo debe ejecutarse dentro de una transacción:

```php
$transactionManager->execute(
    $definition->withPropagation(
        TransactionPropagation::MANDATORY
    ),
    fn () => $this->updateLedger(),
);
```

---

# 178. NEVER example

Operación incompatible con una transacción activa:

```text
Propagation:
    NEVER
```

Si existe TX:

```text
ExistingTransactionNotAllowedException
```

---

# 179. NESTED example

```php
DB::transaction(function () {
    importHeader();

    try {
        DB::transaction(
            fn () => importOptionalDetails(),
            propagation: TransactionPropagation::NESTED,
        );
    } catch (OptionalImportException) {
        // outer may continue if state is safe
    }

    finalizeImport();
});
```

Conceptualmente:

```text
BEGIN
    importHeader

    SAVEPOINT
        importOptionalDetails
    ROLLBACK TO SAVEPOINT

    finalizeImport
COMMIT
```

---

# 180. Transaction state architecture

```text
PhysicalTransactionContext
│
├── PhysicalTransactionId
├── ConnectionLease
├── EffectiveIsolation
├── ReadMode
├── Deadline
├── RollbackOnlyState
├── State
└── LogicalScopes
```

---

# 181. Logical scope architecture

```text
TransactionScope
│
├── ScopeId
├── ParentScopeId
├── Depth
├── Propagation
├── Participation
├── Ownership
├── RequestedDefinition
├── EffectiveRestrictions
├── State
└── Optional Savepoint
```

---

# 182. Context hierarchy

```text
Operation Context
│
└── Transaction Scope Stack
    │
    ├── Scope 1
    │   └── Physical TX-A
    │
    ├── Scope 2
    │   └── joins TX-A
    │
    ├── Scope 3
    │   └── savepoint in TX-A
    │
    └── Scope 4
        └── Physical TX-B
```

---

# 183. TransactionManager responsibilities

`TransactionManager` coordinará:

- context;
- propagation;
- ownership;
- lifecycle;
- commit;
- rollback;
- suspension;
- resume.

Pero delegará:

```text
Propagation decisions
→ PropagationResolver

Savepoints
→ SavepointSystem

Isolation
→ IsolationSystem

Retry
→ RetrySystem

Connection leases
→ ConnectionManager
```

---

# 184. Avoid TransactionManager God Object

No deberá contener directamente toda la lógica de:

```text
isolation
savepoint SQL
retry classification
deadlock parsing
driver syntax
ORM checkpointing
```

---

# 185. Proposed directory structure

```text
src/Quantum/Database/Transaction/
│
├── Nested/
│   ├── TransactionScope.php
│   ├── TransactionScopeId.php
│   ├── TransactionScopeState.php
│   ├── TransactionParticipation.php
│   ├── TransactionScopeOwnership.php
│   │
│   ├── Propagation/
│   │   ├── TransactionPropagation.php
│   │   ├── TransactionPropagationResolver.php
│   │   ├── TransactionPropagationDecision.php
│   │   └── TransactionPhysicalAction.php
│   │
│   ├── Context/
│   │   ├── TransactionScopeFrame.php
│   │   ├── TransactionScopeStack.php
│   │   ├── TransactionScopeStackSnapshot.php
│   │   └── TransactionContextAccessor.php
│   │
│   ├── Compatibility/
│   │   ├── NestedTransactionCompatibilityValidator.php
│   │   ├── IsolationCompatibilityValidator.php
│   │   ├── ReadModeCompatibilityValidator.php
│   │   ├── TimeoutCompatibilityValidator.php
│   │   └── DatabaseContextCompatibilityValidator.php
│   │
│   ├── Suspension/
│   │   ├── SuspendedTransaction.php
│   │   ├── TransactionSuspender.php
│   │   └── TransactionResumer.php
│   │
│   ├── RollbackOnly/
│   │   ├── RollbackOnlyState.php
│   │   ├── RollbackOnlyReason.php
│   │   └── RollbackOnlyRegistry.php
│   │
│   ├── Resource/
│   │   ├── TransactionNestingPolicy.php
│   │   └── TransactionResourceGuard.php
│   │
│   ├── Diagnostics/
│   │   ├── NestedTransactionInspector.php
│   │   └── TransactionScopeDiagnostic.php
│   │
│   └── Exception/
│       └── ...
```

---

# 186. Dependency rules

Permitido:

```text
Nested Transaction System
    ↓
Transaction Context contracts
Transaction Manager contracts
Isolation contracts
Connection lease contracts
Savepoint contracts
Runtime scope contracts
Telemetry contracts
```

No permitido:

```text
Nested Transaction System
    ↓
PDO directly
vendor SQL
HTTP globals
Entity implementation
Repository implementation
static current transaction
```

---

# 187. Architectural invariants

## DB-NTX-001
Toda llamada transaccional creará un scope lógico.

## DB-NTX-002
Logical scope y physical transaction serán conceptos diferentes.

## DB-NTX-003
Múltiples scopes podrán compartir una transacción física.

## DB-NTX-004
Solo el physical owner podrá realizar commit físico.

## DB-NTX-005
Un joined scope no realizará commit físico.

## DB-NTX-006
Un joined scope no realizará rollback físico arbitrariamente.

## DB-NTX-007
Un joined failure podrá marcar rollback-only.

## DB-NTX-008
Rollback-only impedirá commit exitoso.

## DB-NTX-009
Outer owner deberá observar rollback-only antes del commit.

## DB-NTX-010
Unexpected rollback será explícito.

## DB-NTX-011
REQUIRED creará TX si no existe.

## DB-NTX-012
REQUIRED participará si existe TX compatible.

## DB-NTX-013
REQUIRES_NEW creará una nueva transacción física.

## DB-NTX-014
REQUIRES_NEW no se simulará con savepoint.

## DB-NTX-015
REQUIRES_NEW suspenderá la transacción existente.

## DB-NTX-016
SUPPORTS participará cuando exista TX.

## DB-NTX-017
SUPPORTS podrá ejecutar sin TX cuando no exista.

## DB-NTX-018
MANDATORY requerirá una TX existente.

## DB-NTX-019
MANDATORY sin TX fallará.

## DB-NTX-020
NOT_SUPPORTED suspenderá la TX existente.

## DB-NTX-021
NOT_SUPPORTED ejecutará fuera de TX.

## DB-NTX-022
NEVER rechazará una TX existente.

## DB-NTX-023
NESTED utilizará savepoint cuando exista TX.

## DB-NTX-024
NESTED sin TX podrá crear una TX física.

## DB-NTX-025
NESTED no degradará silenciosamente a REQUIRED.

## DB-NTX-026
Savepoint no será una transacción física independiente.

## DB-NTX-027
Savepoint no tendrá isolation independiente.

## DB-NTX-028
Savepoint release no será commit.

## DB-NTX-029
Outer rollback revertirá cambios de savepoints liberados.

## DB-NTX-030
REQUIRES_NEW commit sobrevivirá a outer rollback.

## DB-NTX-031
Scope completion no implicará physical commit.

## DB-NTX-032
Scope state y physical transaction state serán distintos.

## DB-NTX-033
Transaction scope stack será execution-local.

## DB-NTX-034
No existirá static mutable transaction stack.

## DB-NTX-035
No existirá global mutable current transaction.

## DB-NTX-036
FrankenPHP requests tendrán stacks aislados.

## DB-NTX-037
RoadRunner operations tendrán stacks aislados.

## DB-NTX-038
OpenSwoole coroutines tendrán stacks aislados.

## DB-NTX-039
Scope push ocurrirá antes del callback.

## DB-NTX-040
Scope pop será exception-safe.

## DB-NTX-041
Stack ordering será LIFO.

## DB-NTX-042
Pop de scope incorrecto será error de invariante.

## DB-NTX-043
Suspension no equivaldrá a commit.

## DB-NTX-044
Suspension no equivaldrá a rollback.

## DB-NTX-045
Una suspended TX conservará su connection lease.

## DB-NTX-046
Una suspended connection no volverá al pool libre.

## DB-NTX-047
REQUIRES_NEW requerirá una conexión físicamente utilizable.

## DB-NTX-048
No se reutilizará la misma active transactional connection para una independent TX portable.

## DB-NTX-049
Resume ocurrirá después de terminar el inner scope.

## DB-NTX-050
Resume será exception-safe.

## DB-NTX-051
Suspension podrá anidarse.

## DB-NTX-052
Suspension utilizará stack.

## DB-NTX-053
Resume failure será explícito.

## DB-NTX-054
Resume uncertainty podrá taintar el contexto.

## DB-NTX-055
Nesting depth tendrá límites.

## DB-NTX-056
Suspension depth tendrá límites.

## DB-NTX-057
Savepoint count tendrá límites.

## DB-NTX-058
Resource exhaustion no será ignorado.

## DB-NTX-059
Propagation resolution precederá ejecución.

## DB-NTX-060
Compatibility validation precederá participación.

## DB-NTX-061
Joined scope heredará physical isolation.

## DB-NTX-062
Joined scope no podrá elevar isolation físico activo.

## DB-NTX-063
Isolation compatibility será semántica.

## DB-NTX-064
Isolation compatibility no utilizará ordinales simples.

## DB-NTX-065
EXACT conservará semántica exacta.

## DB-NTX-066
AT_LEAST podrá aceptar outer más fuerte cuando sea semánticamente válido.

## DB-NTX-067
READ_ONLY outer rechazará READ_WRITE inner joined.

## DB-NTX-068
READ_WRITE outer podrá imponer scope READ_ONLY lógico.

## DB-NTX-069
Logical read restriction será distinta de physical read mode.

## DB-NTX-070
Inner joined timeout no extenderá outer deadline.

## DB-NTX-071
Joined effective deadline será como máximo outer deadline.

## DB-NTX-072
REQUIRES_NEW podrá tener deadline físico independiente.

## DB-NTX-073
Joined scope no cambiará tenant.

## DB-NTX-074
Joined scope no cambiará database context.

## DB-NTX-075
Joined scope no cambiará shard.

## DB-NTX-076
Cross-shard join no se fingirá como ACID.

## DB-NTX-077
Distributed transaction support no será implícito.

## DB-NTX-078
Joined scope utilizará la active transaction connection.

## DB-NTX-079
Joined query routing conservará connection affinity.

## DB-NTX-080
Savepoint SQL será responsabilidad del Savepoint System.

## DB-NTX-081
Nested System decidirá intención de savepoint, no vendor SQL.

## DB-NTX-082
Savepoint names serán internos.

## DB-NTX-083
Savepoint names no se derivarán directamente de input externo.

## DB-NTX-084
DB rollback to savepoint no implicará ORM rewind.

## DB-NTX-085
ORM uncertainty después de nested rollback será explícita.

## DB-NTX-086
EntityManager podrá ser marcado TAINTED.

## DB-NTX-087
UnitOfWork no será automáticamente restaurado por DB savepoint.

## DB-NTX-088
IdentityMap no será automáticamente restaurado por DB savepoint.

## DB-NTX-089
Snapshots ORM no serán automáticamente restaurados por DB savepoint.

## DB-NTX-090
REQUIRES_NEW podrá requerir PersistenceContext independiente.

## DB-NTX-091
IdentityMap guarantee será context-local.

## DB-NTX-092
Managed entities no cruzarán automáticamente persistence contexts.

## DB-NTX-093
Identificadores serán preferibles para transferencias entre contexts.

## DB-NTX-094
flush no significará commit.

## DB-NTX-095
Inner joined flush no realizará physical commit.

## DB-NTX-096
Outer rollback revertirá inner joined flushed SQL.

## DB-NTX-097
No habrá implicit flush al salir de nested scope.

## DB-NTX-098
Retry boundary será distinto de logical scope boundary.

## DB-NTX-099
Joined inner scope no reiniciará independientemente la TX completa por default.

## DB-NTX-100
Physical owner será el retry boundary base.

## DB-NTX-101
Savepoint retries requerirán política explícita.

## DB-NTX-102
REQUIRES_NEW podrá tener retry independiente.

## DB-NTX-103
Inner independent commit será considerado al reintentar outer.

## DB-NTX-104
Nested System no decidirá deadlock retry.

## DB-NTX-105
Nested System no decidirá serialization retry.

## DB-NTX-106
afterCommit se asociará al physical transaction correspondiente.

## DB-NTX-107
Joined scope completion no disparará physical afterCommit.

## DB-NTX-108
Savepoint release no disparará physical afterCommit.

## DB-NTX-109
REQUIRES_NEW commit podrá disparar su propio afterCommit.

## DB-NTX-110
Scope events serán distintos de physical transaction events.

## DB-NTX-111
Telemetry distinguirá scopes de physical transactions.

## DB-NTX-112
Metrics evitarán IDs de alta cardinalidad.

## DB-NTX-113
Diagnostics mostrarán propagation.

## DB-NTX-114
Diagnostics mostrarán participation.

## DB-NTX-115
Diagnostics mostrarán ownership.

## DB-NTX-116
Diagnostics mostrarán nesting depth.

## DB-NTX-117
Diagnostics mostrarán physical transaction identity cuando sea seguro.

## DB-NTX-118
Rollback-only reason será diagnosticable.

## DB-NTX-119
Callback failure y cleanup failure podrán preservarse simultáneamente.

## DB-NTX-120
Cleanup failure no ocultará silenciosamente primary failure.

## DB-NTX-121
TransactionId no será un resumable token.

## DB-NTX-122
TransactionContext no cruzará HTTP requests.

## DB-NTX-123
TransactionContext no se serializará a queue jobs.

## DB-NTX-124
Queue jobs crearán nuevas physical transactions.

## DB-NTX-125
Current transaction resolution será scope-aware.

## DB-NTX-126
Transaction Manager no dependerá de HTTP globals.

## DB-NTX-127
Nested System no dependerá directamente de PDO.

## DB-NTX-128
Nested System no contendrá vendor SQL.

## DB-NTX-129
Nested System no será un segundo Transaction Manager.

## DB-NTX-130
Propagation Resolver será un componente separado.

## DB-NTX-131
Compatibility Validator será un componente separado.

## DB-NTX-132
Suspension/Resume será un componente separado.

## DB-NTX-133
Savepoint operations serán delegadas.

## DB-NTX-134
Isolation resolution será delegada.

## DB-NTX-135
Connection acquisition será delegada.

## DB-NTX-136
Retry será delegado.

## DB-NTX-137
Rollback-only será compartido por scopes de la misma physical TX.

## DB-NTX-138
Independent TX tendrá rollback-only state independiente.

## DB-NTX-139
Savepoint scope podrá recuperarse sin marcar outer rollback-only cuando la recuperación sea demostrablemente segura.

## DB-NTX-140
UNKNOWN recovery state nunca será tratado como clean.

## DB-NTX-141
Propagation default será REQUIRED.

## DB-NTX-142
REQUIRES_NEW será explícito.

## DB-NTX-143
NESTED será explícito.

## DB-NTX-144
No se inferirá REQUIRES_NEW a partir de nesting.

## DB-NTX-145
No se inferirá NESTED a partir de nesting.

## DB-NTX-146
Una llamada transaction dentro de transaction no implicará automáticamente savepoint.

## DB-NTX-147
Propagation definirá la semántica.

## DB-NTX-148
Physical ownership definirá commit authority.

## DB-NTX-149
Scope ownership definirá savepoint authority.

## DB-NTX-150
Cuando VoltStack no pueda demostrar una recuperación segura conservará explícitamente la incertidumbre.

---

# 188. Anti-patterns

## 188.1 Contador de nesting como única implementación

```php
$depth++;

if ($depth === 1) {
    begin();
}
```

Esto no modela:

- propagation;
- savepoints;
- suspension;
- ownership;
- rollback-only;
- independent transactions.

**Rechazado.**

---

## 188.2 Inner commit físico

```php
DB::transaction(function () {
    DB::transaction(function () {
        commit();
    });
});
```

**Rechazado para joined scopes.**

---

## 188.3 Savepoint como REQUIRES_NEW

```text
REQUIRES_NEW
→ SAVEPOINT
```

**Prohibido.**

---

## 188.4 Catch elimina rollback-only

```php
try {
    innerRequired();
} catch (\Throwable) {
    // transaction magically clean again
}
```

**Rechazado.**

---

## 188.5 Compartir misma conexión para dos TX independientes

```text
Connection A
├── TX outer active
└── TX inner independent
```

como abstracción portable.

**Rechazado.**

---

## 188.6 Global stack

```php
static array $transactions;
```

**Prohibido.**

---

## 188.7 Savepoint rollback restaura automáticamente ORM

```text
ROLLBACK TO SAVEPOINT
→ magically rewind every PHP object
```

**Rechazado.**

---

## 188.8 Inner timeout extiende outer

```text
Outer deadline: 10s
Inner timeout: 60s
→ outer now has 60s
```

**Prohibido.**

---

## 188.9 Cambiar tenant en joined scope

```text
Outer Tenant A
Inner REQUIRED Tenant B
```

**Rechazado.**

---

## 188.10 Continuar ante estado UNKNOWN

```text
resume failed
→ ignore
→ continue queries
```

**Prohibido.**

---

# 189. Testing

La suite deberá cubrir:

```text
PropagationResolverTests
RequiredPropagationTests
RequiresNewPropagationTests
SupportsPropagationTests
MandatoryPropagationTests
NotSupportedPropagationTests
NeverPropagationTests
NestedPropagationTests
RollbackOnlyTests
ScopeOwnershipTests
TransactionStackTests
SuspensionResumeTests
NestedIsolationCompatibilityTests
NestedReadModeTests
NestedTimeoutTests
NestedTenantContextTests
SavepointIntegrationTests
ORMNestedTransactionTests
RetryBoundaryTests
PersistentRuntimeTests
CoroutineIsolationTests
ResourceGovernanceTests
FailureInjectionTests
```

---

# 190. REQUIRED tests

Casos mínimos:

```text
no existing TX → creates physical TX
existing compatible TX → joins
inner success → no commit
inner failure → rollback-only
outer success + rollback-only → rollback
```

---

# 191. REQUIRES_NEW tests

```text
outer active
→ suspend
→ lease second connection
→ begin inner
→ commit/rollback
→ release inner
→ resume outer
```

Verificar también failure en cada transición.

---

# 192. NESTED tests

```text
outer active
→ create savepoint
→ inner success
→ release
```

y:

```text
outer active
→ create savepoint
→ inner failure
→ rollback to savepoint
```

---

# 193. Failure injection

Inyectar fallos en:

```text
suspend
connection lease
BEGIN inner
SAVEPOINT
callback
RELEASE SAVEPOINT
ROLLBACK TO SAVEPOINT
COMMIT inner
ROLLBACK inner
resume
cleanup
```

---

# 194. Rollback-only test

```text
Outer
    Inner REQUIRED throws
    Outer catches

Outer attempts commit
```

Esperado:

```text
physical rollback
+
UnexpectedRollbackException
```

---

# 195. Independent commit test

```text
Outer TX
    Inner REQUIRES_NEW commits
Outer rolls back
```

Verificar:

```text
inner data remains
outer data reverted
```

---

# 196. Savepoint outer rollback test

```text
Outer TX
    Nested savepoint succeeds
Outer rolls back
```

Verificar:

```text
all DB changes reverted
```

---

# 197. Persistent worker test

Secuencia:

```text
Request A:
    nesting depth 5

Request B:
    no transaction
```

B deberá observar:

```text
depth = 0
current = null
suspended = none
```

---

# 198. Coroutine test

Ejecutar simultáneamente:

```text
Coroutine A:
    REQUIRED → NESTED

Coroutine B:
    REQUIRES_NEW
```

Los stacks deberán permanecer completamente independientes.

---

# 199. Resource exhaustion test

Forzar:

```text
max nesting depth
connection pool exhaustion
max suspended contexts
max savepoints
```

y verificar fallos deterministas.

---

# 200. Master formulas

## Propagation

```text
Participation
=
ResolvePropagation(
    RequestedPropagation,
    CurrentTransactionState
)
```

## Joined connection

```text
Connection(JoinedScope)
=
Connection(PhysicalTransaction)
```

## Joined isolation

```text
Isolation(JoinedScope)
=
Isolation(PhysicalTransaction)
```

## Joined deadline

```text
Deadline(JoinedScope)
=
min(
    ParentPhysicalDeadline,
    RequestedScopeDeadline
)
```

## Rollback-only

```text
JoinedFailure
→
PhysicalTransaction.RollbackOnly
```

salvo una política explícita que pueda demostrar recuperación.

## Savepoint

```text
NestedSavepointScope
⊂
ParentPhysicalTransaction
```

## Requires New

```text
PhysicalTransaction(Inner REQUIRES_NEW)
≠
PhysicalTransaction(Outer)
```

---

# 201. Modelo final

```text
                    Transaction Request
                            │
                            ▼
                  Current Context Lookup
                            │
                            ▼
                  Propagation Resolver
                            │
          ┌─────────────────┼─────────────────────┐
          │                 │                     │
          ▼                 ▼                     ▼
      No Current        Current TX            Reject
          │                 │
          │        ┌────────┼──────────────┐
          │        │        │              │
          ▼        ▼        ▼              ▼
        BEGIN     JOIN   SAVEPOINT      SUSPEND
                   │        │              │
                   │        │        ┌─────┴─────┐
                   │        │        ▼           ▼
                   │        │      BEGIN      NO TX
                   │        │
                   ▼        ▼
             Logical Scope Stack
                   │
                   ▼
               Callback
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Success            Failure
          │                 │
          ▼                 ▼
   Scope Completion    Failure Policy
          │                 │
          └────────┬────────┘
                   ▼
           Physical Action
                   │
       ┌───────────┼────────────┐
       ▼           ▼            ▼
     COMMIT      ROLLBACK    SAVEPOINT ACTION
                   │
                   ▼
             Resume Outer
                   │
                   ▼
                Cleanup
```

---

# 202. Decisiones arquitectónicas finales

VoltStack utilizará un modelo de:

```text
Logical Transaction Scopes
+
Explicit Propagation
+
Physical Transaction Ownership
```

La API no interpretará toda llamada anidada como una nueva transacción física.

El default será:

```text
TransactionPropagation::REQUIRED
```

permitiendo composición de servicios sobre una única transacción física.

Se soportarán:

```text
REQUIRED
REQUIRES_NEW
SUPPORTS
MANDATORY
NOT_SUPPORTED
NEVER
NESTED
```

con semántica explícita.

La distinción principal será:

```text
Logical Scope
≠
Physical Transaction
≠
Savepoint
```

`REQUIRES_NEW` siempre significará una transacción física independiente y nunca será simulado mediante savepoint.

`NESTED` utilizará savepoints cuando exista una transacción y la plataforma lo soporte.

Los scopes joined no tendrán autoridad para realizar commit físico.

Sus fallos podrán propagar:

```text
ROLLBACK_ONLY
```

al physical transaction owner.

Los savepoints permitirán rollback parcial de estado de base de datos, pero:

```text
Database Savepoint Rollback
≠
ORM State Rewind
```

por lo que UnitOfWork, IdentityMap y EntityManager deberán manejar explícitamente cualquier incertidumbre.

Las transacciones suspendidas conservarán su connection lease y no devolverán conexiones activas al pool.

El estado transaccional será:

```text
request-local
operation-local
job-local
coroutine-local
```

y nunca global mutable.

Esto permitirá funcionamiento seguro sobre:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

La regla final será:

> **VoltStack tratará cada transacción anidada como un scope lógico cuya semántica física será resuelta explícitamente mediante propagation; ningún scope podrá adquirir más autoridad, aislamiento, independencia o capacidad de rollback de la que realmente proporcione la transacción física subyacente.**

---

# 203. Relación con el bloque Transaction & Concurrency

```text
164_DATABASE_TRANSACTION_ARCHITECTURE
                ↓
165_DATABASE_TRANSACTION_MANAGER_SYSTEM
                ↓
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM
                ↓
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM
                ↓
168_DATABASE_NESTED_TRANSACTION_SYSTEM
                ↓
169_DATABASE_SAVEPOINT_SYSTEM
                ↓
170_DATABASE_TRANSACTION_RETRY_SYSTEM
                ↓
171_DATABASE_DEADLOCK_HANDLING_SYSTEM
                ↓
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM
                ↓
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM
                ↓
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM
                ↓
175_DATABASE_TRANSACTION_EVENT_SYSTEM
```

---

# 204. Siguiente documento

```text
169_DATABASE_SAVEPOINT_SYSTEM.md
```

El siguiente documento deberá definir:

- arquitectura formal de savepoints;
- `SavepointId`;
- `SavepointHandle`;
- savepoint ownership;
- naming;
- lifecycle;
- create/release/rollback;
- nested savepoints;
- savepoint stacks;
- capability model;
- platform differences;
- savepoint SQL compilation;
- state validation;
- rollback-to-savepoint semantics;
- transaction rollback-only interaction;
- ORM/UoW implications;
- persistence checkpoints;
- failure semantics;
- UNKNOWN state;
- connection affinity;
- resource governance;
- telemetry;
- diagnostics;
- testing;
- persistent-runtime safety;
- invariantes arquitectónicas.