# 167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md

# VoltStack Quantum Database
## Database Transaction Isolation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 167 — Database Transaction Isolation System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md`  
**Siguiente documento:** `168_DATABASE_NESTED_TRANSACTION_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Isolation System` define el modelo mediante el cual VoltStack expresa, valida, negocia, aplica y observa el nivel de aislamiento de las transacciones.

El sistema deberá proporcionar una abstracción uniforme para:

- MySQL;
- MariaDB;
- PostgreSQL;
- SQLite;
- drivers futuros;
- plataformas futuras.

sin ocultar las diferencias semánticas reales entre motores.

Regla central:

> **VoltStack modelará el aislamiento como una garantía transaccional solicitada y posteriormente resuelta contra las capacidades reales de la plataforma; nunca asumirá que dos motores ofrecen exactamente las mismas garantías únicamente porque utilizan el mismo nombre SQL para un isolation level.**

Por tanto:

```text
Isolation Name
≠
Isolation Semantics
≠
Platform Implementation
≠
Concurrency Guarantee
```

---

# 2. Objetivos

El sistema deberá:

1. proporcionar un modelo canónico de isolation levels;
2. permitir configuración global y por transacción;
3. resolver requested isolation;
4. determinar effective isolation;
5. validar capacidades de plataforma;
6. impedir downgrades silenciosos;
7. representar limitaciones;
8. integrarse con `TransactionManager`;
9. integrarse con `TransactionContext`;
10. aplicar isolation antes del inicio transaccional cuando la plataforma lo requiera;
11. exponer metadata para Query Engine y ORM;
12. colaborar con retry/deadlock/concurrency systems;
13. soportar read-only transactions;
14. permitir extensiones vendor-specific;
15. generar diagnostics;
16. producir telemetry;
17. ser seguro bajo runtimes persistentes.

---

# 3. No objetivos

Este sistema no será responsable directamente de:

- ejecutar queries de aplicación;
- implementar MVCC;
- adquirir locks físicos;
- resolver deadlocks;
- reintentar transacciones;
- implementar optimistic locking;
- implementar pessimistic locking;
- generar SQL de negocio;
- modificar entidades;
- administrar UnitOfWork;
- decidir automáticamente business consistency;
- implementar distributed transactions.

---

# 4. Isolation como contrato

Una transacción podrá declarar:

```php
DB::transaction(
    callback: function () {
        // ...
    },
    isolation: TransactionIsolation::SERIALIZABLE,
);
```

Esto significa:

> La aplicación solicita una transacción con una determinada semántica de aislamiento.

No significa:

> VoltStack puede fabricar esa garantía aunque la base de datos no la soporte.

---

# 5. Modelo general

```text
Application
    ↓
TransactionDefinition
    ↓
Requested Isolation
    ↓
Isolation Resolver
    ↓
Platform Capabilities
    ↓
Isolation Negotiation
    ↓
Effective Isolation
    ↓
Isolation Application Plan
    ↓
TransactionManager
    ↓
Connection
    ↓
Database
```

---

# 6. Principio requested vs effective

VoltStack distinguirá siempre:

```text
RequestedIsolation
```

de:

```text
EffectiveIsolation
```

Ejemplo:

```text
Requested:
    SERIALIZABLE

Platform:
    supports SERIALIZABLE

Effective:
    SERIALIZABLE
```

Otro caso:

```text
Requested:
    SNAPSHOT

Platform:
    no equivalent guarantee

Effective:
    unresolved
```

Resultado:

```text
UnsupportedTransactionIsolationException
```

por default.

---

# 7. No silent downgrade

Nunca:

```text
SERIALIZABLE requested
        ↓
unsupported
        ↓
silently use READ_COMMITTED
```

Esto estaría prohibido.

Regla:

> **Una garantía de aislamiento nunca podrá reducirse silenciosamente.**

---

# 8. Canonical TransactionIsolation

Modelo inicial:

```php
enum TransactionIsolation: string
{
    case READ_UNCOMMITTED = 'read_uncommitted';
    case READ_COMMITTED   = 'read_committed';
    case REPEATABLE_READ  = 'repeatable_read';
    case SERIALIZABLE     = 'serializable';
}
```

Estos representan niveles canónicos reconocidos por VoltStack.

---

# 9. Isolation extensions

Para semánticas que no encajan limpiamente en los cuatro niveles clásicos se utilizará un descriptor extensible.

Ejemplo:

```php
final readonly class TransactionIsolationDescriptor
{
    public function __construct(
        public TransactionIsolationId $id,
        public IsolationFamily $family,
        public IsolationGuaranteeSet $guarantees,
        public bool $portable,
    ) {}
}
```

Esto permite representar conceptos como:

```text
snapshot isolation
vendor-specific snapshot modes
platform-specific serializable semantics
```

sin contaminar el enum principal con nombres arbitrarios.

---

# 10. IsolationLevel ≠ IsolationGuarantees

Dos plataformas pueden anunciar:

```text
REPEATABLE READ
```

pero implementar diferentes comportamientos respecto a:

- snapshots;
- locking;
- phantom prevention;
- predicate locking;
- gap locking;
- serialization failures.

Por ello VoltStack deberá modelar:

```text
IsolationLevel
+
IsolationGuarantees
+
PlatformSemantics
```

---

# 11. IsolationGuaranteeSet

Modelo conceptual:

```php
final readonly class IsolationGuaranteeSet
{
    public function __construct(
        public DirtyReadGuarantee $dirtyReads,
        public NonRepeatableReadGuarantee $nonRepeatableReads,
        public PhantomReadGuarantee $phantomReads,
        public LostUpdateGuarantee $lostUpdates,
        public WriteSkewGuarantee $writeSkew,
        public SnapshotGuarantee $snapshot,
        public SerializationGuarantee $serialization,
    ) {}
}
```

---

# 12. Guarantee states

Cada propiedad no deberá reducirse necesariamente a `bool`.

Se recomienda:

```php
enum IsolationGuaranteeStatus
{
    case PREVENTED;
    case POSSIBLE;
    case PLATFORM_DEPENDENT;
    case WORKLOAD_DEPENDENT;
    case UNKNOWN;
}
```

---

# 13. UNKNOWN es first-class

Si VoltStack no puede determinar una garantía:

```text
UNKNOWN
```

no significa:

```text
PREVENTED
```

ni:

```text
POSSIBLE
```

---

# 14. Fenómenos de concurrencia

El sistema deberá reconocer al menos:

```text
Dirty Read
Non-Repeatable Read
Phantom Read
Lost Update
Write Skew
Serialization Conflict
```

como fenómenos diferentes.

---

# 15. Dirty Read

Conceptualmente:

```text
Transaction A:
    UPDATE balance = 0
    -- not committed

Transaction B:
    SELECT balance
    → sees 0

Transaction A:
    ROLLBACK
```

B observó información nunca confirmada.

---

# 16. Non-Repeatable Read

```text
Transaction A:
    SELECT balance → 100

Transaction B:
    UPDATE balance = 50
    COMMIT

Transaction A:
    SELECT balance → 50
```

La misma fila produce resultados distintos dentro de A.

---

# 17. Phantom Read

```text
Transaction A:
    SELECT * FROM orders WHERE status = 'pending'
    → 10 rows

Transaction B:
    INSERT matching order
    COMMIT

Transaction A:
    same predicate
    → 11 rows
```

---

# 18. Lost Update

Ejemplo conceptual:

```text
A reads version 10
B reads version 10

A writes 11
B writes 11
```

Una actualización puede sobrescribir otra.

Este fenómeno también podrá combatirse mediante:

```text
Optimistic Locking
```

que será tratado en el documento 172.

---

# 19. Write Skew

Dos transacciones pueden leer un estado compatible y realizar escrituras distintas que, combinadas, violen una invariante.

```text
A reads X,Y
B reads X,Y

A changes X
B changes Y

both commit
```

El resultado global puede ser inválido aunque no exista un write/write conflict directo.

---

# 20. Isolation does not replace domain invariants

Incluso `SERIALIZABLE` no elimina la necesidad de:

- constraints;
- unique constraints;
- foreign keys;
- optimistic locking;
- application invariants;
- idempotency;
- validation.

---

# 21. READ_UNCOMMITTED

Representará la solicitud del nivel más débil del modelo estándar soportado.

Conceptualmente puede permitir:

```text
dirty reads
non-repeatable reads
phantoms
```

pero VoltStack no deberá asumir que cada plataforma implementa literalmente dirty reads bajo ese nombre.

---

# 22. READ_COMMITTED

Principio general:

```text
each statement
→ sees committed data according to platform semantics
```

Puede permitir:

```text
non-repeatable reads
phantoms
write skew
```

dependiendo de plataforma y workload.

---

# 23. REPEATABLE_READ

Busca proporcionar una visión más estable durante la transacción.

Sin embargo:

```text
REPEATABLE_READ(MySQL)
≠
REPEATABLE_READ(PostgreSQL)
```

en todos los detalles observables.

VoltStack no intentará fingir equivalencia absoluta.

---

# 24. SERIALIZABLE

Representa la solicitud de la garantía transaccional estándar más fuerte del modelo base.

Conceptualmente:

```text
ConcurrentTransactions
≈
SomeValidSerialExecution
```

Pero su implementación puede utilizar:

```text
locking
predicate locking
MVCC
serialization detection
abort/retry
```

según plataforma.

---

# 25. Serializable does not mean no concurrency

`SERIALIZABLE` no significa necesariamente:

```text
one transaction at a time
```

Una plataforma puede ejecutar transacciones concurrentemente y abortar aquellas que no pueden formar una historia serializable.

---

# 26. Serialization failure

Por tanto:

```text
SERIALIZABLE
```

puede incrementar:

```text
SerializationFailureException
```

y la arquitectura deberá integrarse con:

```text
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
```

---

# 27. Snapshot isolation

VoltStack reconocerá conceptualmente:

```text
SNAPSHOT
```

como una familia semántica independiente.

No deberá asumirse:

```text
SNAPSHOT
=
REPEATABLE_READ
```

ni:

```text
SNAPSHOT
=
SERIALIZABLE
```

---

# 28. Snapshot semantics

Conceptualmente:

```text
Transaction
    ↓
consistent snapshot
    ↓
reads from snapshot
```

pero write conflict semantics dependen de la plataforma.

---

# 29. Snapshot descriptor

Podrá existir:

```php
final readonly class SnapshotIsolationDescriptor
    implements TransactionIsolationExtension
{
    // ...
}
```

si se necesita exponer explícitamente.

---

# 30. IsolationFamily

Posible modelo:

```php
enum IsolationFamily
{
    case STANDARD;
    case SNAPSHOT;
    case SERIALIZABLE_SNAPSHOT;
    case PLATFORM_SPECIFIC;
}
```

---

# 31. Platform isolation capabilities

Cada `DatabasePlatform` deberá exponer capacidades.

Ejemplo conceptual:

```php
interface TransactionIsolationCapabilities
{
    public function supports(
        TransactionIsolationDescriptor $isolation,
    ): bool;

    public function defaultIsolation(): TransactionIsolationDescriptor;

    public function guarantees(
        TransactionIsolationDescriptor $isolation,
    ): IsolationGuaranteeSet;

    public function applicationMode(
        TransactionIsolationDescriptor $isolation,
    ): IsolationApplicationMode;
}
```

---

# 32. Capability-driven architecture

Prohibido:

```php
if ($driver === 'pgsql') {
    // ...
}
```

Preferido:

```php
if ($platform->transactionIsolation()
             ->supports($requested)) {
    // ...
}
```

---

# 33. Version-aware capabilities

Capabilities podrán depender de:

```text
database family
database version
driver capabilities
connection mode
server configuration
```

Por tanto:

```text
VendorName
≠
Capability
```

---

# 34. MySQL y MariaDB

VoltStack mantendrá:

```text
MySQLPlatform
```

y:

```text
MariaDBPlatform
```

como plataformas independientes.

Nunca:

```text
MariaDB = MySQL alias
```

aunque compartan muchas características.

---

# 35. PostgreSQL

La plataforma PostgreSQL deberá describir sus semánticas efectivas mediante capability descriptors.

El core no codificará:

```text
if PostgreSQL then ...
```

en el Transaction Manager.

---

# 36. SQLite

SQLite deberá modelarse según sus capacidades reales de concurrencia y transaction modes.

No deberá fingirse que:

```text
SQLite SERIALIZABLE
```

es operacionalmente idéntico a una arquitectura MVCC cliente-servidor.

---

# 37. Platform semantic profile

Se recomienda:

```php
final readonly class TransactionIsolationProfile
{
    public function __construct(
        public TransactionIsolationDescriptor $isolation,
        public IsolationGuaranteeSet $guarantees,
        public IsolationApplicationMode $applicationMode,
        public IsolationConflictProfile $conflicts,
        public IsolationRestrictionSet $restrictions,
    ) {}
}
```

---

# 38. Isolation request

Una solicitud podrá expresarse mediante:

```php
final readonly class TransactionIsolationRequest
{
    public function __construct(
        public TransactionIsolationDescriptor $requested,
        public IsolationResolutionPolicy $policy,
    ) {}
}
```

---

# 39. Resolution policy

Políticas posibles:

```php
enum IsolationResolutionPolicy
{
    case EXACT;
    case AT_LEAST;
    case PLATFORM_DEFAULT;
}
```

---

# 40. EXACT

```text
requested isolation
must map to semantically accepted platform isolation
```

Si no:

```text
fail
```

---

# 41. AT_LEAST

Permite una garantía superior compatible.

Ejemplo conceptual:

```text
requested:
    READ_COMMITTED

effective:
    SERIALIZABLE
```

si la plataforma/configuración únicamente puede ofrecer una garantía superior y el perfil de compatibilidad lo considera válido.

---

# 42. Stronger is not always behaviorally equivalent

Importante:

> Un isolation level más fuerte puede cambiar bloqueo, abort rate, latencia y retry behavior.

Por ello `AT_LEAST` deberá ser una política explícita.

No será el default silencioso.

---

# 43. PLATFORM_DEFAULT

Permite:

```text
use database/platform configured default
```

pero deberá capturarse como:

```text
EffectiveIsolation
```

cuando sea determinable.

---

# 44. Default framework policy

Se recomienda:

```text
IsolationResolutionPolicy::EXACT
```

para requests explícitos.

Si el usuario no solicita isolation:

```text
PLATFORM_DEFAULT
```

podrá utilizarse.

---

# 45. Isolation Resolver

Componente:

```text
TransactionIsolationResolver
```

Responsabilidad:

```text
request
+
platform capabilities
+
framework policy
→
resolution
```

---

# 46. Resolution result

```php
final readonly class TransactionIsolationResolution
{
    public function __construct(
        public TransactionIsolationDescriptor $requested,
        public TransactionIsolationDescriptor $effective,
        public IsolationGuaranteeSet $guarantees,
        public IsolationResolutionKind $kind,
        public IsolationApplicationPlan $applicationPlan,
    ) {}
}
```

---

# 47. Resolution kinds

```php
enum IsolationResolutionKind
{
    case EXACT;
    case STRONGER_ACCEPTED;
    case PLATFORM_DEFAULT;
    case PLATFORM_EQUIVALENT;
}
```

---

# 48. Unsupported resolution

No deberá producir un objeto ambiguo.

Debe lanzar o devolver explícitamente:

```text
UnsupportedTransactionIsolation
```

antes de iniciar la transacción.

---

# 49. EffectiveTransactionDefinition integration

Documento 166 estableció:

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

Se refina a:

```php
final readonly class EffectiveTransactionDefinition
{
    public function __construct(
        public TransactionIsolationResolution $isolation,
        public TransactionReadMode $readMode,
        public TransactionPropagation $propagation,
        public ?TransactionTimeout $timeout,
        public NestedTransactionStrategy $nestedStrategy,
    ) {}
}
```

---

# 50. TransactionReadMode

Se recomienda distinguir:

```php
enum TransactionReadMode
{
    case READ_WRITE;
    case READ_ONLY;
}
```

---

# 51. Read-only transaction

Ejemplo:

```php
DB::transaction(
    callback: fn () => $reports->generate(),
    isolation: TransactionIsolation::REPEATABLE_READ,
    readOnly: true,
);
```

---

# 52. Read-only ≠ SELECT-only parser rule

`READ_ONLY` es una propiedad transaccional.

No deberá implementarse exclusivamente mediante:

```text
if query starts with SELECT
```

porque:

- CTEs pueden escribir;
- funciones pueden tener efectos;
- vendor syntax varía;
- DDL/DML puede ocultarse;
- stored procedures pueden mutar.

---

# 53. Layered read-only enforcement

VoltStack podrá usar:

```text
Semantic Query Guard
+
Platform Transaction Mode
+
Database Enforcement
```

cuando estén disponibles.

---

# 54. Isolation application plan

Algunas plataformas requieren que el isolation level se configure:

```text
before BEGIN
```

otras permiten sintaxis asociada al comienzo de la transacción.

Por ello:

```php
final readonly class IsolationApplicationPlan
{
    public function __construct(
        public IsolationApplicationMode $mode,
        public array $commands,
    ) {}
}
```

---

# 55. Application modes

```php
enum IsolationApplicationMode
{
    case CONNECTION_BEFORE_BEGIN;
    case TRANSACTION_BEGIN;
    case TRANSACTION_AFTER_BEGIN;
    case SESSION_PRECONFIGURED;
    case DRIVER_NATIVE;
}
```

---

# 56. Ordering is platform capability

No se deberá asumir:

```text
SET ISOLATION
BEGIN
```

universalmente.

La plataforma determinará el orden.

---

# 57. Transaction start pipeline

```text
Transaction Request
        ↓
Resolve Connection
        ↓
Resolve Platform
        ↓
Resolve Isolation
        ↓
Validate Combination
        ↓
Create Effective Definition
        ↓
Create Transaction Context
        ↓
Apply Pre-Begin Isolation Configuration
        ↓
BEGIN
        ↓
Apply Allowed Post-Begin Configuration
        ↓
Verify Start
        ↓
ACTIVE
```

---

# 58. No ACTIVE before isolation setup

Una transacción no deberá marcarse:

```text
ACTIVE
```

hasta que su configuración transaccional necesaria haya sido aplicada satisfactoriamente.

---

# 59. Configuration failure

Si isolation setup falla antes de `BEGIN`:

```text
TransactionState
→ FAILED
```

sin asumir que existe una transacción física.

---

# 60. Failure around BEGIN

Si el driver no puede determinar si `BEGIN` ocurrió:

```text
UNKNOWN
```

podrá ser necesario.

---

# 61. Session-level contamination

Algunas configuraciones pueden modificar estado de sesión/conexión.

Esto es crítico con pooling.

```text
Connection #7
Request A
    changes isolation/session state

Connection #7
Request B
    inherits state
```

debe evitarse.

---

# 62. Connection reset integration

Cualquier configuración de isolation que pueda persistir más allá de la transacción deberá integrarse con:

```text
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md
```

---

# 63. Session mutation tracking

El `ConnectionState` deberá poder registrar:

```text
isolation modified
read mode modified
transaction mode modified
```

para restauración/reset.

---

# 64. No state leakage

Invariante:

```text
Transaction A configuration
```

nunca deberá contaminar:

```text
Transaction B
```

cuando una conexión se reutilice.

---

# 65. Persistent runtime importance

Esto será obligatorio para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

donde conexiones y servicios pueden vivir durante múltiples operaciones.

---

# 66. Transaction Context integration

El contexto almacenará:

```text
requested isolation
effective isolation
guarantee profile
```

como metadata inmutable después del inicio.

---

# 67. Context example

```text
Transaction:
    tx-120

Requested Isolation:
    SERIALIZABLE

Effective Isolation:
    SERIALIZABLE

Resolution:
    EXACT

Read Mode:
    READ_WRITE

Platform:
    PostgreSQL

State:
    ACTIVE
```

---

# 68. Nested transactions

Una transacción nested que participa en la transacción existente no podrá cambiar arbitrariamente el isolation level.

Ejemplo:

```text
Outer:
    READ_COMMITTED

Inner REQUIRED:
    SERIALIZABLE
```

No puede simplemente transformar la transacción ya activa.

---

# 69. Nested isolation conflict

Por default:

```text
Inner requested isolation
incompatible with outer effective isolation
→
NestedTransactionIsolationConflictException
```

---

# 70. Compatible nested request

Ejemplo:

```text
Outer:
    SERIALIZABLE

Inner:
    READ_COMMITTED
    policy = AT_LEAST
```

podría participar porque el outer satisface una garantía al menos equivalente, si las reglas semánticas lo permiten.

---

# 71. Nested exact request

Si inner solicita:

```text
EXACT READ_COMMITTED
```

y outer es:

```text
SERIALIZABLE
```

no deberá asumirse compatible.

---

# 72. Savepoints do not create new isolation

```text
SAVEPOINT
```

no crea un isolation boundary independiente.

Por tanto:

```text
SavepointIsolation
=
ParentTransactionIsolation
```

---

# 73. REQUIRES_NEW

Una futura transacción:

```text
REQUIRES_NEW
```

sí podrá resolver un isolation level distinto porque utiliza una nueva transacción física.

---

# 74. Isolation and retry

Isolation puede afectar la probabilidad de:

```text
deadlocks
serialization failures
lock waits
```

Por tanto deberá integrarse con:

```text
TransactionRetrySystem
```

sin que Isolation System ejecute retries directamente.

---

# 75. Retry classification

Los errores podrán clasificarse como:

```text
SERIALIZATION_FAILURE
DEADLOCK
LOCK_TIMEOUT
TRANSIENT_CONCURRENCY_FAILURE
NON_RETRYABLE_TRANSACTION_FAILURE
```

---

# 76. Serializable retry

En determinadas plataformas/workloads:

```text
SERIALIZABLE
→ serialization failure
→ entire transaction retry candidate
```

pero solo si:

```text
TransactionRetryPolicy
```

lo permite.

---

# 77. Isolation System does not retry

Regla:

```text
Isolation System
→ describes conflict semantics

Retry System
→ decides retry
```

---

# 78. Deadlock System integration

Documento 171 podrá consumir:

```text
EffectiveIsolation
IsolationConflictProfile
```

para diagnostics y políticas.

---

# 79. Optimistic locking integration

Optimistic locking será independiente del isolation level.

```text
Isolation
+
Version Check
```

pueden utilizarse conjuntamente.

---

# 80. Pessimistic locking integration

Queries como:

```text
FOR UPDATE
```

no son isolation levels.

Por tanto:

```text
Transaction Isolation
≠
Row Lock Mode
```

---

# 81. Lock compatibility

El sistema futuro de pessimistic locking deberá validar:

```text
lock mode
+
effective isolation
+
platform capabilities
```

---

# 82. Query Engine integration

El Query Context podrá contener:

```text
TransactionIsolationContext
```

con información read-only.

---

# 83. Query semantic behavior

El Query Engine podrá utilizar isolation metadata para:

- validar lock clauses;
- seleccionar capacidades;
- generar diagnostics;
- impedir operaciones incompatibles.

Pero no deberá reescribir arbitrariamente queries para simular isolation.

---

# 84. Query Planner

El planner no deberá asumir que puede debilitar isolation para mejorar rendimiento.

Prohibido:

```text
SERIALIZABLE requested
→ planner decides READ_COMMITTED is faster
```

---

# 85. ORM integration

El ORM deberá conocer el isolation únicamente cuando sea necesario para:

- locking;
- consistency diagnostics;
- retry coordination;
- persistence conflict classification.

No deberá implementar isolation por sí mismo.

---

# 86. UnitOfWork

```text
UnitOfWork
≠
Transaction Isolation
```

El UoW representa cambios de entidades.

La base de datos implementa el isolation físico.

---

# 87. IdentityMap

```text
IdentityMap
```

tampoco constituye una garantía de isolation.

Puede mantener una instancia ya cargada aunque la base de datos tenga una realidad distinta.

---

# 88. Database reality vs ORM knowledge

Regla previa:

```text
DatabaseReality
≠
ORMKnowledge
```

continúa siendo válida bajo cualquier isolation level.

---

# 89. Refresh semantics

Un:

```php
$entityManager->refresh($entity);
```

puede producir resultados diferentes según:

```text
effective isolation
snapshot semantics
locking
```

por lo que diagnostics podrán incluir el isolation activo.

---

# 90. Read/write routing

Una transacción activa normalmente deberá mantener connection affinity.

```text
Transaction
→ pinned connection
```

No deberá enviar cada SELECT a replicas diferentes.

---

# 91. Replica restrictions

Una transacción que requiere garantías específicas podrá ser incompatible con una replica.

Ejemplo conceptual:

```text
read-only transaction
+
replica
```

podría ser permitido bajo ciertas políticas.

Pero:

```text
read-write transaction
→ replica
```

normalmente será rechazado.

---

# 92. Replica consistency

`READ_ONLY` no significa automáticamente:

```text
safe to route to any replica
```

porque puede existir:

```text
replica lag
```

y el usuario puede requerir read-your-writes.

---

# 93. Routing input

El futuro `ReadWriteRoutingSystem` deberá considerar:

```text
TransactionReadMode
EffectiveIsolation
ConsistencyRequirement
ReplicaLag
StickyConnectionState
```

---

# 94. Isolation vs read consistency

Distinguir:

```text
Transaction Isolation
```

de:

```text
Distributed Read Consistency
```

Una replica atrasada introduce un problema diferente.

---

# 95. Autocommit

Queries fuera de una transacción explícita pueden ejecutarse bajo semánticas de autocommit.

Esto no deberá confundirse con:

```text
VoltStack managed transaction context
```

---

# 96. Autocommit isolation

El motor puede aplicar un isolation default a cada statement/autocommit transaction.

VoltStack podrá observarlo para diagnostics, pero no creará necesariamente un `TransactionContext` completo por cada statement.

---

# 97. Explicit transaction boundary

Las garantías configuradas mediante:

```php
DB::transaction(...)
```

se aplican al boundary gestionado por VoltStack.

---

# 98. Manual transaction API

También:

```php
$tx = DB::beginTransaction(
    isolation: TransactionIsolation::SERIALIZABLE,
);

try {
    // ...
    $tx->commit();
} catch (\Throwable $e) {
    $tx->rollback();
    throw $e;
}
```

utilizará exactamente el mismo Isolation System.

---

# 99. One engine

No habrá:

```text
ClosureTransactionIsolationEngine
```

y otro:

```text
ManualTransactionIsolationEngine
```

Ambas APIs convergerán en:

```text
TransactionManager
+
IsolationResolver
+
TransactionContext
```

---

# 100. Configuration system

Ejemplo conceptual:

```php
'database' => [
    'transactions' => [
        'isolation' => [
            'default' => 'platform_default',
            'resolution' => 'exact',
            'verify_effective_level' => false,
        ],
    ],
],
```

---

# 101. Connection-specific configuration

Podrá existir:

```php
'connections' => [
    'analytics' => [
        'transaction' => [
            'default_isolation' => 'repeatable_read',
            'read_only' => true,
        ],
    ],
],
```

---

# 102. Precedence

Ejemplo:

```text
Explicit Transaction Request
        ↓
Connection Transaction Policy
        ↓
Application Database Policy
        ↓
Platform Default
```

---

# 103. Explicit wins

Una solicitud explícita deberá prevalecer sobre defaults, siempre que sea soportada.

---

# 104. Policy restrictions

Una aplicación podrá declarar:

```text
minimum isolation = READ_COMMITTED
```

o prohibir:

```text
READ_UNCOMMITTED
```

para determinadas conexiones.

---

# 105. Isolation policy

```php
interface TransactionIsolationPolicy
{
    public function validate(
        TransactionIsolationRequest $request,
        TransactionIsolationProfile $profile,
    ): IsolationPolicyDecision;
}
```

---

# 106. Policy use cases

Ejemplos:

```text
production forbids READ_UNCOMMITTED
financial connection requires SERIALIZABLE
analytics permits REPEATABLE_READ
```

---

# 107. Policy ≠ capability

```text
Platform Capability:
    can database do it?

Application Policy:
    may application use it?
```

Son preguntas distintas.

---

# 108. Effective isolation verification

En algunos escenarios podrá existir:

```text
verify_effective_level = true
```

para comprobar el nivel realmente activo cuando la plataforma lo permita.

---

# 109. Verification cost

No deberá ejecutarse una query de verificación por cada transacción por default.

Esto podría introducir:

```text
latency
extra round trips
load
```

---

# 110. Verification strategies

```php
enum IsolationVerificationMode
{
    case NONE;
    case STARTUP;
    case CONNECTION_ACQUIRE;
    case TRANSACTION_START;
    case DEBUG_ONLY;
}
```

---

# 111. Capability confidence

La plataforma podrá declarar:

```text
STATIC_KNOWN
CONFIGURATION_DEPENDENT
RUNTIME_VERIFIABLE
UNKNOWN
```

respecto a una propiedad.

---

# 112. Isolation confidence

```php
enum IsolationConfidence
{
    case VERIFIED;
    case PLATFORM_DECLARED;
    case CONFIGURATION_DERIVED;
    case ASSUMED;
    case UNKNOWN;
}
```

---

# 113. No false certainty

Diagnostics deberán diferenciar:

```text
effective: SERIALIZABLE
confidence: VERIFIED
```

de:

```text
effective: SERIALIZABLE
confidence: PLATFORM_DECLARED
```

---

# 114. Platform default ambiguity

Si el framework usa:

```text
PLATFORM_DEFAULT
```

pero no puede determinar cuál es:

```text
effective isolation = PLATFORM_DEFAULT
confidence = UNKNOWN
```

podrá conservarse como representación explícita.

---

# 115. Default is not universal

Nunca codificar:

```text
default isolation = READ_COMMITTED
```

como regla universal del framework.

---

# 116. Isolation and DDL

Algunas plataformas tienen comportamientos especiales para DDL dentro de transacciones.

Por ello:

```text
Transaction Isolation
≠
DDL Transactionality
```

---

# 117. DDL capabilities

Schema/Migration systems deberán consultar capacidades independientes:

```text
supportsTransactionalDDL()
```

No inferirlas del isolation level.

---

# 118. Isolation and sequences

Generadores como:

```text
sequences
auto increment
identity
```

pueden tener semánticas fuera del rollback/isolation esperado.

Por tanto:

```text
Transaction rollback
```

no implica necesariamente:

```text
identifier generator state rewind
```

---

# 119. Isolation and external side effects

Una transacción DB no aísla:

```text
HTTP requests
emails
filesystem
external queues
third-party APIs
```

---

# 120. Side effect warning

```text
SERIALIZABLE database transaction
```

no hace serializable una operación distribuida completa.

---

# 121. Transactional events

Eventos externos deberán utilizar mecanismos como:

```text
afterCommit
outbox
```

cuando sea necesario.

---

# 122. Isolation metadata fingerprint

Para caches/diagnostics podrá existir:

```text
IsolationFingerprint
```

calculado a partir de:

```text
IsolationTypeId
PlatformProfileGeneration
GuaranteeProfile
ReadMode
ResolutionPolicy
```

---

# 123. Isolation fingerprint ≠ transaction identity

Dos transacciones pueden compartir el mismo fingerprint pero ser transacciones distintas.

---

# 124. Caching

Podrán cachearse:

```text
platform isolation profiles
compiled application plans
capability resolution
```

No:

```text
active transaction isolation state
```

globalmente.

---

# 125. Persistent-runtime cache

Caches de profiles deberán ser:

```text
immutable
versioned
safe to share
```

---

# 126. Runtime mutable state

Deberá permanecer en:

```text
TransactionContext
ConnectionState
RuntimeScope
```

---

# 127. Telemetry

Se podrán registrar métricas:

```text
db.transaction.isolation.requested
db.transaction.isolation.effective
db.transaction.isolation.resolution
db.transaction.isolation.unsupported
db.transaction.serialization_failure
```

---

# 128. Cardinality control

No utilizar como labels de alta cardinalidad:

```text
TransactionId
UserId
raw SQL
tenant UUID
```

en métricas agregadas.

---

# 129. Trace attributes

Spans podrán incluir:

```text
db.transaction.isolation.requested
db.transaction.isolation.effective
db.transaction.read_only
db.transaction.attempt
db.transaction.outcome
```

con cardinalidad controlada.

---

# 130. Diagnostics

Ejemplo:

```text
TRANSACTION ISOLATION

Requested:
    SERIALIZABLE

Resolution Policy:
    EXACT

Effective:
    SERIALIZABLE

Confidence:
    PLATFORM_DECLARED

Read Mode:
    READ_WRITE

Platform:
    PostgreSQL

Guarantees:
    Dirty Reads: PREVENTED
    Non-Repeatable Reads: PREVENTED
    Phantom Reads: PREVENTED
    Serialization: PLATFORM_DEFINED

Application Mode:
    TRANSACTION_BEGIN
```

---

# 131. Unsupported diagnostic

```text
TRANSACTION ISOLATION RESOLUTION FAILED

Requested:
    custom_snapshot

Policy:
    EXACT

Platform:
    sqlite

Reason:
    requested isolation semantics cannot be represented

Fallback:
    none

Transaction Started:
    no
```

---

# 132. Nested conflict diagnostic

```text
NESTED TRANSACTION ISOLATION CONFLICT

Outer:
    SERIALIZABLE

Inner Requested:
    READ_COMMITTED

Inner Policy:
    EXACT

Strategy:
    JOIN_EXISTING

Resolution:
    incompatible

Physical Transaction:
    unchanged
```

---

# 133. Security

Isolation configuration deberá aceptar únicamente:

```text
registered typed descriptors
```

No:

```php
DB::transaction(
    isolation: $_GET['sql'],
);
```

como SQL arbitrario.

---

# 134. No raw isolation SQL

El core no deberá permitir:

```php
isolation("SET TRANSACTION ... arbitrary SQL");
```

---

# 135. Custom extensions

Custom isolation extensions deberán registrarse durante bootstrap.

```php
$isolationRegistry->register(
    new CustomIsolationDefinition(...)
);
```

---

# 136. Registry freeze

Lifecycle:

```text
BOOTSTRAP
    ↓
REGISTERING
    ↓
VALIDATING
    ↓
COMPILED
    ↓
FROZEN
```

---

# 137. No runtime dynamic isolation registration

Una request no deberá registrar nuevos isolation types.

Esto protege:

- determinismo;
- worker safety;
- security;
- cache correctness.

---

# 138. Error hierarchy

```text
DatabaseTransactionIsolationException
│
├── UnknownTransactionIsolationException
├── UnsupportedTransactionIsolationException
├── TransactionIsolationResolutionException
├── TransactionIsolationPolicyException
├── TransactionIsolationApplicationException
├── TransactionIsolationVerificationException
├── TransactionIsolationCapabilityException
├── TransactionIsolationConfigurationException
├── NestedTransactionIsolationConflictException
├── TransactionIsolationStateException
└── TransactionIsolationInvariantViolationException
```

---

# 139. Unknown isolation

```text
UnknownTransactionIsolationException
```

significa:

> VoltStack no reconoce la definición solicitada.

---

# 140. Unsupported isolation

```text
UnsupportedTransactionIsolationException
```

significa:

> La definición existe, pero la plataforma actual no puede satisfacerla.

---

# 141. Policy exception

```text
TransactionIsolationPolicyException
```

significa:

> La plataforma puede hacerlo, pero la política de aplicación lo prohíbe.

---

# 142. Application exception

```text
TransactionIsolationApplicationException
```

significa:

> La resolución fue válida, pero falló su aplicación sobre la conexión/transacción.

---

# 143. Verification exception

Significa:

> VoltStack intentó verificar la configuración efectiva y no pudo hacerlo de manera válida.

Esto no deberá convertirse automáticamente en:

```text
configuration definitely wrong
```

si el resultado es simplemente desconocido.

---

# 144. Testing architecture

El sistema requerirá:

```text
IsolationUnitTests
IsolationResolutionTests
IsolationCapabilityTests
IsolationPolicyTests
IsolationApplicationTests
IsolationIntegrationTests
IsolationConcurrencyTests
IsolationNestedTransactionTests
IsolationRetryTests
IsolationRuntimeIsolationTests
IsolationPlatformConformanceTests
```

---

# 145. Resolver tests

Casos:

```text
exact supported
exact unsupported
at-least supported
platform default
unknown isolation
policy denied
```

---

# 146. No downgrade test

```text
requested SERIALIZABLE
platform lacks equivalent
```

Esperado:

```text
exception
```

Nunca:

```text
READ_COMMITTED
```

---

# 147. Nested tests

```text
outer READ_COMMITTED
inner SERIALIZABLE EXACT
JOIN_EXISTING
```

deberá fallar.

---

# 148. Savepoint test

```text
outer REPEATABLE_READ
nested SAVEPOINT
```

deberá mantener:

```text
REPEATABLE_READ
```

---

# 149. Read-only tests

Verificar:

- compatible SELECT;
- mutative Query Model rejection cuando sea detectable;
- platform read-only mode;
- state reset;
- no leakage to next transaction.

---

# 150. Session contamination test

```text
Transaction A:
    custom isolation

release connection

Transaction B:
    platform default
```

B no deberá heredar accidentalmente la configuración de A.

---

# 151. Worker test

```text
Request A:
    SERIALIZABLE

Request B:
    READ_COMMITTED
```

en el mismo worker.

No deberá existir contaminación contextual.

---

# 152. Coroutine test

```text
Coroutine A:
    tx SERIALIZABLE

Coroutine B:
    tx READ_COMMITTED
```

deberán conservar contextos independientes.

---

# 153. Conformance testing

Cada plataforma deberá probar al menos:

```text
supported isolation levels
application ordering
read-only mode
state restoration
failure classification
serialization/deadlock mapping
nested compatibility
```

---

# 154. Behavioral tests

Cuando sea razonable se ejecutarán pruebas concurrentes reales para observar:

```text
dirty read behavior
repeatable reads
phantoms
serialization conflicts
```

No depender únicamente del nombre reportado por la plataforma.

---

# 155. Test matrix

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cada uno deberá tener su suite de conformance.

---

# 156. Performance requirements

Resolution deberá ser:

```text
O(1)
```

o cercano a constante mediante profiles compilados.

---

# 157. No hot-path introspection

Evitar:

```text
query database capabilities
```

por cada:

```text
DB::transaction()
```

---

# 158. Capability compilation

Preferir:

```text
connection/platform bootstrap
        ↓
capability discovery
        ↓
compiled isolation profile
        ↓
reused resolution
```

---

# 159. Runtime verification exception

Si una capability depende de estado dinámico del servidor, podrá verificarse según policy.

Pero será explícito.

---

# 160. Proposed directory structure

```text
src/Quantum/Database/Transaction/
│
├── Isolation/
│   ├── TransactionIsolation.php
│   ├── TransactionIsolationId.php
│   ├── TransactionIsolationDescriptor.php
│   ├── IsolationFamily.php
│   ├── IsolationGuaranteeSet.php
│   ├── IsolationGuaranteeStatus.php
│   ├── TransactionIsolationProfile.php
│   └── IsolationConfidence.php
│
├── Isolation/Request/
│   ├── TransactionIsolationRequest.php
│   ├── IsolationResolutionPolicy.php
│   └── TransactionReadMode.php
│
├── Isolation/Resolution/
│   ├── TransactionIsolationResolver.php
│   ├── TransactionIsolationResolution.php
│   ├── IsolationResolutionKind.php
│   └── IsolationApplicationPlan.php
│
├── Isolation/Capability/
│   ├── TransactionIsolationCapabilities.php
│   ├── IsolationApplicationMode.php
│   ├── IsolationConflictProfile.php
│   └── IsolationRestrictionSet.php
│
├── Isolation/Policy/
│   ├── TransactionIsolationPolicy.php
│   ├── IsolationPolicyDecision.php
│   └── DefaultTransactionIsolationPolicy.php
│
├── Isolation/Registry/
│   ├── TransactionIsolationRegistry.php
│   └── CompiledTransactionIsolationRegistry.php
│
├── Isolation/Verification/
│   ├── TransactionIsolationVerifier.php
│   └── IsolationVerificationMode.php
│
├── Isolation/Diagnostics/
│   ├── TransactionIsolationInspector.php
│   └── TransactionIsolationDiagnostic.php
│
└── Isolation/Exception/
    └── ...
```

---

# 161. Dependency model

Permitido:

```text
Transaction Isolation
    ↓
Platform Capabilities
Transaction Definitions
Connection State Contracts
Transaction Context Contracts
Diagnostics Contracts
```

No permitido:

```text
Transaction Isolation
    ↓
ORM Entity
Repository
UnitOfWork
Query execution implementation
HTTP request
Concrete runtime globals
```

---

# 162. Platform integration

```text
Platform
│
├── SQL capabilities
├── Schema capabilities
├── Type capabilities
└── Transaction capabilities
    └── Isolation capabilities
```

Esto mantiene las diferencias vendor-specific en la capa correcta.

---

# 163. Architectural invariants

## DB-TXI-001
Isolation será una propiedad explícita de la definición transaccional.

## DB-TXI-002
Requested isolation y effective isolation serán conceptos distintos.

## DB-TXI-003
VoltStack no realizará silent isolation downgrade.

## DB-TXI-004
Un isolation desconocido será rechazado.

## DB-TXI-005
Un isolation conocido pero no soportado será rechazado por default.

## DB-TXI-006
Platform default no será codificado universalmente.

## DB-TXI-007
Isolation name no equivaldrá automáticamente a isolation semantics.

## DB-TXI-008
Isolation semantics serán capability-driven.

## DB-TXI-009
Vendor name no sustituirá capability detection.

## DB-TXI-010
MySQL y MariaDB serán plataformas independientes.

## DB-TXI-011
PostgreSQL tendrá su propio isolation profile.

## DB-TXI-012
SQLite tendrá su propio isolation profile.

## DB-TXI-013
Core Transaction Manager no contendrá vendor conditionals.

## DB-TXI-014
Platform layer determinará application strategy.

## DB-TXI-015
Application ordering será platform-aware.

## DB-TXI-016
Isolation deberá resolverse antes del inicio transaccional.

## DB-TXI-017
Effective definition deberá existir antes de ACTIVE.

## DB-TXI-018
Isolation setup failure no producirá ACTIVE.

## DB-TXI-019
Ambiguous begin outcome podrá producir UNKNOWN.

## DB-TXI-020
Session-level isolation mutations serán rastreadas.

## DB-TXI-021
Connection reset eliminará state leakage.

## DB-TXI-022
Una transacción no contaminará la siguiente.

## DB-TXI-023
Persistent workers no compartirán mutable isolation state.

## DB-TXI-024
TransactionContext conservará effective isolation.

## DB-TXI-025
Effective isolation será inmutable durante una transacción activa.

## DB-TXI-026
Nested joined transaction no cambiará isolation físico.

## DB-TXI-027
Nested isolation conflicts serán explícitos.

## DB-TXI-028
Savepoints heredarán isolation de la transacción física.

## DB-TXI-029
Savepoint no será un isolation boundary.

## DB-TXI-030
REQUIRES_NEW podrá resolver un isolation distinto.

## DB-TXI-031
READ_UNCOMMITTED será una abstracción canónica, no una promesa de comportamiento vendor-independent.

## DB-TXI-032
READ_COMMITTED será capability-described.

## DB-TXI-033
REPEATABLE_READ será capability-described.

## DB-TXI-034
SERIALIZABLE será capability-described.

## DB-TXI-035
SNAPSHOT no será tratado automáticamente como REPEATABLE_READ.

## DB-TXI-036
SNAPSHOT no será tratado automáticamente como SERIALIZABLE.

## DB-TXI-037
Dirty read será un fenómeno modelado.

## DB-TXI-038
Non-repeatable read será un fenómeno modelado.

## DB-TXI-039
Phantom read será un fenómeno modelado.

## DB-TXI-040
Lost update será un fenómeno modelado.

## DB-TXI-041
Write skew será un fenómeno modelado.

## DB-TXI-042
Serialization failure será un fenómeno modelado.

## DB-TXI-043
Guarantee UNKNOWN permanecerá UNKNOWN.

## DB-TXI-044
UNKNOWN no significará PREVENTED.

## DB-TXI-045
UNKNOWN no significará POSSIBLE.

## DB-TXI-046
Isolation no sustituirá constraints.

## DB-TXI-047
Isolation no sustituirá optimistic locking.

## DB-TXI-048
Isolation no sustituirá pessimistic locking.

## DB-TXI-049
Isolation no sustituirá domain invariants.

## DB-TXI-050
Isolation no sustituirá validation.

## DB-TXI-051
Isolation no implementará retries.

## DB-TXI-052
Retry System consumirá isolation metadata.

## DB-TXI-053
Deadlock System podrá consumir isolation metadata.

## DB-TXI-054
Serialization failures podrán ser retry candidates.

## DB-TXI-055
Retry requerirá política explícita.

## DB-TXI-056
Transaction Isolation no equivaldrá a row locking.

## DB-TXI-057
FOR UPDATE no será un isolation level.

## DB-TXI-058
Query Planner no podrá debilitar isolation.

## DB-TXI-059
Query Optimizer no podrá debilitar isolation.

## DB-TXI-060
ORM no implementará isolation físico.

## DB-TXI-061
UnitOfWork no equivaldrá a isolation.

## DB-TXI-062
IdentityMap no equivaldrá a isolation.

## DB-TXI-063
DatabaseReality seguirá siendo distinta de ORMKnowledge.

## DB-TXI-064
Refresh semantics podrán depender del isolation efectivo.

## DB-TXI-065
Active transaction mantendrá connection affinity.

## DB-TXI-066
SELECT transaccional no será enviado arbitrariamente a replicas.

## DB-TXI-067
Read-only no significará cualquier replica.

## DB-TXI-068
Replica lag será un problema distinto de transaction isolation.

## DB-TXI-069
Distributed read consistency será distinta de transaction isolation.

## DB-TXI-070
Autocommit será distinto de managed transaction context.

## DB-TXI-071
Closure API y manual API usarán el mismo isolation engine.

## DB-TXI-072
No existirán dos motores de isolation.

## DB-TXI-073
Explicit transaction configuration prevalecerá sobre defaults compatibles.

## DB-TXI-074
Application policy será distinta de platform capability.

## DB-TXI-075
Platform capability responderá si puede hacerse.

## DB-TXI-076
Application policy responderá si debe permitirse.

## DB-TXI-077
EXACT será default para solicitudes explícitas.

## DB-TXI-078
AT_LEAST requerirá solicitud/política explícita.

## DB-TXI-079
Stronger isolation no será asumido behavioralmente equivalente.

## DB-TXI-080
PLATFORM_DEFAULT podrá usarse cuando no exista request explícito.

## DB-TXI-081
Platform default desconocido permanecerá explícitamente desconocido.

## DB-TXI-082
Isolation verification será configurable.

## DB-TXI-083
Runtime verification no será obligatoria por transacción por default.

## DB-TXI-084
Capability profiles podrán cachearse.

## DB-TXI-085
Capability caches serán inmutables/versionados.

## DB-TXI-086
Active transaction state no se almacenará en global cache.

## DB-TXI-087
Isolation registry será frozen después de bootstrap.

## DB-TXI-088
Requests no registrarán nuevos isolation types.

## DB-TXI-089
Custom isolation definitions serán typed.

## DB-TXI-090
Raw isolation SQL no será parte del API público base.

## DB-TXI-091
User input no se convertirá en isolation SQL.

## DB-TXI-092
DDL transactionality será distinta de isolation.

## DB-TXI-093
Identifier generator rollback semantics serán distintas de isolation.

## DB-TXI-094
External side effects no estarán aislados por DB transaction.

## DB-TXI-095
SERIALIZABLE no implicará atomicidad distribuida.

## DB-TXI-096
Outbox podrá utilizarse para side effects coordinados.

## DB-TXI-097
Read-only será una propiedad transaccional.

## DB-TXI-098
Read-only no se implementará únicamente mediante string parsing.

## DB-TXI-099
Semantic read-only checks serán defense-in-depth.

## DB-TXI-100
Database read-only enforcement se utilizará cuando exista.

## DB-TXI-101
Read-only state será reseteado al reutilizar conexión.

## DB-TXI-102
Isolation metadata podrá exponerse al Query Context.

## DB-TXI-103
Isolation metadata será read-only para Query Engine.

## DB-TXI-104
Query Engine no cambiará isolation arbitrariamente.

## DB-TXI-105
Query Compiler no decidirá isolation.

## DB-TXI-106
Driver no decidirá application-level isolation policy.

## DB-TXI-107
Platform traducirá semantic request a mecanismo físico.

## DB-TXI-108
Compiler/Platform commands no ejecutarán por sí mismos.

## DB-TXI-109
Connection/Driver ejecutarán la configuración física.

## DB-TXI-110
TransactionContext almacenará el resultado de resolución.

## DB-TXI-111
Isolation resolution será determinista para el mismo profile/policy.

## DB-TXI-112
Resolution failures serán explícitos.

## DB-TXI-113
Policy failures serán distinguibles de capability failures.

## DB-TXI-114
Application failures serán distinguibles de resolution failures.

## DB-TXI-115
Verification failures serán distinguibles de unsupported isolation.

## DB-TXI-116
Telemetry no expondrá credenciales.

## DB-TXI-117
Telemetry no expondrá raw parameter values.

## DB-TXI-118
Metrics evitarán cardinalidad por TransactionId.

## DB-TXI-119
Diagnostics mostrarán requested y effective isolation.

## DB-TXI-120
Diagnostics podrán mostrar confidence.

## DB-TXI-121
Diagnostics podrán mostrar guarantee profile.

## DB-TXI-122
Diagnostics podrán mostrar application mode.

## DB-TXI-123
Isolation fingerprint no será transaction identity.

## DB-TXI-124
Isolation profiles serán worker-safe.

## DB-TXI-125
Mutable transaction isolation state será scope-local.

## DB-TXI-126
FrankenPHP requests estarán aislados.

## DB-TXI-127
RoadRunner operations estarán aisladas.

## DB-TXI-128
OpenSwoole coroutines estarán aisladas.

## DB-TXI-129
Isolation configuration no utilizará static mutable state.

## DB-TXI-130
Isolation configuration no utilizará global mutable state.

## DB-TXI-131
Isolation resolver será independiente del ORM.

## DB-TXI-132
Isolation resolver será testeable sin EntityManager.

## DB-TXI-133
Platform conformance será probada independientemente.

## DB-TXI-134
Behavioral concurrency tests complementarán metadata tests.

## DB-TXI-135
Platform declaration no sustituirá necesariamente behavioral verification.

## DB-TXI-136
No se prometerán garantías que VoltStack no pueda justificar.

## DB-TXI-137
Transaction isolation no resolverá distributed transactions.

## DB-TXI-138
Transaction isolation no resolverá replica lag.

## DB-TXI-139
Transaction isolation no resolverá cache consistency.

## DB-TXI-140
Transaction isolation no resolverá external system consistency.

## DB-TXI-141
Connection affinity se mantendrá hasta completion.

## DB-TXI-142
Isolation resolution precederá connection transaction activation.

## DB-TXI-143
Context ACTIVE implicará effective isolation ya resuelto.

## DB-TXI-144
Context terminal conservará isolation para diagnostics.

## DB-TXI-145
Retry attempt nuevo resolverá nuevamente según política vigente.

## DB-TXI-146
Retry no mutará el contexto terminal anterior.

## DB-TXI-147
Nested transaction resolution utilizará effective outer isolation.

## DB-TXI-148
Nested EXACT no será reinterpretado como AT_LEAST.

## DB-TXI-149
Guarantee comparison será semántica, no ordinal.

## DB-TXI-150
Ante incertidumbre, VoltStack conservará la incertidumbre explícitamente.

---

# 164. Anti-patterns

## 164.1 Hardcode por vendor

```php
if ($connection->driver() === 'mysql') {
    // isolation behavior
}
```

**Rechazado en el core.**

---

## 164.2 Silent downgrade

```text
SERIALIZABLE
→ unsupported
→ READ_COMMITTED
```

**Prohibido.**

---

## 164.3 Tratar niveles como números

```php
if ($requested->value > $current->value) {
    // stronger
}
```

**Rechazado.**

Las garantías no deberán compararse únicamente mediante ordinales.

---

## 164.4 Savepoint cambia isolation

```text
BEGIN READ COMMITTED
SAVEPOINT
SET SERIALIZABLE
```

como abstracción genérica.

**Rechazado.**

---

## 164.5 ORM implementando isolation

```php
$entityManager->simulateSerializable();
```

**Rechazado.**

---

## 164.6 Replica por ser read-only

```text
readOnly = true
→ always use replica
```

**Rechazado.**

---

## 164.7 Global current isolation

```php
static $currentIsolation;
```

**Prohibido.**

---

## 164.8 Query Planner cambia isolation

```text
optimizer:
    downgrade transaction for speed
```

**Prohibido.**

---

## 164.9 Confundir SERIALIZABLE con locks globales

```text
SERIALIZABLE
=
lock entire database
```

**Rechazado como modelo conceptual universal.**

---

## 164.10 Asumir rollback de side effects

```text
DB rollback
→ email unsent
→ HTTP call undone
```

**Falso.**

---

# 165. Ejemplo de API básica

```php
$result = DB::transaction(
    callback: function () use ($orders) {
        return $orders->processPending();
    },
    isolation: TransactionIsolation::SERIALIZABLE,
);
```

Pipeline:

```text
SERIALIZABLE
    ↓
IsolationResolver
    ↓
Platform profile
    ↓
EXACT supported
    ↓
EffectiveTransactionDefinition
    ↓
TransactionContext
    ↓
physical transaction
```

---

# 166. Ejemplo read-only

```php
$report = DB::transaction(
    callback: fn () => $repository->monthlyReport(),
    isolation: TransactionIsolation::REPEATABLE_READ,
    readOnly: true,
);
```

Conceptualmente:

```text
REPEATABLE_READ
+
READ_ONLY
        ↓
Platform validation
        ↓
Transaction start plan
        ↓
Pinned connection
        ↓
report queries
```

---

# 167. Ejemplo de incompatibilidad

```php
DB::transaction(
    callback: fn () => $service->run(),
    isolation: $customSnapshotIsolation,
);
```

Si la plataforma no puede satisfacerlo:

```text
Transaction Request
        ↓
Isolation Resolver
        ↓
UNSUPPORTED
        ↓
exception
```

`BEGIN` nunca deberá ejecutarse.

---

# 168. Ejemplo nested

```php
DB::transaction(
    function () {
        serviceA();
    },
    isolation: TransactionIsolation::READ_COMMITTED,
);
```

Dentro:

```php
DB::transaction(
    function () {
        serviceB();
    },
    isolation: TransactionIsolation::SERIALIZABLE,
);
```

Si utiliza:

```text
JOIN_EXISTING
```

resultado:

```text
Outer Effective:
    READ_COMMITTED

Inner Requested:
    SERIALIZABLE

Can existing physical transaction satisfy request?
    NO

→ NestedTransactionIsolationConflictException
```

---

# 169. Ejemplo SERIALIZABLE + retry

```text
Attempt 1
    ↓
SERIALIZABLE
    ↓
serialization conflict
    ↓
ROLLBACK
    ↓
RetryPolicy evaluates
    ↓
Attempt 2
    ↓
new TransactionContext
    ↓
SERIALIZABLE
    ↓
COMMIT
```

Isolation System:

```text
classifies/describes
```

Retry System:

```text
decides/re-executes
```

---

# 170. Isolation guarantee model

Conceptualmente:

```text
                 TransactionIsolationDescriptor
                              │
                              ▼
                  TransactionIsolationProfile
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
  GuaranteeSet          ConflictProfile      ApplicationMode
        │
 ┌──────┼─────────┬──────────┬──────────┐
 ▼      ▼         ▼          ▼          ▼
Dirty  NonRepeat Phantom  LostUpdate WriteSkew
Reads   Reads
```

---

# 171. Resolution architecture

```text
                TransactionIsolationRequest
                           │
                           ▼
                Application Isolation Policy
                           │
                           ▼
                  Isolation Registry
                           │
                           ▼
                 Platform Capabilities
                           │
                           ▼
                TransactionIsolationResolver
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
          Resolution                 Failure
              │
              ▼
     Effective Isolation
              │
              ▼
     IsolationApplicationPlan
              │
              ▼
       TransactionManager
              │
              ▼
        TransactionContext
```

---

# 172. Master isolation formula

```text
EffectiveIsolation
=
Resolve(
    RequestedIsolation,
    ResolutionPolicy,
    ApplicationPolicy,
    PlatformCapabilities,
    PlatformConfiguration
)
```

---

# 173. Guarantee formula

```text
EffectiveGuarantees(T)
=
PlatformIsolationProfile(
    EffectiveIsolation(T)
)
```

No:

```text
EffectiveGuarantees
=
HardcodedTableByIsolationName
```

---

# 174. Nested compatibility formula

Para una transacción joined:

```text
CompatibleNestedIsolation
=
Satisfies(
    OuterEffectiveGuarantees,
    InnerRequestedGuarantees,
    InnerResolutionPolicy
)
```

---

# 175. Connection isolation invariant

Durante una transacción `T`:

```text
EffectiveIsolation(T)
=
constant
```

salvo una extensión explícita soportada por plataforma y arquitectura.

---

# 176. Isolation vs locking

```text
ConcurrencyBehavior
=
Isolation
+
ExplicitLocks
+
DatabaseConstraints
+
ApplicationConcurrencyControl
+
PlatformMVCCOrLockingSemantics
```

No:

```text
ConcurrencyBehavior
=
IsolationLevelOnly
```

---

# 177. Modelo final

```text
Application
     │
     ▼
Transaction Definition
     │
     ├── isolation
     ├── read mode
     ├── timeout
     ├── propagation
     └── nested strategy
     │
     ▼
Transaction Isolation Request
     │
     ▼
Isolation Policy
     │
     ▼
Platform Capability Profile
     │
     ▼
Isolation Resolver
     │
     ├── requested
     ├── effective
     ├── guarantees
     ├── confidence
     └── application plan
     │
     ▼
Effective Transaction Definition
     │
     ▼
Transaction Context
     │
     ▼
Transaction Manager
     │
     ▼
Pinned Connection
     │
     ▼
Database Platform
```

---

# 178. Decisiones arquitectónicas finales

VoltStack utilizará un:

```text
Capability-Driven Transaction Isolation Model
```

en lugar de un simple enum traducido a SQL.

Los cuatro niveles estándar:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

formarán la API portable principal.

Sin embargo, cada nivel será resuelto mediante:

```text
Platform Isolation Profile
```

que describirá sus garantías reales.

La arquitectura distinguirá:

```text
Requested Isolation
≠
Effective Isolation
≠
Isolation Guarantees
≠
Physical Implementation
```

El default para una solicitud explícita será:

```text
EXACT
```

y se establece como invariante:

> **VoltStack nunca reducirá silenciosamente una garantía de aislamiento solicitada.**

Los nested transactions que compartan la misma transacción física heredarán su isolation efectivo.

```text
Savepoint
≠
New Isolation Boundary
```

`REQUIRES_NEW`, al crear otra transacción física, podrá resolver un isolation independiente.

Isolation estará integrado con:

```text
Transaction Context
Transaction Manager
Connection State
Platform Capabilities
Retry
Deadlock Handling
Optimistic Locking
Pessimistic Locking
Read/Write Routing
Telemetry
```

pero no sustituirá ninguno de esos sistemas.

En runtimes persistentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el estado efectivo permanecerá scope-local y cualquier modificación persistente de la sesión de base de datos deberá restaurarse antes de reutilizar la conexión.

La regla final será:

> **VoltStack prometerá únicamente las garantías de aislamiento que pueda resolver contra capacidades conocidas de la plataforma; cuando la equivalencia sea parcial, dependiente del motor o desconocida, esa condición será representada explícitamente y nunca ocultada detrás de un nombre SQL aparentemente portable.**

---

# 179. Relación con el bloque Transaction & Concurrency

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

# 180. Siguiente documento

```text
168_DATABASE_NESTED_TRANSACTION_SYSTEM.md
```

El siguiente documento deberá definir en profundidad:

- nested transaction semantics;
- logical vs physical transactions;
- transaction participation;
- propagation;
- `REQUIRED`;
- `REQUIRES_NEW`;
- `SUPPORTS`;
- `MANDATORY`;
- `NOT_SUPPORTED`;
- `NEVER`;
- `NESTED`;
- join-existing semantics;
- ownership;
- nesting depth;
- rollback-only propagation;
- inner failure semantics;
- savepoint-backed nesting;
- suspend/resume;
- connection leases;
- isolation compatibility;
- timeout compatibility;
- read-only compatibility;
- transaction context stacks;
- callback behavior;
- UnitOfWork interaction;
- EntityManager implications;
- retry boundaries;
- persistent runtime safety;
- diagnostics;
- telemetry;
- testing;
- invariantes arquitectónicas.