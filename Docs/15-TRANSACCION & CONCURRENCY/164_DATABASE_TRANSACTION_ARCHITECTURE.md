# 164_DATABASE_TRANSACTION_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Transaction Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 164 — Database Transaction Architecture  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md`  
**Siguiente documento:** `165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Architecture` define el modelo arquitectónico general mediante el cual VoltStack administrará transacciones de base de datos de forma:

- explícita;
- segura;
- determinista;
- observable;
- compatible con ORM y Query Engine;
- independiente del driver;
- preparada para runtimes persistentes;
- extensible hacia retries, savepoints, locking y concurrencia.

La regla central será:

> **Una transacción en VoltStack será una unidad explícita de atomicidad y consistencia asociada a un contexto de conexión concreto; no será equivalente a `flush()`, UnitOfWork, EntityManager ni request HTTP, y ningún componente podrá asumir que una operación ORM está comprometida hasta que el Transaction Manager confirme inequívocamente el resultado del commit.**

Por tanto:

```text
Transaction
≠
UnitOfWork
≠
flush()
≠
EntityManager
≠
Connection
≠
HTTP Request
```

---

# 2. Objetivos

La arquitectura deberá resolver:

1. inicio de transacciones;
2. commit;
3. rollback;
4. estado transaccional;
5. ownership;
6. connection affinity;
7. isolation levels;
8. savepoints;
9. nested transaction semantics;
10. rollback-only;
11. transaction retries;
12. deadlocks;
13. pessimistic locking;
14. optimistic locking;
15. failure handling;
16. unknown transaction outcomes;
17. ORM integration;
18. Query Engine integration;
19. read/write routing;
20. callbacks;
21. events;
22. telemetry;
23. persistent runtime isolation;
24. concurrency safety.

---

# 3. Problema fundamental

Una transacción parece conceptualmente simple:

```php
$db->beginTransaction();

try {
    // work
    $db->commit();
} catch (\Throwable $e) {
    $db->rollBack();

    throw $e;
}
```

pero un framework moderno debe responder preguntas más difíciles:

```text
¿Qué pasa si falla COMMIT?

¿Qué pasa si se pierde la conexión después de enviar COMMIT?

¿El servidor confirmó el commit?

¿Puede hacerse retry?

¿El retry duplicará efectos externos?

¿Qué ocurre con las entidades ya modificadas?

¿Rollback restaura objetos PHP?

¿Qué pasa con flush()?

¿Qué ocurre con conexiones read/write?

¿Qué significa nested transaction?

¿Qué pasa si una inner transaction falla?

¿Puede utilizarse un savepoint?

¿Qué isolation level está realmente activo?

¿Quién posee la transacción?

¿Puede otro coroutine utilizar la conexión?
```

VoltStack deberá representar estos estados explícitamente.

---

# 4. Arquitectura general

```text
Application / Domain
        │
        ▼
Transaction API
        │
        ▼
Transaction Manager
        │
        ├── Transaction Context
        ├── Transaction State Machine
        ├── Isolation Policy
        ├── Nested Transaction Policy
        ├── Retry Policy
        ├── Event Pipeline
        └── Diagnostics
        │
        ▼
Connection Manager
        │
        ▼
Transaction-bound Connection
        │
        ▼
Driver Transaction Adapter
        │
        ▼
Database
```

Integraciones:

```text
                    Transaction Manager
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
       ▼                   ▼                    ▼
      ORM              Query Engine          Telemetry
       │                   │                    │
       ▼                   ▼                    ▼
 UnitOfWork          Query Executor          Events
```

---

# 5. Transaction Domain

VoltStack deberá modelar una transacción como concepto de dominio propio.

No deberá limitarse a:

```text
PDO::beginTransaction()
PDO::commit()
PDO::rollBack()
```

Estas operaciones pertenecen al nivel Driver.

Por encima existirá:

```text
Transaction
TransactionManager
TransactionContext
TransactionState
TransactionIsolation
TransactionScope
TransactionOutcome
```

---

# 6. TransactionId

Cada transacción lógica podrá disponer de un identificador interno:

```php
final readonly class TransactionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Este ID será útil para:

- telemetry;
- diagnostics;
- correlation;
- events;
- retry attempts.

No será una identidad proveniente de la base de datos.

---

# 7. Transaction Definition

Modelo conceptual:

```php
final readonly class TransactionDefinition
{
    public function __construct(
        public TransactionIsolation $isolation,
        public bool $readOnly,
        public TransactionPropagation $propagation,
        public TransactionTimeout $timeout,
        public RetryPolicyReference|null $retryPolicy,
    ) {}
}
```

No todas las plataformas soportarán todas las propiedades.

---

# 8. Transaction Scope

Una transacción tendrá un scope explícito:

```text
TransactionScope
```

que delimitará:

```text
BEGIN
   ↓
transactional work
   ↓
COMMIT / ROLLBACK / UNKNOWN
```

---

# 9. Scope ≠ Request

Una request puede contener:

```text
Transaction A
Transaction B
Transaction C
```

o ninguna.

Por tanto:

```text
HTTP Request
≠
Transaction
```

---

# 10. Scope ≠ EntityManager

Un `EntityManager` podrá participar en varias transacciones secuenciales dentro de su scope si su estado sigue siendo válido.

No obstante, resultados inciertos podrán dejarlo:

```text
TAINTED
```

---

# 11. Transaction State Machine

Estado conceptual:

```text
NEW
 │
 ▼
STARTING
 │
 ▼
ACTIVE
 ├───────────────┐
 │               │
 ▼               ▼
COMMITTING    ROLLING_BACK
 │               │
 ▼               ▼
COMMITTED     ROLLED_BACK
 │               │
 └───────┬───────┘
         ▼
      TERMINAL
```

Con fallos:

```text
ACTIVE
   │
   ├── failure → MARKED_ROLLBACK_ONLY
   │
   ├── connection uncertainty → UNKNOWN
   │
   └── fatal transaction failure → FAILED
```

---

# 12. Estados propuestos

```php
enum TransactionState
{
    case NEW;

    case STARTING;

    case ACTIVE;

    case MARKED_ROLLBACK_ONLY;

    case COMMITTING;

    case COMMITTED;

    case ROLLING_BACK;

    case ROLLED_BACK;

    case FAILED;

    case UNKNOWN;
}
```

---

# 13. Terminal states

Estados terminales:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

Sin embargo:

```text
FAILED
```

no necesariamente significa:

```text
database rolled back
```

La información de outcome deberá ser más precisa.

---

# 14. Transaction Outcome

Se separará:

```text
TransactionState
```

de:

```text
TransactionOutcome
```

porque el estado del objeto local y la realidad de la base de datos no son necesariamente idénticos.

---

# 15. Outcomes

Modelo conceptual:

```php
enum TransactionOutcome
{
    case COMMITTED;

    case ROLLED_BACK;

    case NOT_STARTED;

    case UNKNOWN;
}
```

---

# 16. UNKNOWN

`UNKNOWN` será un estado crítico.

Ejemplo:

```text
Client
   │
   │ COMMIT
   ▼
Database
   │
   │ commit succeeds
   ▼
network failure
   X
Client
```

El cliente sabe:

```text
COMMIT was sent
```

pero no sabe:

```text
whether commit completed
```

Resultado:

```text
UNKNOWN
```

Nunca deberá transformarse automáticamente en:

```text
ROLLED_BACK
```

---

# 17. Regla de incertidumbre

> **La pérdida de evidencia no será interpretada como rollback.**

Esto extiende la regla del Persistence Consistency System:

```text
UNKNOWN ≠ FAILURE ≠ ROLLBACK
```

---

# 18. Begin Transaction

Inicio conceptual:

```text
TransactionDefinition
        ↓
TransactionManager
        ↓
Resolve Connection
        ↓
Acquire Transaction Ownership
        ↓
Apply Transaction Options
        ↓
Driver BEGIN
        ↓
ACTIVE
```

---

# 19. Connection resolution before BEGIN

La conexión deberá quedar determinada antes de activar la transacción.

No se permitirá cambiar silenciosamente de conexión una vez activa.

---

# 20. Connection Affinity

Regla:

> **Toda transacción estará fijada a una única conexión física/lógica compatible durante su vida.**

Formalmente:

```text
Transaction T
→ Connection C
```

mientras:

```text
state(T) ∈ ACTIVE states
```

entonces:

```text
connection(T) = C
```

---

# 21. Connection pinning

Una vez iniciada:

```text
Transaction
   ↓
Pinned Connection
```

Toda query perteneciente a esa transacción deberá utilizar esa conexión.

---

# 22. Read/write routing

Antes de la transacción:

```text
SELECT → replica
WRITE  → primary
```

Durante una transacción:

```text
Transaction
     ↓
Pinned transactional connection
     ↓
all participating queries
```

salvo soporte explícito de una arquitectura distribuida futura.

---

# 23. Default write connection

Una transacción general deberá utilizar normalmente una conexión:

```text
WRITE-capable
```

aunque inicialmente solo ejecute `SELECT`.

---

# 24. Read-only transaction

Podrá declararse:

```php
$db->transaction(
    fn () => ...,
    readOnly: true,
);
```

pero será una intención/capability que Platform deberá validar.

---

# 25. Read-only ≠ replica automatically

Una read-only transaction no implica automáticamente:

```text
replica
```

porque pueden existir requisitos de:

- consistency;
- isolation;
- lag;
- session state.

---

# 26. Transaction Manager

El coordinador principal será:

```text
TransactionManager
```

Responsabilidades:

```text
begin
commit
rollback
scope
state
context
connection affinity
callbacks
propagation
nested semantics
```

No será responsable directamente de:

```text
SQL generation
entity hydration
change tracking
query planning
driver protocol
```

---

# 27. Transaction Manager contract

```php
interface TransactionManager
{
    public function begin(
        ?TransactionDefinition $definition = null,
    ): Transaction;

    public function commit(
        Transaction $transaction,
    ): TransactionOutcome;

    public function rollback(
        Transaction $transaction,
    ): TransactionOutcome;

    public function current(): ?TransactionContext;

    public function transactional(
        callable $callback,
        ?TransactionDefinition $definition = null,
    ): mixed;
}
```

Los detalles se desarrollarán en:

```text
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
```

---

# 28. Transaction Context

El contexto deberá contener información scoped:

```text
TransactionContext
├── TransactionId
├── state
├── definition
├── connection identity
├── nesting level
├── savepoint stack
├── rollback-only state
├── retry attempt
├── transaction-local resources
└── diagnostic correlation
```

---

# 29. Context ≠ global singleton

Nunca:

```php
static $currentTransaction;
```

como estado global mutable.

---

# 30. Context scope

Deberá asociarse al:

```text
Request / Job / Coroutine / Operation Context
```

de VoltStack.

---

# 31. Persistent runtime requirement

En FrankenPHP:

```text
Worker
├── Request A
│   └── Transaction A
│
└── Request B
    └── Transaction B
```

Nunca:

```text
Transaction A leaks into Request B
```

---

# 32. Coroutine requirement

En OpenSwoole:

```text
Coroutine A → Transaction A → Connection A
Coroutine B → Transaction B → Connection B
```

aunque ambos vivan en el mismo worker.

---

# 33. Ownership

Una transacción tendrá ownership.

Conceptualmente:

```text
TransactionOwner
```

podrá representar:

```text
request scope
job scope
coroutine scope
explicit operation scope
```

---

# 34. Cross-scope use

Una transacción no podrá utilizarse desde otro scope sin soporte explícito.

Esto evita:

```text
Coroutine A
    ↓
Transaction
    ↑
Coroutine B
```

---

# 35. Commit

Commit pipeline:

```text
ACTIVE
  ↓
Validate State
  ↓
Check rollback-only
  ↓
beforeCommit hooks
  ↓
COMMITTING
  ↓
Driver COMMIT
  ↓
Determine Outcome
  ↓
COMMITTED / UNKNOWN / FAILED
  ↓
afterCommit processing
  ↓
release transaction resources
```

---

# 36. Commit success

Solo se declarará:

```text
COMMITTED
```

cuando exista evidencia suficiente.

---

# 37. Commit failure before dispatch

Si el commit falla antes de enviarse al servidor:

```text
Outcome may remain:
    NOT_COMMITTED
```

pero el Transaction Manager deberá decidir si rollback sigue siendo posible.

---

# 38. Commit failure after dispatch

Si:

```text
COMMIT sent
connection lost
```

el resultado será:

```text
UNKNOWN
```

---

# 39. No blind retry of COMMIT

Nunca:

```text
COMMIT
→ timeout
→ COMMIT again
```

sin garantías específicas.

El segundo commit podría:

- fallar porque la transacción ya terminó;
- ocultar el outcome original;
- generar semántica incorrecta.

---

# 40. Rollback

Rollback pipeline:

```text
ACTIVE / ROLLBACK_ONLY
       ↓
ROLLING_BACK
       ↓
Driver ROLLBACK
       ↓
Outcome
       ↓
ROLLED_BACK / UNKNOWN / FAILED
```

---

# 41. Rollback can also be uncertain

Ejemplo:

```text
ROLLBACK sent
connection lost
```

El servidor probablemente hará rollback al cerrar la sesión, pero el framework no deberá inventar evidencia que no posee.

---

# 42. Connection loss before rollback

Dependiendo del driver/database:

```text
connection loss
```

puede implicar rollback del servidor.

Sin embargo, esa semántica deberá provenir de Platform/Driver capability.

---

# 43. Rollback ≠ object graph rewind

Regla fundamental:

```text
DatabaseRollback
≠
PHPObjectGraphRollback
```

Ejemplo:

```php
$user->name = 'Alice';

$em->flush();

$transaction->rollback();
```

Después del rollback, el objeto puede seguir conteniendo:

```text
name = Alice
```

aunque la base de datos haya restaurado el valor anterior.

---

# 44. ORM state after rollback

El ORM deberá considerar:

```text
DatabaseReality
≠
CurrentObjectGraph
```

y aplicar una política explícita.

Opciones futuras:

```text
clear EntityManager
refresh entities
mark stale
mark transaction-dependent state stale
```

Nunca fingir rewind automático.

---

# 45. flush() ≠ commit()

Esta será una de las invariantes más importantes.

```text
flush()
```

significa:

> sincronizar cambios ORM con la base de datos dentro del contexto transaccional actual.

Mientras:

```text
commit()
```

significa:

> solicitar la confirmación definitiva de la transacción al servidor.

Por tanto:

```text
flush() ≠ commit()
```

---

# 46. Example

```php
$db->transaction(function () use ($em, $user) {
    $user->rename('Alice');

    $em->flush();

    // SQL may have executed,
    // but transaction is still not committed.
});
```

---

# 47. Persistence outcome levels

Distinguir:

```text
Statement executed
Transaction active
Transaction committed
```

No son equivalentes.

---

# 48. Statement success ≠ durable success

```text
UPDATE succeeded
```

no implica:

```text
transaction committed
```

---

# 49. UnitOfWork integration

`UnitOfWork` administra:

```text
entity state
changes
persistence intentions
```

Transaction Manager administra:

```text
database transaction boundary
```

---

# 50. UnitOfWork ≠ Transaction

Una UoW puede:

```text
prepare
plan
flush
```

dentro de una transacción.

Pero no deberá convertirse en el Transaction Manager.

---

# 51. Automatic transaction around flush

VoltStack podrá ofrecer:

```text
flush inside implicit short transaction
```

cuando no exista una transacción activa.

Sin embargo, deberá ser una política explícita del Persistence System.

---

# 52. Explicit transaction wins

Si existe:

```text
Transaction ACTIVE
```

`flush()` deberá reutilizarla.

Nunca abrir otra transacción independiente.

---

# 53. Auto-transaction example

Sin transacción activa:

```text
flush()
  ↓
BEGIN
  ↓
persistence operations
  ↓
COMMIT
```

si la configuración lo permite.

---

# 54. Explicit example

```text
BEGIN
  ↓
flush A
  ↓
domain work
  ↓
flush B
  ↓
COMMIT
```

Ambos flushes pertenecen a la misma transacción.

---

# 55. Failure during flush

Si una operación de persistence falla:

```text
ACTIVE
   ↓
statement failure
```

la transacción puede quedar:

```text
ACTIVE
ROLLBACK_ONLY
ABORTED_BY_DATABASE
UNKNOWN
```

dependiendo de la plataforma y el error.

---

# 56. PostgreSQL-like aborted transaction behavior

Algunas plataformas invalidan el resto de la transacción después de ciertos errores hasta rollback.

VoltStack no deberá asumir que todas se comportan igual.

---

# 57. Platform transaction capabilities

Conceptualmente:

```php
interface TransactionCapabilities
{
    public function supportsTransactions(): bool;

    public function supportsSavepoints(): bool;

    public function supportsIsolation(
        TransactionIsolation $level,
    ): bool;

    public function supportsReadOnlyTransactions(): bool;

    public function transactionAbortsAfterStatementError(): bool;
}
```

---

# 58. Capability-driven behavior

Nunca:

```php
if ($driver === 'pgsql') {
}
```

en Transaction Manager.

Usar:

```text
PlatformCapabilities
```

---

# 59. Transaction Isolation

VoltStack deberá representar niveles lógicos:

```php
enum TransactionIsolation
{
    case DEFAULT;
    case READ_UNCOMMITTED;
    case READ_COMMITTED;
    case REPEATABLE_READ;
    case SERIALIZABLE;
}
```

Podrán existir extensiones platform-specific.

---

# 60. Isolation ≠ exact universal semantics

Aunque SQL use nombres comunes:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

las garantías concretas pueden variar por DB.

VoltStack no deberá afirmar equivalencia perfecta entre plataformas.

---

# 61. Isolation resolution

```text
Requested Isolation
        ↓
Platform Capability
        ↓
Effective Isolation
```

---

# 62. Effective isolation

Transaction Context deberá poder conocer:

```text
requested isolation
effective isolation
```

cuando pueda determinarse.

---

# 63. Unsupported isolation

Policy:

```text
FAIL
DOWNGRADE_EXPLICITLY
PLATFORM_DEFAULT
```

Nunca downgrade silencioso por default.

---

# 64. Isolation lifecycle

Algunas plataformas requieren configurar isolation:

```text
before BEGIN
```

otras permiten diferentes mecanismos.

Platform adapter decidirá.

---

# 65. Nested Transactions

Código como:

```php
$db->transaction(function () {
    // A

    $db->transaction(function () {
        // B
    });
});
```

requiere semántica explícita.

---

# 66. Nested transaction ≠ second physical transaction

En una misma conexión:

```text
BEGIN
BEGIN
```

normalmente no representa dos transacciones físicas independientes.

Por tanto:

```text
Nested Logical Transaction
≠
Nested Physical Transaction
```

---

# 67. Nested strategies

VoltStack podrá soportar:

```text
JOIN_EXISTING
SAVEPOINT
FORBID
```

---

# 68. JOIN_EXISTING

```text
Outer Transaction
       ↓
Inner transaction joins
       ↓
same physical transaction
```

---

# 69. SAVEPOINT

```text
BEGIN
  ↓
SAVEPOINT inner_1
  ↓
inner work
  ↓
RELEASE SAVEPOINT
  ↓
COMMIT
```

---

# 70. FORBID

En entornos donde la semántica sería peligrosa:

```text
nested transaction request
→ exception
```

---

# 71. Nested failure

Con JOIN_EXISTING:

```text
inner failure
→ outer transaction rollback-only
```

por default.

---

# 72. Rollback-only

Estado:

```text
MARKED_ROLLBACK_ONLY
```

significa:

> la transacción continúa existiendo localmente, pero ya no puede terminar válidamente mediante commit.

---

# 73. Commit rollback-only

Si:

```text
state = MARKED_ROLLBACK_ONLY
```

entonces:

```text
commit()
```

deberá rechazarse y normalmente provocar rollback.

---

# 74. Why rollback-only

Evita que código interno capture una excepción y accidentalmente confirme una transacción ya comprometida semánticamente.

---

# 75. Example

```php
$db->transaction(function () {
    try {
        $service->dangerousOperation();
    } catch (\Throwable $e) {
        // swallowed
    }

    // Transaction may already be rollback-only.
});
```

---

# 76. Savepoints

Savepoint será concepto propio:

```text
Savepoint
```

no un string SQL arbitrario.

---

# 77. Savepoint identity

```php
final readonly class SavepointId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 78. Savepoint operations

```text
CREATE
ROLLBACK_TO
RELEASE
```

serán gestionadas mediante Platform/Driver abstraction.

---

# 79. Savepoint ≠ nested transaction

Savepoints son una herramienta física.

Nested transaction es una abstracción lógica.

VoltStack podrá implementar una usando la otra.

---

# 80. Savepoint stack

Transaction Context podrá mantener:

```text
SavepointStack
```

scoped a la transacción.

---

# 81. Savepoint naming

Los nombres deberán:

- generarse internamente;
- evitar colisiones;
- no derivarse directamente de user input;
- respetar Platform identifier constraints.

---

# 82. Transaction propagation

Se podrá modelar:

```php
enum TransactionPropagation
{
    case REQUIRED;
    case REQUIRES_NEW;
    case MANDATORY;
    case NEVER;
    case SUPPORTS;
}
```

No necesariamente todas estarán habilitadas inicialmente.

---

# 83. REQUIRED

```text
existing transaction?
    yes → join
    no  → create
```

---

# 84. MANDATORY

```text
existing transaction?
    yes → use
    no  → error
```

---

# 85. NEVER

```text
existing transaction?
    yes → error
    no  → execute
```

---

# 86. SUPPORTS

```text
existing transaction?
    yes → participate
    no  → execute non-transactionally
```

---

# 87. REQUIRES_NEW complexity

`REQUIRES_NEW` requiere:

```text
suspend current transaction
acquire another connection
run independent transaction
resume outer context
```

No deberá simularse mediante savepoint porque:

```text
savepoint
≠
independent transaction
```

---

# 88. REQUIRES_NEW initial policy

Puede quedar:

```text
unsupported
```

hasta que Connection Manager soporte suspensión segura.

---

# 89. Transaction callbacks

Se necesitarán callbacks:

```text
beforeCommit
afterCommit
afterRollback
afterCompletion
```

---

# 90. Callback semantics

Especialmente:

```text
afterCommit
```

solo se ejecutará cuando outcome sea:

```text
COMMITTED
```

---

# 91. UNKNOWN outcome callbacks

Si outcome:

```text
UNKNOWN
```

no ejecutar:

```text
afterCommit
```

ni:

```text
afterRollback
```

como si existiera certeza.

Usar:

```text
afterCompletion(UNKNOWN)
```

---

# 92. Why afterCommit matters

Permite coordinar operaciones como:

```text
dispatch job after commit
publish domain event after commit
invalidate cache after commit
```

---

# 93. External side effects

La transacción DB no deberá prometer atomicidad con:

```text
HTTP APIs
email
filesystem
message brokers
external storage
```

---

# 94. Example anti-pattern

```php
$db->transaction(function () {
    $paymentGateway->charge();

    $order->save();
});
```

Rollback DB no deshace automáticamente:

```text
external charge
```

---

# 95. Distributed consistency

Casos de DB + broker deberán utilizar patrones superiores como:

```text
Transactional Outbox
Saga
Idempotency
```

cuando corresponda.

No se resolverán fingiendo una transacción distribuida.

---

# 96. No implicit 2PC

VoltStack Database v1 no deberá asumir:

```text
Two-Phase Commit
```

entre múltiples recursos.

---

# 97. Single-resource transaction

La transacción estándar será:

```text
one database connection
one transactional resource
```

---

# 98. Multi-database operations

Ejemplo:

```text
DB A
+
DB B
```

no podrá presentarse como una única transacción ACID por default.

---

# 99. Cross-connection guard

Si una transacción activa intenta utilizar una conexión incompatible:

```text
CrossTransactionConnectionException
```

salvo operación explícitamente separada.

---

# 100. Transaction-scoped resources

El Transaction Context podrá almacenar recursos como:

```text
afterCommit callbacks
afterRollback callbacks
outbox coordination state
transaction-local diagnostics
savepoint metadata
```

---

# 101. Resource lifecycle

```text
BEGIN
 ↓
allocate transaction-local state
 ↓
ACTIVE
 ↓
completion
 ↓
callbacks
 ↓
clear
```

---

# 102. Cleanup invariant

Toda finalización deberá limpiar:

```text
transaction context
connection ownership
savepoint stack
callbacks
temporary state
```

incluso ante excepción.

---

# 103. Cleanup ≠ outcome rewrite

Aunque cleanup local sea exitoso:

```text
UNKNOWN
```

seguirá siendo `UNKNOWN`.

---

# 104. Transaction timeout

VoltStack podrá modelar:

```text
TransactionTimeout
```

independientemente de:

```text
QueryTimeout
```

---

# 105. Query timeout ≠ transaction timeout

```text
Query timeout
```

limita una operación.

```text
Transaction timeout
```

limita el scope transaccional.

---

# 106. Timeout policy

Al expirar:

```text
Transaction
→ rollback-only
```

y se intentará rollback cuando sea seguro.

---

# 107. Database-side timeout

Cuando Platform soporte:

```text
statement timeout
lock timeout
transaction timeout
```

se configurará mediante capabilities explícitas.

---

# 108. Cancellation

Si una request/job es cancelada durante una transacción:

```text
ACTIVE
→ cancellation
→ rollback attempt
```

pero si se pierde evidencia:

```text
UNKNOWN
```

---

# 109. Fatal worker termination

No se podrá garantizar cleanup PHP si el proceso termina abruptamente.

La base de datos normalmente cerrará la conexión y aplicará sus propias garantías.

VoltStack no deberá afirmar que ejecutó rollback si no existe evidencia.

---

# 110. Retry Architecture

Retry deberá ocurrir sobre:

```text
entire transaction operation
```

cuando sea seguro.

No sobre statements arbitrarios dentro de una transacción.

---

# 111. Correct retry model

```text
Attempt 1
BEGIN
work
deadlock
ROLLBACK
        ↓
Retry Policy
        ↓
Attempt 2
BEGIN
work again
COMMIT
```

---

# 112. Wrong retry model

```text
BEGIN
UPDATE A
UPDATE B → deadlock
retry UPDATE B only
COMMIT
```

Esto puede romper invariantes de dominio.

---

# 113. Retry requires replayable work

El callback transaccional debe poder ejecutarse nuevamente.

---

# 114. Replayability

Un callback no es automáticamente replayable si contiene:

```text
external HTTP request
email send
random non-recorded decision
non-idempotent side effect
filesystem mutation
```

---

# 115. Retry classification

Errores podrán clasificarse:

```text
RETRYABLE
NON_RETRYABLE
UNKNOWN
```

---

# 116. UNKNOWN transaction outcome

Nunca realizar retry automático de toda la transacción cuando el commit anterior quedó:

```text
UNKNOWN
```

salvo estrategia de idempotency/reconciliation explícita.

---

# 117. Deadlocks

Deadlock generalmente podrá ser:

```text
retryable transaction failure
```

si:

```text
rollback outcome is known
```

y el trabajo es replayable.

---

# 118. Serialization failures

Podrán ser candidatos a retry dependiendo de:

```text
Platform
Isolation Level
Error Classification
Retry Policy
```

---

# 119. Retry budget

Política:

```text
max attempts
backoff
jitter
deadline
error classes
```

---

# 120. Retry attempt identity

Cada intento tendrá:

```text
TransactionAttempt
```

distinto.

La operación lógica podrá compartir:

```text
TransactionOperationId
```

---

# 121. Example

```text
Operation tx-op-42

Attempt 1:
    tx-101
    DEADLOCK
    ROLLED_BACK

Attempt 2:
    tx-102
    COMMITTED
```

---

# 122. Optimistic locking

La arquitectura de transacciones se integrará posteriormente con:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

Optimistic locking:

```text
version check
```

no será parte del Transaction Manager core.

---

# 123. Pessimistic locking

Se integrará con:

```text
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md
```

y deberá respetar:

```text
transaction connection affinity
```

---

# 124. Lock outside transaction

Algunos locks pierden sentido sin una transacción que mantenga la conexión/contexto.

El Query Engine deberá poder exigir:

```text
active transaction required
```

para ciertas operaciones.

---

# 125. Concurrency control

El documento:

```text
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
```

integrará:

```text
isolation
optimistic locking
pessimistic locking
deadlocks
retry
```

sin mezclar sus responsabilidades.

---

# 126. EntityManager lifecycle

Antes:

```text
EntityManager OPEN
```

Durante transaction:

```text
OPEN
```

Después de commit conocido:

```text
OPEN
```

normalmente.

---

# 127. EntityManager after rollback

Dependiendo de los cambios ejecutados:

```text
OPEN but stale
CLEAR_REQUIRED
TAINTED
```

podrán ser estados/policies.

---

# 128. EntityManager after UNKNOWN

Default seguro:

```text
TAINTED
```

porque:

```text
DatabaseReality
```

es desconocida.

---

# 129. No further flush after UNKNOWN

Por default:

```text
Transaction UNKNOWN
→ EntityManager TAINTED
→ further flush rejected
```

hasta clear/reset/reconciliation explícita.

---

# 130. IdentityMap implications

Rollback no elimina automáticamente:

```text
IdentityMap
```

pero sus entidades pueden dejar de representar la realidad persistente.

---

# 131. Generated identifiers

Caso:

```text
INSERT
→ generated ID 42
→ rollback
```

El objeto PHP puede conservar:

```text
id = 42
```

aunque la fila no exista.

---

# 132. Generated ID rollback policy

Esto deberá tratarse explícitamente por Persistence Consistency.

Nunca asumir que:

```text
rollback → set ID back to null
```

es universalmente correcto.

---

# 133. Auto-increment gaps

Rollback puede no reutilizar IDs.

Por tanto:

```text
generated identifier
```

no implica que la entidad exista después de rollback.

---

# 134. afterCommit entity state

Después de commit conocido:

```text
PersistentSnapshot
```

puede considerarse confirmado según las reglas del Persistence System.

---

# 135. Flush before commit

Antes del commit:

```text
snapshot synchronized with transactional state
```

pero todavía no necesariamente con durable committed reality.

---

# 136. Persistence confidence

Podrán distinguirse:

```text
EXECUTED_IN_TRANSACTION
COMMITTED
UNKNOWN
```

en información interna de consistencia.

---

# 137. Query execution integration

Query Executor deberá consultar:

```text
TransactionContext
```

para resolver:

```text
connection
transaction restrictions
timeout budget
lock rules
```

---

# 138. Executor does not own transaction

Query Executor:

```text
uses transaction
```

pero no decide automáticamente:

```text
commit
rollback
```

---

# 139. Transaction-aware connection resolution

Conceptualmente:

```text
if TransactionContext.active:
    return transaction.pinnedConnection

else:
    normal connection routing
```

---

# 140. Connection pool integration

Una conexión transaccional:

```text
MUST NOT
```

volver al pool mientras la transacción esté activa.

---

# 141. Pool release

Solo después de:

```text
COMMITTED
ROLLED_BACK
known clean connection state
```

podrá reutilizarse normalmente.

---

# 142. UNKNOWN connection

Si outcome:

```text
UNKNOWN
```

la conexión deberá:

```text
discard / quarantine
```

por default.

---

# 143. Failed rollback connection

Igualmente:

```text
rollback failure
```

puede volver la conexión no reusable.

---

# 144. Connection reset

Antes de devolver una conexión al pool deberá verificarse:

```text
no active transaction
no savepoints
session state reset
```

según `DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM`.

---

# 145. Transaction session state

Cambios como:

```text
isolation
read-only mode
timeouts
constraints
```

pueden afectar la sesión.

Deberán restaurarse/resetearse cuando corresponda.

---

# 146. Transaction and schema operations

DDL transactional behavior varía entre plataformas.

VoltStack no deberá asumir:

```text
DDL is always transactional
```

---

# 147. Migration transaction policy

Migration System deberá consultar:

```text
Platform DDL transaction capabilities
```

antes de envolver migrations.

---

# 148. Transactional DDL ≠ ordinary DML transaction

Podrán existir restricciones específicas.

---

# 149. Event architecture

Eventos conceptuales:

```text
TransactionStarting
TransactionStarted
TransactionMarkedRollbackOnly
TransactionCommitting
TransactionCommitted
TransactionRollingBack
TransactionRolledBack
TransactionFailed
TransactionOutcomeUnknown
TransactionRetryScheduled
TransactionRetried
```

---

# 150. Event ordering

Ejemplo:

```text
TransactionStarting
TransactionStarted
TransactionCommitting
TransactionCommitted
```

---

# 151. Unknown commit ordering

```text
TransactionCommitting
TransactionOutcomeUnknown
```

Nunca:

```text
TransactionCommitted
```

sin confirmación.

---

# 152. Event listeners ≠ transaction authority

Un listener no deberá cambiar arbitrariamente:

```text
transaction state
```

fuera de APIs controladas.

---

# 153. Transaction events vs callbacks

Distinguir:

```text
Framework Events
```

de:

```text
Transaction Completion Callbacks
```

Los callbacks forman parte de coordinación local del transaction scope.

Los events sirven para integración/observability.

---

# 154. Telemetry architecture

Cada transacción podrá producir:

```text
database.transaction.started
database.transaction.committed
database.transaction.rolled_back
database.transaction.failed
database.transaction.unknown
database.transaction.duration
database.transaction.retry
database.transaction.deadlock
database.transaction.savepoint
database.transaction.rollback_only
```

---

# 155. Transaction span

Podrá existir:

```text
database.transaction
```

span alrededor del scope.

Atributos bounded:

```text
db.system
transaction.isolation
transaction.read_only
transaction.outcome
transaction.retry_attempt
```

---

# 156. Sensitive information

No incluir:

```text
SQL values
credentials
entity payloads
tenant secrets
```

en transaction telemetry.

---

# 157. Long transaction detection

Profiler podrá detectar:

```text
long-running transaction
```

porque estas pueden:

- mantener locks;
- aumentar MVCC pressure;
- consumir pool connections;
- provocar contention.

---

# 158. Transaction diagnostics

API conceptual:

```php
Database::transactions()
    ->explainCurrent();
```

---

# 159. Diagnostic example

```text
DATABASE TRANSACTION

ID:
    tx-42

State:
    ACTIVE

Outcome:
    pending

Connection:
    primary/default#7

Isolation:
    requested: READ_COMMITTED
    effective: READ_COMMITTED

Read Only:
    false

Nested Level:
    1

Savepoints:
    0

Rollback Only:
    false

Retry Attempt:
    1

Duration:
    84 ms
```

---

# 160. Unknown diagnostic

```text
DATABASE TRANSACTION

ID:
    tx-43

State:
    UNKNOWN

Last Operation:
    COMMIT

Reason:
    Connection lost after commit dispatch

Database Outcome:
    UNKNOWN

Connection:
    quarantined

EntityManager:
    TAINTED

Automatic Retry:
    FORBIDDEN

Recommended Action:
    reconcile application state using an idempotency/business identifier
```

---

# 161. Error hierarchy

```text
DatabaseTransactionException
├── TransactionNotSupportedException
├── TransactionAlreadyActiveException
├── NoActiveTransactionException
├── InvalidTransactionStateException
├── TransactionBeginException
├── TransactionCommitException
├── TransactionRollbackException
├── TransactionOutcomeUnknownException
├── TransactionRollbackOnlyException
├── TransactionConnectionException
├── TransactionConnectionMismatchException
├── TransactionOwnershipException
├── TransactionIsolationException
├── UnsupportedIsolationLevelException
├── NestedTransactionException
├── SavepointException
├── TransactionTimeoutException
├── TransactionCancelledException
├── TransactionRetryException
├── TransactionRetryExhaustedException
├── TransactionDeadlockException
├── TransactionSerializationFailureException
├── TransactionResourceException
├── TransactionRuntimeStateException
└── TransactionInvariantViolationException
```

---

# 162. Error classification

Además del tipo de excepción:

```text
FailureStage
```

podrá indicar:

```text
BEFORE_BEGIN
DURING_BEGIN
DURING_WORK
BEFORE_COMMIT
DURING_COMMIT
AFTER_COMMIT
DURING_ROLLBACK
AFTER_ROLLBACK
```

---

# 163. Outcome confidence

Podrá existir:

```text
CERTAIN
UNCERTAIN
UNKNOWN
```

para diagnostics internos.

---

# 164. Failure ≠ outcome

Ejemplo:

```text
callback afterCommit failed
```

pero:

```text
database transaction = COMMITTED
```

El error posterior no deberá reescribir el outcome.

---

# 165. Critical rule

> **Una excepción después de un commit confirmado no transforma una transacción comprometida en una transacción fallida o revertida.**

---

# 166. afterCommit failure

Ejemplo:

```text
COMMIT confirmed
   ↓
afterCommit callback
   ↓
exception
```

Resultado DB:

```text
COMMITTED
```

Resultado callback:

```text
FAILED
```

Deben reportarse separadamente.

---

# 167. afterRollback failure

Misma regla:

```text
ROLLBACK confirmed
```

no se vuelve `UNKNOWN` porque un callback posterior falló.

---

# 168. Transaction result

Podrá modelarse:

```php
final readonly class TransactionResult
{
    public function __construct(
        public TransactionId $id,
        public TransactionState $state,
        public TransactionOutcome $outcome,
        public TransactionCompletionStatus $completion,
    ) {}
}
```

---

# 169. Completion status

Podrá capturar:

```text
SUCCESS
CALLBACK_FAILURE
CLEANUP_FAILURE
UNKNOWN
```

separado del database outcome.

---

# 170. Transactional API

Developer-facing:

```php
$result = DB::transaction(function () use ($order) {
    $order->confirm();

    return $order;
});
```

---

# 171. Explicit API

También:

```php
$transaction = DB::beginTransaction();

try {
    // ...

    DB::commit($transaction);
} catch (\Throwable $e) {
    DB::rollback($transaction);

    throw $e;
}
```

---

# 172. Preferred API

Callback API será preferible para la mayoría de casos porque facilita:

```text
scope cleanup
rollback
retry
telemetry
context propagation
```

---

# 173. Manual API remains necessary

Para casos avanzados:

```text
stream processing
interactive workflows
framework integrations
```

podrá utilizarse manual transaction API.

---

# 174. Manual API safety

Una transacción manual deberá cerrarse antes de abandonar su scope.

---

# 175. Scope leak

Si termina una request con:

```text
ACTIVE transaction
```

VoltStack deberá:

```text
detect
attempt rollback
discard connection if needed
report invariant violation
```

---

# 176. Never auto-commit on scope end

Regla:

> **Una transacción olvidada jamás será auto-committed durante cleanup.**

---

# 177. Cleanup policy

Preferir:

```text
ACTIVE at scope end
→ rollback attempt
```

no:

```text
→ commit
```

---

# 178. Destructor policy

No confiar en:

```php
__destruct()
```

para garantizar commit/rollback.

---

# 179. Persistent runtime leak detector

Al final del scope:

```text
assert no active transaction contexts
```

en testing/debug mode.

---

# 180. Testing architecture

Deberá existir:

```text
TransactionConformanceSuite
```

para drivers/platforms.

---

# 181. Begin/commit tests

```text
BEGIN
→ ACTIVE
→ COMMIT
→ COMMITTED
```

---

# 182. Rollback tests

```text
BEGIN
→ ACTIVE
→ ROLLBACK
→ ROLLED_BACK
```

---

# 183. Rollback-only tests

```text
BEGIN
→ markRollbackOnly
→ commit request
→ rejected
→ rollback
```

---

# 184. Unknown outcome tests

Simular:

```text
connection failure after COMMIT dispatch
```

y verificar:

```text
UNKNOWN
```

---

# 185. No false rollback test

El framework nunca deberá reportar:

```text
ROLLED_BACK
```

en un escenario de commit incierto.

---

# 186. Nested tests

Probar:

```text
JOIN_EXISTING
SAVEPOINT
FORBID
```

---

# 187. Savepoint tests

```text
BEGIN
SAVEPOINT
work
ROLLBACK TO SAVEPOINT
outer work
COMMIT
```

---

# 188. Isolation tests

Por plataforma:

```text
requested
effective
unsupported
```

---

# 189. Retry tests

Probar:

```text
deadlock
serialization failure
retry exhaustion
non-replayable operation
unknown commit
```

---

# 190. Connection affinity tests

Verificar:

```text
all queries inside transaction
→ same pinned connection
```

---

# 191. Pool tests

Verificar que:

```text
active transactional connection
```

nunca sea prestada a otro scope.

---

# 192. Worker isolation tests

```text
Request A transaction
Request B transaction
```

no comparten:

- context;
- savepoints;
- callbacks;
- pinned connection state.

---

# 193. Coroutine tests

Simular múltiples transacciones concurrentes en un mismo worker.

---

# 194. ORM integration tests

Probar:

```text
persist
flush
rollback
object state
```

y:

```text
persist
flush
commit
```

---

# 195. Generated ID rollback tests

Verificar que VoltStack no suponga:

```text
rollback → identifier never existed
```

---

# 196. Callback tests

```text
afterCommit
afterRollback
afterCompletion
```

deben ejecutarse solo bajo outcomes correctos.

---

# 197. Callback failure tests

Commit confirmado + callback fallido:

```text
DB outcome = COMMITTED
callback outcome = FAILED
```

---

# 198. Transaction timeout tests

Verificar:

```text
timeout
→ rollback-only
→ rollback attempt
```

---

# 199. Cancellation tests

Verificar cleanup y uncertainty preservation.

---

# 200. Cross-platform conformance

Suite mínima:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 201. Directory structure propuesta

```text
src/Quantum/Database/Transaction/
│
├── Contract/
│   ├── Transaction.php
│   ├── TransactionManager.php
│   ├── TransactionContext.php
│   ├── TransactionPlatformAdapter.php
│   └── TransactionCapabilities.php
│
├── Definition/
│   ├── TransactionDefinition.php
│   ├── TransactionId.php
│   ├── TransactionOperationId.php
│   ├── TransactionTimeout.php
│   └── TransactionOwner.php
│
├── State/
│   ├── TransactionState.php
│   ├── TransactionOutcome.php
│   ├── TransactionStateMachine.php
│   ├── TransactionCompletionStatus.php
│   └── RollbackOnlyState.php
│
├── Context/
│   ├── DefaultTransactionContext.php
│   ├── TransactionContextResolver.php
│   ├── TransactionScope.php
│   └── TransactionResourceRegistry.php
│
├── Isolation/
│   ├── TransactionIsolation.php
│   ├── IsolationResolver.php
│   └── EffectiveIsolation.php
│
├── Propagation/
│   ├── TransactionPropagation.php
│   └── TransactionPropagationResolver.php
│
├── Nested/
│   ├── NestedTransactionStrategy.php
│   ├── NestedTransactionCoordinator.php
│   └── NestedTransactionLevel.php
│
├── Savepoint/
│   ├── Savepoint.php
│   ├── SavepointId.php
│   ├── SavepointManager.php
│   └── SavepointStack.php
│
├── Completion/
│   ├── TransactionResult.php
│   ├── TransactionCallbackRegistry.php
│   └── TransactionCompletionCoordinator.php
│
├── Retry/
│   ├── TransactionAttempt.php
│   ├── TransactionRetryPolicy.php
│   └── TransactionFailureClassifier.php
│
├── Event/
│   └── ...
│
├── Telemetry/
│   └── ...
│
├── Diagnostics/
│   ├── TransactionExplainer.php
│   └── TransactionDiagnosticReport.php
│
├── Testing/
│   └── TransactionConformanceSuite.php
│
└── Exception/
    ├── DatabaseTransactionException.php
    ├── TransactionBeginException.php
    ├── TransactionCommitException.php
    ├── TransactionRollbackException.php
    ├── TransactionOutcomeUnknownException.php
    ├── TransactionRollbackOnlyException.php
    ├── TransactionConnectionMismatchException.php
    ├── TransactionOwnershipException.php
    ├── NestedTransactionException.php
    ├── SavepointException.php
    ├── TransactionTimeoutException.php
    └── TransactionInvariantViolationException.php
```

---

# 202. Dependency architecture

Permitido:

```text
Transaction
    ↓
Connection Manager
    ↓
Connection
    ↓
Driver
```

También:

```text
ORM
 ↓
Transaction contracts
```

y:

```text
Query Executor
 ↓
Transaction Context
```

---

# 203. Forbidden dependencies

Transaction core no deberá depender de:

```text
ORM entities
Repositories
Model API
HTTP controllers
Jobs
Authentication
Authorization
specific runtime
```

---

# 204. Runtime adapters

Integraciones específicas:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

deberán conectarse mediante lifecycle/context contracts.

---

# 205. Transaction architecture matrix

| Sistema | Responsabilidad |
|---|---|
| Transaction Manager | Coordinar boundary |
| Transaction Context | Estado scoped |
| Connection Manager | Resolver/pinear conexión |
| Driver | Ejecutar comandos transaccionales |
| Platform | Capabilities/semántica DB |
| Query Executor | Ejecutar queries dentro del contexto |
| UnitOfWork | Detectar y planear cambios |
| Persistence Engine | Convertir cambios en operaciones |
| EntityManager | Coordinar ORM |
| Retry System | Reejecutar operación completa |
| Locking Systems | Control de concurrencia |
| Event System | Publicar lifecycle events |
| Telemetry | Observar |

---

# 206. Architectural invariants

## DB-TX-001
Una transacción tendrá boundary explícito.

## DB-TX-002
Transaction no será equivalente a UnitOfWork.

## DB-TX-003
Transaction no será equivalente a EntityManager.

## DB-TX-004
Transaction no será equivalente a HTTP Request.

## DB-TX-005
Transaction no será equivalente a Connection.

## DB-TX-006
`flush()` no será equivalente a `commit()`.

## DB-TX-007
`persist()` no iniciará necesariamente una transacción.

## DB-TX-008
Statement success no implicará transaction commit.

## DB-TX-009
Transaction commit solo se declarará con evidencia suficiente.

## DB-TX-010
Commit uncertainty producirá UNKNOWN.

## DB-TX-011
UNKNOWN nunca se convertirá silenciosamente en rollback.

## DB-TX-012
Rollback uncertainty podrá producir UNKNOWN.

## DB-TX-013
Transaction tendrá connection affinity.

## DB-TX-014
Una transacción activa mantendrá pinned connection.

## DB-TX-015
Queries participantes usarán la pinned connection.

## DB-TX-016
Connection routing normal será suspendido para queries participantes.

## DB-TX-017
Read-only no implicará replica automáticamente.

## DB-TX-018
Transaction Context será scoped.

## DB-TX-019
Transaction Context no será mutable global static state.

## DB-TX-020
Transaction ownership será explícito.

## DB-TX-021
Cross-scope transaction use será rechazado por default.

## DB-TX-022
Active transaction connection no volverá al pool.

## DB-TX-023
UNKNOWN connection será descartada o quarantined.

## DB-TX-024
Failed rollback connection no será reutilizada sin validación.

## DB-TX-025
Transaction cleanup no cambiará el database outcome.

## DB-TX-026
Rollback DB no rebobinará automáticamente objetos PHP.

## DB-TX-027
Rollback podrá dejar ORM state stale.

## DB-TX-028
UNKNOWN podrá taint EntityManager.

## DB-TX-029
Further flush sobre tainted manager será rechazado por default.

## DB-TX-030
Generated identifier no probará existencia después de rollback.

## DB-TX-031
IdentityMap no será automáticamente restaurado por rollback.

## DB-TX-032
Explicit active transaction será reutilizada por flush.

## DB-TX-033
Flush no abrirá una segunda physical transaction dentro de una activa.

## DB-TX-034
Implicit flush transaction será policy explícita.

## DB-TX-035
Platform transaction semantics serán capability-driven.

## DB-TX-036
Vendor conditionals no pertenecerán al Transaction Manager core.

## DB-TX-037
Requested isolation será distinta de effective isolation.

## DB-TX-038
Isolation downgrade no será silencioso.

## DB-TX-039
Isolation semantics no se asumirán idénticas cross-platform.

## DB-TX-040
Nested logical transaction no será confundida con physical transaction.

## DB-TX-041
Nested strategy será explícita.

## DB-TX-042
JOIN_EXISTING reutilizará la transacción exterior.

## DB-TX-043
Inner failure podrá marcar rollback-only.

## DB-TX-044
Rollback-only transaction no podrá commit exitosamente.

## DB-TX-045
Savepoint no será equivalente a nested transaction.

## DB-TX-046
Savepoints dependerán de Platform capability.

## DB-TX-047
Savepoint identifiers serán generados de forma segura.

## DB-TX-048
REQUIRES_NEW no será simulado con savepoint.

## DB-TX-049
REQUIRES_NEW requerirá transacción independiente.

## DB-TX-050
afterCommit solo se ejecutará después de COMMITTED conocido.

## DB-TX-051
afterRollback solo se ejecutará después de ROLLED_BACK conocido.

## DB-TX-052
UNKNOWN usará completion semantics propias.

## DB-TX-053
Callback failure posterior a commit no cambiará COMMITTED.

## DB-TX-054
Callback failure posterior a rollback no cambiará ROLLED_BACK.

## DB-TX-055
Transaction DB no cubrirá automáticamente external side effects.

## DB-TX-056
No se prometerá atomicidad DB + HTTP.

## DB-TX-057
No se prometerá atomicidad DB + filesystem.

## DB-TX-058
No se prometerá atomicidad DB + broker sin protocolo adicional.

## DB-TX-059
2PC no será implícito.

## DB-TX-060
Transacción estándar abarcará un único transactional resource.

## DB-TX-061
Cross-connection participation será rechazada por default.

## DB-TX-062
Transaction-local resources se limpiarán al finalizar.

## DB-TX-063
Cleanup failure será distinta del DB outcome.

## DB-TX-064
Transaction timeout será distinto de query timeout.

## DB-TX-065
Expired transaction podrá ser rollback-only.

## DB-TX-066
Cancellation intentará rollback cuando sea posible.

## DB-TX-067
Fatal process termination no será reportado falsamente como rollback ejecutado.

## DB-TX-068
Retry operará sobre transaction operation completa.

## DB-TX-069
Statement retry arbitrario dentro de transaction estará prohibido por default.

## DB-TX-070
Retry requerirá failure classification.

## DB-TX-071
Retry requerirá known rollback outcome.

## DB-TX-072
UNKNOWN commit no será auto-retried.

## DB-TX-073
Retry deberá respetar replayability.

## DB-TX-074
External non-idempotent side effects reducirán replayability.

## DB-TX-075
Retry tendrá bounded attempts.

## DB-TX-076
Retry tendrá deadline/backoff policy.

## DB-TX-077
Cada retry podrá tener TransactionId diferente.

## DB-TX-078
Logical transaction operation podrá conservar correlation ID.

## DB-TX-079
Deadlock podrá ser retryable.

## DB-TX-080
Deadlock no será siempre retryable automáticamente.

## DB-TX-081
Serialization failure será platform-classified.

## DB-TX-082
Pessimistic locks respetarán transaction affinity.

## DB-TX-083
Operations requiring active transaction deberán validarlo.

## DB-TX-084
Optimistic locking permanecerá separado del Transaction Manager.

## DB-TX-085
Query Executor consumirá Transaction Context.

## DB-TX-086
Query Executor no poseerá commit authority.

## DB-TX-087
Connection Manager administrará pinning.

## DB-TX-088
Transaction Manager no generará SQL de negocio.

## DB-TX-089
Transaction Manager no hidratará entities.

## DB-TX-090
Transaction Manager no hará change tracking.

## DB-TX-091
Transaction Manager no será Query Planner.

## DB-TX-092
Transaction Manager no será Driver.

## DB-TX-093
DDL transaction behavior será capability-driven.

## DB-TX-094
Migration System no asumirá transactional DDL.

## DB-TX-095
Transaction events reflejarán estados reales.

## DB-TX-096
TransactionCommitted nunca se emitirá para UNKNOWN.

## DB-TX-097
TransactionRolledBack nunca se emitirá para UNKNOWN.

## DB-TX-098
Events no serán autoridad para mutar state machine.

## DB-TX-099
Telemetry no contendrá sensitive DB values.

## DB-TX-100
Long transaction detection será observable.

## DB-TX-101
Transaction diagnostics mostrarán requested/effective isolation.

## DB-TX-102
Transaction diagnostics mostrarán rollback-only.

## DB-TX-103
Transaction diagnostics mostrarán retry attempt.

## DB-TX-104
Transaction diagnostics preservarán UNKNOWN.

## DB-TX-105
Active transaction al finalizar scope será invariant violation.

## DB-TX-106
Scope cleanup nunca auto-commiteará una transacción olvidada.

## DB-TX-107
Scope cleanup intentará rollback.

## DB-TX-108
Destructors no serán transaction correctness mechanism.

## DB-TX-109
Persistent runtime reset verificará transaction leaks.

## DB-TX-110
FrankenPHP requests tendrán transaction contexts aislados.

## DB-TX-111
RoadRunner jobs tendrán transaction contexts aislados.

## DB-TX-112
OpenSwoole coroutines tendrán transaction contexts aislados.

## DB-TX-113
Shared mutable transaction state entre coroutines estará prohibido.

## DB-TX-114
Transaction state transitions serán validadas.

## DB-TX-115
COMMITTED será terminal.

## DB-TX-116
ROLLED_BACK será terminal.

## DB-TX-117
UNKNOWN será terminal para esa transaction instance.

## DB-TX-118
No se reutilizará una Transaction instance terminal.

## DB-TX-119
Double commit será rechazado.

## DB-TX-120
Double rollback será rechazado o tratado idempotentemente solo si el contrato lo define explícitamente.

## DB-TX-121
Commit sobre NEW será rechazado.

## DB-TX-122
Rollback sobre NEW no inventará una transacción.

## DB-TX-123
Begin failure no producirá ACTIVE.

## DB-TX-124
Connection pinning deberá completarse antes de ACTIVE.

## DB-TX-125
Transaction ownership deberá adquirirse antes de ACTIVE.

## DB-TX-126
Savepoint stack será transaction-scoped.

## DB-TX-127
Savepoints no sobrevivirán a transaction completion.

## DB-TX-128
Transaction callbacks serán transaction-scoped.

## DB-TX-129
Callbacks no sobrevivirán al scope.

## DB-TX-130
afterCommit podrá registrar jobs/events pero no alterar el commit ya confirmado.

## DB-TX-131
Transaction Result separará database outcome de completion processing.

## DB-TX-132
Failure después de confirmed commit no será descrito como database rollback.

## DB-TX-133
Connection loss classification deberá considerar operation stage.

## DB-TX-134
Failure stage formará parte de diagnostics.

## DB-TX-135
Transaction retry no ocultará el número de intentos.

## DB-TX-136
Transaction state será observable sin exponer mutable internals.

## DB-TX-137
Platform adapter encapsulará transaction-specific vendor semantics.

## DB-TX-138
Driver ejecutará comandos transaccionales de bajo nivel.

## DB-TX-139
Transaction core permanecerá independiente del ORM.

## DB-TX-140
ORM podrá integrarse mediante contracts sin invertir dependencias.

## DB-TX-141
Transaction architecture conservará DatabaseReality distinta de ORMKnowledge.

## DB-TX-142
Unknown database reality nunca será representada como known clean state.

## DB-TX-143
Transaction lifecycle será determinista respecto a evidencia conocida.

## DB-TX-144
Transaction retries no deberán duplicar external effects silenciosamente.

## DB-TX-145
Resource exhaustion podrá impedir nuevas transacciones antes de adquirir conexión.

## DB-TX-146
Transaction connection acquisition respetará pool/resource governance.

## DB-TX-147
Transaction duration incluirá desde activation hasta terminal outcome según definición explícita.

## DB-TX-148
Telemetry failure no deberá alterar transaction outcome.

## DB-TX-149
Diagnostics failure no deberá alterar transaction outcome.

## DB-TX-150
VoltStack preferirá incertidumbre explícita a certeza falsa.

---

# 207. Anti-patterns

## 207.1 flush = commit

```php
$em->flush();

// Assume durable commit
```

**Rechazado.**

---

## 207.2 Transaction global

```php
static Transaction $current;
```

**Rechazado.**

---

## 207.3 Automatic rollback assumption

```php
try {
    $db->commit();
} catch (\Throwable) {
    // transaction definitely rolled back
}
```

**Rechazado.**

---

## 207.4 Retry uncertain commit

```text
COMMIT timeout
→ rerun transaction
```

**Rechazado por default.**

---

## 207.5 Savepoint = independent transaction

```text
SAVEPOINT
=
REQUIRES_NEW
```

**Incorrecto.**

---

## 207.6 Return active connection to pool

```text
BEGIN
→ pool.release(connection)
```

**Prohibido.**

---

## 207.7 Auto-commit during request cleanup

```text
request finished
→ commit forgotten transaction
```

**Prohibido.**

---

## 207.8 Rollback rewinds entity

```text
ROLLBACK
→ assume PHP objects restored
```

**Incorrecto.**

---

## 207.9 Retry only failed statement

```text
transaction
├── statement A success
└── statement B deadlock
        ↓
    retry B
```

**Rechazado como política general.**

---

## 207.10 External API inside retryable transaction without policy

```php
DB::transaction(function () {
    $gateway->charge();
});
```

con retry automático.

**Peligroso.**

---

# 208. Ejemplo — Transaction básica

```php
$order = DB::transaction(function () use ($em, $order) {
    $order->confirm();

    $em->flush();

    return $order;
});
```

Flujo:

```text
transaction()
    ↓
BEGIN
    ↓
Transaction ACTIVE
    ↓
domain mutation
    ↓
flush()
    ↓
SQL execution
    ↓
COMMIT
    ↓
COMMITTED
    ↓
afterCommit
    ↓
return result
```

---

# 209. Ejemplo — Error durante trabajo

```php
DB::transaction(function () use ($service) {
    $service->execute();

    throw new DomainException();
});
```

Flujo:

```text
BEGIN
 ↓
ACTIVE
 ↓
DomainException
 ↓
ROLLBACK
 ↓
ROLLED_BACK
 ↓
rethrow DomainException
```

---

# 210. Ejemplo — Commit incierto

```text
BEGIN
 ↓
UPDATE
 ↓
COMMIT
 ↓
network connection lost
```

Resultado:

```text
Transaction:
    UNKNOWN

Connection:
    QUARANTINED

EntityManager:
    TAINTED

Automatic retry:
    NO
```

---

# 211. Ejemplo — Nested JOIN_EXISTING

```php
DB::transaction(function () {
    ServiceA::run();

    DB::transaction(function () {
        ServiceB::run();
    });
});
```

Flujo:

```text
BEGIN
 ├── A
 ├── inner joins
 │    └── B
 └── COMMIT
```

Solo existe:

```text
1 physical transaction
```

---

# 212. Ejemplo — Nested failure

```text
BEGIN outer
   ↓
inner joins
   ↓
inner failure
   ↓
outer MARKED_ROLLBACK_ONLY
   ↓
outer attempts commit
   ↓
ROLLBACK
```

---

# 213. Ejemplo — Savepoint

```text
BEGIN
 ↓
operation A
 ↓
SAVEPOINT sp1
 ↓
operation B
 ↓
B fails
 ↓
ROLLBACK TO sp1
 ↓
operation C
 ↓
COMMIT
```

Solo cuando la policy y Platform lo permitan.

---

# 214. Ejemplo — Retry

```php
DB::transaction(
    callback: function () use ($inventory) {
        $inventory->reserve();
    },
    retry: Retry::deadlocks(maxAttempts: 3),
);
```

Internamente:

```text
Attempt 1
    BEGIN
    DEADLOCK
    ROLLBACK confirmed

Attempt 2
    BEGIN
    work
    COMMIT confirmed
```

---

# 215. Ejemplo — afterCommit

```php
DB::transaction(function () use ($order) {
    $order->confirm();

    DB::afterCommit(function () use ($order) {
        DispatchOrderConfirmed::dispatch($order->id);
    });
});
```

El dispatch ocurre únicamente después de:

```text
COMMITTED
```

---

# 216. Master state formula

```text
TransactionState(t+1)
=
Transition(
    TransactionState(t),
    Operation,
    Evidence,
    PlatformSemantics
)
```

---

# 217. Master outcome formula

```text
TransactionOutcome
=
DetermineOutcome(
    LastKnownState,
    DriverEvidence,
    ConnectionEvidence,
    PlatformSemantics
)
```

Cuando la evidencia sea insuficiente:

```text
TransactionOutcome = UNKNOWN
```

---

# 218. Connection affinity formula

Durante una transacción activa:

```text
∀ Query q ∈ Transaction T:

Connection(q)
=
PinnedConnection(T)
```

para todas las queries participantes.

---

# 219. Commit correctness formula

```text
Committed(T)
⇔
SufficientEvidence(
    CommitSucceeded(T)
)
```

No:

```text
NoException
⇒
Committed
```

como única regla universal.

---

# 220. Retry safety formula

```text
RetryAllowed
=
RetryableFailure
∧
KnownRollback
∧
ReplayableWork
∧
RetryBudgetAvailable
```

Si cualquiera falla:

```text
RetryAllowed = false
```

---

# 221. ORM consistency formula

```text
DatabaseRollback
¬⇒
ObjectGraphRollback
```

y:

```text
TransactionUnknown
⇒
PersistenceKnowledgeUncertain
```

---

# 222. Arquitectura final

```text
                     APPLICATION
                          │
                          ▼
                 Transaction API
                          │
                          ▼
                Transaction Manager
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
      Context         State Machine     Retry Policy
         │                │                │
         ├────────────────┼────────────────┤
         │                │                │
         ▼                ▼                ▼
     Isolation        Savepoints        Events
         │                │                │
         └────────────────┼────────────────┘
                          ▼
                 Connection Manager
                          │
                          ▼
                   Pinned Connection
                          │
                          ▼
                Driver Transaction API
                          │
                          ▼
                       DATABASE

                          ▲
                          │
               ┌──────────┴──────────┐
               │                     │
              ORM               Query Executor
               │                     │
           UnitOfWork          Transaction-aware
               │                 execution
               └──────────┬──────────┘
                          │
                     Telemetry
```

---

# 223. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Explicit Transaction Context
```

en lugar de estado global.

Adoptará:

```text
Transaction State Machine
```

en lugar de booleans como:

```text
$isInTransaction
```

Adoptará:

```text
TransactionOutcome::UNKNOWN
```

como estado de primera clase.

Adoptará:

```text
Connection Affinity
```

durante todo transaction scope.

Adoptará:

```text
flush() ≠ commit()
```

como invariante fundamental.

Adoptará:

```text
DatabaseRollback ≠ ObjectGraphRollback
```

como regla ORM.

Adoptará:

```text
Capability-driven Isolation / Savepoints
```

en lugar de vendor conditionals.

Adoptará:

```text
Whole Transaction Retry
```

en lugar de statement retry arbitrario.

Adoptará:

```text
afterCommit / afterRollback / afterCompletion
```

con semántica basada en outcomes conocidos.

Adoptará:

```text
Scoped Transaction State
```

compatible con FrankenPHP, RoadRunner y OpenSwoole.

Y mantendrá:

```text
Transaction
≠
ORM
≠
UnitOfWork
≠
Connection
≠
Driver
```

---

# 224. Resultado del documento

Con esta arquitectura, VoltStack dispone del modelo base sobre el cual podrán construirse los siguientes subsistemas:

```text
Transaction Manager
        ↓
Transaction Context
        ↓
Isolation
        ↓
Nested Transactions
        ↓
Savepoints
        ↓
Retry
        ↓
Deadlock Handling
        ↓
Optimistic Locking
        ↓
Pessimistic Locking
        ↓
Concurrency Control
        ↓
Transaction Events
```

El bloque 15 podrá evolucionar sin mezclar:

```text
transaction boundaries
```

con:

```text
ORM persistence
```

ni confundir:

```text
statement execution
```

con:

```text
durable transaction outcome
```

La arquitectura queda fundamentada sobre una regla general de VoltStack Database:

> **Cuando la realidad de la base de datos sea incierta, el framework preservará la incertidumbre en lugar de fabricar un estado limpio que no puede demostrar.**

---

# 225. Siguiente documento

```text
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
```

El siguiente documento deberá definir en detalle:

- `TransactionManager`;
- contratos públicos;
- `begin()`;
- `commit()`;
- `rollback()`;
- `transactional()`;
- callback transaction API;
- manual transaction API;
- transaction ownership;
- connection acquisition;
- connection pinning;
- state transitions;
- rollback-only coordination;
- nested invocation;
- propagation;
- completion processing;
- callback management;
- error propagation;
- exception precedence;
- cleanup;
- connection release/quarantine;
- integration with ORM;
- integration with Query Executor;
- retry entry points;
- transaction result model;
- persistent runtime behavior;
- telemetry;
- diagnostics;
- testing;
- architectural invariants.