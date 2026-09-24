# 169_DATABASE_SAVEPOINT_SYSTEM.md

# VoltStack Quantum Database
## Database Savepoint System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 169 — Database Savepoint System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `168_DATABASE_NESTED_TRANSACTION_SYSTEM.md`  
**Siguiente documento:** `170_DATABASE_TRANSACTION_RETRY_SYSTEM.md`

---

# 1. Propósito

`Database Savepoint System` define la arquitectura mediante la cual VoltStack podrá establecer puntos de restauración dentro de una transacción física activa.

Un savepoint permite representar conceptualmente:

```text
BEGIN
    operation A

    SAVEPOINT S1

        operation B

    ROLLBACK TO S1

    operation C

COMMIT
```

sin revertir necesariamente toda la transacción.

Sin embargo:

```text
Savepoint
≠
Nested Physical Transaction
```

y:

```text
Rollback To Savepoint
≠
Rollback Transaction
```

Además:

```text
Database Savepoint
≠
ORM Persistence Checkpoint
```

La regla central será:

> **Un savepoint en VoltStack será un checkpoint físico perteneciente a una única transacción de base de datos activa. Permitirá restaurar únicamente el estado que la plataforma pueda garantizar desde dicho punto, sin asumir que UnitOfWork, IdentityMap, entidades PHP, eventos, side effects externos o cualquier otro estado de aplicación hayan sido restaurados automáticamente.**

---

# 2. Objetivos

El sistema deberá proporcionar:

1. abstracción portable de savepoints;
2. `SavepointId`;
3. `SavepointHandle`;
4. lifecycle explícito;
5. ownership;
6. stack de savepoints;
7. nesting;
8. creación;
9. release;
10. rollback-to-savepoint;
11. validación de contexto;
12. connection affinity;
13. capability detection;
14. compilación específica por plataforma;
15. integración con nested transactions;
16. interacción con rollback-only;
17. clasificación de fallos;
18. representación de estados inciertos;
19. integración opcional con checkpoints ORM;
20. resource governance;
21. telemetry;
22. diagnostics;
23. testing;
24. seguridad bajo runtimes persistentes.

---

# 3. No objetivos

Este sistema no deberá:

- iniciar transacciones físicas;
- realizar commits;
- sustituir `TransactionManager`;
- implementar transaction propagation;
- implementar retry;
- resolver deadlocks;
- modificar isolation level;
- restaurar automáticamente objetos PHP;
- restaurar automáticamente UnitOfWork;
- restaurar automáticamente IdentityMap;
- revertir HTTP requests;
- revertir emails;
- revertir archivos;
- revertir mensajes enviados a queues;
- implementar distributed transactions.

---

# 4. Distinciones fundamentales

VoltStack mantendrá:

```text
Savepoint
≠
Transaction
≠
Nested Transaction Scope
≠
Transaction Context
≠
Rollback-Only Marker
≠
ORM Checkpoint
≠
Application Snapshot
```

---

# 5. Modelo conceptual

```text
Physical Transaction
        │
        ▼
Savepoint Manager
        │
        ▼
Savepoint Definition
        │
        ▼
Capability Validation
        │
        ▼
Savepoint Compiler
        │
        ▼
Connection
        │
        ▼
Database
```

El estado lógico se conservará mediante:

```text
TransactionContext
        │
        ▼
SavepointStack
        │
        ├── S1
        ├── S2
        └── S3
```

---

# 6. Requisito fundamental

Un savepoint únicamente podrá existir dentro de:

```text
ACTIVE PHYSICAL TRANSACTION
```

Por tanto:

```text
No Active Transaction
        +
Create Savepoint
        =
Error
```

---

# 7. SavepointId

VoltStack utilizará una identidad lógica propia.

```php
final readonly class SavepointId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
sp-00000042
```

---

# 8. Logical ID vs database identifier

Deberá distinguirse:

```text
SavepointId
```

de:

```text
PhysicalSavepointName
```

El primero es una identidad interna de VoltStack.

El segundo es el identificador que finalmente utiliza el motor SQL.

---

# 9. Razón de la separación

Una plataforma puede imponer restricciones sobre:

- longitud;
- caracteres;
- quoting;
- case sensitivity;
- namespaces;
- sintaxis.

Por ello:

```text
Logical Savepoint Identity
        ↓
Platform Savepoint Name Resolver
        ↓
Physical Identifier
```

---

# 10. PhysicalSavepointName

```php
final readonly class PhysicalSavepointName
{
    public function __construct(
        public string $value,
    ) {}
}
```

Su construcción será interna.

---

# 11. No nombres arbitrarios desde input externo

No deberá permitirse:

```php
$transaction->savepoint($_GET['name']);
```

si ese valor termina directamente en SQL.

Preferido:

```php
$savepoint = $transaction->createSavepoint();
```

---

# 12. SavepointHandle

La API devolverá un handle tipado:

```php
final readonly class SavepointHandle
{
    public function __construct(
        public SavepointId $id,
        public PhysicalTransactionId $transactionId,
        public ConnectionIdentity $connection,
        public int $depth,
        public SavepointSequence $sequence,
    ) {}
}
```

---

# 13. Handle ≠ estado mutable

`SavepointHandle` será:

```text
immutable reference
```

No almacenará internamente:

```text
ACTIVE
RELEASED
ROLLED_BACK
```

como mutable state compartido.

El estado real residirá en:

```text
SavepointRegistry
/
SavepointStack
```

del `TransactionContext`.

---

# 14. Savepoint ownership

Todo savepoint pertenecerá exactamente a:

```text
one PhysicalTransactionId
```

Por tanto:

```text
Owner(Savepoint)
=
PhysicalTransaction
```

---

# 15. Connection affinity

También:

```text
Connection(Savepoint)
=
Connection(PhysicalTransaction)
```

Un savepoint creado sobre Connection A nunca podrá manipularse desde Connection B.

---

# 16. Transaction affinity

Ejemplo inválido:

```text
TX-A
    creates S1

TX-B
    rollback(S1)
```

Resultado:

```text
SavepointTransactionMismatchException
```

---

# 17. Connection mismatch

También será inválido:

```text
S1 created on Connection #4
rollback attempted on Connection #9
```

aunque exista algún error de infraestructura que haya reutilizado accidentalmente IDs.

---

# 18. Lifecycle

Modelo principal:

```text
DECLARED
    ↓
CREATING
    ↓
ACTIVE
   /   \
  /     \
 ▼       ▼
ROLLING  RELEASING
BACK       │
 │         ▼
 ▼      RELEASED
ROLLED_BACK
```

También:

```text
CREATING
ROLLING_BACK
RELEASING
        ↓
      UNKNOWN
```

cuando el resultado físico no pueda determinarse.

---

# 19. SavepointState

```php
enum SavepointState
{
    case DECLARED;
    case CREATING;
    case ACTIVE;
    case ROLLING_BACK;
    case ROLLED_BACK;
    case RELEASING;
    case RELEASED;
    case INVALIDATED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 20. DECLARED

El objeto lógico existe, pero aún no se ha confirmado:

```text
SAVEPOINT physical_name
```

en la base de datos.

---

# 21. CREATING

VoltStack está intentando establecer el savepoint físicamente.

---

# 22. ACTIVE

Solo se alcanzará cuando exista evidencia suficiente de que el savepoint fue establecido.

---

# 23. ROLLING_BACK

Se está ejecutando:

```text
ROLLBACK TO SAVEPOINT
```

o su equivalente de plataforma.

---

# 24. ROLLED_BACK

Significa:

> La plataforma confirmó suficientemente que el estado transaccional físico fue restaurado al savepoint.

No significa:

```text
Application state restored
```

---

# 25. RELEASING

VoltStack intenta liberar el savepoint.

---

# 26. RELEASED

El savepoint ya no podrá utilizarse para rollback.

---

# 27. INVALIDATED

Un savepoint puede quedar inválido debido a:

- rollback de un ancestro;
- rollback completo de la transacción;
- commit;
- comportamiento específico de plataforma;
- invalidación del TransactionContext.

---

# 28. FAILED

La operación falló y VoltStack posee evidencia suficiente para conocer el estado resultante.

---

# 29. UNKNOWN

Si:

```text
command sent
+
connection lost
+
result unknown
```

VoltStack no deberá fabricar:

```text
ACTIVE
```

ni:

```text
RELEASED
```

ni:

```text
ROLLED_BACK
```

Resultado:

```text
UNKNOWN
```

---

# 30. UNKNOWN propagation

Un savepoint con estado:

```text
UNKNOWN
```

podrá volver incierto al:

```text
TransactionContext
```

porque ya no puede garantizarse qué rollback boundaries existen realmente.

---

# 31. Transaction tainting

Por default:

```text
Savepoint Outcome UNKNOWN
        ↓
TransactionContext
        ↓
TAINTED
```

La continuación de operaciones podrá ser rechazada.

---

# 32. SavepointStack

Cada transacción física tendrá:

```php
final class SavepointStack
{
    /** @var SavepointEntry[] */
    private array $entries = [];
}
```

---

# 33. Stack ordering

La estructura lógica seguirá:

```text
S1
 └── S2
      └── S3
```

y deberá conservar orden determinista.

---

# 34. SavepointEntry

```php
final readonly class SavepointEntry
{
    public function __construct(
        public SavepointHandle $handle,
        public SavepointState $state,
        public ?SavepointId $parent,
        public int $depth,
    ) {}
}
```

La implementación real podrá separar metadata inmutable de runtime state mutable.

---

# 35. Nested savepoints

Ejemplo:

```text
BEGIN

SAVEPOINT S1
    A

    SAVEPOINT S2
        B

        SAVEPOINT S3
            C
```

Jerarquía:

```text
TX
└── S1
    └── S2
        └── S3
```

---

# 36. Rollback a un ancestro

Si se ejecuta:

```text
ROLLBACK TO S1
```

los savepoints descendientes:

```text
S2
S3
```

podrán quedar invalidados según las semánticas normalizadas de VoltStack y de la plataforma.

---

# 37. Descendant invalidation

El sistema deberá calcular:

```text
RollbackTarget
        ↓
DescendantSet
        ↓
Invalidate / Reconcile
```

No dejar handles descendientes aparentemente utilizables.

---

# 38. Platform semantics matter

Los motores pueden diferir respecto a qué ocurre con el savepoint objetivo después de:

```text
ROLLBACK TO SAVEPOINT
```

Por tanto VoltStack deberá modelar:

```text
TargetRetentionSemantics
```

---

# 39. TargetRetentionSemantics

```php
enum SavepointTargetRetention
{
    case REMAINS_ACTIVE;
    case CONSUMED;
    case PLATFORM_DEPENDENT;
    case UNKNOWN;
}
```

---

# 40. Normalización

VoltStack podrá normalizar determinadas diferencias mediante:

```text
rollback
+
release/recreate
```

solo cuando:

- sea seguro;
- esté soportado;
- la semántica esté documentada;
- no introduzca falsa atomicidad.

---

# 41. No emulación insegura

Si una plataforma no soporta savepoints:

```text
NESTED
```

no deberá convertirse silenciosamente en:

```text
REQUIRED
```

ni:

```text
REQUIRES_NEW
```

---

# 42. Capability model

Cada plataforma deberá exponer:

```php
interface SavepointCapabilities
{
    public function supportsSavepoints(): bool;

    public function supportsRelease(): bool;

    public function supportsRollbackTo(): bool;

    public function supportsNestedSavepoints(): bool;

    public function maxDepth(): ?int;

    public function targetRetention(): SavepointTargetRetention;
}
```

---

# 43. Capability-driven design

Nunca:

```php
if ($driver === 'mysql') {
    // savepoint
}
```

en el core.

Preferido:

```php
if (!$platform->savepoints()->supportsSavepoints()) {
    throw new UnsupportedSavepointException();
}
```

---

# 44. Version-aware capabilities

El perfil podrá depender de:

```text
platform
version
driver
server configuration
transaction mode
```

---

# 45. MySQL

`MySQLPlatform` tendrá su propio:

```text
SavepointCapabilityProfile
```

---

# 46. MariaDB

`MariaDBPlatform` tendrá otro profile independiente.

Nunca:

```text
MariaDB savepoint semantics
=
MySQL semantics by alias
```

aunque existan similitudes.

---

# 47. PostgreSQL

PostgreSQL expondrá sus capacidades mediante el mismo contrato canónico.

---

# 48. SQLite

SQLite será modelado según sus capacidades reales y restricciones operacionales.

VoltStack no inferirá equivalencia completa únicamente por soportar la palabra:

```text
SAVEPOINT
```

---

# 49. SavepointCompiler

La intención lógica:

```text
CreateSavepoint(S1)
```

será compilada por la plataforma.

```php
interface SavepointCompiler
{
    public function compileCreate(
        PhysicalSavepointName $name,
    ): CompiledSavepointCommand;

    public function compileRollback(
        PhysicalSavepointName $name,
    ): CompiledSavepointCommand;

    public function compileRelease(
        PhysicalSavepointName $name,
    ): CompiledSavepointCommand;
}
```

---

# 50. Compiler does not execute

Regla:

```text
SavepointCompiler
≠
SavepointExecutor
```

El compiler produce representación.

La capa de conexión ejecuta.

---

# 51. CompiledSavepointCommand

```php
final readonly class CompiledSavepointCommand
{
    public function __construct(
        public SavepointCommandType $type,
        public string $sql,
    ) {}
}
```

Este SQL es generado internamente por trusted platform code.

---

# 52. Command types

```php
enum SavepointCommandType
{
    case CREATE;
    case ROLLBACK_TO;
    case RELEASE;
}
```

---

# 53. SavepointManager

Contrato principal:

```php
interface SavepointManager
{
    public function create(
        TransactionContext $transaction,
    ): SavepointHandle;

    public function rollbackTo(
        TransactionContext $transaction,
        SavepointHandle $savepoint,
    ): void;

    public function release(
        TransactionContext $transaction,
        SavepointHandle $savepoint,
    ): void;
}
```

---

# 54. Manager responsibilities

`SavepointManager` coordinará:

- validation;
- allocation;
- naming;
- stack;
- capability checks;
- compilation;
- execution;
- state transition;
- failure classification;
- diagnostics.

No deberá conocer:

- Entity implementation;
- Repository;
- HTTP;
- business services.

---

# 55. Create pipeline

```text
create()
   ↓
Validate Active Transaction
   ↓
Validate Transaction Health
   ↓
Validate Capability
   ↓
Validate Resource Budget
   ↓
Allocate SavepointId
   ↓
Resolve Physical Name
   ↓
Register CREATING
   ↓
Compile CREATE
   ↓
Execute on Pinned Connection
   ↓
Classify Outcome
   ↓
ACTIVE
```

---

# 56. Create preconditions

Deberá verificarse:

```text
TransactionState == ACTIVE
TransactionContext not TAINTED
Connection available
Connection matches transaction
Savepoint capability supported
Depth within limit
```

---

# 57. Create success

Solo después de confirmación suficiente:

```text
CREATING
→
ACTIVE
```

---

# 58. Create failure before send

Si falla antes de enviar el comando:

```text
CREATING
→
FAILED
```

sin asumir que el savepoint existe.

---

# 59. Create ambiguous failure

Si:

```text
command sent
connection lost
```

entonces:

```text
CREATING
→
UNKNOWN
```

---

# 60. Release pipeline

```text
release(S1)
    ↓
Validate Handle
    ↓
Validate Ownership
    ↓
Validate ACTIVE
    ↓
Validate Stack Semantics
    ↓
RELEASING
    ↓
Compile RELEASE
    ↓
Execute
    ↓
Classify
    ↓
RELEASED
```

---

# 61. Release unsupported

Algunas plataformas pueden no ofrecer una operación física equivalente o pueden tratarla de manera distinta.

La capability deberá expresar esta condición.

---

# 62. Logical release

Si una plataforma no requiere o no soporta physical release, VoltStack solo podrá implementar una normalización lógica si puede demostrar que:

```text
future rollback through this handle
```

queda imposibilitado de manera segura.

Nunca fingirá que el servidor eliminó físicamente algo que no puede verificar.

---

# 63. Rollback pipeline

```text
rollbackTo(S1)
    ↓
Validate Handle
    ↓
Validate Ownership
    ↓
Validate State
    ↓
Determine Descendants
    ↓
ROLLING_BACK
    ↓
Compile ROLLBACK TO
    ↓
Execute
    ↓
Classify Outcome
    ↓
Reconcile Stack
    ↓
ROLLED_BACK / ACTIVE / INVALIDATED
```

dependiendo de retention semantics.

---

# 64. Rollback target semantics

Después de rollback, el target podrá:

```text
remain active
```

o ser:

```text
consumed
```

según el profile.

VoltStack deberá reflejar el estado efectivo.

---

# 65. Rollback descendants

Todo descendiente cuyo estado ya no sea físicamente válido deberá marcarse:

```text
INVALIDATED
```

---

# 66. Invalid handle reuse

Intentar:

```php
$savepoints->rollbackTo($transaction, $invalidated);
```

deberá producir:

```text
InvalidSavepointStateException
```

---

# 67. Released handle reuse

Igualmente:

```text
RELEASED
→ cannot rollback
```

---

# 68. Rolled-back handle reuse

Dependerá de:

```text
SavepointTargetRetention
```

No deberá suponerse universalmente.

---

# 69. Savepoint sequence

Cada transacción podrá mantener:

```text
1
2
3
4
...
```

como secuencia monotónica interna.

---

# 70. Sequence reuse

No reutilizar números durante la misma transacción por default.

Ejemplo:

```text
S1 created
S1 released
next → S2
```

Esto simplifica:

- diagnostics;
- tracing;
- debugging;
- uniqueness.

---

# 71. Physical naming strategy

Ejemplo:

```text
vsp_1
vsp_2
vsp_3
```

donde:

```text
vsp
=
VoltStack SavePoint
```

---

# 72. Naming requirements

Los nombres deberán ser:

- deterministas dentro de la transacción;
- válidos para la plataforma;
- libres de input externo;
- suficientemente únicos;
- de cardinalidad controlada.

---

# 73. No business metadata in name

No:

```text
savepoint_user_123_order_9844
```

Preferido:

```text
vsp_7
```

---

# 74. Transaction ID not necessarily embedded

No es obligatorio producir:

```text
vsp_tx_ABC123_7
```

si esto excede límites o expone información.

La relación estará en metadata interna.

---

# 75. Savepoint ownership model

```text
Physical TX-1
│
├── S1
│   └── owner = TX-1
│
└── S2
    └── owner = TX-1
```

No existe:

```text
S1 owner = logical scope only
```

como ownership físico.

---

# 76. Logical scope association

Aun así podrá registrarse:

```text
CreatedByScopeId
```

para:

- diagnostics;
- cleanup;
- NESTED propagation.

---

# 77. Nested transaction integration

Documento 168 estableció:

```text
TransactionPropagation::NESTED
```

Con transacción activa:

```text
Nested Transaction System
        ↓
requests savepoint
        ↓
Savepoint System
```

---

# 78. Separation of responsibilities

```text
Nested System:
    decides WHY a savepoint is needed

Savepoint System:
    decides HOW it is represented safely

Platform:
    decides HOW it is expressed physically
```

---

# 79. NESTED success

```text
Outer TX
    ↓
Create S1
    ↓
Execute Inner Scope
    ↓
Success
    ↓
Release S1
    ↓
Outer continues
```

---

# 80. NESTED failure

```text
Outer TX
    ↓
Create S1
    ↓
Inner fails
    ↓
Rollback To S1
    ↓
Reconcile
    ↓
Outer may continue
```

solo si el estado restante es seguro.

---

# 81. Rollback-only interaction

Un savepoint rollback exitoso no deberá eliminar automáticamente un:

```text
rollback-only
```

preexistente en la transacción.

---

# 82. Preexisting rollback-only

Ejemplo:

```text
TX already rollback-only
    ↓
create/rollback savepoint
```

no transforma:

```text
ROLLBACK_ONLY
→
CLEAN
```

---

# 83. Inner failure recovery

Si un inner `NESTED` falla y el savepoint permite recuperación completa del estado DB correspondiente:

```text
outer rollback-only
```

no tendrá que marcarse necesariamente.

Pero esta decisión depende también del estado ORM y otros recursos.

---

# 84. Failed rollback-to-savepoint

Si el intento de rollback parcial falla:

```text
TransactionContext
→ rollback-only
```

como mínimo.

Dependiendo de la evidencia:

```text
TAINTED
UNKNOWN
```

también podrán ser necesarios.

---

# 85. Savepoint UNKNOWN

Caso crítico:

```text
ROLLBACK TO S1 sent
connection lost
```

No se sabe si:

- rollback ocurrió;
- rollback no ocurrió;
- conexión/transaction abortó;
- servidor mantuvo la transacción.

Resultado:

```text
Transaction outcome knowledge degraded
```

---

# 86. No continuation after unsafe unknown

Por default:

```text
Savepoint operation UNKNOWN
→ reject further application statements
```

hasta completion/recovery apropiada.

---

# 87. Savepoint and full rollback

Cuando ocurre:

```text
ROLLBACK TRANSACTION
```

todos los savepoints de la transacción se vuelven terminales.

---

# 88. Full rollback invalidation

El stack completo deberá marcarse:

```text
INVALIDATED
```

o limpiarse después de conservar diagnostics necesarios.

---

# 89. Savepoint and commit

Cuando ocurre:

```text
COMMIT
```

todos los savepoints dejan de existir.

No deberán permanecer handles activos después del commit.

---

# 90. Transaction completion hook

`SavepointManager` deberá recibir:

```text
onTransactionCompleted()
```

o equivalente para invalidar/limpiar estado.

---

# 91. Cleanup is not physical release

Al finalizar toda la transacción:

```text
clear SavepointStack
```

no significa que sea necesario emitir:

```text
RELEASE
```

para cada savepoint.

La transacción física ya terminó.

---

# 92. ORM problem

Supongamos:

```php
$savepoint = $tx->savepoint();

$user->setEmail('new@example.com');

$entityManager->flush();

$tx->rollbackTo($savepoint);
```

La DB puede recuperar:

```text
old@example.com
```

pero el objeto PHP puede seguir conteniendo:

```text
new@example.com
```

---

# 93. Core ORM invariant

Por tanto:

```text
DatabaseRollbackToSavepoint
≠
EntityStateRollback
```

---

# 94. UnitOfWork problem

Después del rollback:

```text
Database:
    old value

Entity:
    new value

Snapshot:
    potentially new or old depending flush reconciliation
```

Esto puede producir:

```text
DatabaseReality
≠
ORMKnowledge
```

---

# 95. Persistence checkpoint concept

VoltStack podrá introducir:

```text
PersistenceCheckpoint
```

como abstracción separada.

---

# 96. PersistenceCheckpoint

Conceptualmente:

```php
interface PersistenceCheckpoint
{
    public function restore(): PersistenceCheckpointResult;
}
```

Pero este mecanismo:

```text
≠ Savepoint
```

---

# 97. Coordinated checkpoint

Una futura coordinación segura podría utilizar:

```text
TransactionalCheckpoint
│
├── Database Savepoint
└── Persistence Checkpoint
```

---

# 98. No mandatory deep object clone

VoltStack no deberá implementar ORM checkpoint simplemente mediante:

```php
$copy = unserialize(serialize($entityManager));
```

Esto sería:

- inseguro;
- costoso;
- incorrecto;
- incompatible con resources;
- incompatible con proxies;
- incompatible con identity guarantees.

---

# 99. Possible ORM checkpoint strategies

Podrán evaluarse:

```text
UoW snapshot checkpoint
ChangeSet checkpoint
Entity snapshot versioning
Context clear-on-rollback
Context taint-on-rollback
Explicit reload strategy
```

---

# 100. Safe default

Mientras no pueda demostrarse una restauración correcta:

```text
nested DB rollback
+
ORM mutations
→
PersistenceContext may become TAINTED
```

---

# 101. Context clear policy

Una política conservadora podría usar:

```text
rollback to savepoint
    ↓
clear affected PersistenceContext
```

pero tampoco deberá asumirse que esto restaura objetos que el código externo todavía conserva.

---

# 102. Detached references

Después de un clear:

```text
previous entity objects
→ DETACHED
```

y deberán tratarse como tales.

---

# 103. Generated IDs

Caso:

```text
SAVEPOINT S1

INSERT entity
generated ID = 100

ROLLBACK TO S1
```

El objeto PHP puede conservar:

```text
id = 100
```

aunque la fila ya no exista.

---

# 104. Generated ID reconciliation

Por tanto:

```text
DB savepoint rollback
```

no deberá asumir:

```text
generated identifier rewind
```

---

# 105. Sequence/autoincrement behavior

Además, el generador físico puede no retroceder.

Así:

```text
rollback insert ID 100
next insert
→ maybe 101
```

Esto es válido.

---

# 106. IdentityMap hazard

Si una entidad insertada después del savepoint fue registrada como:

```text
EntityKey(User, 100)
```

y luego se revierte la fila:

```text
IdentityMap
```

debe reconciliarse explícitamente.

---

# 107. Relationship state

Cambios como:

```text
add child
remove relation
modify pivot membership
```

también pueden existir únicamente en memoria después del rollback.

---

# 108. Collection state

Un:

```text
PersistentCollection
```

no deberá asumirse restaurado por un savepoint físico.

---

# 109. Event side effects

Supongamos:

```text
SAVEPOINT
    ↓
entity persisted
    ↓
event listener sends email
    ↓
ROLLBACK TO SAVEPOINT
```

El email no desaparece.

---

# 110. Side-effect rule

```text
Savepoint Rollback
≠
External Side Effect Rollback
```

---

# 111. afterCommit integration

Eventos externos que requieran confirmación DB deberían preferir:

```text
afterCommit
```

o:

```text
Outbox
```

según arquitectura.

---

# 112. afterSavepointRelease

Podrá existir un evento interno:

```text
SavepointReleased
```

pero:

```text
SavepointReleased
≠
TransactionCommitted
```

---

# 113. No business commit semantics

No deberá recomendarse:

```php
DB::afterSavepointRelease(
    fn () => sendInvoice()
);
```

como sustituto de `afterCommit`.

---

# 114. Retry integration

Savepoints podrán ser utilizados por:

```text
TransactionRetrySystem
```

solo bajo políticas explícitas.

---

# 115. Full transaction retry remains separate

Regla general:

```text
Transaction Retry
→ retry entire physical transaction boundary
```

no:

```text
automatically rollback to latest savepoint
```

---

# 116. Savepoint-scoped retry

Una futura estrategia:

```text
Create S1
    ↓
Execute operation
    ↓
Transient failure
    ↓
Rollback To S1
    ↓
Retry operation
```

solo será válida si:

- DB state puede restaurarse;
- error no invalida toda la TX;
- ORM state puede restaurarse;
- side effects son retry-safe;
- platform permite continuar;
- policy lo autoriza.

---

# 117. Transaction-aborting errors

Algunas clases de error pueden dejar:

```text
whole transaction unusable
```

aunque exista savepoint.

El Error Classifier deberá indicar:

```text
SAVEPOINT_RECOVERABLE
TRANSACTION_ABORTING
CONNECTION_FATAL
UNKNOWN
```

---

# 118. SavepointRecoveryCapability

```php
enum SavepointRecoveryCapability
{
    case RECOVERABLE;
    case TRANSACTION_ABORTED;
    case CONNECTION_LOST;
    case PLATFORM_DEPENDENT;
    case UNKNOWN;
}
```

---

# 119. Error classification ownership

`SavepointSystem` podrá consumir la clasificación.

No deberá contener cientos de:

```php
if ($sqlState === '...')
```

vendor-specific en el core.

---

# 120. Deadlock interaction

Un deadlock puede provocar:

```text
entire transaction rollback
```

dependiendo del motor.

Por tanto no deberá asumirse:

```text
deadlock
→ rollback to savepoint
→ continue
```

---

# 121. Serialization failure interaction

Igualmente:

```text
serialization failure
```

puede requerir:

```text
full transaction retry
```

---

# 122. Connection failure

Una conexión perdida invalida la capacidad de usar:

```text
SavepointHandle
```

sobre una conexión nueva.

---

# 123. No savepoint failover

Prohibido:

```text
Connection A lost
    ↓
switch to Connection B
    ↓
ROLLBACK TO S1
```

El savepoint pertenece a la transacción física de A.

---

# 124. Read/write routing

Mientras exista una transacción con savepoints:

```text
all commands
→ pinned transactional connection
```

---

# 125. Isolation

Crear un savepoint no modifica:

```text
TransactionIsolation
```

---

# 126. Read mode

Tampoco modifica:

```text
READ_ONLY / READ_WRITE
```

de la transacción física.

---

# 127. Timeout

El savepoint no reinicia:

```text
TransactionDeadline
```

---

# 128. Savepoint lifetime

Debe cumplirse:

```text
Lifetime(Savepoint)
⊂
Lifetime(PhysicalTransaction)
```

---

# 129. No cross-request lifetime

Un `SavepointHandle` no deberá sobrevivir:

```text
HTTP request boundary
```

para ser utilizado después.

---

# 130. No serialization as resumable token

No:

```php
$queue->dispatch(serialize($savepoint));
```

para continuar la misma transacción.

---

# 131. Runtime scope

Mutable savepoint state será:

```text
TransactionContext-local
```

---

# 132. Shared immutable components

Podrán compartirse entre workers:

```text
SavepointCapabilityProfile
SavepointCompiler
SavepointNamingPolicy
Compiled platform metadata
```

si son inmutables.

---

# 133. Mutable components

No deberán compartirse globalmente:

```text
SavepointStack
SavepointSequence
SavepointState
Active Handles
```

---

# 134. FrankenPHP

Cada request deberá terminar con:

```text
no leaked SavepointStack
```

---

# 135. RoadRunner

Cada job/request deberá recibir su propio runtime transaction state.

---

# 136. OpenSwoole

Cada coroutine deberá mantener:

```text
independent SavepointStack
```

---

# 137. Resource governance

Configuración conceptual:

```php
'database' => [
    'transactions' => [
        'savepoints' => [
            'enabled' => true,
            'max_depth' => 32,
            'max_per_transaction' => 128,
        ],
    ],
];
```

---

# 138. Max depth

```text
Depth(Savepoint)
≤
ConfiguredMaxDepth
```

---

# 139. Max count

También podrá limitarse el número total creado durante una transacción:

```text
CreatedSavepoints(TX)
≤
MaxPerTransaction
```

aunque los anteriores hayan sido liberados.

Esto protege frente a loops patológicos.

---

# 140. Limit exception

```text
SavepointResourceLimitExceededException
```

---

# 141. Memory bounds

Diagnostics no conservarán indefinidamente todos los savepoints históricos.

Podrá existir:

```text
bounded history
```

---

# 142. Savepoint history

Para debugging:

```php
final readonly class SavepointHistoryEntry
{
    public function __construct(
        public SavepointId $id,
        public SavepointOperation $operation,
        public SavepointOutcome $outcome,
    ) {}
}
```

---

# 143. History is observational

History:

```text
≠ source of runtime truth
```

La verdad operacional está en el TransactionContext actual.

---

# 144. Telemetry

Métricas posibles:

```text
db.transaction.savepoint.created
db.transaction.savepoint.released
db.transaction.savepoint.rollback
db.transaction.savepoint.failed
db.transaction.savepoint.unknown
db.transaction.savepoint.depth
```

---

# 145. Low-cardinality labels

Permitidos:

```text
platform
operation
outcome
nested=true|false
```

Evitar:

```text
SavepointId
TransactionId
UserId
raw name
```

como metric labels.

---

# 146. Tracing

Span conceptual:

```text
db.transaction.savepoint
```

con atributos:

```text
operation=create
depth=2
outcome=success
```

---

# 147. Diagnostics

Ejemplo:

```text
SAVEPOINT

Logical ID:
    sp-17

Physical Transaction:
    tx-8

Depth:
    2

State:
    ACTIVE

Created By:
    scope-12

Platform:
    PostgreSQL

Connection:
    pinned

Target Retention:
    REMAINS_ACTIVE
```

---

# 148. Rollback diagnostic

```text
SAVEPOINT ROLLBACK

Savepoint:
    sp-17

Previous State:
    ACTIVE

Operation:
    ROLLBACK_TO

Outcome:
    SUCCESS

Descendants Invalidated:
    2

Transaction:
    ACTIVE

Transaction Tainted:
    false
```

---

# 149. Unknown diagnostic

```text
SAVEPOINT OPERATION UNCERTAIN

Savepoint:
    sp-17

Operation:
    ROLLBACK_TO

Connection:
    LOST

Server Outcome:
    UNKNOWN

Transaction:
    TAINTED

Further Statements:
    REJECTED
```

---

# 150. Explain API

Podrá existir:

```php
DB::transactions()
    ->savepoints()
    ->explainCurrent();
```

Resultado:

```text
Physical TX:
    tx-22

Savepoints:

S1
  state: ACTIVE
  depth: 1

S2
  state: ACTIVE
  depth: 2

S3
  state: RELEASED
  depth: 3
```

---

# 151. Security

El sistema deberá impedir:

- savepoint name injection;
- raw SQL injection;
- cross-transaction handles;
- cross-connection handles;
- forged runtime contexts;
- unbounded nesting;
- arbitrary dynamic compiler registration.

---

# 152. Handle validation

Todo handle recibido deberá validarse contra:

```text
TransactionId
ConnectionIdentity
RuntimeScope
RegistryEntry
CurrentState
```

---

# 153. Opaque public handle

La API pública podrá tratar `SavepointHandle` como opaque capability.

La aplicación no deberá modificar:

```text
physicalName
depth
transaction identity
```

---

# 154. No physical names in public API

Idealmente:

```php
$sp = $tx->createSavepoint();
$tx->rollbackTo($sp);
```

en lugar de:

```php
$tx->rollbackTo('vsp_7');
```

---

# 155. Error hierarchy

```text
DatabaseSavepointException
│
├── SavepointNotSupportedException
├── SavepointRequiresActiveTransactionException
├── SavepointCreationException
├── SavepointRollbackException
├── SavepointReleaseException
├── SavepointUnknownOutcomeException
├── SavepointNotFoundException
├── InvalidSavepointStateException
├── SavepointTransactionMismatchException
├── SavepointConnectionMismatchException
├── SavepointOwnershipException
├── SavepointResourceLimitExceededException
├── SavepointStackCorruptionException
├── SavepointCompilationException
├── SavepointCapabilityException
└── SavepointInvariantViolationException
```

---

# 156. Creation exception

Significa:

> El sistema posee evidencia de que la creación no pudo completarse correctamente.

---

# 157. Unknown outcome exception

Significa:

> La operación fue iniciada, pero VoltStack no puede determinar con suficiente certeza el estado físico resultante.

---

# 158. State exception

Ejemplo:

```text
rollbackTo(RELEASED savepoint)
```

---

# 159. Stack corruption

Detectará situaciones como:

```text
logical stack says S3 active
physical/context lifecycle says transaction ended
```

---

# 160. Capability exception

Se utilizará cuando una operación requerida no pueda representarse de forma compatible con las capacidades conocidas.

---

# 161. Testing architecture

La suite deberá incluir:

```text
SavepointUnitTests
SavepointLifecycleTests
SavepointStackTests
SavepointNamingTests
SavepointOwnershipTests
SavepointCapabilityTests
SavepointCompilationTests
SavepointIntegrationTests
NestedSavepointTests
SavepointRollbackTests
SavepointReleaseTests
SavepointFailureTests
SavepointUnknownOutcomeTests
SavepointORMIntegrationTests
SavepointResourceTests
SavepointPersistentRuntimeTests
SavepointPlatformConformanceTests
```

---

# 162. Basic lifecycle test

```text
BEGIN
CREATE S1
ROLLBACK TO S1
RELEASE S1
COMMIT
```

validando cada state transition.

---

# 163. No active transaction test

```text
createSavepoint()
```

sin TX deberá producir:

```text
SavepointRequiresActiveTransactionException
```

---

# 164. Ownership test

```text
TX-A creates S1
TX-B tries release S1
```

deberá fallar.

---

# 165. Connection test

Un handle no podrá utilizarse sobre otra conexión.

---

# 166. Nested stack test

```text
S1
    S2
        S3
```

deberá producir:

```text
depth:
1
2
3
```

---

# 167. Ancestor rollback test

```text
S1
    S2
        S3

ROLLBACK TO S1
```

deberá reconciliar correctamente:

```text
S2
S3
```

según profile.

---

# 168. Release test

Después de:

```text
RELEASE S1
```

`rollbackTo(S1)` deberá fallar.

---

# 169. Full rollback test

```text
BEGIN
S1
S2
ROLLBACK
```

deberá dejar:

```text
no active savepoints
```

---

# 170. Commit cleanup test

```text
BEGIN
S1
COMMIT
```

deberá limpiar/inutilizar el handle.

---

# 171. Unknown create test

Simular:

```text
SAVEPOINT sent
connection lost
```

Resultado:

```text
savepoint UNKNOWN
transaction TAINTED
```

---

# 172. Unknown rollback test

Simular:

```text
ROLLBACK TO sent
connection lost
```

No deberá marcar:

```text
ROLLED_BACK
```

sin evidencia.

---

# 173. ORM generated ID test

```text
S1
INSERT entity
generated ID
flush
ROLLBACK TO S1
```

Verificar que VoltStack no afirme automáticamente que el entity state quedó restaurado.

---

# 174. ORM dirty state test

Modificar entidad después de S1, flush y rollback.

El sistema deberá aplicar la política de reconciliación configurada:

```text
TAINT
CLEAR
CHECKPOINT_RESTORE
```

si dichas estrategias existen.

---

# 175. Side effect test

Un side effect externo realizado después del savepoint no deberá aparecer como revertido en diagnostics.

---

# 176. Resource test

Crear savepoints hasta exceder:

```text
max_per_transaction
```

deberá fallar antes de enviar el siguiente comando físico.

---

# 177. Persistent runtime test

Request A:

```text
TX
S1
S2
ROLLBACK
```

Request B:

```text
new TX
```

deberá comenzar con:

```text
SavepointStack = empty
Sequence = fresh
```

---

# 178. Coroutine test

```text
Coroutine A:
    TX-A
    S1-A

Coroutine B:
    TX-B
    S1-B
```

No deberá existir contaminación entre stacks.

---

# 179. Platform conformance

Cada plataforma deberá verificar:

```text
create
rollback-to
release
nesting
target retention
identifier rules
transaction-end behavior
failure behavior
```

---

# 180. Performance requirements

Las operaciones internas de stack deberán ser aproximadamente:

```text
create → O(1)
top lookup → O(1)
release top → O(1)
```

Rollback a ancestro podrá requerir:

```text
O(number of invalidated descendants)
```

---

# 181. No database introspection per operation

No consultar:

```text
supports savepoints?
```

al servidor por cada savepoint.

Preferir:

```text
Platform Capability Profile
```

precompilado.

---

# 182. Proposed directory structure

```text
src/Quantum/Database/Transaction/
│
├── Savepoint/
│   ├── SavepointId.php
│   ├── SavepointHandle.php
│   ├── SavepointState.php
│   ├── SavepointEntry.php
│   ├── SavepointSequence.php
│   ├── SavepointStack.php
│   ├── SavepointManager.php
│   │
│   ├── Naming/
│   │   ├── SavepointNameResolver.php
│   │   ├── PhysicalSavepointName.php
│   │   └── SavepointNamingPolicy.php
│   │
│   ├── Capability/
│   │   ├── SavepointCapabilities.php
│   │   ├── SavepointCapabilityProfile.php
│   │   ├── SavepointTargetRetention.php
│   │   └── SavepointRecoveryCapability.php
│   │
│   ├── Compilation/
│   │   ├── SavepointCompiler.php
│   │   ├── CompiledSavepointCommand.php
│   │   └── SavepointCommandType.php
│   │
│   ├── Runtime/
│   │   ├── SavepointRegistry.php
│   │   └── SavepointRuntimeState.php
│   │
│   ├── Recovery/
│   │   ├── SavepointRecoveryPolicy.php
│   │   └── SavepointRecoveryResult.php
│   │
│   ├── Diagnostics/
│   │   ├── SavepointInspector.php
│   │   ├── SavepointDiagnostic.php
│   │   └── SavepointHistoryEntry.php
│   │
│   └── Exception/
│       └── ...
```

---

# 183. Dependency model

Permitido:

```text
Savepoint System
    ↓
Transaction Context Contracts
Connection Contracts
Platform Capabilities
Platform Compiler Contracts
Execution/Error Classification
Telemetry Contracts
Runtime Scope Contracts
```

No permitido:

```text
Savepoint System
    ↓
Entity classes
Repositories
HTTP globals
PDO vendor checks in core
Application services
Static mutable runtime state
```

---

# 184. Architectural invariants

## DB-SP-001
Un savepoint requerirá una transacción física activa.

## DB-SP-002
Savepoint no será equivalente a transacción.

## DB-SP-003
Savepoint no será equivalente a nested physical transaction.

## DB-SP-004
Savepoint no será equivalente a ORM checkpoint.

## DB-SP-005
Savepoint pertenecerá a una única physical transaction.

## DB-SP-006
Savepoint conservará connection affinity.

## DB-SP-007
Un savepoint no podrá utilizarse desde otra transacción.

## DB-SP-008
Un savepoint no podrá utilizarse desde otra conexión.

## DB-SP-009
SavepointId será distinto de physical savepoint name.

## DB-SP-010
Physical names serán generados internamente.

## DB-SP-011
Input externo no será insertado directamente como savepoint identifier.

## DB-SP-012
SavepointHandle será tipado.

## DB-SP-013
SavepointHandle no será source of mutable runtime truth.

## DB-SP-014
Runtime state residirá en TransactionContext.

## DB-SP-015
Savepoint stack será transaction-local.

## DB-SP-016
Savepoint stack será execution-local.

## DB-SP-017
Savepoint stack no será static mutable state.

## DB-SP-018
Savepoint sequence será transaction-local.

## DB-SP-019
Savepoint sequence será monotónica por default.

## DB-SP-020
Nombres liberados no serán reutilizados por default dentro de la misma TX.

## DB-SP-021
Savepoint lifecycle será explícito.

## DB-SP-022
CREATING no será ACTIVE.

## DB-SP-023
ROLLING_BACK no será ROLLED_BACK.

## DB-SP-024
RELEASING no será RELEASED.

## DB-SP-025
Ambiguous create producirá UNKNOWN.

## DB-SP-026
Ambiguous rollback producirá UNKNOWN.

## DB-SP-027
Ambiguous release producirá UNKNOWN.

## DB-SP-028
UNKNOWN nunca será convertido silenciosamente a success.

## DB-SP-029
UNKNOWN podrá taintar TransactionContext.

## DB-SP-030
Una transacción tainted podrá rechazar nuevos statements.

## DB-SP-031
Savepoint creation será capability-driven.

## DB-SP-032
Savepoint rollback será capability-driven.

## DB-SP-033
Savepoint release será capability-driven.

## DB-SP-034
Nested savepoint support será capability-driven.

## DB-SP-035
Core no contendrá vendor conditionals para savepoints.

## DB-SP-036
MySQL tendrá profile independiente.

## DB-SP-037
MariaDB tendrá profile independiente.

## DB-SP-038
PostgreSQL tendrá profile independiente.

## DB-SP-039
SQLite tendrá profile independiente.

## DB-SP-040
Vendor name no equivaldrá a capability.

## DB-SP-041
Version podrá afectar capabilities.

## DB-SP-042
Platform compiler generará representación física.

## DB-SP-043
Compiler no ejecutará comandos.

## DB-SP-044
Connection ejecutará comandos físicos.

## DB-SP-045
Savepoint Manager coordinará, no implementará vendor SQL.

## DB-SP-046
Create validará TransactionState.

## DB-SP-047
Create validará TransactionHealth.

## DB-SP-048
Create validará resource limits.

## DB-SP-049
Create validará platform capability.

## DB-SP-050
Create solo producirá ACTIVE con evidencia suficiente.

## DB-SP-051
Rollback validará handle ownership.

## DB-SP-052
Rollback validará handle state.

## DB-SP-053
Release validará handle ownership.

## DB-SP-054
Release validará handle state.

## DB-SP-055
Released savepoint no podrá utilizarse para rollback.

## DB-SP-056
Invalidated savepoint no podrá utilizarse.

## DB-SP-057
Transaction completion invalidará savepoints restantes.

## DB-SP-058
Commit invalidará todos los savepoints.

## DB-SP-059
Full rollback invalidará todos los savepoints.

## DB-SP-060
Transaction cleanup eliminará mutable savepoint state.

## DB-SP-061
Transaction cleanup no requerirá release individual después de completion.

## DB-SP-062
Nested savepoints conservarán jerarquía.

## DB-SP-063
Rollback a ancestro reconciliará descendientes.

## DB-SP-064
Descendientes físicamente inválidos serán marcados INVALIDATED.

## DB-SP-065
Target retention será platform-aware.

## DB-SP-066
Target retention no será asumido universalmente.

## DB-SP-067
VoltStack no realizará emulación insegura.

## DB-SP-068
NESTED no degradará silenciosamente a REQUIRED.

## DB-SP-069
NESTED no degradará silenciosamente a REQUIRES_NEW.

## DB-SP-070
Nested System decidirá cuándo necesita savepoint.

## DB-SP-071
Savepoint System decidirá cómo gestionarlo.

## DB-SP-072
Platform decidirá cómo expresarlo físicamente.

## DB-SP-073
Savepoint release no será commit.

## DB-SP-074
Savepoint rollback no será transaction rollback.

## DB-SP-075
Savepoint creation no modificará isolation.

## DB-SP-076
Savepoint creation no reiniciará timeout.

## DB-SP-077
Savepoint creation no modificará read mode.

## DB-SP-078
Savepoint lifetime será menor que physical transaction lifetime.

## DB-SP-079
Savepoint no cruzará HTTP request boundaries.

## DB-SP-080
Savepoint no se serializará como resumable transaction token.

## DB-SP-081
Connection failover no preservará savepoint.

## DB-SP-082
Savepoint no podrá reconstruirse sobre otra conexión.

## DB-SP-083
Read/write routing respetará pinned connection.

## DB-SP-084
Rollback-only preexistente no será limpiado por savepoint rollback.

## DB-SP-085
Failed savepoint rollback podrá marcar rollback-only.

## DB-SP-086
Unknown rollback podrá taintar toda la transacción.

## DB-SP-087
Database rollback-to-savepoint no restaurará automáticamente entidades.

## DB-SP-088
Database rollback-to-savepoint no restaurará automáticamente UnitOfWork.

## DB-SP-089
Database rollback-to-savepoint no restaurará automáticamente IdentityMap.

## DB-SP-090
Database rollback-to-savepoint no restaurará automáticamente snapshots ORM.

## DB-SP-091
Database rollback-to-savepoint no restaurará automáticamente collections.

## DB-SP-092
Database rollback-to-savepoint no restaurará automáticamente generated IDs.

## DB-SP-093
Database rollback-to-savepoint no restaurará automáticamente external side effects.

## DB-SP-094
Persistence checkpoint será una abstracción separada.

## DB-SP-095
ORM checkpointing no utilizará serialización arbitraria del EntityManager.

## DB-SP-096
ORM uncertainty será representada explícitamente.

## DB-SP-097
PersistenceContext podrá ser tainted después de rollback parcial.

## DB-SP-098
PersistenceContext podrá ser cleared según política.

## DB-SP-099
Cleared entities serán detached.

## DB-SP-100
Generated sequence state no se asumirá reversible.

## DB-SP-101
IdentityMap deberá reconciliar inserts revertidos.

## DB-SP-102
Relationship in-memory state no se asumirá restaurado.

## DB-SP-103
External side effects requerirán coordinación separada.

## DB-SP-104
afterCommit no será equivalente a afterSavepointRelease.

## DB-SP-105
Savepoint release no disparará physical transaction afterCommit.

## DB-SP-106
Savepoint events serán distintos de transaction commit events.

## DB-SP-107
Savepoint retry no será automático.

## DB-SP-108
Full transaction retry seguirá siendo mecanismo independiente.

## DB-SP-109
Savepoint-scoped retry requerirá política explícita.

## DB-SP-110
Savepoint-scoped retry requerirá error recuperable.

## DB-SP-111
Savepoint-scoped retry requerirá estado ORM recuperable.

## DB-SP-112
Savepoint-scoped retry requerirá side effects seguros.

## DB-SP-113
Transaction-aborting error no será tratado como savepoint-recoverable.

## DB-SP-114
Deadlock no será asumido savepoint-recoverable.

## DB-SP-115
Serialization failure no será asumido savepoint-recoverable.

## DB-SP-116
Error classification será platform-aware.

## DB-SP-117
Savepoint System consumirá clasificación de errores.

## DB-SP-118
Core no codificará SQLSTATE vendor-specific disperso.

## DB-SP-119
Resource limits se aplicarán antes de crear nuevos savepoints.

## DB-SP-120
Max depth será configurable.

## DB-SP-121
Max total savepoints será configurable.

## DB-SP-122
Diagnostics history será bounded.

## DB-SP-123
Telemetry será observational.

## DB-SP-124
Telemetry no controlará transaction semantics.

## DB-SP-125
Metrics evitarán SavepointId como label.

## DB-SP-126
Metrics evitarán TransactionId como label.

## DB-SP-127
Diagnostics podrán mostrar SavepointId.

## DB-SP-128
Diagnostics podrán mostrar depth.

## DB-SP-129
Diagnostics podrán mostrar operation outcome.

## DB-SP-130
Diagnostics deberán mostrar UNKNOWN explícitamente.

## DB-SP-131
FrankenPHP no heredará savepoints entre requests.

## DB-SP-132
RoadRunner no heredará savepoints entre operations.

## DB-SP-133
OpenSwoole coroutines no compartirán SavepointStack.

## DB-SP-134
Immutable capability profiles podrán compartirse.

## DB-SP-135
Immutable compilers podrán compartirse.

## DB-SP-136
Mutable SavepointStack no podrá compartirse.

## DB-SP-137
Mutable SavepointSequence no podrá compartirse.

## DB-SP-138
Mutable SavepointState no podrá compartirse globalmente.

## DB-SP-139
SavepointHandle será validado contra runtime context.

## DB-SP-140
Forged handles serán rechazados.

## DB-SP-141
Savepoint IDs no serán security credentials.

## DB-SP-142
Savepoint System no dependerá del ORM.

## DB-SP-143
ORM podrá integrarse mediante contratos separados.

## DB-SP-144
Savepoint System no dependerá de HTTP.

## DB-SP-145
Savepoint System no dependerá de business services.

## DB-SP-146
Savepoint System no será un segundo TransactionManager.

## DB-SP-147
Savepoint System no iniciará physical transactions.

## DB-SP-148
Savepoint System no realizará physical commit.

## DB-SP-149
Savepoint System no modificará transaction isolation.

## DB-SP-150
Cuando VoltStack no pueda demostrar el estado físico de un savepoint conservará explícitamente la incertidumbre.

---

# 185. Anti-patterns

## 185.1 Savepoint como transacción independiente

```text
SAVEPOINT
=
new transaction
```

**Rechazado.**

---

## 185.2 Raw savepoint names

```php
$tx->savepoint($_GET['name']);
```

**Rechazado.**

---

## 185.3 Vendor checks en core

```php
if ($driver === 'pgsql') {
    // savepoint behavior
}
```

**Rechazado.**

---

## 185.4 Reutilizar handle liberado

```text
release(S1)
rollbackTo(S1)
```

**Rechazado.**

---

## 185.5 Reutilizar savepoint desde otra TX

```text
TX-A → S1
TX-B → rollbackTo(S1)
```

**Rechazado.**

---

## 185.6 Restauración mágica del ORM

```text
ROLLBACK TO SAVEPOINT
→ restore all PHP objects
```

**Rechazado.**

---

## 185.7 Continuar después de UNKNOWN

```text
rollback result unknown
→ continue as if success
```

**Prohibido.**

---

## 185.8 Simular NESTED con REQUIRED

```text
savepoints unsupported
→ just join transaction
```

**Prohibido como fallback silencioso.**

---

## 185.9 Simular REQUIRES_NEW

```text
SAVEPOINT
=
REQUIRES_NEW
```

**Prohibido.**

---

## 185.10 Savepoint release como business commit

```text
release savepoint
→ send irreversible external side effects
```

**Rechazado como semántica general.**

---

# 186. Master formulas

## Savepoint ownership

```text
Owner(S)
=
PhysicalTransaction(S)
```

## Connection affinity

```text
Connection(S)
=
Connection(PhysicalTransaction(S))
```

## Lifetime

```text
Lifetime(S)
⊂
Lifetime(PhysicalTransaction(S))
```

## Nested hierarchy

```text
Parent(Sn)
=
NearestActiveAncestorSavepoint
```

cuando exista.

## Rollback effect

```text
RollbackTo(S)
=
RestoreDatabaseStateTo(S)
+
InvalidateAffectedDescendants
+
ReconcileTransactionKnowledge
```

No:

```text
RollbackTo(S)
=
RestoreEntireApplicationState
```

## Uncertainty

```text
UnknownSavepointOutcome
→
TransactionKnowledge = UNCERTAIN
```

## ORM consistency

```text
DatabaseRollbackToSavepoint
+
NoPersistenceCheckpoint
→
ORMConsistencyNotGuaranteed
```

---

# 187. Modelo final

```text
                 TransactionContext
                         │
                         ▼
                  SavepointManager
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
           Create      Rollback     Release
             │           │           │
             └───────────┼───────────┘
                         ▼
                  State Validator
                         │
                         ▼
                Capability Profile
                         │
                         ▼
                 SavepointCompiler
                         │
                         ▼
                Pinned Connection
                         │
                         ▼
                     Database
                         │
                         ▼
                Outcome Classifier
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
       SUCCESS         FAILURE         UNKNOWN
          │              │               │
          ▼              ▼               ▼
    Reconcile Stack   Error State    Taint Context
          │
          ▼
    TransactionContext
```

Con integración ORM opcional:

```text
Nested Scope
     │
     ├──────────────► Database Savepoint
     │
     └──────────────► Persistence Checkpoint
                           │
                           ▼
                  Coordinated Recovery
```

pero:

```text
Database Savepoint
≠
Persistence Checkpoint
```

---

# 188. Decisiones arquitectónicas finales

VoltStack implementará savepoints como un subsistema transaccional independiente y capability-driven.

La API pública preferirá:

```php
$savepoint = $transaction->createSavepoint();

try {
    performOperation();
} catch (\Throwable $e) {
    $transaction->rollbackTo($savepoint);
    throw $e;
}
```

sobre APIs basadas en nombres SQL arbitrarios.

Cada savepoint tendrá:

```text
SavepointId
PhysicalTransactionId
Connection Affinity
Depth
Sequence
Lifecycle State
```

y pertenecerá exclusivamente a una transacción física.

El sistema utilizará:

```text
SavepointStack
```

para mantener la jerarquía y reconciliar descendientes.

Las diferencias entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

serán modeladas mediante:

```text
SavepointCapabilityProfile
+
SavepointCompiler
```

sin introducir vendor conditionals en el core.

Se mantendrá como regla:

```text
SAVEPOINT
≠
TRANSACTION
```

y:

```text
RELEASE SAVEPOINT
≠
COMMIT
```

Además:

```text
ROLLBACK TO SAVEPOINT
≠
ROLLBACK TRANSACTION
```

La arquitectura tampoco prometerá restauración del estado ORM:

```text
DB rollback
≠
UoW rewind
≠
IdentityMap rewind
≠
PHP object rewind
```

Cuando sea necesario, un futuro:

```text
PersistenceCheckpoint
```

podrá coordinarse con el savepoint físico.

Ante un resultado incierto:

```text
UNKNOWN
```

será conservado como estado first-class y el `TransactionContext` podrá quedar:

```text
TAINTED
```

impidiendo que la aplicación continúe sobre una transacción cuyo estado ya no puede determinarse con seguridad.

La regla final será:

> **VoltStack utilizará savepoints únicamente como checkpoints físicos verificables dentro de una transacción existente; jamás los presentará como transacciones independientes ni asumirá que restauran automáticamente el estado de aplicación que existe fuera de la base de datos.**

---

# 189. Relación con el bloque Transaction & Concurrency

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

# 190. Siguiente documento

```text
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
```

El siguiente documento deberá definir en profundidad:

- transaction retry architecture;
- retry boundary;
- full-transaction replay;
- retry attempts;
- retry context;
- retry policies;
- retry budgets;
- transient vs permanent failures;
- serialization failures;
- deadlocks;
- lock timeouts;
- connection failures;
- UNKNOWN outcomes;
- retry eligibility;
- idempotency;
- retry-safe callbacks;
- exponential backoff;
- jitter;
- cancellation;
- deadlines;
- nested transaction interaction;
- `REQUIRES_NEW`;
- savepoint retry;
- UnitOfWork/EntityManager reset;
- external side effects;
- transaction callbacks;
- observability;
- telemetry;
- persistent-runtime safety;
- resource governance;
- testing;
- architectural invariants.