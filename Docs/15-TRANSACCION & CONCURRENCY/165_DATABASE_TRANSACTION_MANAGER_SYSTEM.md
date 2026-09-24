# 165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md

# VoltStack Quantum Database
## Database Transaction Manager System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 165 — Database Transaction Manager System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `164_DATABASE_TRANSACTION_ARCHITECTURE.md`  
**Siguiente documento:** `166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Manager System` define el componente encargado de coordinar el ciclo de vida operativo de las transacciones de base de datos en VoltStack.

El `TransactionManager` será la autoridad responsable de transformar solicitudes como:

```text
begin
commit
rollback
transactional
```

en transiciones transaccionales válidas sobre una conexión correctamente adquirida, fijada y aislada.

Su regla central será:

> **El Transaction Manager de VoltStack coordina el boundary, ownership, estado, conexión y finalización de una transacción; no ejecuta lógica ORM, no genera SQL de negocio y nunca declara COMMITTED o ROLLED_BACK sin evidencia suficiente del resultado correspondiente.**

Por tanto:

```text
TransactionManager
=
Transaction Lifecycle Coordinator
```

y no:

```text
TransactionManager
=
Connection
=
Driver
=
UnitOfWork
=
EntityManager
=
QueryExecutor
```

---

# 2. Relación con la arquitectura anterior

El documento `164_DATABASE_TRANSACTION_ARCHITECTURE.md` definió:

```text
Application
     ↓
Transaction API
     ↓
Transaction Manager
     ↓
Transaction Context
     ↓
Connection Manager
     ↓
Pinned Connection
     ↓
Driver
     ↓
Database
```

Este documento profundiza exclusivamente en:

```text
Transaction Manager
```

El siguiente documento profundizará en:

```text
Transaction Context
```

---

# 3. Responsabilidades principales

El `TransactionManager` deberá coordinar:

1. creación de transacciones;
2. adquisición de conexión;
3. connection pinning;
4. ownership;
5. activación del contexto;
6. inicio físico;
7. commit;
8. rollback;
9. rollback-only;
10. nested transaction coordination;
11. propagation;
12. transaction callbacks;
13. completion;
14. error precedence;
15. cleanup;
16. connection release;
17. connection quarantine;
18. retry entry points;
19. telemetry;
20. diagnostics;
21. runtime isolation.

---

# 4. No responsabilidades

El `TransactionManager` no deberá:

- construir SQL;
- ejecutar queries de negocio;
- hidratar entidades;
- detectar cambios ORM;
- crear ChangeSets;
- planificar inserts;
- planificar updates;
- resolver relaciones;
- administrar IdentityMap;
- decidir reglas de dominio;
- serializar resultados;
- validar HTTP input;
- administrar credenciales directamente.

---

# 5. Modelo conceptual

```text
                    TransactionManager
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
       ▼                   ▼                    ▼
 TransactionContext   StateMachine       ConnectionManager
       │                   │                    │
       ▼                   ▼                    ▼
   Ownership           Lifecycle          Acquire / Pin
       │                   │                    │
       └─────────────┬─────┴──────────────┬─────┘
                     │                    │
                     ▼                    ▼
              Platform Adapter       Driver Adapter
                     │                    │
                     └──────────┬─────────┘
                                ▼
                             DATABASE
```

Integraciones laterales:

```text
TransactionManager
├── Retry Coordinator
├── Callback Registry
├── Event Dispatcher
├── Telemetry
├── Diagnostics
└── Runtime Scope
```

---

# 6. Contrato principal

Contrato conceptual:

```php
interface TransactionManager
{
    public function begin(
        ?TransactionDefinition $definition = null,
    ): Transaction;

    public function commit(
        Transaction $transaction,
    ): TransactionResult;

    public function rollback(
        Transaction $transaction,
    ): TransactionResult;

    public function transactional(
        callable $callback,
        ?TransactionDefinition $definition = null,
    ): mixed;

    public function current(): ?TransactionContext;

    public function isActive(): bool;

    public function markRollbackOnly(
        ?Transaction $transaction = null,
    ): void;
}
```

---

# 7. API pública vs implementación

Se distinguirá:

```text
TransactionManager
```

contrato público,

de:

```text
DefaultTransactionManager
```

implementación estándar.

Esto permitirá:

- testing;
- custom runtimes;
- instrumentation;
- futuras estrategias distribuidas;
- extensiones especializadas.

---

# 8. Transaction object

`begin()` devolverá un objeto transacción explícito:

```php
interface Transaction
{
    public function id(): TransactionId;

    public function state(): TransactionState;

    public function outcome(): ?TransactionOutcome;

    public function isActive(): bool;

    public function isRollbackOnly(): bool;
}
```

El objeto no deberá exponer directamente:

```text
PDO
mysqli
pgsql resource
driver internals
```

---

# 9. Transaction handle

La transacción retornada actuará principalmente como:

```text
Transaction Handle
```

sobre el estado administrado por el manager/context.

No deberá contener un segundo estado transaccional divergente.

---

# 10. Single source of truth

La autoridad de estado será:

```text
TransactionContext
+
TransactionStateMachine
```

No deberán existir simultáneamente:

```text
Transaction::$state
Context::$state
Connection::$transactionState
```

mutándose independientemente.

---

# 11. Begin lifecycle

Pipeline general:

```text
begin(definition)
       ↓
validate runtime scope
       ↓
resolve propagation
       ↓
resolve existing transaction
       ↓
normalize definition
       ↓
resolve platform capabilities
       ↓
acquire connection
       ↓
acquire ownership
       ↓
pin connection
       ↓
create context
       ↓
state NEW → STARTING
       ↓
apply transaction configuration
       ↓
driver BEGIN
       ↓
state STARTING → ACTIVE
       ↓
publish TransactionStarted
       ↓
return Transaction
```

---

# 12. Begin preconditions

Antes de iniciar, el manager deberá verificar:

```text
valid runtime scope
manager operational
connection manager available
transaction definition valid
propagation rules valid
platform supports transaction
```

---

# 13. TransactionDefinition normalization

El usuario podrá especificar parcialmente:

```php
new TransactionDefinition(
    isolation: TransactionIsolation::SERIALIZABLE,
    readOnly: false,
);
```

Los campos restantes se resolverán mediante:

```text
explicit request
    ↓
database configuration
    ↓
transaction defaults
    ↓
platform defaults
```

---

# 14. Default definition

Podrá existir:

```php
final readonly class TransactionDefaults
{
    public function __construct(
        public TransactionIsolation $isolation,
        public bool $readOnly,
        public TransactionPropagation $propagation,
        public ?TransactionTimeout $timeout,
        public ?RetryPolicyReference $retryPolicy,
    ) {}
}
```

---

# 15. Defaults ≠ effective configuration

Distinguir:

```text
RequestedTransactionDefinition
```

de:

```text
EffectiveTransactionDefinition
```

La segunda será resultado de capability resolution.

---

# 16. Capability validation

Antes del `BEGIN`:

```text
Definition
   ↓
TransactionCapabilityResolver
   ↓
Supported?
```

Ejemplo:

```text
SERIALIZABLE requested
        ↓
Platform supports?
        ↓
yes → continue
no  → explicit policy
```

---

# 17. No silent downgrade

Nunca:

```text
SERIALIZABLE
→ unsupported
→ READ_COMMITTED silently
```

Por default deberá fallar.

---

# 18. Existing transaction resolution

Antes de crear una nueva transacción:

```text
TransactionManager
       ↓
TransactionContextResolver
       ↓
Current transaction?
```

La respuesta determinará propagation/nesting.

---

# 19. REQUIRED

Para:

```text
TransactionPropagation::REQUIRED
```

algoritmo:

```text
existing ACTIVE transaction?
    │
    ├── yes → join
    │
    └── no  → create
```

---

# 20. MANDATORY

```text
existing transaction?
    │
    ├── yes → participate
    └── no  → TransactionRequiredException
```

---

# 21. NEVER

```text
existing transaction?
    │
    ├── yes → ExistingTransactionNotAllowedException
    └── no  → execute without transaction
```

---

# 22. SUPPORTS

```text
existing transaction?
    │
    ├── yes → participate
    └── no  → execute without transaction
```

---

# 23. REQUIRES_NEW

Semántica correcta:

```text
Outer Transaction
       ↓
suspend outer context
       ↓
acquire independent connection
       ↓
BEGIN inner transaction
       ↓
complete inner
       ↓
release inner connection
       ↓
resume outer context
```

---

# 24. REQUIRES_NEW initial status

Hasta que VoltStack disponga de suspensión transaccional segura:

```text
REQUIRES_NEW
=
explicitly unsupported
```

podrá ser la política inicial.

Nunca deberá simularse mediante:

```text
SAVEPOINT
```

---

# 25. Transaction ownership

El manager deberá asociar cada transacción con:

```text
TransactionOwner
```

Ejemplos:

```text
RequestScopeId
JobScopeId
CoroutineId
OperationScopeId
```

---

# 26. Ownership validation

Cada operación:

```text
commit
rollback
markRollbackOnly
savepoint
```

deberá validar que el caller se encuentra en un scope compatible.

---

# 27. Cross-scope commit

Caso prohibido:

```text
Coroutine A:
    begin tx-1

Coroutine B:
    commit tx-1
```

Resultado:

```text
TransactionOwnershipException
```

---

# 28. Connection acquisition

El manager no creará conexiones directamente.

Utilizará:

```text
ConnectionManager
```

mediante una solicitud como:

```php
$lease = $connectionManager->acquire(
    ConnectionIntent::TRANSACTION,
    $context,
);
```

---

# 29. Connection lease

Se recomienda modelar:

```text
ConnectionLease
```

para representar ownership temporal de una conexión.

```php
interface ConnectionLease
{
    public function connection(): Connection;

    public function release(): void;

    public function quarantine(): void;
}
```

---

# 30. Why lease

Permite separar:

```text
Connection
```

de:

```text
who currently owns the connection
```

y facilita:

- pooling;
- pinning;
- cleanup;
- quarantine.

---

# 31. Transaction pinning

Una vez adquirida:

```text
TransactionContext
     ↓
ConnectionLease
     ↓
Pinned Connection
```

No podrá cambiar durante la transacción.

---

# 32. Pinning invariant

Formalmente:

```text
∀ operation o ∈ T:

Connection(o) = PinnedConnection(T)
```

para todas las operaciones participantes.

---

# 33. Begin physical transaction

El manager no deberá ejecutar:

```php
$pdo->beginTransaction();
```

directamente.

Usará:

```text
TransactionPlatformAdapter
```

o contrato equivalente.

---

# 34. Adapter contract

```php
interface TransactionPlatformAdapter
{
    public function begin(
        Connection $connection,
        EffectiveTransactionDefinition $definition,
    ): void;

    public function commit(
        Connection $connection,
    ): DriverTransactionOutcome;

    public function rollback(
        Connection $connection,
    ): DriverTransactionOutcome;
}
```

---

# 35. DriverTransactionOutcome

El adapter no deberá limitarse a:

```text
true / false
```

Podrá devolver:

```php
enum DriverTransactionOutcome
{
    case CONFIRMED_SUCCESS;
    case CONFIRMED_FAILURE;
    case CONNECTION_LOST_BEFORE_DISPATCH;
    case CONNECTION_LOST_AFTER_DISPATCH;
    case UNKNOWN;
}
```

---

# 36. Evidence model

El Transaction Manager transformará:

```text
Driver Evidence
+
Platform Semantics
+
Operation Stage
```

en:

```text
TransactionOutcome
```

---

# 37. Begin failure before dispatch

Si `BEGIN` nunca fue enviado:

```text
state:
    FAILED

outcome:
    NOT_STARTED
```

La conexión podrá ser reusable si se conoce limpia.

---

# 38. Begin uncertainty

Si la conexión se pierde durante `BEGIN`:

```text
TransactionState = FAILED / UNKNOWN-start
```

pero no deberá exponerse como una transacción `ACTIVE`.

---

# 39. Begin success

Solo después de confirmación suficiente:

```text
STARTING
→
ACTIVE
```

---

# 40. Activation ordering

Orden recomendado:

```text
acquire connection
pin
configure
BEGIN confirmed
activate context
publish started
```

La implementación deberá evitar que código de aplicación observe una transacción `ACTIVE` antes de que el DB transaction exista realmente.

---

# 41. Commit entry point

```php
$result = $manager->commit($transaction);
```

deberá realizar:

```text
validate handle
validate ownership
validate state
validate current context
check rollback-only
transition COMMITTING
execute beforeCommit callbacks
send COMMIT
classify evidence
set terminal state
execute completion callbacks
cleanup
release/quarantine connection
return TransactionResult
```

---

# 42. Commit preconditions

Estados aceptables normalmente:

```text
ACTIVE
```

No:

```text
NEW
STARTING
COMMITTING
COMMITTED
ROLLING_BACK
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 43. Commit rollback-only

Si:

```text
rollbackOnly = true
```

el manager no deberá enviar `COMMIT`.

Flujo:

```text
commit requested
      ↓
rollback-only detected
      ↓
rollback()
      ↓
throw TransactionRollbackOnlyException
```

---

# 44. beforeCommit

Los callbacks `beforeCommit` ocurren:

```text
ACTIVE
   ↓
beforeCommit
   ↓
COMMITTING
```

o mediante una secuencia equivalente cuidadosamente definida.

Si `beforeCommit` falla antes de enviar COMMIT:

```text
rollback attempt
```

será posible.

---

# 45. Commit dispatch point

El manager deberá rastrear:

```text
CommitDispatchState
```

por ejemplo:

```php
enum CommitDispatchState
{
    case NOT_DISPATCHED;
    case DISPATCHING;
    case DISPATCHED;
    case CONFIRMED;
}
```

Esto es crítico para determinar incertidumbre.

---

# 46. Commit confirmed

Si:

```text
COMMIT confirmed
```

entonces:

```text
TransactionState = COMMITTED
TransactionOutcome = COMMITTED
```

---

# 47. Commit rejected

Si el servidor confirma que commit no ocurrió:

```text
TransactionOutcome
```

deberá reflejar la evidencia concreta.

Puede requerirse rollback/connection disposal según estado.

---

# 48. Commit connection loss

Caso:

```text
send COMMIT
    ↓
network failure
```

Si no puede determinarse si el servidor procesó el comando:

```text
TransactionState = UNKNOWN
TransactionOutcome = UNKNOWN
```

---

# 49. Unknown commit handling

El manager deberá inmediatamente:

```text
mark transaction UNKNOWN
mark persistence context uncertain
quarantine connection
disable normal retry
emit diagnostic event
perform local cleanup
```

---

# 50. Unknown is terminal

No permitir:

```text
UNKNOWN
→ rollback
```

como si el rollback pudiera cambiar retroactivamente un commit posiblemente ejecutado.

---

# 51. No rollback after uncertain commit

Especialmente:

```text
COMMIT dispatched
connection lost
```

no deberá producir automáticamente:

```text
ROLLBACK
```

sobre una nueva conexión.

Eso no resolvería el outcome original.

---

# 52. Rollback entry point

```php
$result = $manager->rollback($transaction);
```

pipeline:

```text
validate
   ↓
ownership
   ↓
state
   ↓
ROLLING_BACK
   ↓
driver rollback
   ↓
classify evidence
   ↓
ROLLED_BACK / UNKNOWN / FAILED
   ↓
callbacks
   ↓
cleanup
```

---

# 53. Rollback known

Con confirmación:

```text
TransactionState = ROLLED_BACK
TransactionOutcome = ROLLED_BACK
```

---

# 54. Rollback uncertainty

Si no puede confirmarse:

```text
TransactionOutcome = UNKNOWN
```

aunque el servidor probablemente haga rollback al cerrar la conexión.

---

# 55. Rollback idempotency

No deberá asumirse que:

```php
rollback();
rollback();
```

es válido.

Por default:

```text
second rollback
→ InvalidTransactionStateException
```

La API podrá ofrecer operaciones de cleanup idempotentes separadas.

---

# 56. transactional()

API principal:

```php
$result = $manager->transactional(
    function () {
        // transactional work
    },
);
```

---

# 57. transactional pipeline

```text
transactional(callback)
        ↓
resolve propagation
        ↓
begin / join
        ↓
execute callback
        ↓
callback success?
    ┌───────┴───────┐
    │               │
   yes              no
    │               │
    ▼               ▼
 commit           rollback
    │               │
    ▼               ▼
 result          rethrow
```

---

# 58. Simplified pseudocode

```php
public function transactional(
    callable $callback,
    ?TransactionDefinition $definition = null,
): mixed {
    $transaction = $this->begin($definition);

    try {
        $result = $callback($transaction);

        $this->commit($transaction);

        return $result;
    } catch (\Throwable $error) {
        $this->rollbackIfPossible($transaction);

        throw $error;
    }
}
```

Pero la implementación real deberá resolver problemas de:

- commit failure;
- rollback failure;
- unknown outcome;
- nested participation;
- retry;
- exception precedence.

---

# 59. Critical catch problem

Este patrón es incorrecto:

```php
try {
    $callback();

    $this->commit();
} catch (\Throwable $e) {
    $this->rollback();
}
```

porque si:

```text
commit()
→ UNKNOWN
```

el catch intentaría rollback incorrectamente.

---

# 60. Correct phase separation

Se distinguirán fases:

```text
CALLBACK
COMMIT
COMPLETION
```

Ejemplo:

```text
callback failure
→ rollback

commit failure
→ classify commit outcome

completion failure
→ preserve DB outcome
```

---

# 61. Transaction operation phases

```php
enum TransactionOperationPhase
{
    case ACQUIRE;
    case BEGIN;
    case WORK;
    case BEFORE_COMMIT;
    case COMMIT;
    case ROLLBACK;
    case COMPLETION;
    case CLEANUP;
}
```

---

# 62. Callback exception

Si el callback de negocio lanza:

```text
DomainException
```

el manager intentará rollback.

---

# 63. Rollback succeeds after callback failure

Resultado:

```text
Database:
    ROLLED_BACK

Application:
    original DomainException rethrown
```

---

# 64. Rollback also fails

Caso:

```text
DomainException
    +
RollbackException
```

se necesita una política de precedencia.

---

# 65. Exception precedence

Nunca deberá perderse el error original.

Se recomienda:

```text
PrimaryException
+
TransactionCompletionException
```

o excepción compuesta.

---

# 66. Composite failure

Modelo:

```php
final class TransactionExecutionException extends DatabaseTransactionException
{
    public function __construct(
        public readonly \Throwable $primaryFailure,
        public readonly ?\Throwable $completionFailure,
        public readonly TransactionResult $transactionResult,
    ) {}
}
```

---

# 67. Original failure preservation

Regla:

> El error que provocó el rollback no será destruido por un error secundario durante rollback o cleanup.

---

# 68. Commit failure precedence

Si el callback terminó correctamente pero commit falla:

```text
commit failure
```

será el error primario.

---

# 69. Commit succeeds, afterCommit fails

Entonces:

```text
DB outcome:
    COMMITTED

completion:
    FAILED
```

y la excepción deberá expresar:

> la transacción sí fue comprometida; falló procesamiento posterior.

---

# 70. TransactionResult

Modelo:

```php
final readonly class TransactionResult
{
    public function __construct(
        public TransactionId $id,
        public TransactionState $state,
        public TransactionOutcome $outcome,
        public TransactionCompletionStatus $completionStatus,
        public int $attempt,
        public ?TransactionFailure $failure,
    ) {}
}
```

---

# 71. Result ≠ callback value

Distinguir:

```text
CallbackResult<T>
```

de:

```text
TransactionResult
```

La API simple podrá devolver `T`.

La API avanzada podrá devolver:

```php
TransactionalResult<T>
```

---

# 72. Advanced result

```php
final readonly class TransactionalResult
{
    public function __construct(
        public mixed $value,
        public TransactionResult $transaction,
    ) {}
}
```

---

# 73. Simple developer API

```php
$order = DB::transaction(function () {
    return Order::create(...);
});
```

---

# 74. Advanced API

```php
$result = DB::transactionResult(function () {
    return Order::create(...);
});

$result->value;
$result->transaction->outcome;
```

---

# 75. Manual begin

```php
$tx = DB::beginTransaction();

try {
    // work

    DB::commit($tx);
} catch (\Throwable $e) {
    if ($tx->isRollbackPossible()) {
        DB::rollBack($tx);
    }

    throw $e;
}
```

---

# 76. Manual transaction safety

La API deberá ofrecer información suficiente para evitar:

```text
rollback after uncertain commit
```

---

# 77. Transaction capabilities on handle

Podría exponerse:

```php
$tx->canCommit();
$tx->canRollback();
$tx->isTerminal();
$tx->isOutcomeKnown();
```

como vistas derivadas del state machine.

---

# 78. No direct state mutation

Nunca:

```php
$transaction->state = TransactionState::COMMITTED;
```

Solo:

```text
TransactionStateMachine
```

podrá producir transiciones.

---

# 79. State machine integration

```text
TransactionManager
       ↓
TransactionStateMachine
       ↓
validate transition
       ↓
apply transition
```

---

# 80. Transition example

```text
ACTIVE
+
commitRequested
=
COMMITTING
```

---

# 81. Invalid transition

```text
COMMITTED
+
rollbackRequested
=
INVALID
```

---

# 82. Rollback-only state

El rollback-only flag deberá formar parte del contexto transaccional.

Podrá incluir razón:

```php
final readonly class RollbackOnlyReason
{
    public function __construct(
        public string $code,
        public ?\Throwable $cause = null,
    ) {}
}
```

---

# 83. Multiple rollback-only causes

Podrá conservarse:

```text
first cause
```

como causa principal y agregar diagnostics bounded para causas adicionales.

---

# 84. markRollbackOnly()

```php
$manager->markRollbackOnly();
```

deberá:

```text
resolve current transaction
validate ownership
validate active state
record reason
transition/mark state
publish event
```

---

# 85. Statement error integration

Query Executor podrá informar:

```text
TransactionFailureEffect
```

al manager/context.

Ejemplo:

```text
statement error
     ↓
Platform says transaction aborted
     ↓
mark rollback-only
```

---

# 86. Query Executor cannot rollback arbitrarily

El executor podrá:

```text
report failure effect
```

pero no apropiarse del lifecycle salvo política explícita.

---

# 87. Database-aborted transaction

Algunas plataformas pueden dejar la transacción físicamente abortada después de ciertos errores.

El estado lógico deberá reflejar:

```text
rollback required
```

aunque la conexión todavía esté dentro del transaction block.

---

# 88. Transaction failure effect

```php
enum TransactionFailureEffect
{
    case NONE;
    case ROLLBACK_RECOMMENDED;
    case ROLLBACK_REQUIRED;
    case CONNECTION_INVALID;
    case OUTCOME_UNKNOWN;
}
```

---

# 89. Nested invocation

Cuando `transactional()` se llama dentro de otra:

```text
Outer Transaction
      ↓
Inner transactional()
```

el manager consultará:

```text
Propagation
+
NestedTransactionStrategy
```

---

# 90. Joined transaction participation

Si inner se une:

```text
inner scope
```

no posee el physical commit.

---

# 91. Transaction ownership levels

Se distinguirá:

```text
PhysicalTransactionOwner
```

de:

```text
TransactionParticipant
```

---

# 92. Inner commit under JOIN_EXISTING

Cuando el inner scope termina correctamente:

```text
no physical COMMIT
```

Solo:

```text
participant success
```

---

# 93. Inner rollback under JOIN_EXISTING

Si falla:

```text
mark outer rollback-only
```

y propagar excepción.

---

# 94. Inner success does not guarantee outer commit

```text
inner completed
```

no significa:

```text
database committed
```

porque el outer scope sigue activo.

---

# 95. Logical transaction scope result

Por ello deberá distinguirse:

```text
ParticipantCompleted
```

de:

```text
TransactionCommitted
```

---

# 96. Savepoint strategy

Si nested strategy:

```text
SAVEPOINT
```

inner scope podrá:

```text
create savepoint
execute
release savepoint
```

o:

```text
rollback to savepoint
```

---

# 97. Savepoint failure

Si `ROLLBACK TO SAVEPOINT` falla:

```text
outer transaction
→ rollback-only
```

por default.

---

# 98. Savepoint release failure

La política dependerá de:

```text
Platform semantics
```

pero no deberá asumirse que el outer transaction sigue sano.

---

# 99. Callback registration

API conceptual:

```php
DB::beforeCommit($callback);
DB::afterCommit($callback);
DB::afterRollback($callback);
DB::afterCompletion($callback);
```

---

# 100. Callback Registry

El manager delegará almacenamiento en:

```text
TransactionCallbackRegistry
```

asociado al contexto.

---

# 101. Callback lifecycle

```text
register
   ↓
transaction completion
   ↓
execute appropriate phase
   ↓
discard
```

---

# 102. beforeCommit ordering

Se recomienda:

```text
FIFO
```

determinista, salvo que exista prioridad explícita.

---

# 103. afterCommit ordering

También deberá ser determinista.

Podrá soportarse:

```text
priority
```

en el futuro.

---

# 104. Callback registration during completion

Por default, registrar nuevos callbacks durante:

```text
afterCommit
```

deberá tener reglas explícitas.

La opción más segura:

```text
reject late registration
```

para la misma fase ya iniciada.

---

# 105. afterCompletion

Siempre que exista finalización local:

```text
COMMITTED
ROLLED_BACK
UNKNOWN
FAILED
```

podrá ejecutarse:

```text
afterCompletion
```

con `TransactionResult`.

---

# 106. afterCompletion cannot rewrite outcome

Aunque falle:

```text
TransactionOutcome
```

permanece intacto.

---

# 107. Completion Coordinator

Se recomienda:

```text
TransactionCompletionCoordinator
```

para separar:

```text
database completion
```

de:

```text
callbacks + cleanup
```

---

# 108. Completion pipeline

```text
Database outcome determined
          ↓
terminal transaction state
          ↓
outcome-specific callbacks
          ↓
afterCompletion
          ↓
resource cleanup
          ↓
connection disposition
```

---

# 109. Connection disposition

Después de la transacción:

```php
enum ConnectionDisposition
{
    case RELEASE;
    case RESET_AND_RELEASE;
    case QUARANTINE;
    case DISCARD;
}
```

---

# 110. Commit success disposition

Normalmente:

```text
COMMITTED
→ reset if required
→ release
```

---

# 111. Rollback success disposition

Normalmente:

```text
ROLLED_BACK
→ reset if required
→ release
```

---

# 112. Unknown disposition

```text
UNKNOWN
→ QUARANTINE / DISCARD
```

---

# 113. Invalid connection disposition

```text
ConnectionFailureEffect::INVALID
→ DISCARD
```

---

# 114. Release happens after unpin

Orden:

```text
terminal outcome
      ↓
cleanup transaction state
      ↓
unpin
      ↓
reset if necessary
      ↓
release
```

---

# 115. No premature unpin

Nunca:

```text
COMMIT dispatched
→ unpin
→ wait for result
```

La conexión pertenece a la transacción hasta que el manager determine su disposición.

---

# 116. Retry entry point

El retry deberá envolver:

```text
entire transactional operation
```

por encima de una transaction instance concreta.

---

# 117. Retry architecture

```text
TransactionOperation
       ↓
Attempt 1
       ↓
TransactionManager
       ↓
failure
       ↓
RetryCoordinator
       ↓
Attempt 2
```

---

# 118. Manager vs RetryCoordinator

`TransactionManager` administrará una transacción.

`TransactionRetryCoordinator` administrará:

```text
multiple transaction attempts
```

---

# 119. Retry pseudocode

```php
for ($attempt = 1; $attempt <= $maxAttempts; $attempt++) {
    try {
        return $manager->transactionalOnce(
            $callback,
            $definition,
            $attempt,
        );
    } catch (TransactionExecutionException $e) {
        if (! $retryPolicy->allows($e, $attempt)) {
            throw $e;
        }
    }
}
```

---

# 120. transactionalOnce()

Conviene separar internamente:

```text
transactional()
```

de:

```text
transactionalOnce()
```

para impedir retry recursivo accidental.

---

# 121. Retry eligibility

```text
RetryAllowed
=
FailureRetryable
∧ KnownRollback
∧ ReplayableOperation
∧ BudgetAvailable
∧ DeadlineAvailable
```

---

# 122. Unknown commit retry

Si:

```text
outcome = UNKNOWN
```

entonces:

```text
RetryAllowed = false
```

por default.

---

# 123. Retry callback result

El resultado de un attempt fallido nunca deberá filtrarse como resultado definitivo.

---

# 124. Transaction operation ID

Todos los attempts podrán compartir:

```text
TransactionOperationId
```

para telemetry.

---

# 125. Attempt ID

Cada intento tendrá:

```text
TransactionId
```

distinto.

---

# 126. ORM integration

EntityManager podrá utilizar:

```text
TransactionManager
```

pero no convertirse en él.

---

# 127. Explicit ORM transaction

```php
DB::transaction(function () use ($em) {
    $em->persist($user);
    $em->flush();
});
```

Flujo:

```text
TransactionManager BEGIN
       ↓
EntityManager / UoW
       ↓
flush
       ↓
Query Executor
       ↓
Pinned Connection
       ↓
TransactionManager COMMIT
```

---

# 128. Auto transaction around flush

Si Persistence Engine configura auto-transaction:

```text
EntityManager.flush()
       ↓
no current transaction?
       ↓
TransactionManager.transactional(...)
```

La implementación deberá evitar dependencia circular.

---

# 129. Dependency inversion for auto flush

Preferible:

```text
PersistenceCoordinator
      ↓
TransactionBoundary contract
```

en lugar de:

```text
UnitOfWork
→ DefaultTransactionManager
```

---

# 130. EntityManager after commit

Cuando outcome sea:

```text
COMMITTED
```

el Persistence Consistency System podrá promover conocimiento persistente según sus propias reglas.

---

# 131. EntityManager after rollback

El Transaction Manager notificará:

```text
ROLLED_BACK
```

pero no manipulará directamente entidades.

---

# 132. EntityManager after UNKNOWN

El manager podrá publicar:

```text
TransactionOutcomeUnknown
```

y el ORM integration layer marcar:

```text
EntityManager = TAINTED
```

---

# 133. Separation rule

Nunca:

```php
$transactionManager->identityMap()->clear();
```

El manager no deberá conocer IdentityMap.

---

# 134. Query Executor integration

Query Executor consultará:

```text
TransactionContextResolver
```

para obtener la conexión activa.

---

# 135. Execution resolution

```text
QueryExecutor
     ↓
TransactionContext active?
   ┌─────┴─────┐
  yes          no
   │            │
   ▼            ▼
Pinned       Normal
Connection   Routing
```

---

# 136. Transaction Manager does not execute query

Nunca:

```php
$transactionManager->execute($query);
```

como responsabilidad base.

---

# 137. Read/write routing integration

Durante transacción:

```text
normal read/write routing
→ bypassed for participating operations
```

utilizando la pinned connection.

---

# 138. Sticky connection distinction

```text
Transaction pinning
≠
Sticky read-after-write routing
```

Son conceptos distintos.

---

# 139. Transaction timeout

El manager deberá establecer:

```text
deadline
```

en el contexto.

---

# 140. Deadline

```php
final readonly class TransactionDeadline
{
    public function __construct(
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

Internamente puede preferirse un monotonic clock para medir duración.

---

# 141. Wall clock ≠ duration clock

Para timeouts:

```text
MonotonicClock
```

será preferible.

Para diagnostics:

```text
WallClock
```

podrá utilizarse.

---

# 142. Timeout check points

El manager podrá verificar deadline:

```text
before callback
before query execution via context
before commit
before retry
```

---

# 143. Timeout expiration

```text
deadline exceeded
→ mark rollback-only
→ rollback when possible
```

---

# 144. Timeout during commit

Si commit ya fue enviado:

```text
timeout
```

no permite asumir rollback.

Puede resultar:

```text
UNKNOWN
```

---

# 145. Cancellation integration

Runtime podrá señalar:

```text
CancellationToken
```

al Transaction Context.

---

# 146. Cancellation before commit

```text
cancelled
→ rollback-only
→ rollback
```

---

# 147. Cancellation during commit

Debe preservarse la semántica de incertidumbre.

No cancelar localmente y reportar falsamente rollback.

---

# 148. Runtime scope lifecycle

`TransactionManager` deberá integrarse con:

```text
RequestScope
JobScope
CoroutineScope
```

mediante contratos, no runtime-specific conditionals.

---

# 149. Scope shutdown

Al terminar:

```text
active transaction?
```

si sí:

```text
report leak
attempt rollback
dispose unsafe connection
clear context
```

---

# 150. Never commit leaked transaction

Esta regla será absoluta:

```text
scope end
+
active transaction
≠
implicit commit
```

---

# 151. Leak diagnostic

Ejemplo:

```text
TRANSACTION LEAK DETECTED

Transaction:
    tx-381

State:
    ACTIVE

Scope:
    request-991

Connection:
    primary#12

Started:
    842 ms ago

Action:
    rollback attempted

Connection disposition:
    RELEASED
```

---

# 152. Persistent worker safety

Estado compartible:

```text
DefaultTransactionManager service
Transaction definitions
Platform capabilities
Policies
```

Estado no compartible:

```text
current transaction
current connection lease
savepoint stack
callbacks
rollback-only state
attempt state
```

---

# 153. FrankenPHP

Modelo:

```text
Worker
├── immutable TransactionManager service
│
├── Request A Scope
│   └── TransactionContext A
│
└── Request B Scope
    └── TransactionContext B
```

---

# 154. RoadRunner

Cada job/request deberá recibir:

```text
fresh transaction scope
```

y cleanup al finalizar.

---

# 155. OpenSwoole

El contexto deberá ser:

```text
coroutine-local
```

o basado en un Runtime Context abstraction equivalente.

---

# 156. No mutable manager-local current transaction

Evitar:

```php
final class DefaultTransactionManager
{
    private ?Transaction $current = null;
}
```

si el manager es compartido por worker.

---

# 157. Context resolver

Preferir:

```php
interface TransactionContextResolver
{
    public function current(): ?TransactionContext;

    public function activate(TransactionContext $context): void;

    public function deactivate(TransactionContext $context): void;
}
```

La implementación se apoyará en el scope runtime.

---

# 158. Events

El manager emitirá eventos mediante interfaces desacopladas.

Eventos:

```text
TransactionStarting
TransactionStarted
TransactionRollbackOnlyMarked
TransactionCommitting
TransactionCommitted
TransactionRollingBack
TransactionRolledBack
TransactionOutcomeUnknown
TransactionFailed
TransactionCompleted
```

---

# 159. Event timing

`TransactionStarted`:

```text
after BEGIN confirmed
```

`TransactionCommitted`:

```text
after COMMIT confirmed
```

`TransactionRolledBack`:

```text
after ROLLBACK confirmed
```

---

# 160. Event dispatcher failure

Observability/event integration no deberá corromper automáticamente una transacción ya confirmada.

---

# 161. Pre-commit event danger

Listeners ejecutados antes del commit sí pueden provocar rollback si forman parte explícita del transaction lifecycle.

Esto deberá diferenciarse de eventos observacionales.

---

# 162. Lifecycle hooks vs observational events

Separar:

```text
Transaction Hooks
```

de:

```text
Transaction Observability Events
```

para evitar que telemetry accidentalmente cambie semántica.

---

# 163. Telemetry

El manager será fuente principal de métricas como:

```text
database.transaction.started
database.transaction.committed
database.transaction.rolled_back
database.transaction.unknown
database.transaction.failed
database.transaction.duration
database.transaction.rollback_only
database.transaction.nested
database.transaction.retry
```

---

# 164. Telemetry failure isolation

Si exporter falla:

```text
TransactionOutcome
```

no cambia.

---

# 165. Trace correlation

Atributos:

```text
transaction.operation_id
transaction.id
transaction.attempt
transaction.isolation
transaction.read_only
transaction.propagation
transaction.outcome
```

Los IDs deberán ser seguros para telemetry.

---

# 166. No sensitive transaction attributes

No incluir:

```text
credentials
raw SQL parameters
entity payload
personal data
```

---

# 167. Diagnostics API

Podrá ofrecerse:

```php
DB::transactions()->inspect();
```

---

# 168. Manager diagnostics

Ejemplo:

```text
TRANSACTION MANAGER

Current Scope:
    request-81

Current Transaction:
    tx-41

State:
    ACTIVE

Owner:
    request-81

Propagation:
    REQUIRED

Nested Level:
    0

Connection:
    primary#4

Isolation:
    READ_COMMITTED

Read Only:
    false

Rollback Only:
    false

Callbacks:
    beforeCommit: 0
    afterCommit: 2
    afterRollback: 1

Elapsed:
    31 ms
```

---

# 169. Explain no transaction

```text
TRANSACTION MANAGER

Current Scope:
    request-82

Current Transaction:
    none

Connection Pin:
    none

Transaction Leak:
    false
```

---

# 170. Exception hierarchy

```text
DatabaseTransactionException
│
├── TransactionManagerException
│   ├── TransactionManagerUnavailableException
│   ├── TransactionScopeException
│   ├── TransactionOwnershipException
│   └── TransactionLifecycleException
│
├── TransactionBeginException
├── TransactionCommitException
├── TransactionRollbackException
├── TransactionOutcomeUnknownException
├── TransactionRollbackOnlyException
├── TransactionRequiredException
├── ExistingTransactionNotAllowedException
├── UnsupportedTransactionPropagationException
├── TransactionConnectionException
├── TransactionConnectionMismatchException
├── TransactionTimeoutException
├── TransactionCancellationException
├── TransactionCompletionException
├── TransactionCleanupException
├── TransactionExecutionException
└── TransactionInvariantViolationException
```

---

# 171. Failure model

Modelo conceptual:

```php
final readonly class TransactionFailure
{
    public function __construct(
        public TransactionOperationPhase $phase,
        public TransactionFailureKind $kind,
        public TransactionFailureEffect $effect,
        public bool $retryable,
        public bool $outcomeKnown,
        public \Throwable $cause,
    ) {}
}
```

---

# 172. Failure kinds

```php
enum TransactionFailureKind
{
    case CONNECTION;
    case TIMEOUT;
    case DEADLOCK;
    case SERIALIZATION;
    case CONSTRAINT;
    case CANCELLATION;
    case CALLBACK;
    case PLATFORM;
    case DRIVER;
    case UNKNOWN;
}
```

---

# 173. Failure effect is separate

Un:

```text
DEADLOCK
```

puede implicar:

```text
ROLLBACK_REQUIRED
```

mientras una:

```text
CONSTRAINT violation
```

puede tener comportamiento distinto según plataforma.

---

# 174. Security model

El manager deberá impedir:

- transaction handle forgery;
- cross-scope transaction hijacking;
- arbitrary savepoint names;
- arbitrary driver access;
- unsafe state mutation;
- user-controlled propagation internals;
- unbounded callback registration.

---

# 175. Callback resource limits

Para evitar abuso:

```text
max callbacks per transaction
max nesting depth
max retry attempts
max transaction lifetime policy
```

podrán formar parte de Resource Governance.

---

# 176. Nested depth limit

Evitar:

```text
transaction
 └── transaction
      └── transaction
           └── ...
```

sin límite.

---

# 177. Transaction lifetime budget

El manager podrá advertir/restringir:

```text
very long transaction
```

según entorno y configuración.

---

# 178. Query count not manager responsibility

Aunque telemetry pueda correlacionar queries por transaction ID:

```text
TransactionManager
```

no deberá contar ni optimizar queries como responsabilidad central.

---

# 179. Testing architecture

Se requerirán:

```text
TransactionManagerUnitTests
TransactionManagerIntegrationTests
TransactionManagerConformanceTests
TransactionManagerFailureInjectionTests
TransactionManagerRuntimeIsolationTests
```

---

# 180. Begin tests

Casos:

```text
begin success
begin unsupported
begin connection acquisition failure
begin configuration failure
begin driver failure
begin uncertain connection failure
```

---

# 181. Commit tests

Casos:

```text
commit success
double commit
commit rollback-only
commit driver rejection
commit connection lost before dispatch
commit connection lost after dispatch
commit unknown
```

---

# 182. Rollback tests

```text
rollback success
rollback without transaction
double rollback
rollback connection failure
rollback unknown
```

---

# 183. transactional tests

```text
callback success
callback failure
commit failure
rollback failure
afterCommit failure
cleanup failure
```

---

# 184. Exception precedence tests

Verificar:

```text
business failure
+
rollback failure
```

preserva ambos.

---

# 185. Unknown tests

Failure injection deberá simular:

```text
COMMIT sent
→ server outcome unavailable
```

y comprobar:

```text
UNKNOWN
connection quarantined
no retry
no afterCommit
no afterRollback
afterCompletion(UNKNOWN)
```

---

# 186. Ownership tests

Probar:

```text
scope A begins
scope B commits
```

y exigir:

```text
TransactionOwnershipException
```

---

# 187. Pinning tests

Dentro de una transaction:

```text
query A
query B
query C
```

deben utilizar:

```text
same pinned connection
```

---

# 188. Pool tests

Una pinned connection no podrá ser:

```text
leased to another request
```

hasta completion.

---

# 189. Nested tests

Probar:

```text
REQUIRED
MANDATORY
NEVER
SUPPORTS
SAVEPOINT
JOIN_EXISTING
```

---

# 190. Rollback-only tests

```text
outer begins
inner joins
inner fails
exception caught
outer tries commit
```

debe resultar:

```text
rollback
TransactionRollbackOnlyException
```

---

# 191. Callback tests

Validar:

```text
ordering
phase
late registration
failure handling
resource limits
cleanup
```

---

# 192. Retry tests

Verificar:

```text
attempt IDs
operation ID
retry eligibility
budget
backoff
unknown commit rejection
```

---

# 193. Persistent runtime tests

Ejecutar secuencialmente:

```text
Request A:
    begin
    commit

Request B:
    no transaction
```

y comprobar ausencia total de state leak.

---

# 194. Leak tests

```text
Request A:
    begin
    forget completion
```

al finalizar:

```text
leak detected
rollback attempted
context cleared
connection safely disposed
```

---

# 195. Coroutine tests

Ejecutar:

```text
Coroutine A → tx-A
Coroutine B → tx-B
```

intercalando ejecución y comprobando aislamiento.

---

# 196. Performance considerations

Hot paths:

```text
current transaction lookup
connection pin lookup
state validation
deadline check
```

deberán ser de bajo costo.

---

# 197. No reflection in hot path

Metadata/policies deberán compilarse durante bootstrap cuando sea posible.

---

# 198. TransactionContext lookup

Debe ser aproximadamente:

```text
O(1)
```

respecto al número de transacciones históricas.

---

# 199. Callback execution complexity

Para `n` callbacks:

```text
O(n)
```

con límites explícitos.

---

# 200. Nested context complexity

Una estructura stack permite:

```text
push/pop O(1)
```

para scopes anidados.

---

# 201. Directory structure

```text
src/Quantum/Database/Transaction/
│
├── Manager/
│   ├── TransactionManager.php
│   ├── DefaultTransactionManager.php
│   ├── TransactionBoundary.php
│   └── TransactionManagerFactory.php
│
├── Transaction/
│   ├── Transaction.php
│   ├── DefaultTransaction.php
│   ├── TransactionId.php
│   └── TransactionOperationId.php
│
├── Definition/
│   ├── TransactionDefinition.php
│   ├── EffectiveTransactionDefinition.php
│   └── TransactionDefaults.php
│
├── State/
│   ├── TransactionState.php
│   ├── TransactionOutcome.php
│   ├── TransactionStateMachine.php
│   ├── TransactionOperationPhase.php
│   └── RollbackOnlyReason.php
│
├── Connection/
│   ├── TransactionConnectionResolver.php
│   ├── TransactionConnectionLease.php
│   ├── ConnectionDisposition.php
│   └── TransactionConnectionCoordinator.php
│
├── Ownership/
│   ├── TransactionOwner.php
│   └── TransactionOwnershipValidator.php
│
├── Propagation/
│   ├── TransactionPropagation.php
│   └── TransactionPropagationResolver.php
│
├── Completion/
│   ├── TransactionResult.php
│   ├── TransactionalResult.php
│   ├── TransactionCompletionStatus.php
│   ├── TransactionCompletionCoordinator.php
│   └── TransactionCallbackRegistry.php
│
├── Failure/
│   ├── TransactionFailure.php
│   ├── TransactionFailureKind.php
│   ├── TransactionFailureEffect.php
│   └── TransactionFailureClassifier.php
│
├── Retry/
│   ├── TransactionRetryCoordinator.php
│   └── TransactionAttempt.php
│
├── Runtime/
│   ├── TransactionContextResolver.php
│   └── TransactionScopeCleanup.php
│
├── Diagnostics/
│   ├── TransactionManagerInspector.php
│   └── TransactionManagerDiagnosticReport.php
│
├── Telemetry/
│   └── TransactionManagerTelemetry.php
│
└── Exception/
    └── ...
```

---

# 202. Dependency rules

Permitido:

```text
DefaultTransactionManager
        ↓
TransactionContext contracts
TransactionStateMachine
ConnectionManager contracts
Platform transaction contracts
Runtime Context contracts
Telemetry contracts
Event contracts
```

---

# 203. Forbidden dependencies

No:

```text
DefaultTransactionManager
→ PDO directly
→ Entity classes
→ UnitOfWork implementation
→ Repository
→ HTTP Request concrete implementation
→ FrankenPHP globals
→ RoadRunner globals
→ OpenSwoole coroutine globals
```

---

# 204. Manager lifecycle

El manager como servicio podrá ser:

```text
application-scoped
```

si todo estado mutable se resuelve mediante:

```text
runtime-scoped TransactionContext
```

---

# 205. Immutable dependencies

El manager podrá conservar referencias compartidas a:

```text
ConnectionManager
PlatformRegistry
TransactionPolicy
Telemetry facade
Clock
ContextResolver
```

si son persistent-runtime safe.

---

# 206. Mutable state rule

> **El Transaction Manager compartido no almacenará directamente el estado de la transacción actual.**

---

# 207. Architectural invariants

## DB-TXM-001
El Transaction Manager será la autoridad de lifecycle transaccional.

## DB-TXM-002
No será la autoridad de ORM state.

## DB-TXM-003
No generará SQL de negocio.

## DB-TXM-004
No ejecutará queries de aplicación.

## DB-TXM-005
No administrará IdentityMap.

## DB-TXM-006
No administrará UnitOfWork.

## DB-TXM-007
No mutará entidades.

## DB-TXM-008
`begin()` validará runtime scope.

## DB-TXM-009
`begin()` resolverá propagation antes de crear una nueva transacción.

## DB-TXM-010
`begin()` validará Platform capabilities.

## DB-TXM-011
No habrá isolation downgrade silencioso.

## DB-TXM-012
Una conexión será adquirida antes del BEGIN físico.

## DB-TXM-013
La conexión será pinned a la transacción.

## DB-TXM-014
Una transacción activa no cambiará de conexión.

## DB-TXM-015
Ownership será establecido antes de exposición de la transacción.

## DB-TXM-016
Cross-scope commit será rechazado.

## DB-TXM-017
Cross-scope rollback será rechazado.

## DB-TXM-018
El manager no almacenará current transaction en mutable global state.

## DB-TXM-019
Transaction Context será source of truth scoped.

## DB-TXM-020
Transaction handle no tendrá estado divergente.

## DB-TXM-021
State transitions pasarán por StateMachine.

## DB-TXM-022
BEGIN no producirá ACTIVE antes de confirmación suficiente.

## DB-TXM-023
Begin failure no devolverá una transacción activa.

## DB-TXM-024
Commit solo se aceptará desde estado válido.

## DB-TXM-025
Double commit será rechazado.

## DB-TXM-026
Rollback solo se aceptará desde estado válido.

## DB-TXM-027
Double rollback será rechazado salvo contrato explícito.

## DB-TXM-028
Rollback-only impedirá COMMIT físico.

## DB-TXM-029
Commit sobre rollback-only provocará finalización segura.

## DB-TXM-030
COMMITTED requerirá evidencia suficiente.

## DB-TXM-031
ROLLED_BACK requerirá evidencia suficiente.

## DB-TXM-032
Commit incierto producirá UNKNOWN.

## DB-TXM-033
Rollback incierto podrá producir UNKNOWN.

## DB-TXM-034
UNKNOWN será terminal para la transaction instance.

## DB-TXM-035
UNKNOWN no será convertido en rollback.

## DB-TXM-036
UNKNOWN commit no provocará rollback en nueva conexión.

## DB-TXM-037
UNKNOWN connection será quarantined/discarded.

## DB-TXM-038
Commit dispatch state será observable internamente.

## DB-TXM-039
Failure stage será preservado.

## DB-TXM-040
Callback failure durante work provocará rollback cuando sea posible.

## DB-TXM-041
Commit failure se clasificará independientemente del work callback.

## DB-TXM-042
Completion failure no reescribirá DB outcome.

## DB-TXM-043
Cleanup failure no reescribirá DB outcome.

## DB-TXM-044
Original application exception será preservada.

## DB-TXM-045
Rollback failure no ocultará original application failure.

## DB-TXM-046
Commit success + afterCommit failure seguirá siendo COMMITTED.

## DB-TXM-047
Rollback success + afterRollback failure seguirá siendo ROLLED_BACK.

## DB-TXM-048
afterCommit solo se ejecutará tras COMMITTED.

## DB-TXM-049
afterRollback solo se ejecutará tras ROLLED_BACK.

## DB-TXM-050
afterCompletion podrá recibir UNKNOWN.

## DB-TXM-051
Callbacks serán transaction-scoped.

## DB-TXM-052
Callbacks se eliminarán tras completion.

## DB-TXM-053
Callback ordering será determinista.

## DB-TXM-054
Late callback registration tendrá política explícita.

## DB-TXM-055
Callback count podrá estar limitado.

## DB-TXM-056
REQUIRED reutilizará una transacción compatible existente.

## DB-TXM-057
MANDATORY fallará sin transacción.

## DB-TXM-058
NEVER fallará con transacción activa.

## DB-TXM-059
SUPPORTS no creará necesariamente una transacción.

## DB-TXM-060
REQUIRES_NEW no será simulado con savepoint.

## DB-TXM-061
JOIN_EXISTING no ejecutará inner physical commit.

## DB-TXM-062
Inner joined success no significará DB commit.

## DB-TXM-063
Inner joined failure podrá marcar outer rollback-only.

## DB-TXM-064
Savepoint semantics permanecerán separadas de propagation.

## DB-TXM-065
Savepoint failure podrá marcar outer rollback-only.

## DB-TXM-066
Nested depth será bounded.

## DB-TXM-067
Transaction connection será obtenida mediante ConnectionManager.

## DB-TXM-068
Manager no instanciará PDO directamente.

## DB-TXM-069
Driver operations pasarán por abstraction.

## DB-TXM-070
Transaction platform behavior será capability-driven.

## DB-TXM-071
No habrá vendor conditionals en core manager.

## DB-TXM-072
Pinned connection no volverá al pool durante ACTIVE.

## DB-TXM-073
Connection disposition será explícita.

## DB-TXM-074
Known clean connection podrá release.

## DB-TXM-075
Unsafe connection será quarantined/discarded.

## DB-TXM-076
Unpin ocurrirá solo durante completion seguro.

## DB-TXM-077
Connection reset ocurrirá antes de reuse cuando sea necesario.

## DB-TXM-078
Transaction session state no deberá filtrarse al siguiente scope.

## DB-TXM-079
`transactional()` separará work, commit y completion phases.

## DB-TXM-080
`transactional()` no atrapará commit failure como si fuera work failure.

## DB-TXM-081
`transactional()` no intentará rollback después de uncertain commit.

## DB-TXM-082
Manual API expondrá state suficiente para safe completion.

## DB-TXM-083
Callback API será la opción recomendada.

## DB-TXM-084
Manual API seguirá disponible.

## DB-TXM-085
TransactionResult separará DB outcome de completion status.

## DB-TXM-086
Callback value será distinto de TransactionResult.

## DB-TXM-087
Advanced API podrá exponer ambos.

## DB-TXM-088
Retry envolverá transaction attempts completos.

## DB-TXM-089
Una transaction instance no se reutilizará como retry attempt.

## DB-TXM-090
Cada retry tendrá TransactionId propio.

## DB-TXM-091
Attempts podrán compartir TransactionOperationId.

## DB-TXM-092
Retry requerirá known rollback.

## DB-TXM-093
Retry requerirá failure retryable.

## DB-TXM-094
Retry requerirá replayability.

## DB-TXM-095
Retry respetará budget.

## DB-TXM-096
UNKNOWN outcome deshabilitará automatic retry por default.

## DB-TXM-097
Manager no realizará statement-level retry arbitrario.

## DB-TXM-098
Timeout será transaction-scoped.

## DB-TXM-099
Transaction timeout será distinto de query timeout.

## DB-TXM-100
Deadline expiration podrá marcar rollback-only.

## DB-TXM-101
Timeout durante commit podrá producir UNKNOWN.

## DB-TXM-102
Cancellation antes de commit intentará rollback.

## DB-TXM-103
Cancellation durante commit preservará uncertainty.

## DB-TXM-104
Scope shutdown detectará transaction leaks.

## DB-TXM-105
Leaked transaction nunca será auto-committed.

## DB-TXM-106
Leaked transaction provocará rollback attempt.

## DB-TXM-107
Failed leak cleanup provocará unsafe connection disposal.

## DB-TXM-108
Persistent worker no conservará transaction context entre requests.

## DB-TXM-109
FrankenPHP requests estarán aisladas.

## DB-TXM-110
RoadRunner jobs estarán aislados.

## DB-TXM-111
OpenSwoole coroutines estarán aisladas.

## DB-TXM-112
Manager compartido será stateless respecto a current transaction.

## DB-TXM-113
Runtime Context resolver será responsable del current context.

## DB-TXM-114
ORM integration será mediante contracts/events.

## DB-TXM-115
Manager no limpiará IdentityMap directamente.

## DB-TXM-116
Manager no rebobinará object graph después de rollback.

## DB-TXM-117
Manager notificará outcome al Persistence Consistency layer.

## DB-TXM-118
Query Executor utilizará pinned connection mediante context.

## DB-TXM-119
Query Executor no será commit authority.

## DB-TXM-120
Read/write routing normal no reemplazará pinned connection.

## DB-TXM-121
Transaction pinning será distinto de sticky connection.

## DB-TXM-122
Events reflejarán estados confirmados.

## DB-TXM-123
TransactionCommitted no se emitirá para UNKNOWN.

## DB-TXM-124
TransactionRolledBack no se emitirá para UNKNOWN.

## DB-TXM-125
Telemetry failure no cambiará transaction state.

## DB-TXM-126
Diagnostics failure no cambiará transaction state.

## DB-TXM-127
Sensitive values no aparecerán en telemetry.

## DB-TXM-128
Transaction IDs no deberán revelar secrets.

## DB-TXM-129
Transaction lifecycle será determinista para la evidencia disponible.

## DB-TXM-130
No se fabricará certeza cuando el driver no pueda determinar outcome.

## DB-TXM-131
Database outcome será distinto de local completion outcome.

## DB-TXM-132
Connection failure classification considerará dispatch stage.

## DB-TXM-133
Begin, work, commit y rollback tendrán failure phases distintas.

## DB-TXM-134
Cleanup será best-effort pero explícito.

## DB-TXM-135
Cleanup no ejecutará business work.

## DB-TXM-136
Completion callbacks no deberán permanecer en manager singleton.

## DB-TXM-137
Transaction resources serán bounded.

## DB-TXM-138
Transaction nesting será bounded.

## DB-TXM-139
Retry attempts serán bounded.

## DB-TXM-140
Long-running transaction será observable.

## DB-TXM-141
Monotonic clock se preferirá para duración/deadline.

## DB-TXM-142
Wall clock podrá usarse para timestamps diagnósticos.

## DB-TXM-143
Connection acquisition failure ocurrirá antes de ACTIVE.

## DB-TXM-144
Connection pin failure ocurrirá antes de ACTIVE.

## DB-TXM-145
Ownership failure ocurrirá antes de ACTIVE.

## DB-TXM-146
Context activation failure deberá limpiar recursos adquiridos.

## DB-TXM-147
Partial begin failure deberá determinar connection disposition.

## DB-TXM-148
Transaction Manager deberá poder ser probado sin ORM.

## DB-TXM-149
Transaction Manager deberá poder ser probado con fake driver/platform.

## DB-TXM-150
VoltStack preferirá un estado UNKNOWN explícito antes que un falso estado limpio.

---

# 208. Anti-patterns

## 208.1 Manager con PDO directo

```php
final class TransactionManager
{
    public function __construct(
        private PDO $pdo,
    ) {}
}
```

**Rechazado.**

Rompe:

- Driver abstraction;
- Connection Manager;
- pooling;
- multi-platform;
- runtime isolation.

---

# 208.2 Current transaction mutable en singleton

```php
private ?Transaction $current = null;
```

en servicio compartido.

**Rechazado.**

---

# 208.3 Catch universal

```php
try {
    $callback();
    $this->commit();
} catch (\Throwable $e) {
    $this->rollback();
}
```

**Rechazado.**

Puede intentar rollback después de un commit incierto.

---

# 208.4 Commit boolean

```php
if ($connection->commit()) {
    // committed
}
```

como único modelo de evidencia.

**Insuficiente.**

---

# 208.5 Retry en el mismo transaction object

```text
tx-1 fails
→ begin tx-1 again
```

**Rechazado.**

Cada attempt tendrá una transaction instance nueva.

---

# 208.6 Inner commit físico

```text
outer BEGIN
inner transaction()
inner COMMIT
outer continues
```

con `JOIN_EXISTING`.

**Rechazado.**

---

# 208.7 Savepoint como REQUIRES_NEW

**Rechazado.**

---

# 208.8 Auto-commit al terminar request

**Prohibido.**

---

# 208.9 Clear EntityManager desde TransactionManager

**Rechazado.**

Debe existir integración desacoplada.

---

# 208.10 Retry de UNKNOWN

```text
COMMIT unknown
→ retry whole callback
```

**Prohibido por default.**

---

# 209. Ejemplo completo — transacción exitosa

```php
$order = DB::transaction(function () use ($em, $order) {
    $order->confirm();

    $em->flush();

    return $order;
});
```

Pipeline:

```text
transactional()
      ↓
resolve REQUIRED
      ↓
no active transaction
      ↓
acquire connection
      ↓
pin connection
      ↓
BEGIN
      ↓
ACTIVE
      ↓
callback
      ↓
flush
      ↓
beforeCommit
      ↓
COMMITTING
      ↓
COMMIT confirmed
      ↓
COMMITTED
      ↓
afterCommit
      ↓
afterCompletion
      ↓
unpin
      ↓
reset/release
      ↓
return $order
```

---

# 210. Ejemplo completo — error de negocio

```text
BEGIN
 ↓
ACTIVE
 ↓
callback
 ↓
DomainException
 ↓
ROLLING_BACK
 ↓
ROLLBACK confirmed
 ↓
ROLLED_BACK
 ↓
afterRollback
 ↓
afterCompletion
 ↓
release
 ↓
rethrow DomainException
```

---

# 211. Ejemplo — error de negocio + rollback failure

```text
DomainException
       ↓
ROLLBACK
       ↓
connection failure
       ↓
TransactionOutcome UNKNOWN
       ↓
connection discarded
```

La excepción final deberá preservar:

```text
primary:
    DomainException

completion:
    TransactionRollbackException

outcome:
    UNKNOWN
```

---

# 212. Ejemplo — commit desconocido

```text
callback success
      ↓
COMMIT
      ↓
dispatch confirmed
      ↓
connection lost
      ↓
UNKNOWN
```

Manager:

```text
no rollback
no retry
no afterCommit
no afterRollback
afterCompletion(UNKNOWN)
quarantine connection
notify persistence consistency layer
```

---

# 213. Ejemplo — rollback-only

```php
DB::transaction(function () {
    try {
        Service::execute();
    } catch (RecoverableApplicationException $e) {
        // application chooses to continue
    }

    // But the transaction was marked rollback-only.
});
```

Completion:

```text
commit requested
      ↓
rollback-only
      ↓
ROLLBACK
      ↓
ROLLED_BACK
      ↓
TransactionRollbackOnlyException
```

---

# 214. Ejemplo — joined nested transaction

```php
DB::transaction(function () {
    ServiceA::run();

    DB::transaction(function () {
        ServiceB::run();
    });

    ServiceC::run();
});
```

Con `REQUIRED + JOIN_EXISTING`:

```text
BEGIN
 ├── A
 ├── B
 ├── C
 └── COMMIT
```

No:

```text
BEGIN
 ├── A
 ├── BEGIN
 │    └── B
 │    └── COMMIT
 ├── C
 └── COMMIT
```

---

# 215. Ejemplo — retry deadlock

```text
TransactionOperation op-92

Attempt 1 / tx-501
    BEGIN
    work
    DEADLOCK
    rollback confirmed
    retryable

Attempt 2 / tx-502
    BEGIN
    work
    COMMIT
```

Resultado:

```text
Operation:
    SUCCESS

Attempts:
    2

Final Transaction:
    tx-502

Outcome:
    COMMITTED
```

---

# 216. Master begin formula

```text
Begin(T)
=
ValidateScope
∧ ResolvePropagation
∧ ResolveDefinition
∧ ValidateCapabilities
∧ AcquireConnection
∧ AcquireOwnership
∧ PinConnection
∧ BeginPhysicalTransaction
```

Solo entonces:

```text
State(T) = ACTIVE
```

---

# 217. Master commit formula

```text
Commit(T)
=
ValidateOwnership(T)
∧ ValidateState(T)
∧ ¬RollbackOnly(T)
∧ BeforeCommit(T)
∧ DispatchCommit(T)
∧ DetermineOutcome(Evidence)
```

---

# 218. Master rollback formula

```text
Rollback(T)
=
ValidateOwnership(T)
∧ ValidateRollbackState(T)
∧ DispatchRollback(T)
∧ DetermineOutcome(Evidence)
```

---

# 219. Completion formula

```text
Completion(T)
=
DatabaseOutcome(T)
+
OutcomeCallbacks(T)
+
AfterCompletion(T)
+
Cleanup(T)
+
ConnectionDisposition(T)
```

Pero:

```text
DatabaseOutcome(T)
```

permanece independiente de fallos posteriores.

---

# 220. Transactional formula

```text
transactional(F)
=
AcquireTransactionScope
→ Execute(F)
→ CompleteTransaction
→ ReleaseTransactionScope
```

con:

```text
WorkFailure
→ RollbackIfPossible
```

pero:

```text
CommitUnknown
¬→ BlindRollback
```

---

# 221. Retry formula

```text
TransactionOperation(F)
=
Attempt₁(F)
∨ Attempt₂(F)
∨ ...
∨ Attemptₙ(F)
```

solo mientras:

```text
RetryPolicyAllows
=
true
```

---

# 222. Context formula

```text
CurrentTransaction
=
TransactionContextResolver(
    CurrentRuntimeScope
)
```

No:

```text
CurrentTransaction
=
GlobalStaticVariable
```

---

# 223. Arquitectura final

```text
                         APPLICATION
                              │
                              ▼
                       DB::transaction()
                              │
                              ▼
                   DefaultTransactionManager
                              │
          ┌───────────────────┼────────────────────┐
          │                   │                    │
          ▼                   ▼                    ▼
   Context Resolver       State Machine      Propagation Resolver
          │                   │                    │
          └─────────────┬─────┴─────────────┬──────┘
                        │                   │
                        ▼                   ▼
                Connection Coordinator   Retry Coordinator
                        │                   │
                        ▼                   │
                 Connection Manager         │
                        │                   │
                        ▼                   │
                  Connection Lease          │
                        │                   │
                        ▼                   │
                 Pinned Connection          │
                        │                   │
                        ▼                   │
              Transaction Platform Adapter  │
                        │                   │
                        ▼                   │
                     DRIVER                 │
                        │                   │
                        ▼                   │
                    DATABASE                │
                                            │
                     ┌──────────────────────┘
                     ▼
              Completion Coordinator
                     │
        ┌────────────┼─────────────┐
        ▼            ▼             ▼
    Callbacks      Events       Telemetry
        │
        ▼
  Connection Disposition
```

---

# 224. Decisiones arquitectónicas finales

VoltStack adoptará un:

```text
Stateless Shared Transaction Manager
```

respecto a la transacción actual.

El estado mutable vivirá en:

```text
Runtime-scoped Transaction Context
```

El manager utilizará:

```text
Connection Lease + Pinning
```

para garantizar afinidad.

El lifecycle utilizará:

```text
Explicit State Machine
```

y no simples flags.

El resultado utilizará:

```text
TransactionOutcome
```

incluyendo:

```text
UNKNOWN
```

como resultado de primera clase.

La API:

```text
transactional()
```

separará estrictamente:

```text
WORK
COMMIT
COMPLETION
CLEANUP
```

para evitar el clásico error de intentar rollback después de un commit incierto.

La arquitectura distinguirá:

```text
Database Outcome
≠
Callback Outcome
≠
Cleanup Outcome
```

Los retries se administrarán por:

```text
TransactionRetryCoordinator
```

sobre attempts completos.

La integración con ORM será:

```text
contract/event based
```

y nunca mediante manipulación directa de `UnitOfWork` o `IdentityMap`.

Finalmente:

> **El Transaction Manager será el coordinador del boundary transaccional, pero la realidad de la base de datos seguirá siendo la autoridad final; cuando esa realidad no pueda conocerse, VoltStack conservará explícitamente el estado `UNKNOWN`.**

---

# 225. Relación con los siguientes documentos

Este documento establece el manager general.

Los siguientes documentos especializarán:

```text
165 Transaction Manager
        │
        ▼
166 Transaction Context
        │
        ▼
167 Transaction Isolation
        │
        ▼
168 Nested Transactions
        │
        ▼
169 Savepoints
        │
        ▼
170 Transaction Retry
        │
        ▼
171 Deadlock Handling
        │
        ▼
172 Optimistic Locking
        │
        ▼
173 Pessimistic Locking
        │
        ▼
174 Concurrency Control
        │
        ▼
175 Transaction Events
```

---

# 226. Siguiente documento

```text
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
```

El siguiente documento deberá definir detalladamente:

- `TransactionContext`;
- identidad del contexto;
- relación entre Transaction y Context;
- runtime scope;
- transaction ownership;
- connection lease;
- pinned connection;
- effective transaction definition;
- transaction state;
- transaction outcome;
- rollback-only state;
- nesting;
- savepoint stack;
- retry attempt;
- callbacks;
- transaction-local resources;
- deadlines;
- cancellation;
- context activation;
- context suspension;
- context restoration;
- propagation stack;
- lifecycle cleanup;
- context leak detection;
- FrankenPHP isolation;
- RoadRunner isolation;
- OpenSwoole coroutine isolation;
- context snapshots para diagnostics;
- concurrency safety;
- testing;
- invariantes arquitectónicas.