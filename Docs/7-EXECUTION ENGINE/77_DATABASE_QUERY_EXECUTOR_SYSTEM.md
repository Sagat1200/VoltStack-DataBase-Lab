# 77_DATABASE_QUERY_EXECUTOR_SYSTEM.md

# VoltStack Quantum Database
## Query Executor System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 77 — Query Executor System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`QueryExecutor` es el componente responsable de coordinar la ejecución runtime de un `ExecutionPlan`.

Su responsabilidad principal es transformar:

```text
ExecutionRequest
+
ExecutionPlan
+
RuntimeBindingSet
+
ExecutionContext
```

en:

```text
ExecutionOutcome
```

mediante la ejecución ordenada de las unidades y stages definidos previamente por el `Execution Plan System`.

Formalmente:

```text
ExecuteQuery(
    ExecutionRequest
)
    ↓
ExecutionOutcome
```

El `QueryExecutor` no decide qué significa una query, cómo optimizarla, qué estrategia física utilizar ni cómo generar SQL.

Su función es:

> **orquestar correctamente un plan de ejecución ya resuelto.**

---

# 2. Principio fundamental

```text
QueryExecutor
=
Execution Plan Orchestrator
```

No:

```text
QueryExecutor
=
Query Engine
+
Planner
+
SQL Compiler
+
Driver
```

Principio:

> **The QueryExecutor follows the ExecutionPlan; it does not redesign it.**

---

# 3. Posición arquitectónica

```text
Query AST
   ↓
Semantic Engine
   ↓
Optimizer
   ↓
Logical Plan
   ↓
Physical Plan
   ↓
Execution Plan
   ↓
SQL Compiler
   ↓
CompiledDatabaseCommand
   ↓
PreparedStatementBlueprint
   ↓
══════════════════════════════
       EXECUTION RUNTIME
══════════════════════════════
   ↓
QueryExecutor
   ↓
Execution Scheduler
   ↓
ExecutionUnitExecutor
   ├── DatabaseExecutionUnitExecutor
   ├── FrameworkExecutionUnitExecutor
   └── ExtensionExecutionUnitExecutor
   ↓
Statement Execution System
   ↓
Connection / Driver
   ↓
Database
```

---

# 4. Responsabilidad exacta

El `QueryExecutor` deberá coordinar:

```text
ExecutionRequest validation
ExecutionInstance creation
ExecutionPlan traversal
stage scheduling
unit scheduling
dependency resolution
runtime binding propagation
correlation propagation
materialization coordination
unit dispatch
result propagation
failure propagation
cancellation propagation
resource ownership
cleanup coordination
final outcome construction
```

---

# 5. No responsabilidades

El `QueryExecutor` no deberá:

```text
parse SQL
generate SQL
prepare SQL syntax
optimize predicates
reorder joins
select indexes
choose join algorithms
infer types
resolve columns
hydrate entities
track UnitOfWork
manage IdentityMap
implement transaction primitives
implement driver protocol
invent retries
```

---

# 6. QueryExecutor ≠ StatementExecutor

Distinción fundamental:

```text
QueryExecutor
    ↓
executes an ExecutionPlan
```

mientras:

```text
StatementExecutor
    ↓
executes one prepared database statement
```

Por tanto:

```text
QueryExecutor
≠
StatementExecutor
```

---

# 7. Ejemplo

Un `ExecutionPlan` podría contener:

```text
Stage 1
   ├── DatabaseUnit A
   └── DatabaseUnit B

Stage 2
   └── FrameworkMergeUnit C

Stage 3
   └── FrameworkLimitUnit D
```

El `QueryExecutor` coordina todo el plan.

`StatementExecutor` sólo participa dentro de:

```text
DatabaseUnit A
DatabaseUnit B
```

---

# 8. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Contract;

interface QueryExecutor
{
    public function execute(
        ExecutionRequest $request,
    ): ExecutionOutcome;
}
```

---

# 9. Contrato alternativo interno

Internamente puede utilizarse:

```php
interface ExecutionPlanExecutor
{
    public function execute(
        ExecutionPlan $plan,
        ExecutionInstance $instance,
    ): ExecutionOutcome;
}
```

La API pública podrá ocultar esta distinción.

---

# 10. Flujo principal

```text
ExecutionRequest
       ↓
Request Validation
       ↓
ExecutionInstanceFactory
       ↓
ExecutionInstance
       ↓
Preflight Checks
       ↓
ExecutionPlan Activation
       ↓
Stage Scheduler
       ↓
Execution Unit Dispatch
       ↓
Result Propagation
       ↓
Completion Validation
       ↓
Cleanup / Ownership Transfer
       ↓
ExecutionOutcome
```

---

# 11. QueryExecutor como coordinador

La implementación no deberá convertirse en:

```php
final class QueryExecutor
{
    public function execute(...)
    {
        // 3000 lines
    }
}
```

El componente deberá delegar responsabilidades especializadas.

---

# 12. Arquitectura interna

```text
QueryExecutor
│
├── ExecutionRequestValidator
├── ExecutionInstanceFactory
├── ExecutionPlanValidator
├── ExecutionScheduler
├── ExecutionStageExecutor
├── ExecutionUnitDispatcher
├── RuntimeDependencyResolver
├── RuntimeBindingResolver
├── CorrelationCoordinator
├── MaterializationCoordinator
├── ResultPropagationCoordinator
├── FailureCoordinator
├── CancellationCoordinator
├── ResourceCoordinator
├── CleanupCoordinator
└── ExecutionOutcomeFactory
```

---

# 13. ExecutionRequest

El QueryExecutor recibe:

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

---

# 14. Request immutability

`ExecutionRequest` deberá ser immutable.

El estado mutable deberá residir en:

```text
ExecutionInstance
```

---

# 15. ExecutionInstance creation

Flujo:

```text
ExecutionRequest
      ↓
ExecutionInstanceFactory
      ↓
ExecutionInstance
```

---

# 16. ExecutionInstance

Representa una ejecución concreta del plan.

```php
final class ExecutionInstance
{
    public function __construct(
        private readonly ExecutionOperationId $id,
        private readonly ExecutionPlan $plan,
        private ExecutionState $state,
        private readonly RuntimeBindingStore $bindings,
        private readonly ExecutionResourceLedger $resources,
        private readonly ExecutionRuntimeState $runtimeState,
    ) {}
}
```

---

# 17. Plan ≠ instance

```text
ExecutionPlan
=
reusable execution template
```

```text
ExecutionInstance
=
one mutable runtime execution
```

---

# 18. Ejemplo

```text
ExecutionPlan P1
   │
   ├── ExecutionInstance E1
   │      user_id = 10
   │
   ├── ExecutionInstance E2
   │      user_id = 20
   │
   └── ExecutionInstance E3
          user_id = 30
```

---

# 19. Runtime state

El `ExecutionInstance` podrá contener:

```text
stage states
unit states
runtime bindings
correlation bindings
materializations
resource leases
intermediate results
failure state
cancellation state
deadline state
execution metrics
```

---

# 20. Estado compartido prohibido

Nunca:

```php
static ?ExecutionInstance $current;
```

---

# 21. Persistent runtime safety

Cada ejecución deberá poseer su propio:

```text
ExecutionInstance
```

incluso dentro del mismo FrankenPHP worker.

---

# 22. Preflight

Antes de comenzar el plan:

```text
validate request
validate plan compatibility
validate required bindings
validate execution capabilities
validate extension availability
validate resource policy
validate transaction requirements
validate cancellation
validate deadline
```

---

# 23. Preflight before resources

Siempre que sea posible:

```text
validation
```

deberá ocurrir antes de adquirir:

```text
connections
statements
cursors
temporary resources
```

---

# 24. ExecutionPlan validation

El QueryExecutor puede verificar invariantes runtime como:

```text
root exists
stages are valid
dependencies are valid
units have executors
required parameters exist
required extensions exist
resource requirements are representable
```

No deberá volver a verificar semántica SQL.

---

# 25. Plan traversal

El QueryExecutor no deberá asumir que el plan es simplemente:

```text
recursive tree
```

El ExecutionPlan puede ser:

```text
directed execution graph
```

con regiones estructuradas.

---

# 26. Execution graph

```text
        Unit A
       /      \
      ▼        ▼
   Unit B    Unit C
      \        /
       ▼      ▼
        Unit D
```

---

# 27. Graph edges

Podrán existir:

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

# 28. Edge semantics

Cada edge deberá tener semántica explícita.

No deberá existir un genérico:

```text
dependsOn
```

para representar todas las relaciones.

---

# 29. DATA edge

Representa flujo de resultados:

```text
Unit A
  │ rows
  ▼
Unit B
```

---

# 30. CONTROL edge

Representa orden de ejecución sin necesariamente transferir datos.

```text
Unit A
   ↓ completed
Unit B
```

---

# 31. DEPENDENCY edge

Representa una dependencia requerida para activar una unidad.

---

# 32. PARAMETER edge

Representa producción/consumo de parámetros runtime.

---

# 33. CORRELATION edge

Representa dependencia de valores del outer execution scope.

---

# 34. MATERIALIZATION edge

Representa dependencia de un resultado materializado.

---

# 35. RESOURCE edge

Expresa requisitos/ownership de recursos.

---

# 36. CANCELLATION edge

Expresa propagación de cancelación.

---

# 37. CLEANUP edge

Expresa orden de liberación.

---

# 38. RECURSION edge

Sólo permitido dentro de una región recursiva estructurada.

---

# 39. Arbitrary cycles

Prohibidos:

```text
A → B → C → A
```

salvo que formen parte de una estructura recursiva declarada.

---

# 40. Stage model

El ExecutionPlan puede agrupar unidades en:

```text
ExecutionStage
```

---

# 41. Ejemplo

```text
ExecutionPlan
│
├── Stage 0
│   └── Unit A
│
├── Stage 1
│   ├── Unit B
│   └── Unit C
│
└── Stage 2
    └── Unit D
```

---

# 42. Stage purpose

Los stages pueden representar:

```text
execution boundaries
materialization boundaries
dependency barriers
delegation boundaries
resource boundaries
```

---

# 43. Stage ≠ transaction

Nunca inferir:

```text
one stage = one transaction
```

---

# 44. Stage ≠ thread

Tampoco:

```text
one stage = one thread
```

---

# 45. Stage state

```php
enum ExecutionStageState
{
    case PENDING;
    case READY;
    case RUNNING;
    case DRAINING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
}
```

---

# 46. Unit state

```php
enum ExecutionUnitState
{
    case PENDING;
    case BLOCKED;
    case READY;
    case RUNNING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case SKIPPED;
}
```

---

# 47. State storage

Estos estados viven en:

```text
ExecutionInstance
```

No dentro del immutable `ExecutionPlan`.

---

# 48. Execution scheduler

Contrato:

```php
interface ExecutionScheduler
{
    public function next(
        ExecutionInstance $instance,
    ): ExecutionScheduleDecision;
}
```

---

# 49. Scheduler responsibility

Determina:

```text
which units are READY
which units are BLOCKED
which stage may advance
whether execution has completed
```

---

# 50. Scheduler does not choose strategy

No decide:

```text
HashJoin vs NestedLoop
```

Eso pertenece al Physical Planner.

---

# 51. Scheduler does not compile

No genera SQL.

---

# 52. Deterministic scheduling

Para un mismo plan y mismas condiciones runtime:

```text
scheduler decisions
```

deberán ser deterministas cuando no exista concurrencia externa que lo impida.

---

# 53. Tie-breaking

Cuando varias unidades estén listas:

```text
Unit B
Unit C
```

deberá existir un criterio estable.

Por ejemplo:

```text
stage ordinal
+
unit ordinal
+
stable plan identity
```

---

# 54. No hash-map ordering

No depender de:

```text
incidental array/hash iteration order
```

como semántica de scheduling.

---

# 55. Sequential execution profile

VoltStack V1 podrá utilizar:

```text
SequentialExecutionScheduler
```

---

# 56. Sequential scheduler

```text
Ready Units
    ↓
Stable Order
    ↓
Execute First
    ↓
Update Dependencies
    ↓
Repeat
```

---

# 57. Future concurrent scheduler

Posteriormente:

```text
ConcurrentExecutionScheduler
```

podrá ejecutar unidades independientes simultáneamente.

---

# 58. Parallelizable flag

Un plan podrá declarar:

```text
parallelizable = true
```

pero el runtime podrá seguir ejecutándolo secuencialmente.

---

# 59. Parallel safety

Para ejecutar realmente en paralelo deberán cumplirse:

```text
no dependency conflict
resource availability
transaction compatibility
connection compatibility
runtime capability
result ordering guarantees
cancellation compatibility
```

---

# 60. Unit dispatcher

```php
interface ExecutionUnitDispatcher
{
    public function dispatch(
        ExecutionUnit $unit,
        ExecutionUnitRuntimeContext $context,
    ): ExecutionUnitOutcome;
}
```

---

# 61. Unit executor registry

```text
ExecutionUnitKind
      ↓
ExecutionUnitExecutorRegistry
      ↓
ExecutionUnitExecutor
```

---

# 62. Executor types

```text
DatabaseExecutionUnitExecutor
FrameworkExecutionUnitExecutor
ExtensionExecutionUnitExecutor
```

---

# 63. Typed dispatch

Preferible:

```text
DatabaseExecutionUnit
→ DatabaseExecutionUnitExecutor
```

No:

```php
if ($unit->type === 'db') {
    ...
}
```

disperso por todo el sistema.

---

# 64. Registry lifecycle

```text
bootstrap
   ↓
discover
   ↓
validate
   ↓
resolve conflicts
   ↓
freeze
```

---

# 65. No runtime mutation

Una request no podrá registrar nuevos unit executors.

---

# 66. No last-wins

Si dos ejecutores reclaman la misma unidad:

```text
ambiguous executor
```

deberá producir error.

---

# 67. DatabaseExecutionUnit

Representa trabajo que debe delegarse al database.

Conceptualmente:

```php
final readonly class DatabaseExecutionUnit implements ExecutionUnit
{
    public function __construct(
        public ExecutionUnitId $id,
        public CompiledDatabaseCommandId $command,
        public ConnectionRequirement $connection,
        public ResultRequirement $result,
    ) {}
}
```

---

# 68. Database unit execution

Flujo:

```text
DatabaseExecutionUnit
       ↓
Resolve Compiled Command
       ↓
Resolve Runtime Bindings
       ↓
Statement Execution System
       ↓
Driver
       ↓
Database
       ↓
ExecutionResult
```

---

# 69. Statement details

El QueryExecutor no deberá implementar directamente:

```text
prepare
bind
driver execute
fetch
```

Estos pertenecen a documentos `78–83`.

---

# 70. Database unit result

La unidad devolverá un:

```text
ExecutionUnitOutcome
```

estructurado.

---

# 71. FrameworkExecutionUnit

Representa trabajo seleccionado explícitamente para ejecución dentro de VoltStack.

---

# 72. Ejemplos futuros

```text
FrameworkFilter
FrameworkProjection
FrameworkSort
FrameworkHashAggregate
FrameworkHashJoin
FrameworkMerge
FrameworkLimit
FrameworkMaterialization
CrossShardMerge
FederatedUnion
```

---

# 73. Framework operator boundary

El QueryExecutor sólo despacha.

No deberá contener implementaciones como:

```php
private function hashJoin(...) {}
```

---

# 74. Framework executor registry

```text
FrameworkOperatorKind
        ↓
FrameworkOperatorExecutorRegistry
        ↓
FrameworkOperatorExecutor
```

---

# 75. V1 strategy

La mayoría de queries SQL deberán convertirse en:

```text
one or few DatabaseExecutionUnits
```

aprovechando el optimizador del motor SQL.

---

# 76. No accidental second RDBMS

VoltStack no deberá ejecutar en PHP operaciones que el planner decidió delegar al database.

---

# 77. ExtensionExecutionUnit

Permite incorporar operaciones adicionales sin modificar el core.

---

# 78. Extension contract

```php
interface ExtensionExecutionUnitExecutor
{
    public function supports(
        ExtensionExecutionUnit $unit,
    ): bool;

    public function execute(
        ExtensionExecutionUnit $unit,
        ExecutionUnitRuntimeContext $context,
    ): ExecutionUnitOutcome;
}
```

---

# 79. Extension constraints

Una extension deberá declarar:

```text
capabilities
resource requirements
cancellation behavior
cleanup behavior
failure behavior
result contract
thread/coroutine safety
```

---

# 80. ExecutionUnitRuntimeContext

```php
final readonly class ExecutionUnitRuntimeContext
{
    public function __construct(
        public ExecutionInstance $execution,
        public RuntimeBindingView $bindings,
        public RuntimeDependencyView $dependencies,
        public ExecutionResourceScope $resources,
        public ExecutionDeadline $deadline,
        public CancellationToken $cancellation,
        public ExecutionInstrumentationScope $instrumentation,
    ) {}
}
```

---

# 81. Narrow context

Preferir views/capabilities específicas en lugar de entregar acceso irrestricto al `ExecutionInstance`.

---

# 82. Runtime dependency resolver

```php
interface RuntimeDependencyResolver
{
    public function resolve(
        ExecutionUnit $unit,
        ExecutionInstance $instance,
    ): RuntimeDependencySet;
}
```

---

# 83. Dependency readiness

Una unidad estará `READY` sólo cuando se satisfagan sus dependencies.

---

# 84. Readiness equation

Conceptualmente:

```text
Ready(U)
=
RequiredDependencies(U)
⊆
SatisfiedDependencies
```

más:

```text
not cancelled
deadline valid
resources potentially obtainable
```

---

# 85. Data dependency

Si:

```text
A →DATA B
```

B no podrá ejecutarse hasta que el output necesario de A esté disponible.

---

# 86. Streaming dependency

En streaming, "available" no necesariamente significa:

```text
A completed
```

Puede significar:

```text
A opened stream
```

si el plan soporta pipelining.

---

# 87. Completion dependency ≠ stream availability

Deben modelarse separadamente.

---

# 88. Pipelining

Arquitectura futura:

```text
Database Unit
     ↓ rows
Framework Unit
     ↓ rows
Consumer
```

sin materializar completamente cada etapa.

---

# 89. Pipeline safety

Requiere:

```text
backpressure
resource ownership
cancellation
failure propagation
ordering guarantees
```

---

# 90. Intermediate result

El QueryExecutor puede gestionar:

```text
IntermediateExecutionResult
```

---

# 91. Intermediate result types

```text
scalar
row
row stream
buffered rows
materialization handle
affected rows
generated value
extension result
```

---

# 92. Intermediate results scope

Deberán vivir únicamente mientras sean necesarios.

---

# 93. Reference counting

Puede utilizarse un modelo como:

```text
remaining consumers
```

para liberar resultados intermedios cuando ya no tengan consumidores.

---

# 94. No implicit retention

No conservar todos los intermediate results hasta el final por defecto.

---

# 95. Result propagation

```text
Producer Unit
     ↓
ExecutionResultHandle
     ↓
Consumer Unit(s)
```

---

# 96. Result handle

Un handle permite evitar pasar objetos vivos arbitrariamente por el graph.

```php
interface ExecutionResultHandle
{
    public function resultId(): ExecutionResultId;
}
```

---

# 97. Result store

El `ExecutionInstance` podrá mantener:

```text
ExecutionResultStore
```

operation-scoped.

---

# 98. Result store ≠ cache

No deberá confundirse con:

```text
Query Result Cache
```

---

# 99. Result store lifetime

```text
ExecutionInstance lifetime
```

o menor.

---

# 100. Root result

El `ExecutionPlan` deberá identificar:

```text
root result producer
```

o output contract equivalente.

---

# 101. Completion

Un plan no está completo simplemente porque:

```text
all SQL statements were sent
```

Debe satisfacerse su:

```text
ExecutionOutputContract
```

---

# 102. Completion condition

Conceptualmente:

```text
PlanCompleted
=
RequiredUnitsTerminal
∧
RootOutputAvailable
∧
NoUnresolvedFailure
∧
RequiredCleanup/TransferSatisfied
```

---

# 103. Streaming completion

Un plan que devuelve un stream presenta dos niveles:

```text
execution handoff complete
```

y posteriormente:

```text
stream consumption complete
```

---

# 104. Handoff completion

El QueryExecutor puede devolver:

```text
StreamingResult
```

antes de que el database haya enviado todas las filas.

---

# 105. Ownership transfer

En ese momento deberá transferirse:

```text
cursor
statement lease
connection lease
```

o un composite owner equivalente.

---

# 106. ExecutionOutcome and stream

```text
ExecutionOutcome SUCCESS
```

en este contexto significa:

```text
stream successfully established and transferred
```

No necesariamente:

```text
all rows successfully consumed
```

---

# 107. Stream failures after handoff

Deben reportarse mediante el propio stream/cursor result contract.

---

# 108. Materialization coordinator

```php
interface MaterializationCoordinator
{
    public function materialize(
        MaterializationRequirement $requirement,
        ExecutionResultHandle $source,
        ExecutionInstance $instance,
    ): MaterializationHandle;
}
```

---

# 109. Materialization only when planned

El QueryExecutor no deberá decidir arbitrariamente:

```text
"this stream is inconvenient, buffer everything"
```

---

# 110. Materialization forms

Podrán existir:

```text
MEMORY
TEMPORARY_STORAGE
DATABASE_TEMPORARY
EXTENSION
```

según el Physical/Execution Plan.

---

# 111. Materialization lifetime

Deberá declararse:

```text
UNIT
STAGE
EXECUTION
TRANSFERRED
```

---

# 112. Materialization cleanup

Debe existir ownership explícito.

---

# 113. Correlation

El QueryExecutor deberá soportar subplanes correlacionados cuando el ExecutionPlan los contenga.

---

# 114. Correlation example

```text
Outer Unit
    ↓ row
Correlation Binding
    ↓
Inner Unit
```

---

# 115. CorrelationBindingSet

```php
final readonly class CorrelationBindingSet
{
    /**
     * @var array<OuterSymbolId, RuntimeBinding>
     */
    private array $bindings;
}
```

---

# 116. Semantic identity

La correlación deberá utilizar:

```text
OuterSymbolId
ParameterId
CorrelationSlotId
```

según corresponda.

No aliases textuales ambiguos.

---

# 117. Per-row execution

Un subplan puede declarar:

```text
PER_OUTER_ROW
```

---

# 118. Multiplicity

Valores posibles:

```text
ONCE
PER_OUTER_ROW
PER_BATCH
MATERIALIZED_ONCE
RECURSIVE
DELEGATED
```

---

# 119. Multiplicity ≠ scheduling guess

El QueryExecutor deberá obedecer la multiplicidad definida.

---

# 120. Correlated execution cost

El QueryExecutor no deberá decidir que ejecutar N veces es costoso y reoptimizar.

---

# 121. Runtime batching

Sólo podrá agrupar correlaciones si el plan lo permite explícitamente.

---

# 122. Recursion

Las regiones recursivas serán estructuras explícitas.

---

# 123. RecursiveExecutionRegion

```php
final readonly class RecursiveExecutionRegion
{
    public function __construct(
        public ExecutionUnitId $anchor,
        public ExecutionUnitId $recursiveMember,
        public RecursiveBindingContract $bindings,
        public RecursiveTerminationContract $termination,
        public RecursiveResourcePolicy $resources,
    ) {}
}
```

---

# 124. Recursive flow

```text
Anchor
  ↓
Seed Result
  ↓
Recursive Member
  ↓
Iteration Result
  ├── termination reached → exit
  └── continue → Recursive Member
```

---

# 125. DB delegated recursion

Cuando el recursive CTE esté dentro de un único SQL statement:

```text
QueryExecutor
```

no ejecuta las iteraciones.

El database lo hace.

---

# 126. Framework recursion

Sólo se utiliza si el ExecutionPlan lo seleccionó.

---

# 127. Recursion limits

Deberán existir:

```text
iteration limit
deadline
resource budget
result budget
```

para framework-controlled recursion.

---

# 128. Cancellation checkpoints

El QueryExecutor deberá comprobar cancellation en puntos definidos.

---

# 129. Checkpoints

Ejemplos:

```text
before execution starts
before stage starts
before unit starts
after unit completes
before materialization
during framework loops
before scheduling next unit
```

---

# 130. Driver cancellation

Mientras un statement está ejecutándose, la cancelación real se delegará al Statement/Driver layer.

---

# 131. Cancellation propagation

```text
Execution Cancelled
       ↓
Mark instance cancelling
       ↓
Stop scheduling new work
       ↓
Propagate to running units
       ↓
Cancel dependent units
       ↓
Cleanup
```

---

# 132. Cancellation state

Conviene distinguir:

```text
RUNNING
CANCELLING
CANCELLED
```

internamente, aunque el state machine público pueda simplificarse.

---

# 133. Timeout propagation

Timeout seguirá flujo parecido:

```text
deadline exceeded
      ↓
stop scheduling
      ↓
cancel active units
      ↓
cleanup
      ↓
TIMED_OUT
```

---

# 134. Cancellation ≠ timeout

El QueryExecutor deberá preservar la causa.

---

# 135. Failure propagation

Cuando una unidad falla:

```text
Unit Failure
    ↓
Failure Coordinator
    ↓
Determine dependent units
    ↓
Mark them blocked/cancelled
    ↓
Stop invalid future work
    ↓
Cleanup
```

---

# 136. Failure graph

La propagación deberá seguir dependencies reales.

---

# 137. Independent branches

En futuros modos best-effort:

```text
A fails

B independent
```

podría permitirse continuar B si el plan lo especifica.

---

# 138. Default policy

Para query execution normal:

```text
fail fast
```

será la política preferida.

---

# 139. Fail-fast ≠ skip cleanup

Siempre deberá ejecutarse cleanup.

---

# 140. Failure categories

El QueryExecutor consume categorías estructuradas provenientes de unidades.

Ejemplos:

```text
CONNECTION
PREPARATION
BINDING
EXECUTION
FETCH
TIMEOUT
CANCELLATION
RESOURCE
TRANSACTION
EXTENSION
INVARIANT
UNKNOWN
```

---

# 141. Primary failure

El primer error causal relevante deberá preservarse como:

```text
PrimaryExecutionFailure
```

---

# 142. Secondary failures

Podrán registrarse:

```text
SuppressedExecutionFailure
```

para cleanup/cancellation/follow-up failures.

---

# 143. QueryExecutor no normaliza SQLSTATE

La normalización detallada del driver pertenece a:

```text
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
```

---

# 144. Retry boundary

El QueryExecutor podrá participar en retries, pero no decidirlos unilateralmente.

---

# 145. Retry Coordinator

```text
QueryExecutor
    ↓ failure
RetryCoordinator
    ↓
RetryDecision
```

---

# 146. Retry decision

Puede ser:

```text
DO_NOT_RETRY
RETRY_EXECUTION
RETRY_STAGE
RETRY_UNIT
RECONNECT_AND_RETRY
OUTCOME_UNKNOWN
```

aunque las variantes finales serán definidas en doc 86.

---

# 147. Retry scope safety

No todo plan puede reiniciarse desde cualquier punto.

---

# 148. Example

Si:

```text
Unit A = INSERT
Unit B = SELECT
```

y B falla, no puede simplemente reiniciarse todo el plan si A pudo haber sido committed.

---

# 149. Retry checkpoints

El ExecutionPlan podrá declarar safe retry boundaries.

---

# 150. QueryExecutor must not guess

Nunca:

```text
catch Throwable
sleep
execute plan again
```

como política genérica.

---

# 151. Transaction coordination

El QueryExecutor deberá respetar:

```text
TransactionExecutionRequirements
```

del plan.

---

# 152. Transaction execution context

```php
final readonly class TransactionExecutionContext
{
    public function __construct(
        public TransactionRequirement $requirement,
        public ?TransactionHandle $existing,
        public TransactionOwnership $ownership,
    ) {}
}
```

---

# 153. Transaction boundaries

Si el plan requiere:

```text
one transaction across units A+B+C
```

el QueryExecutor deberá coordinarlo con Transaction Manager.

---

# 154. Transaction Manager responsibility

QueryExecutor no implementará:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

directamente.

---

# 155. Transaction ownership

Distinguir:

```text
CALLER_OWNED
EXECUTION_OWNED
SHARED_CONTEXT
```

---

# 156. Caller-owned

El QueryExecutor no deberá hacer commit de una transacción propiedad del caller.

---

# 157. Execution-owned

El QueryExecutor será responsable de solicitar commit/rollback mediante Transaction Manager.

---

# 158. Transaction failure

Un error podrá dejar la transacción:

```text
ACTIVE
FAILED
ROLLED_BACK
UNKNOWN
```

según plataforma/driver.

---

# 159. UNKNOWN transaction state

Debe propagarse conservadoramente.

---

# 160. Resource coordination

El QueryExecutor mantiene la visión global de recursos del plan.

---

# 161. Unit resources

Cada unidad puede requerir:

```text
connection
statement
cursor
memory
temporary storage
transaction
extension resource
```

---

# 162. Resource scope

```text
UNIT
STAGE
EXECUTION
TRANSFERRED
```

---

# 163. Resource acquisition timing

Preferir:

```text
acquire as late as practical
release as early as safe
```

---

# 164. Late acquisition

Evita mantener conexiones mientras se ejecuta trabajo que no las necesita.

---

# 165. Early release

Reduce:

```text
pool pressure
open cursor count
memory pressure
lock duration
```

---

# 166. Resource dependency

Sin embargo, no liberar antes de satisfacer:

```text
cursor lifetime
transaction affinity
session affinity
materialization requirements
```

---

# 167. Resource ledger

El QueryExecutor podrá utilizar:

```text
ExecutionResourceLedger
```

para verificar ownership.

---

# 168. Resource conservation

Al terminar:

```text
Acquired
=
Released
∪
Transferred
```

---

# 169. No leaked resources

Si existe:

```text
Acquired
-
Released
-
Transferred
≠
∅
```

deberá considerarse:

```text
ExecutionInvariantViolation
```

---

# 170. Cleanup coordinator

```php
interface ExecutionCleanupCoordinator
{
    public function cleanup(
        ExecutionInstance $instance,
        CleanupReason $reason,
    ): CleanupOutcome;
}
```

---

# 171. Cleanup reasons

```text
SUCCESS
FAILURE
CANCELLATION
TIMEOUT
PARTIAL_PREPARATION
CALLER_ABORT
```

---

# 172. Cleanup order

Debe seguir el graph de ownership/dependencies.

Ejemplo:

```text
Cursor
  ↓
Statement
  ↓
Connection
```

---

# 173. Cleanup and transferred resources

No deberán cerrarse recursos transferidos al result consumer.

---

# 174. Streaming transfer example

```text
QueryExecutor
     │
     └── transfers
          ↓
StreamingResult
          ↓ owns
Cursor
StatementLease
ConnectionLease
```

---

# 175. Cleanup idempotence

El QueryExecutor deberá tolerar cleanup repetido cuando los contracts lo permitan.

---

# 176. Final outcome

Después de ejecutar:

```text
ExecutionOutcomeFactory
```

produce el resultado final.

---

# 177. ExecutionOutcome

```php
final readonly class ExecutionOutcome
{
    public function __construct(
        public ExecutionOutcomeStatus $status,
        public ?ExecutionResult $result,
        public ExecutionMetadata $metadata,
        public ?ExecutionFailure $failure,
    ) {}
}
```

---

# 178. Outcome invariants

Por ejemplo:

```text
SUCCESS
→ failure = null
```

```text
FAILURE
→ failure != null
```

---

# 179. CANCELLED

Debe conservar:

```text
cancellation metadata
```

---

# 180. TIMED_OUT

Debe conservar:

```text
deadline/timeout metadata
```

---

# 181. UNKNOWN

Puede aparecer cuando los efectos de una operación no pueden determinarse.

---

# 182. Result contract

El QueryExecutor deberá comprobar que el root result satisface:

```text
ExecutionOutputContract
```

---

# 183. Result kinds

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

# 184. Result mismatch

Si el plan esperaba:

```text
SCALAR
```

y la unidad produce:

```text
ROW_STREAM
```

sin adapter definido:

```text
ExecutionResultContractException
```

---

# 185. No implicit coercion

No convertir arbitrariamente:

```text
rows → scalar
```

sin contract.

---

# 186. Scalar cardinality

Si el plan requiere exactamente un scalar y el database produce múltiples rows, deberá respetarse el cardinality contract.

---

# 187. Cardinality enforcement location

Puede realizarse en:

```text
database
execution plan
result system
```

según la estrategia seleccionada.

El QueryExecutor sólo coordina la enforcement indicada.

---

# 188. Result ordering

No deberá alterar el orden observable.

---

# 189. Concurrent branch merge

Si varias ramas se ejecutan concurrentemente:

```text
merge order
```

debe estar definido por el plan.

---

# 190. No completion-order merge

Nunca asumir:

```text
whichever branch finishes first
```

si el resultado exige orden determinista.

---

# 191. Result duplicates

No deduplicar intermediate/final rows salvo que el plan tenga un operador `DISTINCT` correspondiente.

---

# 192. Result buffering

No convertir streams a buffers automáticamente.

---

# 193. Result ownership

Todo resultado vivo deberá tener:

```text
owner
lifetime
cleanup semantics
```

---

# 194. QueryExecutor instrumentation

El QueryExecutor deberá emitir instrumentation points de alto nivel.

---

# 195. Suggested events

```text
QueryExecutionStarted
ExecutionInstanceCreated
StageReady
StageStarted
UnitReady
UnitStarted
UnitCompleted
UnitFailed
IntermediateResultProduced
MaterializationStarted
MaterializationCompleted
StageCompleted
QueryExecutionCompleted
QueryExecutionFailed
QueryExecutionCancelled
QueryExecutionTimedOut
QueryExecutionCleanupCompleted
```

---

# 196. Event payload

Debe usar IDs y metadata estructurada.

No live mutable internals.

---

# 197. Observer restrictions

Observers no podrán:

```text
change bindings
change plan
skip units
replace result
alter transaction
```

salvo que exista un extension contract explícito diferente.

---

# 198. Instrumentation ≠ extension

Mantener:

```text
Observer
=
observe
```

```text
Extension
=
declared behavior
```

---

# 199. Query execution trace

Debug mode podrá producir:

```text
Execution #E42

Stage 0
  U1 DatabaseExecutionUnit
     READY
     RUNNING
     COMPLETED 2.1 ms

Stage 1
  U2 FrameworkMergeUnit
     READY
     RUNNING
     COMPLETED 0.3 ms

Result
  ROW_STREAM

Resources
  connection #3 transferred
  statement #9 transferred
  cursor #2 transferred
```

---

# 200. Trace redaction

Nunca incluir valores sensibles sin política explícita.

---

# 201. Persistent workers

QueryExecutor deberá diseñarse para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 202. Shared safe state

Puede compartirse:

```text
frozen executor registry
immutable execution descriptors
stateless schedulers
immutable configuration
compiled artifacts
```

---

# 203. Operation-local state

Debe ser local:

```text
ExecutionInstance
bindings
results
resources
transactions
cursors
materializations
correlations
deadlines
cancellation
failures
```

---

# 204. FrankenPHP request lifecycle

```text
Worker Start
   ↓
Bootstrap QueryExecutor
   ↓
Freeze registries
   ↓
Request A
   ↓
ExecutionInstance A
   ↓
destroy/reset
   ↓
Request B
   ↓
ExecutionInstance B
```

---

# 205. No request leakage

A no podrá dejar en B:

```text
result
binding
transaction
tenant
cursor
failure
deadline
```

---

# 206. OpenSwoole concurrency

Dos execution instances podrán coexistir:

```text
Coroutine A
  ExecutionInstance A

Coroutine B
  ExecutionInstance B
```

---

# 207. No mutable singleton context

Prohibido:

```php
$this->currentExecution = $instance;
```

si `QueryExecutor` es singleton compartido.

---

# 208. Stateless coordinator preference

Preferible:

```text
shared QueryExecutor
+
explicit ExecutionInstance argument
```

---

# 209. Async future

La API podrá evolucionar hacia:

```php
executeAsync(...)
```

sin modificar el `ExecutionPlan`.

---

# 210. Sync vs async

Debe ser una propiedad del runtime execution mechanism.

No de query semantics.

---

# 211. Awaitable result future

Podría existir:

```text
ExecutionFuture
```

en adapters async.

---

# 212. Core neutrality

El core no deberá depender obligatoriamente de Fibers, ReactPHP, Amp o Swoole.

---

# 213. Backpressure coordination

Cuando una unidad consume un stream:

```text
Producer
   ↓
Consumer
```

el scheduler deberá permitir un modelo pull o bounded push.

---

# 214. Pull V1

Recomendado:

```text
consumer asks
   ↓
producer fetches
```

---

# 215. Pipeline cancellation

Si consumer deja de consumir:

```text
upstream
```

deberá poder cerrarse.

---

# 216. Example

```text
Database Cursor
      ↓
Framework Limit 10
      ↓
Consumer
```

Después de 10 rows:

```text
Framework Limit
      ↓
close upstream
      ↓
Cursor closes
```

si el plan permite esa ejecución.

---

# 217. Short-circuit execution

Algunos operadores pueden terminar upstream temprano.

---

# 218. Short-circuit semantics

Debe estar definido por el executor del operador y por el plan.

No inferido arbitrariamente por QueryExecutor.

---

# 219. Framework LIMIT

Si existe:

```text
FrameworkLimitExecutor
```

puede cerrar upstream después de N filas.

---

# 220. Database LIMIT

Si LIMIT fue delegado a SQL:

```text
QueryExecutor
```

no implementa ningún límite adicional.

---

# 221. Error isolation

Cada `ExecutionUnitOutcome` deberá indicar claramente:

```text
success
failure
result
resource effects
```

---

# 222. Unit outcome

```php
final readonly class ExecutionUnitOutcome
{
    public function __construct(
        public ExecutionUnitOutcomeStatus $status,
        public ?ExecutionResultHandle $result,
        public ?ExecutionFailure $failure,
        public ExecutionResourceEffectSet $resources,
    ) {}
}
```

---

# 223. Atomic state update

Después de una unidad:

```text
unit state
result store
dependency state
resource ledger
```

deberán actualizarse de manera coherente.

---

# 224. Runtime state transaction

No necesariamente una DB transaction.

Conceptualmente se necesita:

```text
atomic ExecutionInstance state transition
```

para evitar estados parciales internos.

---

# 225. Unit completion invariant

Una unidad no podrá estar:

```text
COMPLETED
```

si su required result no fue registrado correctamente.

---

# 226. Unit failure invariant

Una unidad:

```text
FAILED
```

no podrá publicar un result como si fuera completo salvo contract explícito de partial output.

---

# 227. SKIPPED

Sólo podrá usarse cuando el plan/policy lo permita.

No como forma de ocultar errores.

---

# 228. Runtime dependency state

Puede representarse con:

```text
ExecutionDependencyTracker
```

---

# 229. Dependency tracker

```php
interface ExecutionDependencyTracker
{
    public function markSatisfied(
        ExecutionDependencyId $dependency,
    ): void;

    public function isReady(
        ExecutionUnitId $unit,
    ): bool;
}
```

---

# 230. Efficient readiness

No será necesario recorrer todo el graph después de cada unidad.

Puede utilizarse:

```text
remaining dependency counters
```

---

# 231. Complexity

Para un DAG:

```text
Scheduling ≈ O(V + E)
```

en el caso común.

---

# 232. No quadratic scheduler

Evitar:

```text
for every completed unit
    scan every unit
        scan every dependency
```

---

# 233. Execution budget

El QueryExecutor deberá respetar:

```text
ExecutionBudget
```

---

# 234. Budget dimensions

Podrán incluir:

```text
max units executed
max stages
max recursive iterations
max framework rows processed
max intermediate results
max materialized bytes
max connections
max open cursors
max temporary bytes
```

---

# 235. Budget is not optimizer cost

```text
ExecutionBudget
≠
PlannerCost
```

---

# 236. Budget exhaustion

Produce failure estructurado.

---

# 237. No silent truncation

Nunca:

```text
budget exhausted
→ return whatever rows we have
```

salvo que partial-result semantics estén explícitamente autorizadas.

---

# 238. Execution plan compatibility

El QueryExecutor deberá comprobar compatibilidad entre:

```text
ExecutionPlanVersion
RuntimeExecutionEngineVersion
RequiredExtensionVersions
RequiredCapabilities
```

---

# 239. Stale plan

Si un cached execution plan ya no es compatible:

```text
ExecutionPlanCompatibilityException
```

y la capa superior podrá replanificar.

---

# 240. QueryExecutor does not replan

Nunca:

```text
plan incompatible
→ QueryExecutor rebuilds plan
```

---

# 241. Replanning boundary

```text
Execution failure due to stale/incompatible plan
        ↓
higher query orchestration layer
        ↓
invalidate/recompile/replan if policy allows
```

---

# 242. Compiled artifact compatibility

Igualmente:

```text
CompiledDatabaseCommand
PreparedStatementBlueprint
```

deberán ser compatibles con la execution target.

---

# 243. Connection target validation

Si el command fue compilado para:

```text
PostgreSQL
```

no podrá ejecutarse en una conexión:

```text
MySQL
```

---

# 244. Capability fingerprint

Puede verificarse contra:

```text
CompiledQueryFingerprint
PlatformCapabilityFingerprint
DriverCompilationContractFingerprint
```

cuando corresponda.

---

# 245. Security execution context

El QueryExecutor recibe security metadata ya resuelta.

---

# 246. Security requirements

Puede verificar:

```text
mandatory security artifact present
tenant context present
required connection role allowed
dangerous operation capability present
raw SQL trust requirement satisfied
```

---

# 247. No authorization engine

No deberá decidir:

```text
user X may update row Y
```

---

# 248. Security failure

Si falta un requisito:

```text
ExecutionSecurityRequirementException
```

antes de ejecutar.

---

# 249. Tenant context

Debe ser explícito:

```php
final readonly class ExecutionTenantContext
{
    public function __construct(
        public ?TenantExecutionIdentity $tenant,
        public TenantIsolationMode $isolation,
    ) {}
}
```

---

# 250. Core optionality

Database core no deberá depender obligatoriamente del paquete Multitenancy.

Puede trabajar con:

```text
null / neutral tenant context
```

---

# 251. Multitenancy integration

El paquete oficial podrá aportar:

```text
TenantConnectionResolver
TenantExecutionContextFactory
TenantExecutionValidator
```

sin modificar QueryExecutor core.

---

# 252. Connection routing context

El QueryExecutor pasa:

```text
tenant-aware connection requirement
```

al Connection Manager.

---

# 253. QueryExecutor does not select database credentials

Eso pertenece al Connection/Tenant integration layer.

---

# 254. Exception hierarchy

Raíz:

```php
class QueryExecutionException extends DatabaseExecutionException
{
}
```

---

# 255. Suggested exceptions

```text
QueryExecutionValidationException
ExecutionPlanCompatibilityException
ExecutionPlanTraversalException
ExecutionDependencyException
ExecutionDeadlockException
ExecutionUnitDispatchException
ExecutionUnitFailureException
ExecutionStageException
ExecutionResultContractException
ExecutionMaterializationException
ExecutionCorrelationException
ExecutionRecursionException
ExecutionResourceException
ExecutionCancellationException
ExecutionTimeoutException
ExecutionCleanupException
ExecutionSecurityRequirementException
ExecutionInvariantException
```

---

# 256. Execution deadlock

No confundir:

```text
Execution graph deadlock
```

con:

```text
Database transaction deadlock
```

---

# 257. Graph deadlock

Ejemplo:

```text
Unit A waits for B
Unit B waits for A
```

fuera de recursive region.

Eso es:

```text
ExecutionPlanInvariantViolation
```

---

# 258. Database deadlock

Proviene del DBMS y se clasifica por Execution Error System.

---

# 259. Diagnostics

Cuando falle scheduling:

```text
Unit U7 blocked
Missing dependencies:
  D12 from U3
  D15 from U5
```

---

# 260. Explain execution

VoltStack podrá exponer:

```text
EXPLAIN EXECUTION
```

conceptual.

---

# 261. Explain view

```text
ExecutionPlan E1

Stage 0
  U1 DATABASE
    command: C14
    result: ROW_STREAM

Stage 1
  U2 FRAMEWORK_LIMIT
    input: U1
    limit: 20

Dependencies
  U1 DATA → U2

Resources
  U1 requires READ connection
  U1 transfers cursor ownership through U2

Cancellation
  E1 → U1 → statement
```

---

# 262. Explain ≠ execute

La vista explain no deberá adquirir conexiones ni ejecutar statements.

---

# 263. Explain runtime trace

Separadamente podrá existir:

```text
actual execution trace
```

después de ejecutar.

---

# 264. Plan vs actual

Mantener:

```text
planned execution
≠
actual execution observations
```

---

# 265. Testing strategy

El QueryExecutor requerirá:

```text
unit tests
graph tests
stage tests
dependency tests
dispatch tests
result propagation tests
materialization tests
correlation tests
recursion tests
failure tests
cancellation tests
timeout tests
resource tests
transaction tests
retry-boundary tests
persistent-worker tests
concurrency tests
property tests
performance tests
```

---

# 266. DAG tests

Casos:

```text
linear
fan-out
fan-in
diamond
multiple stages
independent branches
```

---

# 267. Linear

```text
A → B → C
```

---

# 268. Fan-out

```text
    A
   / \
  B   C
```

---

# 269. Fan-in

```text
B   C
 \ /
  D
```

---

# 270. Diamond

```text
    A
   / \
  B   C
   \ /
    D
```

---

# 271. Invalid cycle

```text
A → B
↑   ↓
└── C
```

deberá rechazarse salvo recursive region válida.

---

# 272. Deterministic scheduling test

El mismo DAG deberá producir el mismo sequential schedule.

---

# 273. Failure propagation tests

Fallar cada unidad y comprobar:

```text
dependent state
cleanup
root outcome
resource release
```

---

# 274. Resource property test

Propiedad:

```text
∀ acquired resource r:

terminal(execution)
⇒
released(r) XOR transferred(r)
```

---

# 275. State property test

```text
COMPLETED(unit)
⇒
all required completion invariants satisfied
```

---

# 276. Readiness property

```text
RUNNING(unit)
⇒
READY(unit) immediately before transition
```

---

# 277. Dependency property

```text
READY(unit)
⇒
all mandatory dependencies satisfied
```

---

# 278. Cancellation property

Después de cancellation:

```text
no new non-cleanup unit
```

deberá comenzar salvo policy explícita.

---

# 279. Timeout property

Después del deadline:

```text
no new normal execution work
```

deberá comenzar.

---

# 280. Persistent runtime property

Después de cerrar `ExecutionInstance A`:

```text
MutableRuntimeState(A)
∩
ExecutionInstance(B)
=
∅
```

---

# 281. Performance model

Para un plan DAG normal:

```text
QueryExecutor overhead
≈
O(V + E)
```

más el costo real de cada unidad.

---

# 282. Hot path

Para una query SQL simple:

```text
ExecutionPlan
└── DatabaseExecutionUnit
```

el flujo deberá ser corto.

---

# 283. Simple query fast path

```text
Request
 ↓
Validate
 ↓
Create Instance
 ↓
Dispatch DB Unit
 ↓
Statement System
 ↓
Result
 ↓
Cleanup/Transfer
```

---

# 284. Avoid abstraction tax

La arquitectura conceptual puede tener muchos contracts.

La implementación no deberá crear necesariamente un objeto heap por cada concepto.

---

# 285. Specialized fast path

Podrá existir:

```text
SingleDatabaseUnitExecutionPath
```

si mantiene exactamente los mismos invariantes.

---

# 286. Fast path eligibility

Sólo cuando:

```text
one unit
no framework operator
no correlation
no recursion
no materialization
no complex dependencies
```

---

# 287. Fast path ≠ semantic shortcut

No podrá omitir:

```text
binding safety
resource ownership
cancellation
deadline
failure normalization
cleanup
```

---

# 288. Directory structure

Propuesta:

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Query/
                ├── Contract/
                │   ├── QueryExecutor.php
                │   ├── ExecutionPlanExecutor.php
                │   └── ExecutionUnitExecutor.php
                │
                ├── Executor/
                │   ├── DefaultQueryExecutor.php
                │   ├── DatabaseExecutionUnitExecutor.php
                │   ├── FrameworkExecutionUnitExecutor.php
                │   └── ExtensionExecutionUnitExecutor.php
                │
                ├── Instance/
                │   ├── ExecutionInstance.php
                │   ├── ExecutionInstanceFactory.php
                │   ├── ExecutionRuntimeState.php
                │   └── ExecutionStateMachine.php
                │
                ├── Scheduler/
                │   ├── ExecutionScheduler.php
                │   ├── SequentialExecutionScheduler.php
                │   ├── ExecutionScheduleDecision.php
                │   ├── ExecutionDependencyTracker.php
                │   └── ExecutionReadinessResolver.php
                │
                ├── Stage/
                │   ├── ExecutionStageExecutor.php
                │   ├── ExecutionStageRuntime.php
                │   └── ExecutionStageState.php
                │
                ├── Unit/
                │   ├── ExecutionUnitDispatcher.php
                │   ├── ExecutionUnitRuntime.php
                │   ├── ExecutionUnitRuntimeContext.php
                │   ├── ExecutionUnitOutcome.php
                │   └── ExecutionUnitState.php
                │
                ├── Dependency/
                │   ├── RuntimeDependencyResolver.php
                │   ├── RuntimeDependencySet.php
                │   └── RuntimeDependencyState.php
                │
                ├── Result/
                │   ├── ExecutionResultStore.php
                │   ├── ExecutionResultHandle.php
                │   ├── ResultPropagationCoordinator.php
                │   └── RootResultResolver.php
                │
                ├── Correlation/
                │   ├── CorrelationCoordinator.php
                │   ├── CorrelationBindingSet.php
                │   └── CorrelationRuntimeScope.php
                │
                ├── Materialization/
                │   ├── MaterializationCoordinator.php
                │   ├── MaterializationHandle.php
                │   └── MaterializationRuntimeScope.php
                │
                ├── Recursion/
                │   ├── RecursiveExecutionCoordinator.php
                │   ├── RecursiveExecutionRegionRuntime.php
                │   └── RecursiveIterationState.php
                │
                ├── Resource/
                │   ├── QueryExecutionResourceCoordinator.php
                │   └── ExecutionResourceScope.php
                │
                ├── Failure/
                │   ├── QueryFailureCoordinator.php
                │   └── QueryExecutionFailure.php
                │
                ├── Cleanup/
                │   ├── QueryExecutionCleanupCoordinator.php
                │   └── CleanupReason.php
                │
                ├── Outcome/
                │   ├── ExecutionOutcomeFactory.php
                │   └── ExecutionMetadataFactory.php
                │
                ├── Registry/
                │   ├── ExecutionUnitExecutorRegistry.php
                │   └── FrameworkOperatorExecutorRegistry.php
                │
                ├── Diagnostic/
                │   ├── QueryExecutionDiagnostic.php
                │   └── QueryExecutionTrace.php
                │
                └── Exception/
                    ├── QueryExecutionException.php
                    ├── ExecutionPlanTraversalException.php
                    ├── ExecutionDependencyException.php
                    ├── ExecutionUnitDispatchException.php
                    ├── ExecutionResultContractException.php
                    ├── ExecutionCorrelationException.php
                    ├── ExecutionMaterializationException.php
                    ├── ExecutionRecursionException.php
                    └── QueryExecutionInvariantException.php
```

---

# 289. Architectural dependency rule

```text
QueryExecutor
     ↓
Execution Contracts
     ↓
Statement Execution System
     ↓
Connection
     ↓
Driver
```

---

# 290. Forbidden dependency

```text
QueryExecutor
     ↓
Query Optimizer
```

Prohibido.

---

# 291. Forbidden dependency

```text
QueryExecutor
     ↓
SQL Compiler
```

durante ejecución normal.

Los artifacts deberán estar preparados antes.

---

# 292. Lazy compilation

Si en el futuro se permite lazy compilation:

```text
CompilationCoordinator
```

deberá ser una frontera explícita superior.

No lógica oculta dentro de unit execution.

---

# 293. Forbidden ORM dependency

```text
QueryExecutor
     ↓
EntityManager
```

prohibido.

---

# 294. Architectural invariants

## DB-QEXEC-001

QueryExecutor ejecutará `ExecutionPlan`.

## DB-QEXEC-002

QueryExecutor no interpretará Query AST.

## DB-QEXEC-003

QueryExecutor no realizará semantic analysis.

## DB-QEXEC-004

QueryExecutor no realizará query optimization.

## DB-QEXEC-005

QueryExecutor no realizará physical planning.

## DB-QEXEC-006

QueryExecutor no generará SQL.

## DB-QEXEC-007

QueryExecutor no hidratará ORM entities.

## DB-QEXEC-008

QueryExecutor será distinto de StatementExecutor.

## DB-QEXEC-009

ExecutionRequest será immutable.

## DB-QEXEC-010

ExecutionInstance será mutable y operation-scoped.

## DB-QEXEC-011

ExecutionPlan no almacenará runtime state.

## DB-QEXEC-012

ExecutionInstance no será global.

## DB-QEXEC-013

Persistent workers no compartirán ExecutionInstance.

## DB-QEXEC-014

Preflight deberá ocurrir antes de adquirir recursos cuando sea posible.

## DB-QEXEC-015

Preflight no repetirá semantic analysis.

## DB-QEXEC-016

ExecutionPlan podrá ser graph.

## DB-QEXEC-017

Graph edge kinds tendrán semántica explícita.

## DB-QEXEC-018

DATA edge será distinto de CONTROL edge.

## DB-QEXEC-019

CORRELATION edge será distinto de DATA edge.

## DB-QEXEC-020

CLEANUP edge será distinto de normal dependency.

## DB-QEXEC-021

Arbitrary cycles estarán prohibidos.

## DB-QEXEC-022

Recursion será estructurada.

## DB-QEXEC-023

Stage será distinto de transaction.

## DB-QEXEC-024

Stage será distinto de thread.

## DB-QEXEC-025

Runtime stage state no mutará ExecutionPlan.

## DB-QEXEC-026

Runtime unit state no mutará ExecutionPlan.

## DB-QEXEC-027

Scheduler no elegirá physical strategy.

## DB-QEXEC-028

Scheduler no compilará SQL.

## DB-QEXEC-029

Sequential scheduling será determinista.

## DB-QEXEC-030

Scheduling no dependerá de incidental hash order.

## DB-QEXEC-031

Parallelizable no implicará parallel execution.

## DB-QEXEC-032

Concurrent execution preservará dependencies.

## DB-QEXEC-033

Unit dispatch será tipado.

## DB-QEXEC-034

Executor registry será frozen.

## DB-QEXEC-035

Executor registry no usará last-wins.

## DB-QEXEC-036

Database unit execution se delegará al Statement System.

## DB-QEXEC-037

QueryExecutor no implementará driver preparation.

## DB-QEXEC-038

QueryExecutor no implementará runtime binding internals.

## DB-QEXEC-039

QueryExecutor no implementará driver fetching.

## DB-QEXEC-040

Framework execution sólo ocurrirá si fue planificada.

## DB-QEXEC-041

QueryExecutor no inventará framework fallback.

## DB-QEXEC-042

V1 favorecerá database delegation.

## DB-QEXEC-043

Extensions tendrán executor contract explícito.

## DB-QEXEC-044

Extensions declararán resources.

## DB-QEXEC-045

Extensions declararán cancellation behavior.

## DB-QEXEC-046

Extensions declararán cleanup behavior.

## DB-QEXEC-047

ExecutionUnitRuntimeContext será scoped.

## DB-QEXEC-048

Unit executors no recibirán acceso global innecesario.

## DB-QEXEC-049

Unit readiness requerirá dependencies satisfechas.

## DB-QEXEC-050

Streaming availability será distinta de producer completion.

## DB-QEXEC-051

Pipelining requerirá explicit plan support.

## DB-QEXEC-052

Intermediate results serán execution-scoped.

## DB-QEXEC-053

Intermediate results no serán persistent cache.

## DB-QEXEC-054

Intermediate results se liberarán cuando ya no sean necesarios.

## DB-QEXEC-055

Root output será explícito.

## DB-QEXEC-056

Plan completion requerirá output contract satisfecho.

## DB-QEXEC-057

Streaming handoff será distinto de stream exhaustion.

## DB-QEXEC-058

Streaming resource ownership será transferido explícitamente.

## DB-QEXEC-059

Post-handoff stream failures serán observables.

## DB-QEXEC-060

Materialization sólo ocurrirá si fue planificada.

## DB-QEXEC-061

Materialization será distinta de result cache.

## DB-QEXEC-062

Materialization tendrá lifetime explícito.

## DB-QEXEC-063

Materialization tendrá cleanup explícito.

## DB-QEXEC-064

Correlation utilizará structured bindings.

## DB-QEXEC-065

Correlation no utilizará SQL string substitution.

## DB-QEXEC-066

Execution multiplicity será explícita.

## DB-QEXEC-067

Multiplicity será distinta de cardinality.

## DB-QEXEC-068

QueryExecutor no reoptimizará correlated execution.

## DB-QEXEC-069

Runtime batching requerirá autorización del plan.

## DB-QEXEC-070

Framework recursion será bounded.

## DB-QEXEC-071

Database delegated recursion no será iterada por QueryExecutor.

## DB-QEXEC-072

Cancellation será comprobada en checkpoints.

## DB-QEXEC-073

Cancellation detendrá nuevo trabajo normal.

## DB-QEXEC-074

Cancellation se propagará a running units.

## DB-QEXEC-075

Cancellation será distinta de timeout.

## DB-QEXEC-076

Timeout detendrá nuevo trabajo normal.

## DB-QEXEC-077

Failure propagation seguirá dependency graph.

## DB-QEXEC-078

Default query execution será fail-fast.

## DB-QEXEC-079

Fail-fast no omitirá cleanup.

## DB-QEXEC-080

Primary failure será preservado.

## DB-QEXEC-081

Cleanup failures podrán ser suppressed failures.

## DB-QEXEC-082

QueryExecutor no implementará SQLSTATE normalization.

## DB-QEXEC-083

Retry requerirá RetryCoordinator.

## DB-QEXEC-084

QueryExecutor no retryará automáticamente Throwable.

## DB-QEXEC-085

Retry scope deberá ser seguro.

## DB-QEXEC-086

Ambiguous writes no serán reiniciados ciegamente.

## DB-QEXEC-087

Transaction requirements serán respetados.

## DB-QEXEC-088

QueryExecutor no implementará transaction primitives.

## DB-QEXEC-089

Caller-owned transaction no será committed por QueryExecutor.

## DB-QEXEC-090

Execution-owned transaction tendrá lifecycle explícito.

## DB-QEXEC-091

Unknown transaction state será preservado.

## DB-QEXEC-092

Resources se adquirirán lo más tarde posible cuando sea seguro.

## DB-QEXEC-093

Resources se liberarán lo antes posible cuando sea seguro.

## DB-QEXEC-094

Cursor lifetime podrá impedir early connection release.

## DB-QEXEC-095

Transaction affinity será preservada.

## DB-QEXEC-096

Session affinity será preservada.

## DB-QEXEC-097

Resource ledger será execution-scoped.

## DB-QEXEC-098

Todo recurso adquirido será released o transferred.

## DB-QEXEC-099

Leaked resource será invariant violation.

## DB-QEXEC-100

Cleanup respetará ownership graph.

## DB-QEXEC-101

Transferred resources no serán limpiados por el antiguo owner.

## DB-QEXEC-102

Cleanup será idempotente cuando el contract lo permita.

## DB-QEXEC-103

ExecutionOutcome respetará status invariants.

## DB-QEXEC-104

UNKNOWN outcome será representable.

## DB-QEXEC-105

Root result respetará ExecutionOutputContract.

## DB-QEXEC-106

Result kinds no se coercionarán implícitamente.

## DB-QEXEC-107

Result ordering observable será preservado.

## DB-QEXEC-108

Concurrent branch merge no usará completion order si cambia semantics.

## DB-QEXEC-109

Rows no serán deduplicadas sin operador explícito.

## DB-QEXEC-110

Streams no serán bufferizados implícitamente.

## DB-QEXEC-111

Result ownership será explícito.

## DB-QEXEC-112

Instrumentation será observacional.

## DB-QEXEC-113

Observers no modificarán execution state.

## DB-QEXEC-114

Instrumentation será distinta de extensions.

## DB-QEXEC-115

Trace redaction será obligatoria para sensitive data.

## DB-QEXEC-116

Shared QueryExecutor deberá ser stateless respecto a ejecución actual.

## DB-QEXEC-117

Bindings serán execution-local.

## DB-QEXEC-118

Results serán execution-local.

## DB-QEXEC-119

Resources serán execution-local salvo transfer explícito.

## DB-QEXEC-120

Transactions serán execution/context-local.

## DB-QEXEC-121

Materializations serán execution-local.

## DB-QEXEC-122

Correlation state será execution-local.

## DB-QEXEC-123

Deadlines serán execution-local.

## DB-QEXEC-124

Cancellation será execution-local.

## DB-QEXEC-125

FrankenPHP workers no filtrarán state entre requests.

## DB-QEXEC-126

RoadRunner deberá soportar el mismo execution model.

## DB-QEXEC-127

OpenSwoole deberá soportar concurrent execution isolation.

## DB-QEXEC-128

Core QueryExecutor no dependerá de async framework específico.

## DB-QEXEC-129

Async execution no cambiará query semantics.

## DB-QEXEC-130

Backpressure será preservado en streaming pipelines.

## DB-QEXEC-131

Early downstream completion podrá cerrar upstream cuando el plan lo permita.

## DB-QEXEC-132

Short-circuit behavior pertenecerá al planned operator.

## DB-QEXEC-133

Unit outcomes serán estructurados.

## DB-QEXEC-134

Unit state/result/resource updates serán coherentes.

## DB-QEXEC-135

COMPLETED unit tendrá required result satisfecho.

## DB-QEXEC-136

FAILED unit no publicará complete result inválido.

## DB-QEXEC-137

SKIPPED no ocultará errores.

## DB-QEXEC-138

Dependency tracking deberá ser bounded.

## DB-QEXEC-139

Scheduler común deberá aspirar a O(V+E).

## DB-QEXEC-140

ExecutionBudget será distinto de PlannerCost.

## DB-QEXEC-141

Budget exhaustion no producirá silent partial success.

## DB-QEXEC-142

ExecutionPlan compatibility será verificada.

## DB-QEXEC-143

QueryExecutor no replanificará stale plan.

## DB-QEXEC-144

Compiled command target deberá coincidir con connection target.

## DB-QEXEC-145

Required security execution metadata será validada.

## DB-QEXEC-146

QueryExecutor no ejecutará authorization policy.

## DB-QEXEC-147

Tenant context será explícito.

## DB-QEXEC-148

Multitenancy no será mandatory core dependency.

## DB-QEXEC-149

QueryExecutor no resolverá database credentials directamente.

## DB-QEXEC-150

Execution graph deadlock será distinto de database deadlock.

## DB-QEXEC-151

Explain execution no ejecutará el plan.

## DB-QEXEC-152

Planned execution será distinto de actual trace.

## DB-QEXEC-153

Fast paths deberán preservar todos los invariantes de seguridad.

## DB-QEXEC-154

Fast paths no omitirán cancellation/deadline checks requeridos.

## DB-QEXEC-155

Fast paths no omitirán resource ownership.

## DB-QEXEC-156

Fast paths no omitirán failure normalization.

## DB-QEXEC-157

Fast paths no omitirán cleanup.

## DB-QEXEC-158

Execution shall preserve ExecutionPlan semantics.

## DB-QEXEC-159

QueryExecutor shall orchestrate, not reinterpret.

## DB-QEXEC-160

Runtime state shall never contaminate reusable plan state.

---

# 295. Invariante maestro de scheduling

Para cada unidad `U`:

```text
RUNNING(U)
⇒
Ready(U)
```

y:

```text
Ready(U)
⇒
MandatoryDependencies(U)
⊆
SatisfiedDependencies
```

---

# 296. Invariante maestro de resultados

```text
Completed(U)
⇒
RequiredOutputContract(U)
is satisfied
```

---

# 297. Invariante maestro de ownership

Al cerrar el scope del QueryExecutor:

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

# 298. Invariante maestro de failure

```text
Failure(U)
⇒
No semantically dependent unit
may execute as though U succeeded
```

---

# 299. Invariante maestro de cancellation

```text
CancellationRequested
⇒
No new normal work
after the applicable cancellation boundary
```

---

# 300. Invariante maestro de plan

```text
ExecutionInstance
may mutate runtime state

ExecutionInstance
must never mutate ExecutionPlan
```

---

# 301. Invariante maestro persistent-runtime

```text
MutableExecutionState
∩
SharedWorkerState
=
∅
```

salvo recursos compartidos expresamente diseñados con ownership/concurrency contract.

---

# 302. Flujo completo de query simple

```text
ExecutionRequest
       ↓
QueryExecutor
       ↓
Validate
       ↓
ExecutionInstance
       ↓
Single Database Unit READY
       ↓
Dispatch
       ↓
DatabaseExecutionUnitExecutor
       ↓
Statement Execution System
       ↓
Database Result
       ↓
Root Result
       ↓
Cleanup / Transfer
       ↓
ExecutionOutcome
```

---

# 303. Flujo de plan compuesto

```text
ExecutionRequest
      ↓
ExecutionInstance
      ↓
Stage 0
      ↓
Database Unit A
      ↓
Result A
      ↓
Stage 1
   ┌──┴──┐
   ▼     ▼
Unit B  Unit C
   │     │
   └──┬──┘
      ▼
Stage 2
      ↓
Merge Unit D
      ↓
Root Result
      ↓
ExecutionOutcome
```

---

# 304. Flujo correlacionado

```text
Outer Unit
    ↓
Outer Row
    ↓
CorrelationBindingSet
    ↓
Inner Unit
    ↓
Inner Result
    ↓
Correlation Consumer
```

---

# 305. Flujo streaming

```text
Database Unit
      ↓
Open Cursor
      ↓
Result Handle
      ↓
Root Streaming Result
      ↓
Ownership Transfer
      ↓
Caller
      ↓
fetch
fetch
fetch
close
      ↓
Final Resource Cleanup
```

---

# 306. Flujo de fallo

```text
Unit A
  ↓ success
Unit B
  ↓ failure
Failure Coordinator
  ↓
Mark dependent units cancelled
  ↓
Stop scheduling
  ↓
Cleanup
  ↓
Preserve primary failure
  ↓
ExecutionOutcome FAILURE
```

---

# 307. Fórmula arquitectónica

```text
Query Executor System
=
Execution Request Validation
+
Execution Instance Management
+
Execution Graph Traversal
+
Stage Scheduling
+
Unit Readiness Resolution
+
Typed Unit Dispatch
+
Runtime Dependency Tracking
+
Intermediate Result Propagation
+
Correlation Coordination
+
Materialization Coordination
+
Recursion Coordination
+
Resource Coordination
+
Transaction Coordination
+
Cancellation Propagation
+
Timeout Propagation
+
Failure Propagation
+
Retry Boundaries
+
Cleanup Coordination
+
Root Result Resolution
+
Execution Outcome Construction
+
Persistent Runtime Isolation
```

---

# 308. Regla final

El `QueryExecutor` deberá responder exclusivamente a:

```text
"What execution work is ready,
how do I coordinate it,
and how do I safely propagate its runtime effects?"
```

Nunca deberá volver a responder:

```text
"What does this query mean?"

"Can I optimize this predicate?"

"Which join algorithm should I choose?"

"How should this SQL be rendered?"
```

Esas decisiones pertenecen a capas anteriores.

Por tanto:

> **QueryExecutor coordinates the runtime realization of an ExecutionPlan without changing the plan's meaning or physical strategy.**

---

# 309. Relación con el siguiente subsistema

El QueryExecutor delegará la ejecución concreta de un:

```text
DatabaseExecutionUnit
```

hacia:

```text
Statement Execution System
```

La frontera será:

```text
QueryExecutor
      ↓
DatabaseExecutionUnitExecutor
      ↓
StatementExecutionRequest
      ↓
Statement Execution System
```

---

# 310. Siguiente documento

```text
78_DATABASE_STATEMENT_EXECUTION_SYSTEM.md
```

El siguiente documento deberá definir en profundidad:

```text
Statement Execution System
├── StatementExecutionRequest
├── StatementExecutionContext
├── Statement Lifecycle
├── Connection Lease Integration
├── PreparedStatement Acquisition
├── Statement Preparation
├── Runtime Binding Integration
├── Driver Statement Execution
├── Statement State Machine
├── Driver Result Acquisition
├── Affected Rows
├── Generated Values
├── RETURNING Results
├── Statement Timeout
├── Statement Cancellation
├── Statement Failure Classification
├── Statement Resource Ownership
├── Statement Reset
├── Statement Reuse
├── Statement Cleanup
├── Instrumentation
├── Persistent Runtime Safety
└── Driver Abstraction Boundaries
```

manteniendo la separación:

```text
QueryExecutor
=
plan orchestration

StatementExecutor
=
single database statement runtime execution
```