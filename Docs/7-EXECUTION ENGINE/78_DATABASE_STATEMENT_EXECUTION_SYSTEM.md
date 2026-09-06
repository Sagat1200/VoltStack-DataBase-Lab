# 78_DATABASE_STATEMENT_EXECUTION_SYSTEM.md

# VoltStack Quantum Database
## Statement Execution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 78 — Statement Execution System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Statement Execution System` define la infraestructura responsable de ejecutar **una operación concreta contra una conexión de base de datos**.

Su frontera conceptual es:

```text
CompiledDatabaseCommand
+
RuntimeBindingSet
+
StatementExecutionContext
        ↓
Statement Execution System
        ↓
StatementExecutionOutcome
```

Formalmente:

```text
ExecuteStatement(
    CompiledDatabaseCommand,
    RuntimeBindingSet,
    StatementExecutionContext
)
    →
StatementExecutionOutcome
```

El sistema coordina:

- adquisición de conexión;
- preparación/adquisición del statement;
- binding runtime;
- ejecución;
- obtención inicial del resultado;
- affected rows;
- generated values cuando el contrato lo permita;
- establecimiento de cursores;
- timeout;
- cancellation;
- normalización inicial de errores;
- ownership de recursos;
- cleanup.

No interpreta la query ni decide cómo ejecutarla físicamente.

---

# 2. Principio fundamental

```text
StatementExecutor
=
Single Database Statement Runtime Coordinator
```

No:

```text
StatementExecutor
=
QueryExecutor
+
SQL Compiler
+
Prepared Statement Cache
+
Parameter Converter
+
Result Hydrator
+
Transaction Manager
```

Principio:

> **A StatementExecutor executes an already-defined database command; it does not redesign, reinterpret, or recompile it.**

---

# 3. Posición arquitectónica

```text
QueryExecutor
     ↓
DatabaseExecutionUnitExecutor
     ↓
StatementExecutionRequest
     ↓
════════════════════════════════
   STATEMENT EXECUTION SYSTEM
════════════════════════════════
     ↓
StatementExecutor
     ├── Connection Lease
     ├── Prepared Statement
     ├── Parameter Binding
     ├── Driver Execution
     ├── Result Acquisition
     ├── Timeout / Cancellation
     └── Cleanup / Ownership
     ↓
Driver Statement
     ↓
Database
```

---

# 4. Relación con documentos vecinos

```text
77 Query Executor
        ↓
78 Statement Execution
        ↓
79 Prepared Statement
        ↓
80 Parameter Binding
        ↓
81 Result System
        ↓
82 Result Cursor
        ↓
83 Streaming Result
        ↓
84 Timeout / Cancellation
        ↓
85 Execution Error
        ↓
86 Execution Retry
```

Cada subsistema tiene responsabilidades distintas.

---

# 5. QueryExecutor ≠ StatementExecutor

`QueryExecutor` coordina:

```text
ExecutionPlan
├── Stage A
├── Stage B
└── Stage C
```

`StatementExecutor` coordina:

```text
one database command
```

Por tanto:

```text
QueryExecutor
≠
StatementExecutor
```

---

# 6. StatementExecutor ≠ PreparedStatement

También:

```text
StatementExecutor
=
orchestrator
```

mientras:

```text
PreparedStatement
=
prepared driver-side statement resource
```

---

# 7. StatementExecutor ≠ DriverStatement

```text
DriverStatement
```

es la abstracción runtime de bajo nivel suministrada por el driver.

El `StatementExecutor` utiliza esa abstracción.

No la reemplaza.

---

# 8. StatementExecutor ≠ SQL Compiler

El SQL ya debe estar compilado antes de esta capa.

```text
Query
 ↓
Compiler
 ↓
CompiledDatabaseCommand
 ↓
StatementExecutor
```

Nunca:

```text
StatementExecutor
 ↓
generate SQL
```

---

# 9. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Statement\Contract;

interface StatementExecutor
{
    public function execute(
        StatementExecutionRequest $request,
    ): StatementExecutionOutcome;
}
```

---

# 10. StatementExecutionRequest

```php
final readonly class StatementExecutionRequest
{
    public function __construct(
        public CompiledDatabaseCommand $command,
        public RuntimeBindingSet $bindings,
        public StatementExecutionContext $context,
    ) {}
}
```

---

# 11. Request immutability

`StatementExecutionRequest` será immutable.

El estado mutable de una ejecución concreta residirá en:

```text
StatementExecutionInstance
```

---

# 12. StatementExecutionContext

Representa el contexto explícito necesario para ejecutar el statement.

```php
final readonly class StatementExecutionContext
{
    public function __construct(
        public ConnectionRequirement $connection,
        public TransactionExecutionContext $transaction,
        public ExecutionDeadline $deadline,
        public CancellationToken $cancellation,
        public StatementExecutionPolicy $policy,
        public StatementInstrumentationContext $instrumentation,
        public ExecutionSecurityContext $security,
    ) {}
}
```

---

# 13. Contexto explícito

No deberán resolverse implícitamente:

```text
current tenant
current transaction
current connection
current request
current user
current deadline
```

mediante estado global.

---

# 14. StatementExecutionInstance

Una ejecución concreta podrá representarse mediante:

```php
final class StatementExecutionInstance
{
    public function __construct(
        private readonly StatementExecutionId $id,
        private StatementExecutionState $state,
        private readonly StatementExecutionRequest $request,
        private readonly StatementResourceLedger $resources,
    ) {}
}
```

---

# 15. Command ≠ execution instance

```text
CompiledDatabaseCommand
=
reusable immutable command description
```

```text
StatementExecutionInstance
=
one runtime execution attempt
```

---

# 16. Un mismo command

Puede ejecutarse múltiples veces:

```text
CompiledCommand C1
    ├── Execution E1
    ├── Execution E2
    ├── Execution E3
    └── Execution E4
```

con diferentes bindings.

---

# 17. Runtime bindings

Ejemplo:

```text
C1:
SELECT * FROM users WHERE id = ?
```

Ejecuciones:

```text
E1 → id = 10
E2 → id = 20
E3 → id = 30
```

El SQL compilado permanece immutable.

---

# 18. Pipeline principal

```text
StatementExecutionRequest
          ↓
Request Validation
          ↓
Compatibility Validation
          ↓
StatementExecutionInstance
          ↓
Connection Resolution
          ↓
Connection Lease Acquisition
          ↓
Transaction Affinity Validation
          ↓
Prepared Statement Acquisition
          ↓
Runtime Parameter Binding
          ↓
Pre-Execution Cancellation Check
          ↓
Timeout Configuration
          ↓
Driver Statement Execution
          ↓
Initial Result Acquisition
          ↓
Result Contract Validation
          ↓
Ownership Resolution
          ↓
Cleanup / Transfer
          ↓
StatementExecutionOutcome
```

---

# 19. Fases propuestas

```text
S0  Request Acceptance
S1  Preflight Validation
S2  Execution Instance Creation
S3  Connection Resolution
S4  Connection Lease Acquisition
S5  Session/Transaction Validation
S6  Prepared Statement Acquisition
S7  Runtime Binding
S8  Execution Controls Installation
S9  Driver Execution
S10 Result Acquisition
S11 Result Contract Validation
S12 Resource Ownership Resolution
S13 Cleanup / Transfer
S14 Outcome Finalization
```

---

# 20. S0 — Request Acceptance

Se valida estructuralmente:

```text
command exists
bindings object exists
context exists
command kind supported
execution policy valid
```

No se abre ninguna conexión.

---

# 21. S1 — Preflight Validation

Debe comprobarse:

```text
platform compatibility
driver compatibility
binding completeness
result contract compatibility
security requirements
transaction requirements
timeout state
cancellation state
prepared-statement capability
```

---

# 22. Fail before resource acquisition

Siempre que sea posible:

```text
invalid request
→ fail
```

antes de:

```text
acquire connection
prepare statement
allocate cursor
```

---

# 23. Command compatibility

Un command compilado para:

```text
PostgreSQL
```

no podrá ejecutarse sobre:

```text
MySQL connection
```

---

# 24. Compatibility identities

Podrán verificarse:

```text
PlatformIdentity
DialectIdentity
DriverCompilationContractIdentity
CapabilityFingerprint
CompiledCommandVersion
ExtensionVersionSet
```

---

# 25. Compatibility ≠ equality of everything

No toda diferencia runtime invalida un command.

La compatibilidad deberá depender sólo de propiedades relevantes para su representación/ejecución.

---

# 26. Capability validation

Ejemplo:

```text
command requires:
  prepared statements
  positional placeholders
  returning result
```

La execution target deberá satisfacer esos requisitos.

---

# 27. No runtime compiler fallback

Si no son compatibles:

```text
StatementExecutionCompatibilityException
```

No:

```text
"let's compile another SQL here"
```

---

# 28. S2 — Instance Creation

Después del preflight:

```text
StatementExecutionInstanceFactory
```

crea el runtime state.

---

# 29. Initial state

```text
CREATED
```

---

# 30. Statement state machine

Propuesta:

```text
CREATED
   ↓
VALIDATING
   ↓
ACQUIRING_CONNECTION
   ↓
PREPARING
   ↓
BINDING
   ↓
READY
   ↓
EXECUTING
   ↓
RESULT_PENDING
   ↓
RESULT_READY
   ↓
TRANSFERRING / CLEANING
   ↓
COMPLETED
```

Estados terminales alternativos:

```text
FAILED
CANCELLED
TIMED_OUT
CLOSED
```

---

# 31. State transitions

Deberán estar controladas.

No:

```php
$instance->state = $whatever;
```

desde componentes arbitrarios.

---

# 32. State transition API

```php
interface StatementExecutionStateMachine
{
    public function transition(
        StatementExecutionInstance $instance,
        StatementExecutionState $target,
    ): void;
}
```

---

# 33. Illegal transition

Ejemplo:

```text
CREATED
→ EXECUTING
```

sin conexión/preparation/binding:

```text
StatementExecutionInvariantException
```

---

# 34. S3 — Connection Resolution

El command/context expresan:

```text
ConnectionRequirement
```

No una conexión global implícita.

---

# 35. ConnectionRequirement

Puede contener:

```text
connection role
read/write intent
transaction affinity
consistency requirement
tenant routing metadata
database identity
platform requirement
session requirement
```

---

# 36. StatementExecutor no decide routing policy

La resolución se delega:

```text
StatementExecutor
       ↓
ConnectionManager
       ↓
ConnectionResolver
```

---

# 37. Read/write intent

Ejemplos:

```text
SELECT → READ
INSERT → WRITE
UPDATE → WRITE
DELETE → WRITE
```

pero la clasificación real deberá venir del command/plan.

El StatementExecutor no deberá inferirla analizando SQL text.

---

# 38. Locking SELECT

Un:

```sql
SELECT ... FOR UPDATE
```

puede requerir:

```text
WRITE-capable / primary connection
```

aunque sea sintácticamente SELECT.

Esto deberá estar expresado en `ConnectionRequirement`.

---

# 39. No SQL text inspection

Prohibido:

```php
if (str_starts_with($sql, 'SELECT')) {
    useReplica();
}
```

---

# 40. S4 — Connection Lease

La conexión deberá adquirirse mediante:

```text
ConnectionLease
```

o abstracción equivalente.

---

# 41. ConnectionLease

Representa:

```text
temporary execution ownership
```

sobre una conexión.

---

# 42. Lease ≠ connection

```text
Connection
=
database communication/session abstraction
```

```text
ConnectionLease
=
runtime ownership right over that connection
```

---

# 43. Lease ownership

Debe registrarse en:

```text
StatementResourceLedger
```

---

# 44. Existing transaction connection

Si existe una transacción activa:

```text
TransactionHandle
    ↓
bound connection
```

el StatementExecutor deberá respetarla.

---

# 45. No transaction hopping

Dentro de una transacción:

```text
statement A → connection X
statement B → connection X
```

según el contrato de transacción.

Nunca mover B a una replica arbitrariamente.

---

# 46. Session affinity

Algunos statements pueden depender de:

```text
temporary tables
session variables
transaction state
locks
prepared session state
```

El plan/context deberá expresar la affinity necesaria.

---

# 47. Connection validation

Después de adquirirla:

```text
connection platform
driver
capabilities
session profile
transaction affinity
```

deberán ser compatibles.

---

# 48. Stale connection

Si la conexión está:

```text
closed
broken
invalid
expired
```

el Connection Manager determinará si puede reemplazarla.

---

# 49. StatementExecutor no implementa reconnect policy

Eso pertenece a:

```text
Connection System
Resilience System
Retry System
```

---

# 50. S5 — Transaction Validation

Antes de preparar/ejecutar:

```text
transaction requirement
```

debe satisfacerse.

---

# 51. TransactionRequirement examples

```text
NONE
OPTIONAL
REQUIRED
REQUIRES_EXISTING
FORBIDS_TRANSACTION
EXECUTION_OWNED
CALLER_OWNED
```

---

# 52. Transaction lifecycle

El StatementExecutor puede colaborar con:

```text
TransactionManager
```

pero no implementará manualmente:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

---

# 53. Single statement transaction

Si el ExecutionPlan exige una transacción para una sola operación:

```text
QueryExecutor / TransactionCoordinator
```

deberá establecerla.

No debe esconderse arbitrariamente dentro del StatementExecutor.

---

# 54. Driver implicit transactions

Si el DBMS posee comportamiento implícito, éste deberá modelarse mediante:

```text
PlatformCapabilities
TransactionCapabilities
StatementEffects
```

---

# 55. S6 — Prepared Statement Acquisition

Flujo:

```text
CompiledDatabaseCommand
        ↓
PreparedStatementSystem
        ↓
PreparedStatementHandle
```

---

# 56. Prepared statement responsibility

Documento 79 definirá:

```text
statement preparation
statement reuse
statement cache
statement reset
statement compatibility
statement lifecycle
```

---

# 57. StatementExecutor role

Sólo solicita:

```text
"give me an executable prepared statement for this command and connection"
```

---

# 58. PreparedStatementRequest

Conceptualmente:

```php
final readonly class PreparedStatementRequest
{
    public function __construct(
        public CompiledDatabaseCommand $command,
        public ConnectionLease $connection,
        public PreparedStatementPolicy $policy,
    ) {}
}
```

---

# 59. Prepared statement result

```text
PreparedStatementLease
```

preferiblemente.

---

# 60. PreparedStatementLease

Permite distinguir:

```text
statement object
```

de:

```text
temporary execution ownership
```

---

# 61. Prepared statement cache

StatementExecutor no deberá implementar:

```php
$this->statements[$sql] ??= $pdo->prepare($sql);
```

---

# 62. Cache identity

No deberá basarse exclusivamente en SQL text.

Puede requerir:

```text
compiled command fingerprint
connection/session identity
driver mode
placeholder contract
result options
statement options
```

---

# 63. Prepared statement ≠ compiled command

```text
CompiledDatabaseCommand
=
connection-independent reusable artifact
```

```text
PreparedStatement
=
connection/driver-bound runtime resource
```

---

# 64. Persistent runtime implication

Un compiled command puede compartirse.

Un prepared statement puede no ser portable entre conexiones.

---

# 65. S7 — Runtime Binding

Después de obtener el statement:

```text
RuntimeBindingSet
      ↓
ParameterBindingSystem
      ↓
BoundPreparedStatement
```

conceptualmente.

---

# 66. Binding responsibility

Documento 80 definirá:

```text
parameter lookup
runtime conversion
driver type mapping
placeholder occurrence mapping
null handling
binary handling
large values
sensitive values
binding diagnostics
```

---

# 67. StatementExecutor no convierte tipos manualmente

No:

```php
if ($value instanceof DateTime) {
    $value = $value->format(...);
}
```

disperso dentro del executor.

---

# 68. Binding layout

El command ya contiene:

```text
CompiledBindingLayout
```

---

# 69. Runtime values

El request contiene:

```text
RuntimeBindingSet
```

---

# 70. Binding formula

```text
CompiledBindingLayout
+
RuntimeBindingSet
+
DriverBindingContract
        ↓
ParameterBindingSystem
        ↓
DriverBindings
```

---

# 71. Missing parameter

Debe fallar antes de ejecutar.

```text
MissingRuntimeBindingException
```

---

# 72. Extra parameters

La política deberá ser explícita.

Preferiblemente:

```text
unexpected binding
→ validation error
```

para evitar bugs silenciosos.

---

# 73. Sensitive parameters

Nunca deberán aparecer automáticamente en:

```text
logs
exceptions
traces
telemetry
debug toolbar
```

---

# 74. Binding failure

Un fallo de conversión/binding ocurre antes de ejecutar el statement.

Por tanto:

```text
database side effects = none
```

salvo comportamiento excepcional del driver durante prepare.

---

# 75. Execution side-effect boundary

Es importante distinguir:

```text
before execute()
```

de:

```text
after execute() was dispatched
```

---

# 76. Outcome certainty

Antes de dispatch:

```text
side effects definitely not executed
```

Después de dispatch con fallo de conexión:

```text
side effects may be unknown
```

---

# 77. S8 — Execution Controls

Antes de ejecutar deben instalarse los controles necesarios.

---

# 78. Controls

```text
deadline
statement timeout
cancellation registration
driver execution options
resource limits
fetch mode prerequisites
```

---

# 79. Deadline check

Antes de dispatch:

```text
if deadline expired
→ TIMED_OUT
```

sin enviar el statement.

---

# 80. Cancellation check

Antes de dispatch:

```text
if cancellation requested
→ CANCELLED
```

---

# 81. Timeout architecture

Documento 84 definirá el sistema completo.

StatementExecutor sólo integra el control.

---

# 82. Timeout mechanisms

Dependiendo de plataforma/driver:

```text
driver timeout
statement timeout
server-side timeout
connection-level timeout
client deadline
cancellation fallback
```

---

# 83. Timeout mechanism selection

No deberá estar hard-coded dentro del executor.

---

# 84. ExecutionControlStrategy

```php
interface StatementExecutionControlStrategy
{
    public function install(
        PreparedStatementLease $statement,
        StatementExecutionContext $context,
    ): StatementExecutionControlHandle;
}
```

---

# 85. Control cleanup

Cualquier cambio temporal de session state deberá restaurarse.

---

# 86. Example

Si para aplicar timeout se usa una propiedad session-scoped:

```text
save old state
set temporary timeout
execute
restore old state
```

deberá existir ownership y cleanup explícitos.

---

# 87. Persistent worker danger

No podrá dejarse:

```text
statement timeout from Request A
```

activo accidentalmente para:

```text
Request B
```

---

# 88. S9 — Driver Execution

Punto central:

```text
PreparedStatement
      ↓
DriverStatementExecutor
      ↓
Database
```

---

# 89. Driver contract

Conceptualmente:

```php
interface DriverStatementExecutor
{
    public function execute(
        DriverPreparedStatement $statement,
        DriverBindingSet $bindings,
        DriverExecutionOptions $options,
    ): DriverExecutionResult;
}
```

---

# 90. Driver abstraction

El StatementExecutor no deberá conocer directamente:

```text
PDOStatement
mysqli_stmt
PgSql\Result
SQLite3Stmt
```

en su core.

---

# 91. Driver adapters

Ejemplo:

```text
PDO Driver Adapter
Native MySQL Adapter
Native PostgreSQL Adapter
SQLite Adapter
Extension Driver Adapter
```

podrán implementar los contracts correspondientes.

---

# 92. Execution dispatch marker

Antes de llamar al driver deberá registrarse:

```text
EXECUTION_DISPATCH_STARTED
```

o estado equivalente.

---

# 93. Why dispatch marker matters

Permite distinguir:

```text
failed before dispatch
```

de:

```text
failed after dispatch began
```

---

# 94. Side-effect certainty

Especialmente importante para:

```text
INSERT
UPDATE
DELETE
DDL
stored procedures
```

---

# 95. ExecutionAttempt

Puede registrarse:

```php
final class StatementExecutionAttempt
{
    private bool $dispatchStarted = false;
    private bool $driverAcknowledged = false;
    private bool $resultAcquired = false;
}
```

---

# 96. Attempt ≠ retry

`StatementExecutionAttempt` representa una ejecución.

No decide si debe reintentarse.

---

# 97. Driver acknowledgement

El significado exacto dependerá del driver.

No deberá fingirse que:

```text
execute() returned
=
transaction committed
```

---

# 98. Execute success

Significa solamente que la operación alcanzó el estado de éxito definido por el driver y el statement contract.

---

# 99. Transaction commit separate

Si existe transaction:

```text
statement success
≠
transaction committed
```

---

# 100. Autocommit

Incluso con autocommit:

```text
driver statement completion
```

y:

```text
durability guarantees
```

no deben confundirse conceptualmente.

---

# 101. S10 — Result Acquisition

Después de ejecutar:

```text
DriverExecutionResult
```

debe transformarse hacia la abstracción de resultados de VoltStack.

---

# 102. Result system boundary

Documento 81 definirá:

```text
DatabaseResult
ScalarResult
AffectedRowsResult
ReturningRowsResult
NoResult
ResultMetadata
```

---

# 103. Cursor boundary

Documento 82 definirá:

```text
ResultCursor
```

---

# 104. Streaming boundary

Documento 83 definirá:

```text
StreamingResult
```

---

# 105. StatementExecutor responsibility

Sólo deberá adquirir el resultado inicial y transferirlo al sistema correcto.

---

# 106. Result kinds

Según `CompiledResultContract`:

```text
ROW_STREAM
BUFFERED_ROWS
SCALAR
AFFECTED_ROWS
RETURNING_ROWS
NO_RESULT
EXTENSION_RESULT
```

---

# 107. Result selection

No deberá decidirse mediante SQL text.

No:

```php
if (str_starts_with($sql, 'SELECT')) {
    fetchAll();
}
```

---

# 108. CompiledResultContract

Debe guiar:

```text
result acquisition strategy
```

---

# 109. SELECT

Podría producir:

```text
ROW_STREAM
```

---

# 110. UPDATE

Podría producir:

```text
AFFECTED_ROWS
```

---

# 111. UPDATE RETURNING

Podría producir:

```text
RETURNING_ROWS
```

cuando el target/plan lo soporte.

---

# 112. INSERT

Puede requerir:

```text
affected rows
generated values
returning rows
```

dependiendo del command contract.

---

# 113. Generated values

No deberá asumirse que toda inserción tiene:

```text
lastInsertId()
```

---

# 114. Generated value contract

Debe existir algo como:

```text
GeneratedValueRequirement
```

---

# 115. GeneratedValueRequirement

Ejemplos:

```text
NONE
SINGLE_ID
RETURNING_COLUMNS
DRIVER_GENERATED_KEYS
PLATFORM_SPECIFIC
```

---

# 116. No implicit lastInsertId

Prohibido:

```php
$id = $connection->lastInsertId();
```

para toda inserción.

---

# 117. Generated value semantics

Deben depender de:

```text
platform capability
compiled command
driver capability
result contract
```

---

# 118. Affected rows

También requieren contrato explícito.

---

# 119. Affected rows semantics

Diferentes motores/drivers pueden reportar:

```text
matched rows
changed rows
affected rows
```

de manera diferente.

---

# 120. NormalizedAffectedRows

VoltStack deberá definir qué semántica ofrece.

---

# 121. Driver-specific normalization

Pertenece al:

```text
Driver Result Adapter
```

no al QueryExecutor.

---

# 122. Result metadata

Podrá incluir:

```text
column count
column labels
driver types
database types
affected rows
generated value metadata
cursor capability
result-set count
```

---

# 123. Multiple result sets

Deberán ser una capability explícita.

---

# 124. No hidden multi-result behavior

Si un command puede producir múltiples result sets:

```text
CompiledResultContract
```

debe indicarlo.

---

# 125. Multi-statement SQL

No debe confundirse con multiple result sets.

VoltStack no deberá generar stacked multi-statements por defecto.

---

# 126. S11 — Result Contract Validation

Después de adquirir el resultado:

```text
ActualDriverResult
```

deberá satisfacer:

```text
CompiledResultContract
```

---

# 127. Example mismatch

Esperado:

```text
AFFECTED_ROWS
```

Recibido:

```text
ROW_CURSOR
```

sin adaptación definida:

```text
StatementResultContractException
```

---

# 128. Validation level

No significa inspeccionar todas las filas.

Significa verificar el contrato estructural disponible.

---

# 129. Streaming result validation

No puede validarse completamente la ejecución futura del cursor antes de consumirlo.

Por eso se distingue:

```text
cursor establishment success
```

de:

```text
cursor consumption success
```

---

# 130. Cursor handoff

```text
StatementExecutor
     ↓
ResultCursor
     ↓
StreamingResult
     ↓
Caller
```

---

# 131. S12 — Resource Ownership Resolution

Antes de terminar, debe decidirse qué recursos:

```text
release
transfer
retain temporarily
```

---

# 132. Resource categories

```text
ConnectionLease
PreparedStatementLease
ResultCursor
ExecutionControlHandle
TemporarySessionState
DriverResultHandle
ExtensionResource
```

---

# 133. Buffered result

Para:

```text
BUFFERED_ROWS
```

puede ser posible:

```text
fetch all
release cursor
release statement
release connection
return buffered result
```

---

# 134. Streaming result

Para:

```text
ROW_STREAM
```

puede requerirse:

```text
cursor
statement
connection
```

hasta que termine el stream.

---

# 135. Ownership transfer graph

```text
StatementExecutor
      │
      └── transfer
             ↓
        StreamingResult
             │
             ├── ResultCursor
             ├── PreparedStatementLease
             └── ConnectionLease
```

---

# 136. Composite ownership

Podrá utilizarse:

```text
StreamingStatementResourceOwner
```

para agrupar recursos dependientes.

---

# 137. Resource release ordering

Generalmente:

```text
Cursor
   ↓
PreparedStatement
   ↓
ConnectionLease
```

aunque el driver contract podrá especializarlo.

---

# 138. No premature release

No devolver una conexión al pool mientras un cursor dependiente siga activo.

---

# 139. No double release

Una vez transferido un recurso:

```text
StatementExecutor
```

deja de ser su owner.

---

# 140. Ownership ledger

```php
interface StatementResourceLedger
{
    public function acquired(ResourceHandle $resource): void;

    public function released(ResourceHandle $resource): void;

    public function transferred(
        ResourceHandle $resource,
        ResourceOwnerId $newOwner,
    ): void;
}
```

---

# 141. Conservation invariant

Al cerrar el execution scope:

```text
Acquired
=
Released
∪
Transferred
```

---

# 142. Exclusive disposition

```text
Released
∩
Transferred
=
∅
```

---

# 143. S13 — Cleanup

Cleanup debe ejecutarse para:

```text
SUCCESS
FAILURE
CANCELLATION
TIMEOUT
PARTIAL_PREPARATION
BINDING_FAILURE
RESULT_FAILURE
```

---

# 144. Cleanup coordinator

```php
interface StatementCleanupCoordinator
{
    public function cleanup(
        StatementExecutionInstance $instance,
        StatementCleanupReason $reason,
    ): StatementCleanupOutcome;
}
```

---

# 145. Partial initialization

Ejemplo:

```text
connection acquired
statement preparation fails
```

Debe liberarse:

```text
connection lease
```

---

# 146. Another partial case

```text
connection acquired
statement prepared
binding fails
```

Debe limpiarse:

```text
prepared statement execution state
connection lease
temporary session controls
```

---

# 147. Execute failure

```text
execute()
throws
```

puede requerir:

```text
statement reset
cursor cleanup
connection health evaluation
transaction state evaluation
```

---

# 148. Connection health after error

El StatementExecutor no deberá asumir:

```text
all errors leave connection healthy
```

ni:

```text
all errors break connection
```

---

# 149. Driver health classification

El Driver/Connection layer deberá indicar:

```text
HEALTHY
SUSPECT
BROKEN
UNKNOWN
```

o modelo equivalente.

---

# 150. Pool return

Sólo conexiones aptas deberán regresar al pool.

---

# 151. Cleanup failure

No deberá ocultar el primary execution error.

---

# 152. Example

```text
Primary:
  DeadlockException

Cleanup:
  StatementCloseException
```

Resultado:

```text
primary failure = deadlock
suppressed failure = close failure
```

---

# 153. Cleanup idempotence

Siempre que sea posible:

```text
close(close(resource))
```

no deberá causar corrupción.

---

# 154. Statement reset

Un prepared statement reusable puede requerir:

```text
reset
clear bindings
close cursor
clear driver result
restore options
```

antes de volver al pool/cache.

---

# 155. Reset belongs Prepared Statement System

StatementExecutor solicita el reset.

No deberá conocer detalles internos del driver.

---

# 156. S14 — Outcome Finalization

Resultado:

```php
final readonly class StatementExecutionOutcome
{
    public function __construct(
        public StatementExecutionStatus $status,
        public ?DatabaseResult $result,
        public StatementExecutionMetadata $metadata,
        public ?ExecutionFailure $failure,
    ) {}
}
```

---

# 157. Status

```php
enum StatementExecutionStatus
{
    case SUCCESS;
    case FAILURE;
    case CANCELLED;
    case TIMED_OUT;
    case OUTCOME_UNKNOWN;
}
```

---

# 158. OUTCOME_UNKNOWN

Es esencial para operaciones con side effects.

---

# 159. Example unknown outcome

```text
INSERT dispatched
     ↓
connection lost
     ↓
client receives no acknowledgement
```

No puede saberse automáticamente si:

```text
INSERT committed
```

---

# 160. Dangerous simplification

Nunca transformar automáticamente:

```text
connection lost after write
```

en:

```text
safe to retry
```

---

# 161. Retry system

Documento 86 decidirá, utilizando:

```text
statement kind
transaction state
idempotency
dispatch state
error classification
outcome certainty
retry policy
```

---

# 162. Execution metadata

Puede contener:

```text
execution id
command fingerprint
connection identity
driver identity
statement identity
start/end monotonic timing
affected rows
result kind
dispatch status
outcome certainty
resource disposition
retry eligibility metadata
```

---

# 163. Timing

Para duración deberá preferirse:

```text
monotonic clock
```

sobre wall clock.

---

# 164. Clock abstraction

```php
interface MonotonicClock
{
    public function now(): MonotonicTimestamp;
}
```

---

# 165. Deterministic planning vs runtime timing

Runtime timing obviamente varía.

No deberá formar parte de:

```text
CompiledQueryFingerprint
```

---

# 166. Instrumentation

Eventos sugeridos:

```text
StatementExecutionStarted
ConnectionResolutionStarted
ConnectionLeaseAcquired
PreparedStatementAcquisitionStarted
PreparedStatementAcquired
ParameterBindingStarted
ParameterBindingCompleted
StatementDispatchStarted
StatementDispatchCompleted
ResultAcquisitionStarted
ResultAcquired
StatementResourcesTransferred
StatementCleanupStarted
StatementCleanupCompleted
StatementExecutionCompleted
StatementExecutionFailed
StatementExecutionCancelled
StatementExecutionTimedOut
```

---

# 167. Instrumentation payload

Preferir:

```text
IDs
fingerprints
durations
counts
types
statuses
```

No valores sensibles.

---

# 168. SQL telemetry

El SQL puede ser registrado según política.

Pero:

```text
SQL
+
bindings
```

no deberán combinarse reconstruyendo una query con secretos.

---

# 169. Safe query display

Preferible:

```sql
SELECT * FROM `users` WHERE `email` = ?
```

con:

```text
parameter_count = 1
```

---

# 170. Binding telemetry

Podrá registrar:

```text
parameter type
null/non-null
size category
sensitivity classification
```

sin valor.

---

# 171. Sensitive value rule

```text
sensitive runtime binding
→ never logged by default
```

---

# 172. Execution security

El StatementExecutor constituye una frontera importante.

Debe verificar que:

```text
command trust level
connection permissions
raw SQL requirements
dangerous command requirements
tenant execution metadata
```

sean compatibles.

---

# 173. No authorization re-evaluation

No ejecutará policies de aplicación.

Sólo comprobará execution requirements ya resueltos.

---

# 174. Raw SQL

Raw SQL puede llegar como `CompiledDatabaseCommand`.

Pero deberá conservar:

```text
RawSqlTrustMetadata
```

---

# 175. Multi-statement protection

El core deberá preferir:

```text
one CompiledDatabaseCommand
=
one database statement
```

---

# 176. Semicolon

El canonical SQL normalmente no requiere terminador `;`.

---

# 177. Stacked statements

No deberán habilitarse implícitamente.

---

# 178. Dangerous database commands

Operaciones como:

```text
LOAD DATA
INTO OUTFILE
COPY PROGRAM
administrative commands
filesystem-related commands
```

si alguna plataforma las soporta, deberán tener:

```text
explicit command kind
explicit capability
explicit security policy
```

No deberán pasar como queries ordinarias sin clasificación.

---

# 179. Statement side-effect model

Cada command deberá declarar:

```text
StatementEffectProfile
```

---

# 180. Possible effects

```text
READ_ONLY
DATA_WRITE
SCHEMA_WRITE
SESSION_MUTATION
TRANSACTION_MUTATION
LOCK_ACQUISITION
EXTERNAL_EFFECT
UNKNOWN
```

---

# 181. Multiple effects

Un statement puede poseer varios efectos.

---

# 182. Effect profile usage

Sirve para:

```text
routing
retry safety
cleanup
session reset
telemetry
security
transaction requirements
```

---

# 183. Effect profile not inferred from SQL string

Debe ser compilado/estructurado.

---

# 184. Session mutation

Statements que modifican session state requieren especial atención bajo persistent workers.

---

# 185. Session state invariant

Después de liberar una conexión reusable:

```text
SessionState
=
PoolBaselineState
```

salvo estado expresamente permitido.

---

# 186. Session reset system

La restauración real pertenece a:

```text
Connection State and Reset System
```

El StatementExecutor solicita/verifica la acción necesaria.

---

# 187. Temporary statement controls

Timeouts, fetch modes u opciones temporales también deberán limpiarse.

---

# 188. Connection poisoning

Si no puede garantizarse el reset:

```text
connection
→ discard
```

en lugar de regresar una conexión contaminada al pool.

---

# 189. Persistent runtime model

VoltStack debe funcionar correctamente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 190. Shared safe components

Podrán compartirse:

```text
StatementExecutor
frozen registries
immutable driver descriptors
immutable policies
compiled commands
stateless validators
```

---

# 191. Operation-local components

Deberán ser locales:

```text
StatementExecutionInstance
runtime bindings
connection lease
prepared statement lease
cursor
driver result
timeout handle
cancellation registration
resource ledger
failure state
```

---

# 192. No current statement singleton

Prohibido:

```php
$this->currentStatement = $statement;
```

en un singleton compartido.

---

# 193. Concurrent executions

Debe soportarse:

```text
Execution A
  Statement A

Execution B
  Statement B
```

simultáneamente en runtimes concurrentes.

---

# 194. Connection concurrency

Una misma conexión sólo podrá usarse concurrentemente si:

```text
DriverCapability
+
ConnectionCapability
+
TransactionContract
```

lo permiten explícitamente.

---

# 195. Default conservative rule

Preferible:

```text
one active statement execution
per non-multiplexable connection
```

---

# 196. Cursor conflict

Algunos drivers no permiten:

```text
active streaming cursor
+
another statement
```

en la misma conexión.

Esto deberá expresarse como capability.

---

# 197. Connection busy state

El Connection System podrá manejar:

```text
IDLE
LEASED
EXECUTING
STREAMING
TRANSACTION_BOUND
BROKEN
```

o modelo equivalente.

---

# 198. StatementExecutor respects connection state

No deberá saltarse ese state machine.

---

# 199. Prepared statement concurrency

Un mismo prepared statement handle no deberá compartirse concurrentemente salvo capability explícita.

---

# 200. Default statement ownership

```text
one PreparedStatementLease
→ one active execution
```

---

# 201. Statement reuse

Reuse significa:

```text
same prepared resource
used sequentially
```

No necesariamente simultáneamente.

---

# 202. Rebinding

Antes de reutilizar:

```text
old bindings
```

deberán limpiarse según driver contract.

---

# 203. Result draining

Algunos drivers requieren consumir/cerrar resultados antes de reutilizar statement/connection.

---

# 204. Drain policy

Debe modelarse explícitamente:

```text
CLOSE
DRAIN
DISCARD_CONNECTION
DRIVER_RESET
```

según capabilities.

---

# 205. Statement execution policy

```php
final readonly class StatementExecutionPolicy
{
    public function __construct(
        public PreparedStatementPolicy $preparation,
        public ResultAcquisitionPolicy $result,
        public ResourceReleasePolicy $resources,
        public ExecutionControlPolicy $controls,
    ) {}
}
```

---

# 206. Policy ≠ semantics

Una policy puede controlar:

```text
resource reuse
instrumentation
buffering when already allowed
prepared statement reuse
```

pero no cambiar el significado de la query.

---

# 207. No policy semantic mutation

No:

```text
timeout disabled
→ remove locking clause
```

No:

```text
stream inconvenient
→ add LIMIT
```

---

# 208. Driver exception boundary

Todas las excepciones del driver deberán cruzar una capa de adaptación.

---

# 209. No PDO leakage

La API pública no deberá obligar al usuario a manejar:

```text
PDOException
```

como abstracción principal de VoltStack.

---

# 210. Preserve native cause

Sin embargo, la excepción nativa podrá conservarse como:

```text
previous/cause
```

para diagnóstico.

---

# 211. ExecutionFailure

Modelo estructurado:

```php
final readonly class ExecutionFailure
{
    public function __construct(
        public ExecutionFailureCategory $category,
        public ExecutionFailureCode $code,
        public OutcomeCertainty $certainty,
        public RetryClassification $retry,
        public ConnectionHealthEffect $connectionHealth,
        public Throwable $cause,
    ) {}
}
```

---

# 212. Error normalization

La clasificación completa se define en documento 85.

---

# 213. Outcome certainty

Propuesta:

```text
NOT_DISPATCHED
DISPATCHED_NO_EFFECT_CONFIRMED
EFFECT_CONFIRMED
OUTCOME_UNKNOWN
```

---

# 214. Read query certainty

Para SELECT, unknown side effect puede ser menos grave, pero:

```text
locks
session effects
volatile functions
```

pueden complicarlo.

---

# 215. Do not infer idempotency from SELECT keyword

Debe utilizarse metadata estructurada.

---

# 216. Statement idempotency

Puede declararse:

```text
IDEMPOTENT
CONDITIONALLY_IDEMPOTENT
NON_IDEMPOTENT
UNKNOWN
```

---

# 217. Idempotency ≠ read-only

Una operación write puede ser idempotente bajo ciertas condiciones.

Una operación aparentemente read puede producir efectos mediante funciones/procedures.

---

# 218. Retry metadata

StatementExecutor recopila hechos.

RetrySystem decide.

---

# 219. Cancellation race

Puede ocurrir:

```text
execute succeeds
cancellation arrives
```

casi simultáneamente.

---

# 220. Terminal outcome ordering

Debe existir una política definida para resolver:

```text
execution completion
vs
cancellation observation
```

---

# 221. Completion-before-cancellation

Si el resultado fue confirmado antes de observar cancellation:

```text
SUCCESS
```

puede ser correcto.

---

# 222. Cancellation-before-dispatch

```text
CANCELLED
```

sin side effects.

---

# 223. Cancellation-during-dispatch

Puede terminar como:

```text
CANCELLED
OUTCOME_UNKNOWN
FAILURE
SUCCESS
```

dependiendo del driver y de lo observable.

---

# 224. Timeout race

Misma consideración para timeout.

---

# 225. State transition synchronization

En runtimes concurrentes deberá evitarse que dos terminal states ganen simultáneamente.

---

# 226. Terminal state invariant

Exactamente un estado terminal principal:

```text
SUCCESS
FAILURE
CANCELLED
TIMED_OUT
OUTCOME_UNKNOWN
```

---

# 227. Atomic terminalization

Conceptualmente:

```text
compare-and-set terminal outcome
```

o mecanismo runtime apropiado.

---

# 228. No thread-specific assumption

El core no deberá requerir mutex OS si el runtime puede ofrecer otra primitiva.

---

# 229. Runtime adapter

Podrá existir:

```text
ExecutionSynchronizationPrimitive
```

abstracta.

---

# 230. StatementExecutionBudget

El sistema deberá aceptar límites.

---

# 231. Budget dimensions

```text
max binding count
max binding bytes
max result metadata
max generated keys
max result sets
max execution duration
max cursor lifetime policy
max driver diagnostic size
```

---

# 232. SQL size budget

El SQL ya fue limitado durante compilation.

Statement execution puede volver a aplicar un defensive limit.

---

# 233. Parameter count

Debe verificarse contra:

```text
CompiledBindingLayout
DriverCapabilities
PlatformCapabilities
```

---

# 234. Large objects

LOBs requieren tratamiento especial.

---

# 235. LOB binding

Puede ser:

```text
BUFFERED
STREAMED
DRIVER_NATIVE
```

---

# 236. LOB resource ownership

Un input stream utilizado para binding deberá tener ownership definido.

---

# 237. Caller-owned input stream

StatementExecutor no deberá cerrarlo automáticamente salvo contract.

---

# 238. Execution-owned input stream

Deberá cerrarse durante cleanup.

---

# 239. Output LOB

Puede mantener dependencias con:

```text
cursor
statement
connection
```

y por ello afecta ownership transfer.

---

# 240. Batch execution

Batch statements no deberán esconderse dentro del executor normal.

---

# 241. Batch contract

Si se soporta:

```text
BatchStatementExecutionRequest
```

será explícito.

---

# 242. One request ≠ hidden batch

Un `StatementExecutionRequest` normal representa una ejecución lógica definida.

---

# 243. Multi-row INSERT

No es necesariamente batch.

Puede ser un único SQL statement:

```sql
INSERT INTO ... VALUES (...), (...), (...)
```

---

# 244. Batch ≠ multi-row statement

Distinción obligatoria.

---

# 245. Stored procedures

Si se soportan deberán declarar:

```text
input parameters
output parameters
result sets
side effects
transaction behavior
```

---

# 246. OUT parameters

No deberán modelarse como ordinary result rows.

---

# 247. Statement kinds

Propuesta:

```text
QUERY
MUTATION
DDL
PROCEDURE
UTILITY
ADMINISTRATIVE
EXTENSION
```

---

# 248. V1 focus

Principalmente:

```text
QUERY
MUTATION
```

---

# 249. DDL

El mismo execution infrastructure podrá reutilizarse posteriormente.

---

# 250. Explain

Un:

```text
EXPLAIN SELECT ...
```

es otro command explícitamente compilado.

StatementExecutor no ejecuta EXPLAIN automáticamente.

---

# 251. Query profiling

No:

```text
execute query
then automatically EXPLAIN it
```

en esta capa.

---

# 252. Extension system

Podrán registrarse:

```text
StatementExecutionMiddleware
StatementExecutionObserver
DriverExecutionAdapter
ResultAdapter
ExecutionControlProvider
```

pero con contratos estrictos.

---

# 253. Middleware danger

No debe permitirse que middleware arbitrario:

```text
replace SQL
change bindings
reroute transaction
retry writes
```

sin contratos especializados.

---

# 254. Preferred extension model

Preferir:

```text
typed hooks
```

sobre middleware universal.

---

# 255. Observer hooks

```text
beforeConnectionAcquire
afterConnectionAcquire
beforePrepare
afterPrepare
beforeBind
afterBind
beforeDispatch
afterDispatch
beforeResultAcquire
afterResultAcquire
beforeCleanup
afterCleanup
```

---

# 256. Observer non-interference

Observers no modifican el execution.

---

# 257. Interceptors

Si en el futuro existen interceptors mutables, deberán ser:

```text
explicit
typed
ordered
security-reviewed
fingerprinted where relevant
```

---

# 258. No runtime registry mutation

Registries deberán estar frozen después del bootstrap.

---

# 259. Registry conflict

Ambigüedad:

```text
two result adapters
support same driver result
with same priority
```

deberá fallar durante bootstrap.

---

# 260. No last-wins

Regla general de VoltStack Database.

---

# 261. Directory structure

Propuesta:

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Statement/
                ├── Contract/
                │   ├── StatementExecutor.php
                │   ├── DriverStatementExecutor.php
                │   ├── StatementExecutionStateMachine.php
                │   └── StatementCleanupCoordinator.php
                │
                ├── Executor/
                │   └── DefaultStatementExecutor.php
                │
                ├── Request/
                │   └── StatementExecutionRequest.php
                │
                ├── Context/
                │   ├── StatementExecutionContext.php
                │   └── StatementExecutionPolicy.php
                │
                ├── Instance/
                │   ├── StatementExecutionInstance.php
                │   ├── StatementExecutionInstanceFactory.php
                │   ├── StatementExecutionState.php
                │   └── StatementExecutionAttempt.php
                │
                ├── Validation/
                │   ├── StatementRequestValidator.php
                │   ├── StatementCompatibilityValidator.php
                │   ├── StatementSecurityValidator.php
                │   └── StatementResultContractValidator.php
                │
                ├── Connection/
                │   ├── StatementConnectionCoordinator.php
                │   └── StatementConnectionRequirement.php
                │
                ├── Preparation/
                │   └── PreparedStatementCoordinator.php
                │
                ├── Binding/
                │   └── StatementBindingCoordinator.php
                │
                ├── Control/
                │   ├── StatementExecutionControlStrategy.php
                │   ├── StatementExecutionControlHandle.php
                │   └── StatementExecutionControlCoordinator.php
                │
                ├── Dispatch/
                │   ├── StatementDispatchCoordinator.php
                │   └── StatementDispatchState.php
                │
                ├── Result/
                │   ├── StatementResultAcquirer.php
                │   ├── StatementResultAdapter.php
                │   └── GeneratedValueAcquirer.php
                │
                ├── Resource/
                │   ├── StatementResourceLedger.php
                │   ├── StatementResourceCoordinator.php
                │   └── StatementResourceDisposition.php
                │
                ├── Cleanup/
                │   ├── DefaultStatementCleanupCoordinator.php
                │   ├── StatementCleanupReason.php
                │   └── StatementCleanupOutcome.php
                │
                ├── Outcome/
                │   ├── StatementExecutionOutcome.php
                │   ├── StatementExecutionStatus.php
                │   ├── StatementExecutionMetadata.php
                │   └── OutcomeCertainty.php
                │
                ├── Effect/
                │   ├── StatementEffectProfile.php
                │   └── StatementEffect.php
                │
                ├── Instrumentation/
                │   ├── StatementExecutionObserver.php
                │   └── StatementExecutionTrace.php
                │
                ├── Registry/
                │   ├── StatementResultAdapterRegistry.php
                │   └── StatementExecutionExtensionRegistry.php
                │
                └── Exception/
                    ├── StatementExecutionException.php
                    ├── StatementExecutionValidationException.php
                    ├── StatementExecutionCompatibilityException.php
                    ├── StatementConnectionException.php
                    ├── StatementPreparationException.php
                    ├── StatementBindingException.php
                    ├── StatementDispatchException.php
                    ├── StatementResultException.php
                    ├── StatementResultContractException.php
                    ├── StatementResourceException.php
                    ├── StatementCleanupException.php
                    └── StatementExecutionInvariantException.php
```

---

# 262. Dependency direction

```text
QueryExecutor
      ↓
Statement Execution
      ↓
Prepared Statement
      ↓
Parameter Binding
      ↓
Driver Statement
      ↓
Connection
      ↓
Driver
```

El Result System es consumido como abstracción de salida:

```text
Statement Execution
      ↓
Result System
```

---

# 263. Forbidden dependencies

Prohibido:

```text
StatementExecutor
→ QueryOptimizer

StatementExecutor
→ QueryPlanner

StatementExecutor
→ EntityManager

StatementExecutor
→ UnitOfWork

StatementExecutor
→ IdentityMap

StatementExecutor
→ Repository
```

---

# 264. SQL compiler dependency

Durante ejecución normal:

```text
StatementExecutor
```

no deberá necesitar:

```text
SqlCompiler
```

El command ya está compilado.

---

# 265. Lazy compilation boundary

Si alguna API superior soporta lazy compilation:

```text
Query Execution Coordinator
       ↓
Compilation
       ↓
Statement Execution
```

La frontera seguirá siendo explícita.

---

# 266. Architectural invariants

## DB-STMTEXEC-001

StatementExecutor ejecutará un database command ya definido.

## DB-STMTEXEC-002

StatementExecutor no interpretará Query AST.

## DB-STMTEXEC-003

StatementExecutor no realizará semantic analysis.

## DB-STMTEXEC-004

StatementExecutor no optimizará queries.

## DB-STMTEXEC-005

StatementExecutor no realizará physical planning.

## DB-STMTEXEC-006

StatementExecutor no generará SQL.

## DB-STMTEXEC-007

StatementExecutor no hidratará entities.

## DB-STMTEXEC-008

StatementExecutor será distinto de QueryExecutor.

## DB-STMTEXEC-009

StatementExecutor será distinto de PreparedStatement.

## DB-STMTEXEC-010

StatementExecutor será distinto de DriverStatement.

## DB-STMTEXEC-011

StatementExecutionRequest será immutable.

## DB-STMTEXEC-012

StatementExecutionInstance será operation-scoped.

## DB-STMTEXEC-013

Runtime state no será almacenado en CompiledDatabaseCommand.

## DB-STMTEXEC-014

El mismo command podrá ejecutarse múltiples veces.

## DB-STMTEXEC-015

Bindings runtime no modificarán el command.

## DB-STMTEXEC-016

Preflight ocurrirá antes de adquirir recursos cuando sea posible.

## DB-STMTEXEC-017

Platform compatibility será validada.

## DB-STMTEXEC-018

Driver compatibility será validada.

## DB-STMTEXEC-019

Capability compatibility será validada.

## DB-STMTEXEC-020

StatementExecutor no recompilará commands incompatibles.

## DB-STMTEXEC-021

Statement state transitions serán controladas.

## DB-STMTEXEC-022

Illegal state transitions fallarán.

## DB-STMTEXEC-023

ConnectionRequirement será explícito.

## DB-STMTEXEC-024

Routing no se inferirá del SQL text.

## DB-STMTEXEC-025

SELECT no implicará automáticamente replica.

## DB-STMTEXEC-026

Locking requirements serán preservados.

## DB-STMTEXEC-027

Connection acquisition utilizará ownership explícito.

## DB-STMTEXEC-028

Transaction affinity será preservada.

## DB-STMTEXEC-029

Session affinity será preservada.

## DB-STMTEXEC-030

StatementExecutor no implementará reconnect policy.

## DB-STMTEXEC-031

Transaction requirements serán verificadas.

## DB-STMTEXEC-032

StatementExecutor no implementará BEGIN directamente.

## DB-STMTEXEC-033

StatementExecutor no implementará COMMIT directamente.

## DB-STMTEXEC-034

StatementExecutor no implementará ROLLBACK directamente.

## DB-STMTEXEC-035

Prepared statement acquisition se delegará al Prepared Statement System.

## DB-STMTEXEC-036

Prepared statement cache no vivirá dentro de StatementExecutor.

## DB-STMTEXEC-037

PreparedStatement será connection/driver-bound.

## DB-STMTEXEC-038

CompiledDatabaseCommand será conceptualmente reusable.

## DB-STMTEXEC-039

Runtime binding se delegará al Parameter Binding System.

## DB-STMTEXEC-040

Type conversion no estará dispersa en StatementExecutor.

## DB-STMTEXEC-041

Missing bindings fallarán antes de dispatch.

## DB-STMTEXEC-042

Sensitive bindings no serán registrados por defecto.

## DB-STMTEXEC-043

Pre-dispatch failure no tendrá database side effects.

## DB-STMTEXEC-044

Post-dispatch failure podrá tener outcome desconocido.

## DB-STMTEXEC-045

Cancellation será comprobada antes de dispatch.

## DB-STMTEXEC-046

Deadline será comprobado antes de dispatch.

## DB-STMTEXEC-047

Execution controls tendrán cleanup explícito.

## DB-STMTEXEC-048

Session-scoped timeout state será restaurado.

## DB-STMTEXEC-049

Driver execution se realizará mediante contract.

## DB-STMTEXEC-050

Core no dependerá de PDOStatement.

## DB-STMTEXEC-051

Dispatch start será observable internamente.

## DB-STMTEXEC-052

StatementExecutionAttempt será distinto de retry attempt policy.

## DB-STMTEXEC-053

Statement success no implicará transaction commit.

## DB-STMTEXEC-054

Statement success no implicará durability universal.

## DB-STMTEXEC-055

Result acquisition seguirá CompiledResultContract.

## DB-STMTEXEC-056

Result kind no se inferirá desde SQL text.

## DB-STMTEXEC-057

Generated values requerirán contrato explícito.

## DB-STMTEXEC-058

lastInsertId no será asumido universalmente.

## DB-STMTEXEC-059

Affected-row semantics serán normalizadas explícitamente.

## DB-STMTEXEC-060

Multiple result sets serán capability explícita.

## DB-STMTEXEC-061

Multiple result sets serán distintos de multi-statements.

## DB-STMTEXEC-062

Actual result deberá satisfacer result contract.

## DB-STMTEXEC-063

Cursor establishment será distinto de cursor consumption.

## DB-STMTEXEC-064

Streaming result transferirá ownership explícitamente.

## DB-STMTEXEC-065

Buffered results podrán liberar recursos tempranamente.

## DB-STMTEXEC-066

Streaming results no liberarán recursos prematuramente.

## DB-STMTEXEC-067

Todo recurso tendrá owner.

## DB-STMTEXEC-068

Todo recurso adquirido será released o transferred.

## DB-STMTEXEC-069

Un recurso no podrá ser simultaneously released y transferred.

## DB-STMTEXEC-070

Cleanup ocurrirá en success.

## DB-STMTEXEC-071

Cleanup ocurrirá en failure.

## DB-STMTEXEC-072

Cleanup ocurrirá en cancellation.

## DB-STMTEXEC-073

Cleanup ocurrirá en timeout.

## DB-STMTEXEC-074

Partial initialization será limpiada.

## DB-STMTEXEC-075

Connection health después de error será clasificada.

## DB-STMTEXEC-076

Broken connection no regresará normalmente al pool.

## DB-STMTEXEC-077

Cleanup failure no ocultará primary failure.

## DB-STMTEXEC-078

Statement reset se delegará al Prepared Statement System.

## DB-STMTEXEC-079

Outcome status será estructurado.

## DB-STMTEXEC-080

OUTCOME_UNKNOWN será representable.

## DB-STMTEXEC-081

Unknown write outcome no será tratado como safe retry automáticamente.

## DB-STMTEXEC-082

Runtime metadata no formará parte del compiled fingerprint.

## DB-STMTEXEC-083

Monotonic timing se preferirá para durations.

## DB-STMTEXEC-084

Instrumentation será observacional.

## DB-STMTEXEC-085

Sensitive bindings no aparecerán en telemetry por defecto.

## DB-STMTEXEC-086

SQL y bindings no se concatenarán para logging.

## DB-STMTEXEC-087

Execution security requirements serán validadas.

## DB-STMTEXEC-088

StatementExecutor no ejecutará authorization policies.

## DB-STMTEXEC-089

Raw SQL conservará trust metadata.

## DB-STMTEXEC-090

Un command normal representará un statement.

## DB-STMTEXEC-091

Stacked statements no se habilitarán implícitamente.

## DB-STMTEXEC-092

Dangerous commands tendrán clasificación explícita.

## DB-STMTEXEC-093

Statement effects serán metadata estructurada.

## DB-STMTEXEC-094

Statement effects no se inferirán mediante string matching.

## DB-STMTEXEC-095

Session mutation deberá resetearse antes de pool return.

## DB-STMTEXEC-096

Unresettable contaminated connection deberá descartarse.

## DB-STMTEXEC-097

Shared StatementExecutor será stateless respecto a ejecución actual.

## DB-STMTEXEC-098

StatementExecutionInstance será execution-local.

## DB-STMTEXEC-099

Runtime bindings serán execution-local.

## DB-STMTEXEC-100

ConnectionLease será execution-local salvo transfer.

## DB-STMTEXEC-101

PreparedStatementLease será execution-local salvo transfer.

## DB-STMTEXEC-102

Cursor será execution-local salvo transfer.

## DB-STMTEXEC-103

ExecutionControlHandle será execution-local.

## DB-STMTEXEC-104

Failure state será execution-local.

## DB-STMTEXEC-105

FrankenPHP no filtrará statement state entre requests.

## DB-STMTEXEC-106

RoadRunner no filtrará statement state entre jobs.

## DB-STMTEXEC-107

OpenSwoole soportará aislamiento concurrente.

## DB-STMTEXEC-108

Connection concurrent use requerirá capability explícita.

## DB-STMTEXEC-109

Prepared statement concurrent use requerirá capability explícita.

## DB-STMTEXEC-110

Default prepared statement execution será exclusive.

## DB-STMTEXEC-111

Statement reuse no implicará concurrent reuse.

## DB-STMTEXEC-112

Old bindings serán limpiados antes del reuse cuando sea necesario.

## DB-STMTEXEC-113

Pending results serán cerrados/drenados antes del reuse cuando sea necesario.

## DB-STMTEXEC-114

Drain policy será capability-driven.

## DB-STMTEXEC-115

Execution policy no cambiará query semantics.

## DB-STMTEXEC-116

Driver-native exceptions serán adaptadas.

## DB-STMTEXEC-117

Native exception podrá preservarse como cause.

## DB-STMTEXEC-118

Outcome certainty será explícita.

## DB-STMTEXEC-119

Idempotency será distinta de read-only.

## DB-STMTEXEC-120

Retry eligibility será distinta de retry decision.

## DB-STMTEXEC-121

Cancellation race tendrá resolución definida.

## DB-STMTEXEC-122

Timeout race tendrá resolución definida.

## DB-STMTEXEC-123

Sólo un terminal outcome principal será seleccionado.

## DB-STMTEXEC-124

Terminalization será concurrency-safe.

## DB-STMTEXEC-125

Core no asumirá una primitiva específica de threading.

## DB-STMTEXEC-126

Execution budget será bounded.

## DB-STMTEXEC-127

Budget exhaustion no producirá partial success implícito.

## DB-STMTEXEC-128

LOB ownership será explícito.

## DB-STMTEXEC-129

Caller-owned streams no se cerrarán sin contract.

## DB-STMTEXEC-130

Execution-owned streams se limpiarán.

## DB-STMTEXEC-131

Output LOB podrá transferir resource ownership.

## DB-STMTEXEC-132

Batch execution será explícita.

## DB-STMTEXEC-133

Multi-row INSERT será distinto de batch.

## DB-STMTEXEC-134

Stored procedure output parameters serán explícitos.

## DB-STMTEXEC-135

Statement kinds serán tipados.

## DB-STMTEXEC-136

EXPLAIN será command explícito.

## DB-STMTEXEC-137

StatementExecutor no ejecutará EXPLAIN automáticamente.

## DB-STMTEXEC-138

Extensions utilizarán contracts tipados.

## DB-STMTEXEC-139

Observers no modificarán execution state.

## DB-STMTEXEC-140

Runtime registries estarán frozen.

## DB-STMTEXEC-141

Registry conflicts no usarán last-wins.

## DB-STMTEXEC-142

StatementExecutor no dependerá de ORM.

## DB-STMTEXEC-143

StatementExecutor no dependerá de Optimizer.

## DB-STMTEXEC-144

StatementExecutor no dependerá de Planner.

## DB-STMTEXEC-145

StatementExecutor no dependerá de EntityManager.

## DB-STMTEXEC-146

StatementExecutor no dependerá de UnitOfWork.

## DB-STMTEXEC-147

StatementExecutor no dependerá de IdentityMap.

## DB-STMTEXEC-148

Normal statement execution no dependerá del SQL Compiler.

## DB-STMTEXEC-149

Resource ownership deberá permanecer consistente incluso ante excepciones.

## DB-STMTEXEC-150

Connection session state deberá ser seguro antes de reuse.

## DB-STMTEXEC-151

Prepared statement state deberá ser seguro antes de reuse.

## DB-STMTEXEC-152

Cursor state deberá ser seguro antes de connection reuse.

## DB-STMTEXEC-153

Result semantics serán preservadas.

## DB-STMTEXEC-154

Transaction semantics serán preservadas.

## DB-STMTEXEC-155

Locking semantics serán preservadas.

## DB-STMTEXEC-156

Security execution requirements serán preservadas.

## DB-STMTEXEC-157

Statement side effects nunca se asumirán inexistentes por sintaxis superficial.

## DB-STMTEXEC-158

Unknown execution outcome deberá tratarse conservadoramente.

## DB-STMTEXEC-159

Statement execution no cambiará el significado del compiled command.

## DB-STMTEXEC-160

StatementExecutor coordinará runtime execution sin absorber responsabilidades de capas vecinas.

---

# 267. Invariante maestro de representación

```text
Semantics(
    StatementExecution(
        CompiledDatabaseCommand
    )
)
=
Semantics(
    CompiledDatabaseCommand
)
```

El executor ejecuta la representación.

No la reinterpreta.

---

# 268. Invariante maestro de dispatch

Antes de:

```text
DriverExecute
```

debe existir:

```text
ValidCommand
∧
CompatibleConnection
∧
ValidPreparedStatement
∧
CompleteBindings
∧
SatisfiedTransactionRequirements
∧
NotCancelled
∧
DeadlineValid
```

---

# 269. Invariante maestro de side effects

```text
FailureBeforeDispatch
⇒
DatabaseStatementNotExecuted
```

Mientras:

```text
FailureAfterDispatch
⇏
DatabaseStatementNotExecuted
```

Por ello:

```text
PostDispatchFailure
```

puede requerir:

```text
OutcomeCertainty = UNKNOWN
```

---

# 270. Invariante maestro de recursos

Al cerrar la ejecución:

```text
AcquiredResources
=
ReleasedResources
∪
TransferredResources
```

y:

```text
ReleasedResources
∩
TransferredResources
=
∅
```

---

# 271. Invariante maestro de streaming

```text
StreamingResultAlive
⇒
RequiredStreamingResourcesAlive
```

---

# 272. Invariante maestro de reuse

Antes de reutilizar:

```text
Connection
PreparedStatement
```

deberá cumplirse:

```text
NoOwnedCursor
∧
NoPendingResult
∧
NoStaleBindings
∧
NoTemporaryExecutionControls
∧
ValidSessionState
∧
HealthyResourceState
```

---

# 273. Invariante maestro de persistent runtime

```text
Request A Statement Runtime State
∩
Request B Statement Runtime State
=
∅
```

excepto recursos explícitamente reutilizables y correctamente reseteados.

---

# 274. Ejemplo completo — SELECT

Command:

```sql
SELECT `id`, `name`
FROM `users`
WHERE `status` = ?
ORDER BY `id`
LIMIT ?
```

Flujo:

```text
StatementExecutionRequest
        ↓
Validate
        ↓
Acquire READ ConnectionLease
        ↓
Acquire PreparedStatementLease
        ↓
Bind:
  status → active
  limit  → 50
        ↓
Install Deadline
        ↓
Driver Execute
        ↓
Acquire ResultCursor
        ↓
Validate ROW_STREAM
        ↓
Transfer:
  Cursor
  StatementLease
  ConnectionLease
        ↓
StreamingResult
```

El StatementExecutor no:

```text
optimiza ORDER BY
elige índice
interpreta status
hidrata User entities
```

---

# 275. Ejemplo completo — UPDATE

Command:

```sql
UPDATE `users`
SET `status` = ?
WHERE `id` = ?
```

Flujo:

```text
StatementExecutionRequest
        ↓
Validate WRITE requirement
        ↓
Acquire primary/write connection
        ↓
Acquire prepared statement
        ↓
Bind parameters
        ↓
Execute
        ↓
Acquire affected-row metadata
        ↓
Normalize according to driver contract
        ↓
Release statement
        ↓
Release connection
        ↓
AffectedRowsResult
```

---

# 276. Ejemplo — failure before dispatch

```text
Acquire connection
      ↓
Prepare statement
      ↓
Missing binding
      ↓
FAILURE
```

Garantía:

```text
statement was not dispatched
```

Por tanto:

```text
OutcomeCertainty
=
NOT_DISPATCHED
```

---

# 277. Ejemplo — ambiguous write

```text
UPDATE dispatched
      ↓
network connection lost
      ↓
no acknowledgement
```

Resultado:

```text
StatementExecutionOutcome
    status = OUTCOME_UNKNOWN
```

No:

```text
retry automatically
```

---

# 278. Ejemplo — streaming ownership

```text
StatementExecutor
      ↓
execute SELECT
      ↓
open cursor
      ↓
transfer resources
      ↓
StreamingResult
      │
      ├── owns Cursor
      ├── owns StatementLease
      └── owns ConnectionLease
```

Cuando:

```text
StreamingResult::close()
```

se ejecuta:

```text
close cursor
reset/release statement
release connection
```

según el ownership contract.

---

# 279. Ejemplo — cancellation before dispatch

```text
Request
  ↓
Acquire connection
  ↓
Cancellation detected
  ↓
Do not execute
  ↓
Cleanup statement/connection
  ↓
CANCELLED
```

---

# 280. Ejemplo — timeout during execution

```text
execute()
   ↓
deadline expires
   ↓
Cancellation Strategy
   ↓
driver cancellation attempt
   ↓
classify actual outcome
   ↓
cleanup
```

El resultado puede depender de lo que el driver pueda confirmar.

---

# 281. Anti-patterns

## Anti-pattern 1 — God StatementExecutor

```php
final class StatementExecutor
{
    public function execute(...)
    {
        // connection routing
        // transaction management
        // SQL generation
        // prepare cache
        // type conversion
        // retry loops
        // hydration
        // telemetry
        // everything
    }
}
```

Prohibido.

---

# 282. Anti-pattern 2 — SQL text routing

```php
if (str_starts_with($sql, 'SELECT')) {
    $connection = $replica;
}
```

Incorrecto.

---

# 283. Anti-pattern 3 — interpolated bindings

```php
$sql = str_replace('?', $value, $sql);
```

Prohibido.

---

# 284. Anti-pattern 4 — automatic write retry

```php
try {
    execute();
} catch (ConnectionException) {
    execute();
}
```

Prohibido.

---

# 285. Anti-pattern 5 — premature connection release

```text
execute SELECT
↓
return connection to pool
↓
caller still consuming cursor
```

Prohibido.

---

# 286. Anti-pattern 6 — persistent state leakage

```php
final class StatementExecutor
{
    private ?Connection $currentConnection;
    private array $currentBindings;
}
```

si el servicio es compartido entre requests.

---

# 287. Anti-pattern 7 — universal lastInsertId

```php
$result->id = $connection->lastInsertId();
```

después de toda inserción.

Incorrecto.

---

# 288. Anti-pattern 8 — swallowing cleanup failure

```php
try {
    cleanup();
} catch (\Throwable) {
}
```

sin diagnóstico.

---

# 289. Anti-pattern 9 — returning poisoned connection

```text
session state corrupted
↓
return to pool anyway
```

Prohibido.

---

# 290. Anti-pattern 10 — ORM hydration

```php
return User::hydrate($rows);
```

dentro del StatementExecutor.

Prohibido.

---

# 291. Fórmula arquitectónica

```text
Statement Execution System
=
Request Validation
+
Execution Instance
+
Connection Lease Coordination
+
Transaction Affinity Validation
+
Prepared Statement Acquisition
+
Runtime Binding Coordination
+
Execution Control Installation
+
Driver Dispatch
+
Result Acquisition
+
Result Contract Validation
+
Outcome Certainty Tracking
+
Resource Ownership
+
Cleanup
+
Failure Metadata
+
Instrumentation
+
Persistent Runtime Isolation
```

---

# 292. Flujo final simplificado

```text
CompiledDatabaseCommand
        +
RuntimeBindingSet
        +
ExecutionContext
        ↓
StatementExecutor
        ↓
ConnectionLease
        ↓
PreparedStatementLease
        ↓
ParameterBindingSystem
        ↓
Driver Execute
        ↓
Result System
        ↓
Release / Transfer Resources
        ↓
StatementExecutionOutcome
```

---

# 293. Principio final

El Statement Execution System deberá seguir esta regla:

> **Prepare what has already been compiled, bind what has already been declared, execute what has already been planned, and return exactly the result contract that was requested.**

En forma resumida:

```text
StatementExecutor
=
Runtime Coordinator

not

Query Decision Engine
```

Y su invariante principal:

```text
Execute
without
reinterpretation
```

---

# 294. Relación con el siguiente documento

La ejecución de un statement requiere una abstracción robusta para representar, adquirir, reutilizar, resetear y liberar statements preparados.

La siguiente frontera es:

```text
StatementExecutor
      ↓
PreparedStatementSystem
      ↓
PreparedStatementLease
      ↓
DriverPreparedStatement
```

---

# 295. Siguiente documento

```text
79_DATABASE_PREPARED_STATEMENT_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Prepared Statement System
├── PreparedStatement Architecture
├── PreparedStatementDescriptor
├── PreparedStatementHandle
├── PreparedStatementLease
├── Preparation Request
├── Preparation Context
├── Statement Preparation
├── Driver Preparation Contract
├── Connection Affinity
├── Session Affinity
├── Prepared Statement Identity
├── Prepared Statement Fingerprint
├── Statement Reuse
├── Statement Cache
├── Statement Cache Key
├── Statement Reset
├── Binding Reset
├── Cursor Reset
├── Result Drain
├── Statement Health
├── Statement Invalidation
├── Connection Invalidation
├── Capability Compatibility
├── Resource Ownership
├── Concurrency Safety
├── Persistent Runtime Safety
├── Instrumentation
├── Security
├── Extensions
└── Architectural Invariants
```

manteniendo siempre:

```text
CompiledDatabaseCommand
≠
PreparedStatement
≠
StatementExecutionInstance
```