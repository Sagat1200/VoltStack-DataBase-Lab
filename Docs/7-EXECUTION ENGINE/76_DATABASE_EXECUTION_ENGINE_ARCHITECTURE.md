# 76_DATABASE_EXECUTION_ENGINE_ARCHITECTURE.md

# VoltStack Quantum Database
## Execution Engine Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 76 — Execution Engine Architecture  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Execution Engine` define la arquitectura responsable de consumir los artefactos previamente construidos por el Query Engine y convertirlos en operaciones reales contra el sistema de base de datos.

Su responsabilidad comienza cuando VoltStack ya conoce:

```text
qué operación ejecutar
cómo está planificada
qué SQL utilizar
qué parámetros necesita
qué resultado espera
qué recursos requiere
```

Formalmente:

```text
Execute(
    ExecutionPlan,
    CompiledDatabaseCommand,
    RuntimeBindings,
    ExecutionContext
)
    ↓
ExecutionOutcome
```

El Execution Engine constituye la frontera entre:

```text
declarative / compiled database knowledge
```

y:

```text
live database runtime interaction
```

---

# 2. Principio fundamental

```text
Execution Engine
=
Runtime Orchestration
```

No:

```text
Execution Engine
=
Query Interpretation
+
Optimization
+
Planning
+
SQL Generation
```

La regla principal será:

> **The Execution Engine executes decisions. It does not rediscover them.**

---

# 3. Posición arquitectónica

```text
Query Model / AST
        ↓
Semantic Engine
        ↓
Optimizer
        ↓
Logical Planner
        ↓
Physical Planner
        ↓
Execution Planner
        ↓
SQL Compiler
        ↓
Prepared Statement Compiler
        ↓
Compiled Artifact Cache
        ↓
════════════════════════════════
      EXECUTION BOUNDARY
════════════════════════════════
        ↓
Execution Engine
        ↓
Connection Manager
        ↓
Connection
        ↓
Driver
        ↓
Database Server
```

---

# 4. Cambio de dominio

Hasta el documento 75 predominaban objetos como:

```text
SemanticQueryArtifact
LogicalQueryPlan
PhysicalQueryPlan
ExecutionPlan
CompiledDatabaseCommand
PreparedStatementBlueprint
```

que son principalmente:

```text
immutable
reusable
cacheable
serializable
runtime-independent
```

A partir del Execution Engine aparecen:

```text
ConnectionLease
PreparedStatement
RuntimeBindingSet
ExecutionInstance
ResultCursor
CancellationToken
Deadline
TransactionContext
```

que son:

```text
mutable
operation-scoped
resource-owning
runtime-specific
not generally cacheable
```

---

# 5. Frontera fundamental

```text
Reusable Knowledge
────────────────────────────
ExecutionPlan
CompiledDatabaseCommand
PreparedStatementBlueprint

════════ EXECUTION ════════

Mutable Runtime State
────────────────────────────
ExecutionInstance
ConnectionLease
PreparedStatement
Runtime Bindings
Cursor
Buffers
Cancellation State
```

Esta frontera deberá mantenerse estrictamente.

---

# 6. Objetivos

El Execution Engine deberá proporcionar:

1. ejecución determinista del plan recibido;
2. adquisición y liberación segura de conexiones;
3. preparación o reutilización de statements;
4. binding seguro de parámetros;
5. ejecución de statements;
6. producción estructurada de resultados;
7. soporte para cursores y streaming;
8. propagación de backpressure;
9. timeouts y deadlines;
10. cancelación;
11. manejo de errores;
12. cleanup determinista;
13. integración transaccional;
14. límites claros para retries;
15. ownership explícito de recursos;
16. aislamiento entre requests;
17. compatibilidad con persistent workers;
18. hooks de telemetría;
19. extensibilidad controlada;
20. independencia del ORM.

---

# 7. No objetivos

El Execution Engine no deberá:

```text
parse SQL
build Query AST
resolve symbols
infer query types
optimize predicates
reorder joins
select indexes
choose physical join algorithms
generate SQL
decide ORM hydration
track entities
perform UnitOfWork
decide authorization policy
discover tenant semantics
```

---

# 8. Separación de responsabilidades

```text
Semantic Engine
    ↓
determines meaning

Optimizer
    ↓
chooses equivalent logical form

Planner
    ↓
chooses execution strategy

Compiler
    ↓
produces database representation

Execution Engine
    ↓
orchestrates runtime execution

Connection
    ↓
manages database session/channel

Driver
    ↓
performs protocol/API interaction
```

---

# 9. Invariante principal

```text
Semantics(
    Execute(
        ExecutionPlan,
        RuntimeBindings
    )
)
=
Semantics(ExecutionPlan)
```

El Execution Engine no podrá modificar silenciosamente la semántica recibida.

---

# 10. Arquitectura general

```text
                  ExecutionRequest
                         │
                         ▼
              Execution Coordinator
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
       Validation    Resources    Runtime State
             │           │           │
             └───────────┼───────────┘
                         ▼
                 ExecutionInstance
                         │
                         ▼
                 Execution Scheduler
                         │
             ┌───────────┼────────────┐
             │           │            │
             ▼           ▼            ▼
       DB Execution   Framework    Extension
           Unit         Unit          Unit
             │
             ▼
       Connection Manager
             │
             ▼
       Connection Lease
             │
             ▼
       Statement Manager
             │
             ▼
       Prepared Statement
             │
             ▼
       Runtime Binder
             │
             ▼
           Driver
             │
             ▼
       Database Server
             │
             ▼
        Driver Result
             │
             ▼
        Result Adapter
             │
             ▼
       ExecutionOutcome
```

---

# 11. ExecutionRequest

La entrada pública de bajo nivel podrá representarse mediante:

```php
final readonly class ExecutionRequest
{
    public function __construct(
        public ExecutionPlan $plan,
        public RuntimeBindingSet $bindings,
        public ExecutionContext $context,
    ) {}
}
```

El `ExecutionPlan` podrá referenciar los:

```text
CompiledDatabaseCommand
PreparedStatementBlueprint
```

necesarios.

---

# 12. ExecutionRequest ≠ Query

`ExecutionRequest` no será:

```text
QueryBuilder
Query AST
SQL string
Entity query
```

Es una solicitud para ejecutar conocimiento previamente preparado.

---

# 13. ExecutionContext

```php
final readonly class ExecutionContext
{
    public function __construct(
        public ExecutionOperationId $operationId,
        public ConnectionRequirementSet $connections,
        public TransactionExecutionContext $transaction,
        public RuntimeBindingContext $bindings,
        public ExecutionDeadline $deadline,
        public CancellationContext $cancellation,
        public ResourceExecutionPolicy $resources,
        public RetryExecutionPolicy $retry,
        public ExecutionInstrumentationContext $instrumentation,
        public ExecutionSecurityContext $security,
        public ExecutionTenantContext $tenant,
    ) {}
}
```

---

# 14. ExecutionContext no será Service Locator

Incorrecto:

```php
final class ExecutionContext
{
    public Container $container;
}
```

Correcto:

```text
explicit runtime dependencies
+
explicit runtime policies
+
explicit operation state
```

---

# 15. Execution context ≠ HTTP context

Database Execution Engine no deberá conocer:

```text
Request
Response
Controller
Route
Session
Cookie
```

La integración HTTP deberá transformar esos conceptos antes de llegar al Database System.

---

# 16. ExecutionInstance

`ExecutionPlan` es reusable.

`ExecutionInstance` es mutable.

```php
final class ExecutionInstance
{
    // runtime state
}
```

Conceptualmente:

```text
ExecutionPlan
    │
    ├── ExecutionInstance A
    │      bindings = A
    │
    ├── ExecutionInstance B
    │      bindings = B
    │
    └── ExecutionInstance C
           bindings = C
```

---

# 17. Regla

```text
ExecutionPlan
≠
ExecutionInstance
```

---

# 18. Estado del ExecutionInstance

Estado recomendado:

```php
enum ExecutionState
{
    case CREATED;
    case PREPARING;
    case READY;
    case RUNNING;
    case DRAINING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case TIMED_OUT;
    case CLEANING;
    case CLOSED;
}
```

---

# 19. State machine

```text
CREATED
   ↓
PREPARING
   ↓
READY
   ↓
RUNNING
   ├──────────────→ FAILED
   ├──────────────→ CANCELLED
   ├──────────────→ TIMED_OUT
   │
   ↓
DRAINING
   ↓
COMPLETED
   │
   ▼
CLEANING
   ↓
CLOSED
```

Los estados terminales de error también deberán pasar por cleanup.

---

# 20. Transiciones inválidas

Ejemplos:

```text
CLOSED → RUNNING
FAILED → READY
COMPLETED → RUNNING
```

deberán rechazarse.

---

# 21. Execution Coordinator

Contrato conceptual:

```php
interface ExecutionEngine
{
    public function execute(
        ExecutionRequest $request,
    ): ExecutionOutcome;
}
```

Sin embargo, `ExecutionEngine` será una fachada/coordinador.

No deberá implementar todas las responsabilidades internamente.

---

# 22. Componentes principales

```text
ExecutionEngine
│
├── ExecutionRequestValidator
├── ExecutionInstanceFactory
├── ExecutionScheduler
├── ConnectionLeaseManager
├── StatementExecutionManager
├── RuntimeParameterBinder
├── ResultProducer
├── CursorManager
├── CancellationCoordinator
├── DeadlineManager
├── FailureCoordinator
├── CleanupCoordinator
├── ResourceGovernor
├── TransactionExecutionCoordinator
├── RetryCoordinator
├── InstrumentationDispatcher
└── ExecutionExtensionRegistry
```

---

# 23. Execution validation

Antes de tocar recursos vivos deberán validarse:

```text
plan compatibility
binding completeness
binding types
execution capabilities
transaction requirements
connection requirements
security requirements
tenant execution context
resource limits
deadline state
extension availability
```

---

# 24. Validation ≠ semantic analysis

Execution validation no preguntará:

```text
"¿Qué significa esta expresión?"
```

Preguntará:

```text
"¿Tenemos todo lo necesario para ejecutar este plan?"
```

---

# 25. Preflight validation

```text
ExecutionRequest
       ↓
Preflight Validator
       ├── plan valid?
       ├── bindings complete?
       ├── capabilities valid?
       ├── transaction compatible?
       ├── deadline already expired?
       ├── cancelled already?
       └── resources permitted?
       ↓
Execution Preparation
```

---

# 26. Validation before resource acquisition

Siempre que sea posible:

```text
invalid request
```

deberá detectarse antes de adquirir una conexión.

---

# 27. Resource ownership

Uno de los principios centrales será:

> **Every live resource must have an explicit owner.**

---

# 28. Recursos vivos

Ejemplos:

```text
ConnectionLease
PreparedStatement
Cursor
ResultStream
TemporaryBuffer
Spool
TemporaryFile
MemoryReservation
DriverHandle
CancellationHandle
```

---

# 29. ResourceOwner

```php
interface ExecutionResourceOwner
{
    public function resourceOwnerId(): ExecutionResourceOwnerId;
}
```

---

# 30. Ownership graph

```text
ExecutionInstance
│
├── ConnectionLease
│   ├── PreparedStatement
│   │   └── DriverResult
│   │       └── ResultCursor
│   │
│   └── Session State
│
├── MemoryReservation
└── Temporary Resources
```

---

# 31. Ownership transfer

Algunos recursos podrán transferirse.

Ejemplo:

```text
ExecutionInstance
    ↓
ResultCursor
    ↓
StreamingResult
```

Cuando el resultado es devuelto al caller:

```text
cursor ownership
```

puede pasar del engine al `StreamingResult`.

---

# 32. Transferencia explícita

Nunca:

```text
"the cursor probably stays alive"
```

Siempre:

```text
OwnershipTransfer
```

estructurado.

---

# 33. Connection acquisition

Execution Engine no creará directamente PDO.

Usará:

```text
Connection Manager
```

---

# 34. Connection requirement

El plan podrá declarar:

```php
final readonly class ConnectionRequirement
{
    public function __construct(
        public ConnectionRole $role,
        public ConnectionConsistency $consistency,
        public ConnectionCapabilityRequirementSet $capabilities,
        public ConnectionRoutingRequirement $routing,
    ) {}
}
```

---

# 35. ConnectionRole

Ejemplos:

```text
READ
WRITE
TRANSACTIONAL
ADMINISTRATIVE
EXTENSION
```

---

# 36. Read/write routing

Execution Engine consume la decisión o requisito.

La política real de selección podrá ser responsabilidad del:

```text
Connection Manager
+
Read/Write Routing System
```

---

# 37. ConnectionLease

No deberá entregarse simplemente una conexión sin ownership.

```php
interface ConnectionLease
{
    public function connection(): Connection;

    public function release(): void;
}
```

---

# 38. Lease semantics

```text
acquire
   ↓
use
   ↓
reset if required
   ↓
release
```

---

# 39. Connection reuse

Especialmente bajo FrankenPHP:

```text
connection reuse
```

es deseable.

Pero:

```text
session state leakage
```

es inaceptable.

---

# 40. Connection state

Una conexión puede contener estado como:

```text
transaction
isolation level
session variables
temporary tables
SQL modes
timezone
charset
search_path
role
locks
```

según la plataforma.

---

# 41. Connection reset contract

Antes de reutilizar una conexión deberán respetarse las garantías definidas por:

```text
DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
```

---

# 42. Execution Engine no implementa reset internamente

Debe solicitarlo a la capa responsable.

---

# 43. Statement acquisition

Flujo:

```text
CompiledDatabaseCommand
        ↓
PreparedStatementBlueprint
        ↓
Statement Acquisition
        ↓
PreparedStatement
```

---

# 44. Prepared blueprint ≠ live statement

```text
PreparedStatementBlueprint
=
reusable immutable preparation knowledge
```

```text
PreparedStatement
=
live connection-bound resource
```

---

# 45. Statement manager

```php
interface StatementManager
{
    public function acquire(
        ConnectionLease $connection,
        PreparedStatementBlueprint $blueprint,
    ): StatementLease;
}
```

---

# 46. StatementLease

```php
interface StatementLease
{
    public function statement(): PreparedStatement;

    public function release(): void;
}
```

---

# 47. Connection binding

Un live prepared statement pertenece a:

```text
specific connection/session
```

salvo que un driver declare otra semántica explícita.

---

# 48. No global statement handles

Nunca:

```text
static PDOStatement
```

compartido entre conexiones.

---

# 49. Runtime parameter binding

La compilación produjo:

```text
CompiledBindingLayout
```

La ejecución aporta:

```text
RuntimeBindingSet
```

---

# 50. Binding equation

```text
CompiledBindingLayout
+
RuntimeBindingSet
+
Driver Binding Contract
=
Driver Bindings
```

---

# 51. RuntimeBindingSet

```php
final readonly class RuntimeBindingSet
{
    /**
     * @var array<ParameterId, RuntimeBinding>
     */
    private array $bindings;
}
```

---

# 52. RuntimeBinding

```php
final readonly class RuntimeBinding
{
    public function __construct(
        public ParameterId $parameter,
        public mixed $value,
        public QueryType $type,
        public BindingSensitivity $sensitivity,
    ) {}
}
```

---

# 53. Runtime values live only in execution scope

Nunca deberán escribirse en:

```text
CompiledDatabaseCommand
PreparedStatementBlueprint
Compiled Query Cache
global static state
```

---

# 54. Binding identity

Mantener:

```text
ParameterId
≠
ExecutionParameterSlotId
≠
SqlPlaceholderId
≠
DriverBindingPosition
```

---

# 55. Binding pipeline

```text
RuntimeBinding
      ↓
Type Validation
      ↓
Conversion Plan
      ↓
Driver Value Conversion
      ↓
Placeholder Mapping
      ↓
Driver Binding
```

---

# 56. Type conversion

Conversiones runtime pertenecen a esta frontera o al Type/Driver binding subsystem.

No al SQL renderer.

---

# 57. Binding security

Runtime values jamás deberán convertirse en SQL mediante concatenación.

---

# 58. Sensitive bindings

Bindings sensibles deberán marcarse.

Ejemplos:

```text
PASSWORD
TOKEN
SECRET
PII
CREDENTIAL
```

---

# 59. Diagnostic redaction

```text
parameter P3 = [REDACTED]
```

no:

```text
parameter P3 = actual_password
```

---

# 60. Statement execution

Contrato conceptual:

```php
interface StatementExecutor
{
    public function execute(
        PreparedStatement $statement,
        DriverBindingSet $bindings,
        StatementExecutionContext $context,
    ): DriverExecutionResult;
}
```

---

# 61. StatementExecutor ≠ QueryExecutor

`StatementExecutor` opera sobre una unidad database concreta.

`QueryExecutor` podrá coordinar un `ExecutionPlan` completo.

---

# 62. Execution unit

El plan puede contener:

```text
DatabaseExecutionUnit
FrameworkExecutionUnit
ExtensionExecutionUnit
```

---

# 63. DatabaseExecutionUnit

Ejecuta trabajo delegado a la base de datos.

---

# 64. FrameworkExecutionUnit

Ejecuta operaciones que el Physical/Execution Planner decidió realizar en VoltStack.

Ejemplos futuros:

```text
cross-shard merge
framework materialization
federated merge
application-level sort
```

---

# 65. No compiler fallback

El Execution Engine no deberá decidir:

```text
"DB does not support this, I'll implement it in PHP."
```

Eso debió ser decidido por el Planner.

---

# 66. ExecutionControlLevel

Se preservará:

```php
enum ExecutionControlLevel
{
    case FRAMEWORK_CONTROLLED;
    case COMPILER_INFLUENCED;
    case DATABASE_DELEGATED;
    case HYBRID;
}
```

---

# 67. V1 strategy

Para SQL databases maduras:

```text
maximize safe database delegation
```

será la estrategia preferida.

---

# 68. Database delegation

Ejemplo:

```text
Filter
Join
Aggregate
Sort
Limit
```

podrán fusionarse en un único:

```text
DatabaseExecutionUnit
```

que ejecute un statement SQL.

---

# 69. Execution graph

El Execution Engine deberá consumir el graph definido por `ExecutionPlan`.

No reconstruirlo.

---

# 70. Execution edges

Tipos conceptuales:

```text
DATA
CONTROL
DEPENDENCY
PARAMETER
CORRELATION
MATERIALIZATION
RESOURCE
CANCELLATION
CLEANUP
RECURSION
```

---

# 71. Scheduling

Execution Scheduler determina cuándo una unidad puede ejecutarse conforme a las dependencias existentes.

---

# 72. Scheduler ≠ thread scheduler

No deberá asumir:

```text
OS thread
fiber
coroutine
process
```

---

# 73. Runtime-neutral scheduling

El plan expresa:

```text
dependencies
parallelizability
resource requirements
```

El runtime adapter decide cómo materializarlo.

---

# 74. Sequential V1

VoltStack V1 podrá ejecutar la mayoría de planes secuencialmente.

La arquitectura no deberá impedir futura concurrencia.

---

# 75. Parallelizable ≠ parallel

Que una unidad pueda ejecutarse en paralelo no obliga al runtime a hacerlo.

---

# 76. Execution stage

```text
ExecutionPlan
├── Stage 1
├── Stage 2
└── Stage 3
```

---

# 77. Stage ≠ transaction

Un stage es una frontera de orchestration.

No implica automáticamente una transacción.

---

# 78. Stage ≠ connection

Un stage puede usar una o varias conexiones según el plan.

---

# 79. Stage lifecycle

```text
PENDING
READY
RUNNING
COMPLETED
FAILED
CANCELLED
```

---

# 80. Result architecture

El Driver produce un resultado driver-specific.

Execution Engine deberá transformarlo a contracts internos.

---

# 81. Result pipeline

```text
DriverExecutionResult
        ↓
Driver Result Adapter
        ↓
Database Result Contract
        ↓
ExecutionOutcome
```

---

# 82. Result kinds

```php
enum ExecutionResultKind
{
    case ROW_STREAM;
    case BUFFERED_ROWS;
    case SCALAR;
    case AFFECTED_ROWS;
    case RETURNING_ROWS;
    case NO_RESULT;
    case EXTENSION_RESULT;
}
```

---

# 83. Result ≠ hydration

Execution Engine produce:

```text
database result
```

No:

```text
ORM Entity
```

---

# 84. ORM boundary

```text
Execution Engine
      ↓
Database Result
      ↓
Hydration System
      ↓
Entity / Scalar / DTO
```

---

# 85. Result contract

```php
interface ExecutionResult
{
    public function kind(): ExecutionResultKind;
}
```

---

# 86. Buffered result

```text
Database
   ↓
fetch all
   ↓
memory buffer
   ↓
consumer
```

Adecuado para resultados pequeños.

---

# 87. Streaming result

```text
Database
   ↓
Cursor
   ↓
Row
   ↓
Consumer
   ↓
Row
   ↓
Consumer
```

Adecuado para grandes datasets.

---

# 88. Streaming is first-class

No deberá implementarse como:

```text
fetchAll()
+
yield
```

porque eso no es streaming real.

---

# 89. ResultCursor

```php
interface ResultCursor
{
    public function fetch(): ?DatabaseRow;

    public function close(): void;
}
```

---

# 90. Cursor ownership

Mientras el cursor permanezca abierto:

```text
connection
statement
driver result
```

pueden permanecer ocupados.

---

# 91. Cursor resource chain

```text
StreamingResult
     ↓ owns
ResultCursor
     ↓ owns/depends
StatementLease
     ↓ depends
ConnectionLease
```

---

# 92. Closing stream

```text
StreamingResult::close()
```

deberá provocar cleanup de toda la cadena que corresponda.

---

# 93. Early consumer termination

Ejemplo:

```php
foreach ($rows as $row) {
    if ($row['id'] === 100) {
        break;
    }
}
```

deberá liberar correctamente el cursor.

---

# 94. Destructor is not enough

No depender exclusivamente de:

```php
__destruct()
```

para resource correctness.

---

# 95. Explicit close

Los resultados streaming deberán soportar lifecycle explícito.

---

# 96. Automatic cleanup

Podrán añadirse mecanismos automáticos como safety net.

Pero no sustituirán el ownership model.

---

# 97. Backpressure

Streaming deberá soportar backpressure.

---

# 98. Pull-based V1

Modelo recomendado:

```text
Consumer requests next row
        ↓
Cursor fetches next row
```

---

# 99. Pull model

```php
while (($row = $cursor->fetch()) !== null) {
    consume($row);
}
```

es naturalmente backpressure-aware.

---

# 100. Push-based future

Podrá existir para runtimes async.

Pero requerirá:

```text
flow control
buffer limits
cancellation
```

explícitos.

---

# 101. Memory governance

Execution Engine deberá controlar:

```text
buffer sizes
materialization memory
hash memory
sort memory
result buffering
temporary structures
```

para framework-controlled operations.

---

# 102. MemoryReservation

```php
interface MemoryReservation
{
    public function bytes(): int;

    public function release(): void;
}
```

---

# 103. Resource Governor

```php
interface ExecutionResourceGovernor
{
    public function reserve(
        ExecutionResourceRequest $request,
    ): ExecutionResourceReservation;
}
```

---

# 104. Resource limits

Podrán existir:

```text
max execution memory
max buffered rows
max temporary bytes
max open cursors
max statements
max connections per execution
```

---

# 105. Resource exhaustion

Debe producir error estructurado.

Nunca:

```text
unbounded allocation
```

---

# 106. Spill

Si un PhysicalPlan declaró capacidad de spill:

```text
memory
  ↓ threshold
temporary storage
```

podrá utilizarse.

---

# 107. Spill ≠ hidden fallback

No podrá aparecer arbitrariamente durante ejecución si el plan no lo permite.

---

# 108. Temporary resource ownership

Todo temporary file/spool deberá tener:

```text
owner
lifetime
cleanup
budget
```

---

# 109. Cancellation

Cancellation será first-class.

---

# 110. CancellationToken

```php
interface CancellationToken
{
    public function isCancellationRequested(): bool;
}
```

---

# 111. CancellationSource

El caller podrá controlar una:

```text
CancellationSource
```

que active el token.

---

# 112. Cancellation propagation

```text
ExecutionInstance
        ↓
Stage
        ↓
ExecutionUnit
        ↓
Statement
        ↓
Driver
```

cuando el driver permita cancelación.

---

# 113. Driver cancellation capability

No todos los drivers tienen la misma capacidad.

Por ello:

```text
supportsStatementCancellation()
```

deberá ser capability-driven.

---

# 114. Cancellation when unsupported

Si no puede cancelarse físicamente el statement:

```text
mark cancellation
stop future work
discard result when safe
cleanup after driver returns
```

según las garantías del driver.

---

# 115. No fake cancellation

VoltStack no deberá afirmar:

```text
"query cancelled"
```

si sólo dejó de esperar mientras el servidor continúa consumiendo recursos, salvo que esa semántica esté claramente representada.

---

# 116. Cancellation outcome

```text
CANCELLED
```

deberá distinguirse de:

```text
FAILED
```

---

# 117. Timeout

Timeout será distinto de cancellation manual.

---

# 118. ExecutionDeadline

Preferible:

```text
absolute monotonic deadline
```

dentro del runtime.

Conceptualmente:

```text
deadline = start + allowed_duration
```

---

# 119. Monotonic time

Duraciones deberán medirse con clock monotónico cuando esté disponible.

---

# 120. Timeout levels

Podrán existir:

```text
whole execution timeout
statement timeout
connection acquisition timeout
cursor idle timeout
cleanup timeout
```

---

# 121. Deadline propagation

Un timeout global deberá reducir los budgets internos.

Ejemplo:

```text
Global remaining = 500 ms
Statement configured = 5 s

Effective statement budget <= 500 ms
```

---

# 122. Server-side timeout

Cuando la plataforma soporte statement timeout:

```text
database-side timeout
```

puede ser utilizado.

---

# 123. Client-side timeout

Puede existir adicionalmente.

---

# 124. Server timeout ≠ client timeout

Ambos tienen diferentes garantías.

---

# 125. Timeout outcome

```text
TIMED_OUT
```

deberá distinguirse de:

```text
CANCELLED
FAILED
```

---

# 126. Failure architecture

Execution Engine deberá preservar el error original.

---

# 127. Failure categories

```text
ConnectionAcquisitionFailure
ConnectionLost
StatementPreparationFailure
ParameterBindingFailure
StatementExecutionFailure
ConstraintViolation
Deadlock
SerializationFailure
Timeout
Cancellation
ResultFetchFailure
ResourceExhaustion
CleanupFailure
DriverFailure
ExtensionFailure
InvariantFailure
```

---

# 128. Driver error normalization

Driver-specific errors deberán convertirse en una taxonomía VoltStack sin destruir el error original.

---

# 129. Error chain

```text
DatabaseExecutionException
    ↓ cause
DriverException
    ↓ metadata
native error code
SQLSTATE
driver message
```

---

# 130. SQLSTATE

Cuando exista deberá conservarse como metadata estructurada.

---

# 131. Source map integration

`SqlSourceMap` podrá utilizarse para relacionar errores con:

```text
Query node
Semantic node
Plan node
Execution unit
SQL range
Parameter
```

---

# 132. Source map limitation

Si el servidor sólo devuelve:

```text
syntax error near ...
```

sin offset preciso, VoltStack no deberá inventar precisión.

---

# 133. Error redaction

Database errors pueden incluir SQL o values sensibles.

La capa diagnóstica deberá sanitizarlos.

---

# 134. Failure propagation

```text
Unit Failure
     ↓
Execution Failure Coordinator
     ↓
Cancel dependent work
     ↓
Cleanup resources
     ↓
Produce structured failure
```

---

# 135. Failure ≠ cleanup

Son fases diferentes.

---

# 136. Cleanup always runs

Conceptualmente:

```php
try {
    execute();
} finally {
    cleanup();
}
```

aunque la implementación real sea más compleja.

---

# 137. Cleanup architecture

```text
Execution
   ↓
SUCCESS / FAILURE / CANCEL / TIMEOUT
   ↓
Cleanup Coordinator
   ↓
cursor cleanup
statement release
temporary resource cleanup
connection release/reset
memory release
instrumentation close
```

---

# 138. Cleanup ordering

El orden deberá respetar dependencias.

Ejemplo:

```text
Cursor
 ↓
Statement
 ↓
Connection
```

No al revés.

---

# 139. Cleanup graph

ExecutionPlan ya podrá declarar:

```text
CLEANUP edges
```

El runtime deberá respetarlos.

---

# 140. Cleanup idempotence

Siempre que sea posible:

```text
close()
close()
```

no deberá causar corrupción.

---

# 141. Cleanup failures

Si existe un error primario:

```text
StatementExecutionFailure
```

y después:

```text
CursorCleanupFailure
```

el cleanup error no deberá ocultar al primero.

---

# 142. Suppressed failures

Podrá utilizarse:

```text
primary failure
+
suppressed cleanup failures
```

---

# 143. Partial preparation failure

Si se adquirieron:

```text
connection
statement A
memory
```

y falla:

```text
statement B
```

todo recurso ya adquirido deberá limpiarse.

---

# 144. Resource ledger

Se recomienda mantener:

```text
ExecutionResourceLedger
```

operation-scoped.

---

# 145. Resource ledger example

```text
Resource                    State
────────────────────────────────────
ConnectionLease #1          acquired
StatementLease #4           released
Cursor #8                   open
MemoryReservation #3        released
TempSpool #2                open
```

---

# 146. Resource ledger ≠ global registry

Debe existir sólo dentro de la ejecución.

---

# 147. Transaction integration

Execution Engine no deberá inventar transacciones.

---

# 148. Transaction requirement

El ExecutionPlan podrá requerir:

```text
NONE
OPTIONAL
REQUIRED
EXISTING_REQUIRED
NEW_REQUIRED
```

según contratos posteriores.

---

# 149. Transaction Manager

La responsabilidad real de:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

pertenece al Transaction System.

Execution Engine lo coordina cuando el plan lo exige.

---

# 150. Existing transaction

Si existe una transacción exterior:

```text
Execution Engine
```

deberá respetar ownership.

---

# 151. Transaction ownership

```text
Caller-owned transaction
```

no podrá ser committed automáticamente por el Execution Engine.

---

# 152. Engine-owned transaction

Cuando el plan/policy autorice una transacción propia:

```text
Execution Engine
```

podrá solicitarla al Transaction Manager y será responsable de su lifecycle.

---

# 153. Transaction lease

Conceptualmente:

```php
interface TransactionLease
{
    public function ownership(): TransactionOwnership;
}
```

---

# 154. Transaction failure

Si el statement falla dentro de una transacción:

```text
rollback behavior
```

dependerá del ownership y transaction policy.

---

# 155. No implicit cross-database transaction

Un ExecutionPlan con dos databases no implicará atomicidad distribuida.

---

# 156. Distributed transaction

Deberá ser una capability/sistema explícito futuro.

---

# 157. Retry architecture

Retries son especialmente peligrosos.

---

# 158. Retry ≠ execute again

Antes de retry deberá conocerse:

```text
failure category
operation idempotency
transaction state
partial effects
driver state
connection state
deadline
retry policy
```

---

# 159. Retry eligibility

```php
interface RetryEligibilityDecider
{
    public function decide(
        ExecutionFailure $failure,
        RetryExecutionContext $context,
    ): RetryDecision;
}
```

---

# 160. Retry metadata

ExecutionPlan podrá declarar:

```text
IDEMPOTENT
CONDITIONALLY_IDEMPOTENT
NON_IDEMPOTENT
UNKNOWN
```

---

# 161. UNKNOWN default

```text
UNKNOWN
```

deberá tratarse conservadoramente.

---

# 162. Read retries

Algunas lecturas podrán ser retry-safe.

Pero:

```text
locking read
volatile function
transactional read
```

pueden alterar esa conclusión.

---

# 163. Write retries

Nunca deberán repetirse ciegamente.

---

# 164. Ambiguous commit

Caso crítico:

```text
COMMIT sent
connection lost
```

No siempre puede saberse si la transacción fue committed.

---

# 165. Ambiguous outcome

Debe existir una categoría como:

```text
ExecutionOutcomeUnknown
```

o error equivalente.

---

# 166. Never hide ambiguous writes

No realizar:

```text
"connection lost, retry INSERT"
```

sin idempotency proof.

---

# 167. Retry belongs to bounded policy

```text
max attempts
backoff
deadline
retryable failures
idempotency requirement
```

deberán ser explícitos.

---

# 168. Execution Engine vs doc 86

Este documento define sólo la frontera arquitectónica.

El comportamiento completo se desarrollará en:

```text
86_DATABASE_EXECUTION_RETRY_SYSTEM.md
```

---

# 169. DML execution

INSERT, UPDATE y DELETE deberán preservar:

```text
mutation semantics
affected rows contract
RETURNING contract
transaction requirements
locking behavior
```

---

# 170. Affected rows

No deberá asumirse que todos los drivers/platforms reportan affected rows exactamente igual.

---

# 171. Result normalization

La Platform/Driver layer deberá proporcionar un contrato normalizado.

---

# 172. Generated identifiers

La recuperación de:

```text
generated IDs
```

deberá seguir el compiled result contract.

No hacks genéricos dentro del Execution Engine.

---

# 173. RETURNING

Cuando el command usa RETURNING:

```text
RETURNING rows
```

se tratarán como result contract estructurado.

---

# 174. No post-write SELECT invention

Execution Engine no deberá transformar:

```text
INSERT
```

en:

```text
INSERT
+
SELECT
```

porque la plataforma no soporta RETURNING.

Tal estrategia debió decidirse antes.

---

# 175. Batch execution

Un ExecutionPlan podrá contener múltiples commands explícitos.

---

# 176. Explicit batch

```text
Command A
Command B
Command C
```

deberán existir explícitamente en el plan.

---

# 177. Hidden multi-statement forbidden

Execution Engine no deberá concatenar:

```text
SQL A; SQL B; SQL C;
```

como atajo.

---

# 178. Batch failure semantics

El plan deberá definir:

```text
stop on failure
transactional
best effort
collect failures
```

cuando corresponda.

---

# 179. Bulk execution

Bulk operations deberán usar capacidades específicas del driver/plataforma cuando hayan sido seleccionadas por el plan.

---

# 180. Execution extensions

El sistema deberá ser extensible sin permitir hooks arbitrarios sobre cualquier estado.

---

# 181. Extension points

Ejemplos:

```text
ExecutionUnitExecutor
ResourceProvider
CancellationAdapter
ResultAdapter
DriverExecutionAdapter
ExecutionObserver
FrameworkOperatorExecutor
```

---

# 182. Extension registry

```text
discover
   ↓
validate
   ↓
resolve conflicts
   ↓
freeze
```

---

# 183. No last-wins

Dos extensions que reclamen la misma unidad sin prioridad/resolución explícita deberán producir error de configuración.

---

# 184. Extension runtime state

Cada extension deberá mantener estado mutable en:

```text
ExecutionInstance
```

no en singleton compartido.

---

# 185. Extension cleanup

Toda extension que adquiera recursos deberá declarar cleanup.

---

# 186. Extension cancellation

Toda operación larga de extension deberá definir comportamiento de cancellation.

---

# 187. Extension failure

No podrá retornar:

```text
null
```

para representar fallo ambiguo.

---

# 188. Framework operators

Podrán existir:

```text
FrameworkFilterExecutor
FrameworkProjectExecutor
FrameworkHashJoinExecutor
FrameworkSortExecutor
FrameworkAggregateExecutor
FrameworkMergeExecutor
```

en futuras capacidades.

---

# 189. V1 restraint

No implementar un segundo database engine PHP completo en V1.

Priorizar:

```text
database delegation
```

---

# 190. Federation future

Framework operators serán especialmente útiles para:

```text
database federation
sharding
cross-database joins
distributed aggregation
distributed ordering
distributed limits
```

---

# 191. Runtime adapter architecture

Execution Engine deberá ser runtime-neutral.

---

# 192. Runtime adapter

```php
interface DatabaseRuntimeAdapter
{
    public function executionEnvironment(): ExecutionEnvironment;

    public function monotonicClock(): MonotonicClock;

    public function cancellationSupport(): RuntimeCancellationSupport;
}
```

---

# 193. FrankenPHP

Runtime predeterminado:

```text
FrankenPHP
```

El Execution Engine deberá aprovechar persistent workers.

---

# 194. FrankenPHP architecture

```text
FrankenPHP Worker
│
├── immutable/frozen
│   ├── execution descriptors
│   ├── executor registries
│   ├── driver descriptors
│   └── compiled caches
│
├── Request A
│   └── ExecutionInstance A
│
├── Request B
│   └── ExecutionInstance B
│
└── Request C
    └── ExecutionInstance C
```

---

# 195. Request isolation

Nunca:

```text
ExecutionInstance A
      ↓ leaks into
ExecutionInstance B
```

---

# 196. Persistent-safe shared state

Permitido:

```text
immutable descriptors
frozen registries
compiled artifacts
stateless executors
immutable configuration
```

---

# 197. Persistent-unsafe shared state

Prohibido:

```text
current connection lease
current transaction
current bindings
current cursor
current tenant
current deadline
current cancellation token
current result
```

---

# 198. RoadRunner

El mismo modelo deberá funcionar mediante un adapter futuro.

---

# 199. OpenSwoole

Requerirá especial atención a:

```text
coroutines
concurrent requests
connection ownership
context isolation
```

---

# 200. Coroutine safety

Nunca almacenar execution context en:

```text
static $currentExecution
```

---

# 201. Fiber/coroutine context

Si se usa almacenamiento contextual deberá estar abstraído por Runtime System y no convertirse en dependencia semántica del Database Engine.

---

# 202. Telemetry architecture

Execution Engine deberá exponer instrumentation points.

---

# 203. Instrumentation events

Ejemplos:

```text
ExecutionStarted
ConnectionAcquisitionStarted
ConnectionAcquired
StatementPreparationStarted
StatementPrepared
BindingStarted
BindingCompleted
StatementExecutionStarted
StatementExecutionCompleted
CursorOpened
RowFetched
CursorClosed
ExecutionCompleted
ExecutionFailed
ExecutionCancelled
ExecutionTimedOut
CleanupStarted
CleanupCompleted
```

---

# 204. RowFetched caution

Telemetry por row puede ser demasiado costosa.

Deberá estar deshabilitada por defecto o agregada.

---

# 205. Metrics

Ejemplos:

```text
database.execution.duration
database.execution.count
database.execution.failure
database.execution.timeout
database.execution.cancelled
database.connection.acquire.duration
database.statement.prepare.duration
database.statement.execute.duration
database.result.rows
database.result.bytes
```

---

# 206. Telemetry ≠ execution policy

Observers no podrán cambiar el resultado.

---

# 207. No mandatory OpenTelemetry

La integración real llegará mediante:

```text
DATABASE_TELEMETRY_ARCHITECTURE
DATABASE_QUERY_TELEMETRY_SYSTEM
```

---

# 208. Execution trace

Debug mode podrá construir:

```text
ExecutionTrace
```

---

# 209. Example trace

```text
Execution #E14
────────────────────────────

00.000 ms  execution.start
00.080 ms  connection.acquire.start
00.510 ms  connection.acquire.complete
00.550 ms  statement.prepare.start
00.820 ms  statement.prepare.complete
00.840 ms  parameter.bind.start
00.910 ms  parameter.bind.complete
00.930 ms  statement.execute.start
03.420 ms  statement.execute.complete
03.500 ms  cursor.open
04.800 ms  cursor.close
04.910 ms  cleanup.complete
```

---

# 210. Sensitive trace data

No deberá incluir valores sensibles salvo opt-in explícito y policy segura.

---

# 211. Security boundary

Execution Engine representa una frontera crítica porque maneja valores runtime reales.

---

# 212. Security rules

```text
compiled SQL
+
driver bindings
```

deberán permanecer separados.

---

# 213. No SQL interpolation

Incluso si un driver no soporta cierto binding, la solución no será automáticamente concatenar el valor.

---

# 214. Identifier runtime input

Dynamic identifiers deberán haberse convertido en estructuras validadas antes de compilación.

No deberán llegar como runtime binding ordinario.

---

# 215. Raw SQL

Raw SQL seguirá sujeto a:

```text
explicit trust boundary
parameter binding
security metadata
execution policy
```

---

# 216. Multi-statements

Deshabilitados por default para commands normales.

---

# 217. Dangerous operations

Operaciones como:

```text
LOAD DATA
INTO OUTFILE
administrative commands
filesystem-access SQL
```

deberán requerir capabilities/security contracts explícitos.

---

# 218. Tenant security

Execution Engine no decide qué tenant corresponde.

Consume:

```text
ExecutionTenantContext
```

ya resuelto.

---

# 219. Tenant connection routing

Puede pasar el tenant-aware routing requirement al Connection Manager.

---

# 220. No global current tenant

Especialmente bajo persistent workers.

---

# 221. Authorization

Mandatory authorization/security predicates ya deberán estar incorporados en el artifact.

Execution Engine sólo verifica requirements/provenance cuando sea necesario.

---

# 222. No predicate dropping

Nunca podrá retirar:

```text
tenant predicate
authorization predicate
security filter
```

para "optimizar" ejecución.

---

# 223. Execution budgets

Cada ejecución deberá ser bounded.

---

# 224. ExecutionBudget

```php
final readonly class ExecutionBudget
{
    public function __construct(
        public int $maxStatements,
        public int $maxConnections,
        public int $maxOpenCursors,
        public int $maxBufferedBytes,
        public int $maxTemporaryBytes,
        public int $maxFrameworkOperations,
    ) {}
}
```

---

# 225. Budget + deadline

```text
Resource Budget
+
Time Deadline
```

son controles complementarios.

---

# 226. Budget exhaustion

Debe producir:

```text
ExecutionResourceLimitException
```

o equivalente.

---

# 227. No partial silent result

Si el budget se agota a mitad de una consulta:

```text
partial result
```

no deberá presentarse como resultado completo.

---

# 228. Partial results

Sólo podrán existir si el result contract los permite explícitamente.

---

# 229. Streaming failure

Si falla después de entregar 500 filas:

```text
stream failure
```

deberá ser observable.

No podrá fingirse EOF normal.

---

# 230. Streaming semantics

```text
EOF
≠
ERROR
≠
CANCELLED
≠
TIMED_OUT
```

---

# 231. Result stream state

```text
OPEN
EXHAUSTED
FAILED
CANCELLED
TIMED_OUT
CLOSED
```

---

# 232. ExecutionOutcome

Para operaciones no streaming:

```php
final readonly class ExecutionOutcome
{
    public function __construct(
        public ExecutionOutcomeStatus $status,
        public ?ExecutionResult $result,
        public ExecutionMetadata $metadata,
    ) {}
}
```

---

# 233. Outcome statuses

```php
enum ExecutionOutcomeStatus
{
    case SUCCESS;
    case FAILURE;
    case CANCELLED;
    case TIMED_OUT;
    case UNKNOWN;
}
```

---

# 234. Exception vs outcome

La API pública podrá usar exceptions para failures.

Internamente conviene preservar:

```text
structured outcome
```

para cleanup, telemetry y retries.

---

# 235. UNKNOWN

`UNKNOWN` es importante para operaciones cuyos efectos no pueden determinarse con certeza.

---

# 236. Execution metadata

Podrá contener:

```text
operation ID
duration
connection role
statement count
row count
affected rows
retry count
resource usage
cache metadata
```

sin exponer secretos.

---

# 237. Execution IDs

```text
ExecutionOperationId
```

es operativo.

No deberá afectar semantics/fingerprints.

---

# 238. Determinism

Execution runtime no puede garantizar idénticos resultados para queries dependientes de:

```text
current database state
volatile functions
concurrency
time
```

Pero sí deberá garantizar:

```text
same plan execution protocol
```

bajo el mismo contexto.

---

# 239. Volatility

Execution Engine no reevalúa clasificación de volatilidad.

La respeta.

---

# 240. Ordering

Si el result contract garantiza ordering:

```text
Execution Engine
```

deberá preservarlo.

---

# 241. No result reordering

Framework buffering/concurrency no podrá cambiar orden observable salvo que el plan lo permita.

---

# 242. Duplicate preservation

Bag semantics deberán preservarse.

---

# 243. NULL preservation

Driver result normalization deberá mantener:

```text
SQL NULL
```

como valor distinguible según el Type System.

---

# 244. Driver value normalization

Valores driver-specific deberán pasar por:

```text
Database Type System
```

cuando corresponda.

---

# 245. Raw driver values

Execution Engine podrá producir una representación interna de fila previa a hydration.

---

# 246. DatabaseRow

Conceptualmente:

```php
final readonly class DatabaseRow
{
    /** @var list<DatabaseValue> */
    private array $values;
}
```

---

# 247. Positional identity

Internamente será preferible conservar:

```text
column position
+
compiled output identity
```

en lugar de depender únicamente de labels textuales.

---

# 248. Duplicate aliases

SQL puede producir nombres repetidos.

El Result System deberá poder representarlos.

---

# 249. Internal alias mapping

`CompiledResultContract` proporciona la correspondencia segura.

---

# 250. Result shape validation

Cuando sea viable, el engine deberá verificar que el resultado driver sea compatible con el compiled result contract.

---

# 251. Result mismatch

Un mismatch puede indicar:

```text
stale compiled artifact
driver incompatibility
schema drift
compiler bug
```

y deberá tratarse como invariant/compatibility failure.

---

# 252. Statement preparation cache

Live statement reuse será connection-scoped.

---

# 253. Live statement cache

Conceptualmente:

```text
Connection
└── Statement Cache
    ├── Statement A
    ├── Statement B
    └── Statement C
```

---

# 254. Not global

No:

```text
Application
└── Global PDOStatement Cache
```

---

# 255. Statement reset

Antes de reuse deberán limpiarse:

```text
previous bindings
cursor state
driver result state
errors
```

según driver contract.

---

# 256. Non-reusable statements

Algunos statements deberán descartarse después de uso.

---

# 257. Statement reuse capability

Será explícita.

---

# 258. Connection failure

Si una conexión muere:

```text
statement handles
cursors
transaction state
```

asociados quedan invalidados.

---

# 259. Failure fan-out

```text
Connection Failure
      ↓
Invalidate Statements
      ↓
Invalidate Cursors
      ↓
Mark Transaction Failed/Unknown
      ↓
Execution Failure
```

---

# 260. Connection replacement

Una conexión nueva no convierte automáticamente un old statement en reusable.

---

# 261. Transaction affinity

Statements dentro de una transacción deberán permanecer en la conexión correspondiente.

---

# 262. Connection affinity

ExecutionPlan podrá expresar:

```text
same connection required
```

para determinadas unidades.

---

# 263. Session affinity

También podrá existir:

```text
same database session required
```

cuando existan temporary/session-level constructs.

---

# 264. Affinity ≠ global pinning

Debe limitarse al execution/transaction scope.

---

# 265. Recursive execution

ExecutionPlan puede contener `RecursiveExecutionRegion`.

---

# 266. No arbitrary graph cycles

Execution Scheduler no deberá aceptar ciclos arbitrarios.

---

# 267. Recursive region

La recursión tendrá:

```text
entry
iteration contract
termination condition
resource budget
exit
```

explícitos.

---

# 268. Recursion budget

Debe existir protección contra recursión no acotada cuando el framework controle la operación.

---

# 269. DB-controlled recursion

Si un recursive CTE se delega al database:

```text
DatabaseExecutionUnit
```

lo maneja como un statement normal.

---

# 270. Framework-controlled recursion

Sólo si el PhysicalPlan la seleccionó explícitamente.

---

# 271. Correlated execution

Algunos subplans pueden requerir:

```text
PER_OUTER_ROW
```

---

# 272. Execution multiplicity

Preservar:

```text
ONCE
PER_OUTER_ROW
REUSABLE
MATERIALIZED_ONCE
RECURSIVE
DELEGATED
```

---

# 273. Multiplicity ≠ cardinality

Un subplan ejecutado una vez puede devolver millones de filas.

---

# 274. Correlation binding

Outer values deberán entrar mediante:

```text
CorrelationBindingSet
```

estructurado.

---

# 275. No string substitution

Correlated execution tampoco interpolará valores en SQL.

---

# 276. Materialization

Framework materialization deberá estar explícita en el plan.

---

# 277. Materialization ≠ cache

```text
Execution Materialization
```

vive durante una ejecución.

No es:

```text
Query Result Cache
```

---

# 278. Materialization lifecycle

```text
create
fill
consume
release
```

---

# 279. Materialization scope

Normalmente:

```text
ExecutionInstance
```

---

# 280. Instrumentation overhead

Instrumentation deberá tener bounded overhead.

---

# 281. Fast path

Producción podrá usar:

```text
minimal execution instrumentation
```

---

# 282. Debug path

Debug podrá activar:

```text
detailed trace
resource ledger
binding metadata
source mapping
stage timings
```

con redaction.

---

# 283. Execution profiles

Podrán existir:

```php
enum ExecutionProfile
{
    case PRODUCTION;
    case DEBUG;
    case VALIDATION_HEAVY;
    case BENCHMARK;
}
```

---

# 284. Profile restriction

ExecutionProfile puede modificar:

```text
diagnostics
validation intensity
telemetry detail
```

pero no query semantics.

---

# 285. Testing architecture

El Execution Engine requerirá pruebas amplias.

---

# 286. Unit tests

Para:

```text
state machine
resource ownership
binding
deadline
cancellation
cleanup
result adaptation
retry eligibility
```

---

# 287. Integration tests

Contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 288. Driver conformance tests

Cada driver deberá demostrar:

```text
prepare
bind
execute
fetch
cursor close
statement close
connection failure
transaction behavior
error mapping
```

---

# 289. Failure injection

Deberán simularse fallos en:

```text
connection acquisition
prepare
bind
execute
first fetch
middle fetch
cursor close
statement release
connection release
transaction commit
```

---

# 290. Cleanup tests

Para cada failure point:

```text
all acquired resources
```

deberán quedar correctamente liberados o transferidos.

---

# 291. Cancellation tests

Cancelar durante:

```text
connection acquisition
preparation
execution
streaming
framework operation
cleanup
```

---

# 292. Timeout tests

Igualmente para cada etapa.

---

# 293. Persistent runtime tests

Múltiples requests sobre el mismo worker deberán demostrar:

```text
no binding leakage
no cursor leakage
no transaction leakage
no tenant leakage
no deadline leakage
no cancellation leakage
```

---

# 294. Concurrency tests

Especialmente para futuras integraciones OpenSwoole:

```text
Execution A
Execution B
Execution C
```

simultáneas.

---

# 295. Streaming tests

Casos:

```text
complete consumption
partial consumption
consumer exception
driver failure mid-stream
cancellation
timeout
explicit close
double close
```

---

# 296. Property tests

Propiedad:

```text
Every acquired resource
must eventually be
released
or
explicitly transferred
```

---

# 297. Resource conservation invariant

Formalmente:

```text
AcquiredResources
=
ReleasedResources
∪
TransferredResources
```

al terminar el ownership scope.

---

# 298. Performance tests

Medir:

```text
execution overhead
binding overhead
statement acquisition
cursor overhead
streaming throughput
memory usage
cleanup overhead
telemetry overhead
```

---

# 299. Zero/low overhead goal

Para un simple:

```sql
SELECT id FROM users WHERE id = ?
```

la arquitectura no deberá introducir una cadena excesivamente costosa de allocations virtuales en hot path.

---

# 300. Architecture vs implementation

La existencia conceptual de:

```text
Validator
Coordinator
Binder
ResourceGovernor
```

no obliga a crear docenas de objetos por query.

---

# 301. Compile-time composition

Muchos componentes podrán resolverse durante bootstrap y permanecer frozen.

---

# 302. Hot path specialization

El runtime podrá usar implementaciones optimizadas siempre que preserve contracts.

---

# 303. Suggested namespace

```text
VoltStack\Quantum\Database\Execution
```

---

# 304. Directory structure

```text
Execution/
├── Contract/
│   ├── ExecutionEngine.php
│   ├── QueryExecutor.php
│   ├── StatementExecutor.php
│   └── ExecutionUnitExecutor.php
│
├── Request/
│   ├── ExecutionRequest.php
│   └── ExecutionRequestValidator.php
│
├── Context/
│   ├── ExecutionContext.php
│   ├── ExecutionOperationId.php
│   └── ExecutionProfile.php
│
├── Instance/
│   ├── ExecutionInstance.php
│   ├── ExecutionInstanceFactory.php
│   ├── ExecutionState.php
│   └── ExecutionStateMachine.php
│
├── Coordinator/
│   ├── ExecutionCoordinator.php
│   ├── FailureCoordinator.php
│   └── CleanupCoordinator.php
│
├── Scheduler/
│   ├── ExecutionScheduler.php
│   ├── ExecutionStageRuntime.php
│   └── ExecutionUnitRuntime.php
│
├── Connection/
│   ├── ConnectionLeaseManager.php
│   ├── ConnectionRequirement.php
│   └── ConnectionAffinity.php
│
├── Statement/
│   ├── StatementManager.php
│   ├── StatementLease.php
│   ├── StatementExecutionContext.php
│   └── StatementReusePolicy.php
│
├── Binding/
│   ├── RuntimeBindingSet.php
│   ├── RuntimeBinding.php
│   ├── RuntimeParameterBinder.php
│   ├── DriverBindingSet.php
│   └── BindingSensitivity.php
│
├── Result/
│   ├── ExecutionResult.php
│   ├── ExecutionResultKind.php
│   ├── ExecutionOutcome.php
│   ├── ExecutionOutcomeStatus.php
│   ├── DatabaseRow.php
│   └── DriverResultAdapter.php
│
├── Cursor/
│   ├── ResultCursor.php
│   ├── CursorManager.php
│   ├── StreamingResult.php
│   └── CursorState.php
│
├── Resource/
│   ├── ExecutionResourceOwner.php
│   ├── ExecutionResourceLedger.php
│   ├── ExecutionResourceGovernor.php
│   ├── ExecutionResourceRequest.php
│   ├── ExecutionResourceReservation.php
│   └── MemoryReservation.php
│
├── Cancellation/
│   ├── CancellationToken.php
│   ├── CancellationSource.php
│   └── CancellationCoordinator.php
│
├── Timeout/
│   ├── ExecutionDeadline.php
│   ├── DeadlineManager.php
│   └── MonotonicClock.php
│
├── Transaction/
│   ├── TransactionExecutionCoordinator.php
│   ├── TransactionExecutionContext.php
│   └── TransactionOwnership.php
│
├── Retry/
│   ├── RetryCoordinator.php
│   ├── RetryEligibilityDecider.php
│   ├── RetryExecutionContext.php
│   └── RetryDecision.php
│
├── Failure/
│   ├── ExecutionFailure.php
│   ├── ExecutionFailureCategory.php
│   └── SuppressedExecutionFailure.php
│
├── Runtime/
│   ├── DatabaseRuntimeAdapter.php
│   ├── FrankenPhpDatabaseRuntimeAdapter.php
│   ├── RoadRunnerDatabaseRuntimeAdapter.php
│   └── OpenSwooleDatabaseRuntimeAdapter.php
│
├── Instrumentation/
│   ├── ExecutionObserver.php
│   ├── ExecutionEvent.php
│   └── ExecutionTrace.php
│
├── Extension/
│   ├── ExecutionExtension.php
│   ├── ExecutionExtensionRegistry.php
│   └── ExecutionExtensionDescriptor.php
│
└── Exception/
    ├── DatabaseExecutionException.php
    ├── ExecutionValidationException.php
    ├── ExecutionStateException.php
    ├── ConnectionAcquisitionException.php
    ├── StatementPreparationException.php
    ├── ParameterBindingException.php
    ├── StatementExecutionException.php
    ├── ResultFetchException.php
    ├── ExecutionCancelledException.php
    ├── ExecutionTimeoutException.php
    ├── ExecutionResourceLimitException.php
    ├── ExecutionCleanupException.php
    ├── ExecutionOutcomeUnknownException.php
    └── ExecutionInvariantException.php
```

---

# 305. Dependency rules

Permitido:

```text
Execution
   ↓
Connection Contracts
   ↓
Driver Contracts
```

También:

```text
Execution
   ↓
Transaction Contracts
```

No:

```text
Driver
   ↓
Execution Engine
```

---

# 306. ORM dependency

Prohibido:

```text
Execution Engine
    ↓
ORM EntityManager
```

---

# 307. HTTP dependency

Prohibido:

```text
Execution Engine
    ↓
HTTP Request
```

---

# 308. Telemetry dependency

Preferible:

```text
Execution
    ↓
Instrumentation Contract
```

y posteriormente:

```text
Telemetry Adapter
    ↓
Instrumentation Contract
```

---

# 309. Runtime dependency

Preferible:

```text
Execution
    ↓
Runtime Abstraction
```

No:

```text
if FrankenPHP
else if RoadRunner
else if Swoole
```

disperso por el engine.

---

# 310. Invariantes arquitectónicos

## DB-EXEC-001

Execution Engine ejecutará decisiones previamente tomadas.

## DB-EXEC-002

Execution Engine no realizará semantic analysis.

## DB-EXEC-003

Execution Engine no realizará query optimization.

## DB-EXEC-004

Execution Engine no realizará logical planning.

## DB-EXEC-005

Execution Engine no realizará physical planning.

## DB-EXEC-006

Execution Engine no generará SQL.

## DB-EXEC-007

Execution Engine no interpretará ORM entities.

## DB-EXEC-008

ExecutionPlan será distinto de ExecutionInstance.

## DB-EXEC-009

ExecutionPlan podrá ser reusable.

## DB-EXEC-010

ExecutionInstance será operation-scoped.

## DB-EXEC-011

Runtime bindings serán operation-scoped.

## DB-EXEC-012

Connections vivos no se almacenarán en reusable plans.

## DB-EXEC-013

Statements vivos no se almacenarán en reusable plans.

## DB-EXEC-014

Cursors vivos no se almacenarán en reusable plans.

## DB-EXEC-015

Transactions vivas no se almacenarán en reusable plans.

## DB-EXEC-016

Todo live resource tendrá owner explícito.

## DB-EXEC-017

Todo resource adquirido será liberado o transferido.

## DB-EXEC-018

Ownership transfer será explícito.

## DB-EXEC-019

Connection acquisition ocurrirá mediante Connection Manager.

## DB-EXEC-020

Execution Engine no construirá PDO directamente.

## DB-EXEC-021

ConnectionLease tendrá lifecycle explícito.

## DB-EXEC-022

Connection reuse requerirá state isolation.

## DB-EXEC-023

PreparedStatementBlueprint será distinto de PreparedStatement.

## DB-EXEC-024

Live statements serán connection-bound salvo capability explícita.

## DB-EXEC-025

No existirán global live statement handles.

## DB-EXEC-026

Runtime values no se escribirán en compiled artifacts.

## DB-EXEC-027

Runtime values no se escribirán en compiled cache.

## DB-EXEC-028

ParameterId será distinto de DriverBindingPosition.

## DB-EXEC-029

Runtime values usarán parameter binding.

## DB-EXEC-030

Execution Engine no concatenará runtime values a SQL.

## DB-EXEC-031

Sensitive bindings serán redactables.

## DB-EXEC-032

StatementExecutor será distinto de QueryExecutor.

## DB-EXEC-033

Framework execution sólo ocurrirá cuando el plan lo especifique.

## DB-EXEC-034

Execution Engine no inventará fallback PHP por capability faltante.

## DB-EXEC-035

ExecutionControlLevel será preservado.

## DB-EXEC-036

V1 favorecerá safe database delegation.

## DB-EXEC-037

Execution graph no será reconstruido por el runtime.

## DB-EXEC-038

Execution dependencies serán respetadas.

## DB-EXEC-039

Scheduler será runtime-neutral.

## DB-EXEC-040

Parallelizable no implicará ejecución paralela.

## DB-EXEC-041

ExecutionStage será distinto de transaction.

## DB-EXEC-042

ExecutionStage será distinto de connection.

## DB-EXEC-043

Driver result será adaptado a internal result contract.

## DB-EXEC-044

Execution result será distinto de ORM hydration.

## DB-EXEC-045

Streaming real no usará fetch-all como implementación oculta.

## DB-EXEC-046

Streaming cursors tendrán lifecycle explícito.

## DB-EXEC-047

Early stream termination deberá liberar recursos.

## DB-EXEC-048

Resource correctness no dependerá exclusivamente de destructors.

## DB-EXEC-049

Streaming soportará backpressure.

## DB-EXEC-050

V1 podrá utilizar pull-based streaming.

## DB-EXEC-051

Framework-controlled memory será bounded.

## DB-EXEC-052

Temporary resources tendrán ownership.

## DB-EXEC-053

Spill requerirá autorización del plan/capability.

## DB-EXEC-054

Cancellation será first-class.

## DB-EXEC-055

Cancellation será distinta de failure.

## DB-EXEC-056

Cancellation será distinta de timeout.

## DB-EXEC-057

Cancellation propagation respetará capabilities del driver.

## DB-EXEC-058

VoltStack no declarará cancelación server-side inexistente.

## DB-EXEC-059

Timeout será first-class.

## DB-EXEC-060

Deadlines usarán clock monotónico cuando sea posible.

## DB-EXEC-061

Global deadline limitará suboperation deadlines.

## DB-EXEC-062

Server timeout será distinto de client timeout.

## DB-EXEC-063

Timeout será distinto de generic failure.

## DB-EXEC-064

Driver errors conservarán native cause metadata.

## DB-EXEC-065

SQLSTATE será preservado cuando exista.

## DB-EXEC-066

Source maps podrán enriquecer execution errors.

## DB-EXEC-067

Source maps no inventarán precisión inexistente.

## DB-EXEC-068

Errors sensibles serán redactados.

## DB-EXEC-069

Failure propagation será distinta de cleanup.

## DB-EXEC-070

Cleanup ocurrirá tras success.

## DB-EXEC-071

Cleanup ocurrirá tras failure.

## DB-EXEC-072

Cleanup ocurrirá tras cancellation.

## DB-EXEC-073

Cleanup ocurrirá tras timeout.

## DB-EXEC-074

Cleanup respetará dependency ordering.

## DB-EXEC-075

Cleanup failures no ocultarán primary failure.

## DB-EXEC-076

Partial preparation failure liberará recursos previos.

## DB-EXEC-077

Resource ledger será operation-scoped.

## DB-EXEC-078

Execution Engine no inventará transaction boundaries.

## DB-EXEC-079

Transaction Manager será responsable de primitives transaccionales.

## DB-EXEC-080

Caller-owned transactions no serán committed por el engine.

## DB-EXEC-081

Engine-owned transaction tendrá ownership explícito.

## DB-EXEC-082

Cross-database atomicity no será asumida.

## DB-EXEC-083

Retries requerirán eligibility explícita.

## DB-EXEC-084

UNKNOWN idempotency será tratada conservadoramente.

## DB-EXEC-085

Writes no serán retryados ciegamente.

## DB-EXEC-086

Ambiguous commit será representable.

## DB-EXEC-087

Unknown execution outcome no será ocultado.

## DB-EXEC-088

Retry policy será bounded.

## DB-EXEC-089

Retry respetará global deadline.

## DB-EXEC-090

DML preservará mutation semantics.

## DB-EXEC-091

Affected rows serán platform/driver normalized.

## DB-EXEC-092

Generated IDs seguirán compiled result contract.

## DB-EXEC-093

Execution Engine no inventará post-write SELECT.

## DB-EXEC-094

Batches serán explícitos.

## DB-EXEC-095

Hidden SQL multi-statements estarán prohibidos.

## DB-EXEC-096

Batch failure semantics serán explícitas.

## DB-EXEC-097

Extensions tendrán contracts tipados.

## DB-EXEC-098

Extension registries serán frozen.

## DB-EXEC-099

Extension conflicts no usarán last-wins.

## DB-EXEC-100

Mutable extension state será execution-scoped.

## DB-EXEC-101

Extensions que adquieran recursos declararán cleanup.

## DB-EXEC-102

Extensions largas declararán cancellation behavior.

## DB-EXEC-103

V1 no intentará recrear un RDBMS completo en PHP.

## DB-EXEC-104

Execution Engine será runtime-neutral.

## DB-EXEC-105

FrankenPHP será runtime predeterminado.

## DB-EXEC-106

Persistent workers reutilizarán sólo safe shared state.

## DB-EXEC-107

ExecutionInstance no sobrevivirá request boundary.

## DB-EXEC-108

Bindings no sobrevivirán request boundary.

## DB-EXEC-109

Cursor state no sobrevivirá request boundary accidentalmente.

## DB-EXEC-110

Transaction context no sobrevivirá request boundary.

## DB-EXEC-111

Tenant runtime state no sobrevivirá request boundary.

## DB-EXEC-112

Cancellation state no sobrevivirá request boundary.

## DB-EXEC-113

Deadline state no sobrevivirá request boundary.

## DB-EXEC-114

No habrá static current execution.

## DB-EXEC-115

RoadRunner deberá poder usar la misma architecture.

## DB-EXEC-116

OpenSwoole deberá poder usar la misma architecture.

## DB-EXEC-117

Concurrent executions permanecerán aisladas.

## DB-EXEC-118

Telemetry será observacional.

## DB-EXEC-119

OpenTelemetry no será mandatory dependency.

## DB-EXEC-120

Per-row telemetry estará bounded.

## DB-EXEC-121

Sensitive telemetry será redactada.

## DB-EXEC-122

Execution Engine manejará runtime values como security boundary.

## DB-EXEC-123

Runtime identifiers no serán values ordinarios.

## DB-EXEC-124

Raw SQL mantendrá explicit trust boundary.

## DB-EXEC-125

Multi-statements estarán deshabilitados por default.

## DB-EXEC-126

Dangerous database operations requerirán capabilities explícitas.

## DB-EXEC-127

Execution Engine no resolverá current tenant globalmente.

## DB-EXEC-128

Tenant routing será explícito.

## DB-EXEC-129

Security predicates no podrán eliminarse.

## DB-EXEC-130

Execution resources serán bounded.

## DB-EXEC-131

Budget exhaustion será observable.

## DB-EXEC-132

Partial result no se presentará como complete result.

## DB-EXEC-133

Streaming EOF será distinto de streaming failure.

## DB-EXEC-134

Streaming cancellation será observable.

## DB-EXEC-135

Streaming timeout será observable.

## DB-EXEC-136

ExecutionOutcome podrá representar UNKNOWN.

## DB-EXEC-137

Execution IDs no afectarán semantics.

## DB-EXEC-138

Observable ordering será preservado.

## DB-EXEC-139

Duplicate semantics serán preservadas.

## DB-EXEC-140

SQL NULL semantics serán preservadas.

## DB-EXEC-141

Result columns no dependerán únicamente de textual labels.

## DB-EXEC-142

Duplicate output labels serán representables.

## DB-EXEC-143

Compiled result contract podrá validar driver output.

## DB-EXEC-144

Live statement cache será connection-scoped.

## DB-EXEC-145

Statement reuse requerirá reset adecuado.

## DB-EXEC-146

Connection failure invalidará dependent statements.

## DB-EXEC-147

Connection failure invalidará dependent cursors.

## DB-EXEC-148

Connection failure podrá afectar transaction certainty.

## DB-EXEC-149

Transaction affinity será preservada.

## DB-EXEC-150

Session affinity será execution-scoped.

## DB-EXEC-151

Arbitrary execution graph cycles estarán prohibidos.

## DB-EXEC-152

Recursion será estructurada.

## DB-EXEC-153

Framework-controlled recursion será bounded.

## DB-EXEC-154

Execution multiplicity será distinta de row cardinality.

## DB-EXEC-155

Correlation bindings serán estructurados.

## DB-EXEC-156

Materialization será distinta de persistent cache.

## DB-EXEC-157

Materialization lifetime será explícito.

## DB-EXEC-158

Execution profiles no cambiarán semantics.

## DB-EXEC-159

Every acquired resource shall be released or transferred.

## DB-EXEC-160

Execution shall preserve the semantics of the ExecutionPlan.

---

# 311. Invariante maestro de recursos

Al finalizar un ownership scope:

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

Ningún recurso deberá quedar sin ownership.

---

# 312. Invariante maestro de estado

```text
Reusable Execution Knowledge
∩
Mutable Runtime State
=
∅
```

---

# 313. Invariante maestro de ejecución

```text
Execution Engine
may orchestrate execution

Execution Engine
must not reinterpret query meaning
```

---

# 314. Invariante de failure safety

```text
Failure
+
Cancellation
+
Timeout
```

no deberán impedir:

```text
Resource Cleanup
```

salvo recursos cuyo ownership haya sido transferido explícitamente.

---

# 315. Invariante de retry safety

```text
Retry
requires
Provable Retry Eligibility
```

Nunca:

```text
Failure
→ automatically execute again
```

---

# 316. Invariante de streaming

```text
Streaming Result
=
Result Contract
+
Cursor Ownership
+
Backpressure
+
Failure Semantics
+
Cancellation
+
Cleanup
```

No:

```text
fetchAll() + generator
```

---

# 317. Invariante persistent-runtime

```text
Shared Across Requests
=
Immutable / Frozen / Stateless
```

```text
Scoped Per Execution
=
Connections
Statements
Bindings
Cursors
Transactions
Deadlines
Cancellation
Tenant State
Mutable Buffers
```

---

# 318. Flujo completo de ejecución simple

```text
ExecutionRequest
      ↓
Validate
      ↓
Create ExecutionInstance
      ↓
Acquire ConnectionLease
      ↓
Acquire / Prepare Statement
      ↓
Resolve Runtime Bindings
      ↓
Convert Driver Values
      ↓
Bind
      ↓
Execute
      ↓
Adapt Driver Result
      ↓
Produce ExecutionResult
      ↓
Cleanup / Transfer Resources
      ↓
ExecutionOutcome
```

---

# 319. Flujo streaming

```text
ExecutionRequest
      ↓
Validate
      ↓
ExecutionInstance
      ↓
ConnectionLease
      ↓
StatementLease
      ↓
Bind
      ↓
Execute
      ↓
Open Cursor
      ↓
Transfer Cursor Ownership
      ↓
StreamingResult
      ↓
Consumer
      ├── fetch
      ├── fetch
      ├── fetch
      └── close
             ↓
        Cursor Cleanup
             ↓
        Statement Release
             ↓
        Connection Release
```

---

# 320. Flujo de error

```text
Execution
    ↓
Statement Failure
    ↓
Normalize Error
    ↓
Mark Execution FAILED
    ↓
Cancel Dependent Units
    ↓
Cleanup Cursor
    ↓
Release Statement
    ↓
Reset/Release Connection
    ↓
Release Resources
    ↓
Attach Suppressed Cleanup Errors
    ↓
Propagate Primary Failure
```

---

# 321. Flujo de timeout

```text
Deadline Reached
      ↓
Mark TIMED_OUT
      ↓
Request Driver Cancellation
      ↓
Stop Future Work
      ↓
Wait/Abort According To Capability
      ↓
Cleanup
      ↓
Timeout Outcome
```

---

# 322. Flujo de retry futuro

```text
Execution Failure
      ↓
Classify Failure
      ↓
Determine Side-Effect Certainty
      ↓
Check Idempotency
      ↓
Check Transaction State
      ↓
Check Deadline
      ↓
Check Retry Budget
      ↓
┌──────────────┐
│ Retry Safe?  │
└──────────────┘
   │        │
  yes       no
   │        │
   ▼        ▼
Retry     Propagate
```

---

# 323. Relación con documentos del bloque

Este documento establece el marco general.

Los siguientes documentos especializarán:

```text
76 Execution Engine Architecture
        ↓
77 Query Executor System
        ↓
78 Statement Execution System
        ↓
79 Prepared Statement System
        ↓
80 Parameter Binding System
        ↓
81 Result System
        ↓
82 Result Cursor System
        ↓
83 Streaming Result System
        ↓
84 Query Timeout and Cancellation System
        ↓
85 Execution Error System
        ↓
86 Execution Retry System
```

---

# 324. Límites documentales

`76` define:

```text
architecture
boundaries
ownership
runtime model
main contracts
cross-cutting invariants
```

`77–86` definirán el comportamiento detallado de cada subsistema.

---

# 325. Arquitectura resumida

```text
                  COMPILED WORLD
─────────────────────────────────────────────────

ExecutionPlan
CompiledDatabaseCommand
PreparedStatementBlueprint
CompiledBindingLayout
CompiledResultContract

══════════════ EXECUTION BOUNDARY ═══════════════

                  RUNTIME WORLD
─────────────────────────────────────────────────

ExecutionRequest
       ↓
ExecutionInstance
       ↓
Execution Scheduler
       ↓
Connection Lease
       ↓
Prepared Statement
       ↓
Runtime Bindings
       ↓
Driver Execution
       ↓
Result / Cursor
       ↓
ExecutionOutcome

Cross-cutting:

Transaction
Cancellation
Timeout
Resource Governance
Failure Handling
Cleanup
Security
Tenant Isolation
Telemetry
Runtime Isolation
```

---

# 326. Fórmula arquitectónica final

```text
Database Execution Engine
=
Execution Plan Consumption
+
Runtime Context
+
Execution Instance
+
Resource Ownership
+
Connection Acquisition
+
Statement Acquisition
+
Runtime Parameter Binding
+
Driver Invocation
+
Result Production
+
Cursor Lifecycle
+
Streaming
+
Backpressure
+
Transaction Coordination
+
Cancellation
+
Timeout
+
Failure Propagation
+
Retry Boundaries
+
Deterministic Cleanup
+
Resource Governance
+
Security Boundaries
+
Tenant Isolation
+
Instrumentation
+
Persistent Runtime Safety
```

---

# 327. Principio final

La arquitectura deberá mantener siempre esta separación:

```text
Planner
    decides how execution should happen

Compiler
    produces executable database representation

Execution Engine
    orchestrates that execution

Connection
    owns database session state

Driver
    communicates with the database

Result System
    represents database output

ORM
    interprets that output as entities when required
```

Por tanto:

> **The Execution Engine owns runtime orchestration, not query meaning.**

---

# 328. Siguiente documento

```text
77_DATABASE_QUERY_EXECUTOR_SYSTEM.md
```

Este documento deberá profundizar en el componente que consume `ExecutionPlan` y coordina:

```text
QueryExecutor
├── ExecutionRequest
├── ExecutionInstance
├── Plan Traversal
├── Stage Execution
├── Execution Unit Dispatch
├── Database Unit Execution
├── Framework Unit Execution
├── Dependency Scheduling
├── Correlation
├── Materialization
├── Result Propagation
├── Failure Propagation
├── Cancellation Propagation
├── Resource Ownership
└── Final ExecutionOutcome
```

sin invadir las responsabilidades específicas de:

```text
Statement Execution
Prepared Statements
Parameter Binding
Results
Cursors
Streaming
Timeout
Cancellation
Errors
Retries
```

que serán definidas en los documentos `78–86`.