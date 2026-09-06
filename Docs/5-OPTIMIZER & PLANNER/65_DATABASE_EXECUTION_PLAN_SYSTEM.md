# 65_DATABASE_EXECUTION_PLAN_SYSTEM.md

# VoltStack Quantum Database
## Execution Plan System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 65 — Execution Plan System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Planner / Execution Planning  
**Versión:** 1.0

---

# 1. Propósito

`Execution Plan System` define la representación ejecutable intermedia que transforma un:

```text
PhysicalQueryPlan
```

en una topología operacional concreta que posteriormente podrá ser consumida por el `Execution Engine`.

Su pregunta fundamental es:

> ¿Cómo debe convertirse una estrategia física seleccionada en unidades de trabajo ejecutables, coordinables, cancelables, observables y limpiables?

Formalmente:

```text
ExecutionPlanning(
    PhysicalQueryPlan,
    ExecutionPlanningContext
)
→
ExecutionPlan
```

La distinción central será:

```text
PhysicalQueryPlan
=
which physical strategies should implement the query

ExecutionPlan
=
how those strategies become executable work

Execution Engine
=
runs that work
```

---

# 2. Posición arquitectónica

```text
Query Builder
     │
     ▼
Query AST
     │
     ▼
Semantic Analysis
     │
     ▼
SemanticQueryArtifact
     │
     ▼
Query Optimizer
     │
     ▼
OptimizedQueryArtifact
     │
     ▼
Logical Plan
     │
     ▼
LogicalQueryPlan
     │
     ▼
Physical Plan
     │
     ▼
PhysicalQueryPlan
     │
     ▼
┌──────────────────────────────────────┐
│        Execution Plan System         │
│                                      │
│ Execution Topology                   │
│ Stage Construction                   │
│ Operator Lowering                    │
│ Runtime Parameter Slots              │
│ Database Execution Units             │
│ Framework Execution Units            │
│ Resource Ownership                   │
│ Scheduling Dependencies              │
│ Cancellation Propagation             │
│ Cleanup Planning                     │
│ Instrumentation Points               │
└──────────────────────────────────────┘
     │
     ▼
ExecutionPlan
     │
     ▼
Execution Engine
     │
     ▼
Query Executor
     │
     ▼
Connection / Driver / Database
```

---

# 3. Frontera arquitectónica

El `Execution Plan System` puede decidir:

```text
execution stages
execution units
operator topology
dependencies
parameter slots
resource ownership
buffer ownership
cursor ownership
materialization boundaries
database delegation boundaries
framework execution boundaries
scheduling constraints
cancellation relationships
cleanup relationships
instrumentation points
```

pero no debe:

```text
open database connections
begin transactions
prepare live statements
bind runtime values
execute statements
fetch rows
hydrate entities
commit transactions
rollback transactions
```

Esas operaciones pertenecen al runtime de ejecución.

---

# 4. Execution Plan ≠ Physical Plan

El `PhysicalQueryPlan` describe estrategia.

Ejemplo:

```text
HashJoin
├── SequentialScan(users)
└── SequentialScan(orders)
```

El `ExecutionPlan` describe cómo esa estrategia se convierte en trabajo.

Ejemplo conceptual:

```text
ExecutionStage S1
└── BuildHashTableUnit
    └── DatabaseReadUnit(users)

ExecutionStage S2
└── ProbeHashTableUnit
    └── DatabaseReadUnit(orders)

ExecutionStage S3
└── ResultProjectionUnit
```

---

# 5. Execution Plan ≠ Execution

La existencia de:

```text
ExecutionPlan
```

no significa que la query haya comenzado.

Siempre:

```text
ExecutionPlan
≠
RunningExecution
```

---

# 6. Execution Plan ≠ SQL Compiler

El `Execution Plan System` puede determinar que una región se delegará a la base:

```text
DatabaseExecutionUnit
```

pero la representación SQL concreta pertenece al:

```text
SQL Compiler
```

---

# 7. Execution Plan ≠ Prepared Statement

Nunca:

```text
ExecutionPlan
=
PDOStatement
```

Un statement preparado es un recurso runtime.

---

# 8. Execution Plan ≠ Query Result

Igualmente:

```text
ExecutionPlan
≠
Result
≠
ResultCursor
≠
HydratedEntity
```

---

# 9. Objetivo principal

El sistema debe proporcionar una frontera estable entre:

```text
Planning
```

y:

```text
Execution
```

de forma que el `Execution Engine` no tenga que reinterpretar el `PhysicalQueryPlan`.

---

# 10. Principio maestro

```text
The Execution Plan is executable structure,
not executing state.
```

---

# 11. Entrada principal

El input principal será:

```php
PhysicalQueryPlan
```

acompañado por:

```php
ExecutionPlanningContext
```

---

# 12. ExecutionPlanningContext

Modelo conceptual:

```php
final readonly class ExecutionPlanningContext
{
    public function __construct(
        public PlatformCapabilitySnapshot $capabilities,
        public ExecutionCapabilitySnapshot $executionCapabilities,
        public RuntimeResourceProfile $resources,
        public ExecutionPlanningConfiguration $configuration,
        public ExecutionPlanningBudget $budget,
        public ExecutionExtensionSet $extensions,
        public InstrumentationConfiguration $instrumentation,
    ) {}
}
```

---

# 13. Contexto explícito

El planner no utilizará:

```text
global current connection
global current transaction
global current tenant
static runtime state
service locator
```

---

# 14. RuntimeResourceProfile

Puede describir límites conocidos como:

```text
memory budget
temporary storage availability
worker concurrency
streaming support
buffer limits
maximum execution units
```

---

# 15. Resource profile ≠ resource allocation

El perfil sólo describe disponibilidad/restricciones.

No reserva recursos.

---

# 16. ExecutionPlan

Modelo conceptual:

```php
final readonly class ExecutionPlan
{
    public function __construct(
        public ExecutionPlanId $id,
        public ExecutionPlanGraph $graph,
        public ExecutionStageGraph $stages,
        public ExecutionParameterLayout $parameters,
        public ExecutionResourcePlan $resources,
        public ExecutionCancellationPlan $cancellation,
        public ExecutionCleanupPlan $cleanup,
        public ExecutionInstrumentationPlan $instrumentation,
        public PhysicalExecutionMapping $mapping,
        public ExecutionDependencySet $dependencies,
        public ExecutionPlanMetadata $metadata,
        public ExecutionPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 17. Inmutabilidad

Una vez construido:

```text
ExecutionPlan
```

será inmutable.

---

# 18. ExecutionPlanId

Se distinguirán:

```text
LogicalPlanId
PhysicalPlanId
ExecutionPlanId
ExecutionInstanceId
```

Siempre:

```text
ExecutionPlanId
≠
ExecutionInstanceId
```

---

# 19. Plan vs instancia

Un mismo:

```text
ExecutionPlan
```

puede potencialmente ser reutilizado para múltiples:

```text
ExecutionInstance
```

si sus dependencias siguen siendo válidas.

---

# 20. Runtime state

El estado runtime pertenece a:

```text
ExecutionInstance
```

no al plan.

---

# 21. Ejemplo

```text
ExecutionPlan
     │
     ├── ExecutionInstance #1
     │       parameter id = 10
     │
     ├── ExecutionInstance #2
     │       parameter id = 20
     │
     └── ExecutionInstance #3
             parameter id = 30
```

---

# 22. Runtime bindings

Por tanto:

```text
ExecutionPlan
```

contiene slots.

```text
ExecutionInstance
```

contiene valores.

---

# 23. ExecutionParameterSlot

Modelo:

```php
final readonly class ExecutionParameterSlot
{
    public function __construct(
        public ExecutionParameterSlotId $id,
        public ParameterId $parameter,
        public QueryType $semanticType,
        public BindingType $bindingType,
        public ParameterSensitivity $sensitivity,
    ) {}
}
```

---

# 24. Parameter slot ≠ runtime value

Siempre:

```text
ExecutionParameterSlot
≠
RuntimeParameterValue
```

---

# 25. Parameter layout

```php
final readonly class ExecutionParameterLayout
{
    public function __construct(
        public ExecutionParameterSlotMap $slots,
        public ParameterToExecutionSlotMap $mapping,
    ) {}
}
```

---

# 26. Placeholder ≠ slot

Otra distinción:

```text
ParameterId
≠
ExecutionParameterSlotId
≠
SQLPlaceholder
≠
DriverBindingPosition
```

---

# 27. Placeholder ownership

Los placeholders concretos pertenecen al SQL Compiler.

---

# 28. Driver binding ownership

Los detalles finales:

```text
PDO::PARAM_INT
driver-native parameter type
positional binding index
```

pertenecen a Compilation/Execution/Driver integration.

---

# 29. ExecutionPlanGraph

El plan será modelado como grafo.

```php
final readonly class ExecutionPlanGraph
{
    public function __construct(
        public ExecutionNodeMap $nodes,
        public ExecutionEdgeSet $edges,
        public ExecutionNodeIdSet $roots,
    ) {}
}
```

---

# 30. ExecutionNode

Contrato base:

```php
interface ExecutionNode
{
    public function id(): ExecutionNodeId;

    public function kind(): ExecutionNodeKind;

    public function inputs(): ExecutionInputSet;
}
```

---

# 31. ExecutionNodeKind

V1 podrá incluir:

```text
DATABASE_QUERY
DATABASE_MUTATION
DATABASE_CURSOR
FRAMEWORK_FILTER
FRAMEWORK_PROJECT
FRAMEWORK_HASH_BUILD
FRAMEWORK_HASH_PROBE
FRAMEWORK_SORT
FRAMEWORK_AGGREGATE
FRAMEWORK_DISTINCT
FRAMEWORK_WINDOW
FRAMEWORK_SET_OPERATION
MATERIALIZE
BUFFER
SPOOL
PARAMETER_BIND
RESULT_FORWARD
RESULT_MERGE
RESULT_LIMIT
CLEANUP
EXTENSION
```

---

# 32. ExecutionUnit

El concepto operativo principal será:

```text
ExecutionUnit
```

Una unidad representa trabajo ejecutable independiente o coordinable.

---

# 33. ExecutionNode ≠ ExecutionUnit

No todos los nodos necesariamente necesitan convertirse en unidades aisladas.

Varias operaciones pueden fusionarse.

---

# 34. Ejemplo

```text
PhysicalFilter
     │
PhysicalProject
     │
DatabaseDelegatedScan
```

puede convertirse en:

```text
DatabaseExecutionUnit
```

si las tres operaciones son delegables.

---

# 35. Operator fusion

El sistema podrá fusionar operaciones sólo si:

```text
semantics preserved
ownership preserved
instrumentation contract preserved
cancellation preserved
resource behavior remains valid
```

---

# 36. ExecutionStage

Las unidades se agruparán en:

```text
ExecutionStage
```

---

# 37. Stage model

```php
final readonly class ExecutionStage
{
    public function __construct(
        public ExecutionStageId $id,
        public ExecutionUnitSet $units,
        public ExecutionStageDependencySet $dependencies,
        public ExecutionStagePropertySet $properties,
    ) {}
}
```

---

# 38. Stage purpose

Un stage agrupa trabajo que comparte una frontera de ejecución.

Ejemplos:

```text
database statement boundary
materialization boundary
blocking operator boundary
parallel scheduling boundary
transaction-sensitive boundary
```

---

# 39. Stage ≠ transaction

Un stage no crea automáticamente una transacción.

---

# 40. Stage ≠ thread

Un stage tampoco equivale necesariamente a:

```text
thread
process
worker
fiber
coroutine
```

---

# 41. ExecutionStageGraph

```text
Stage A
   │
   ▼
Stage B
   │
   ├────► Stage C
   │
   ▼
Stage D
```

permitirá representar dependencias.

---

# 42. Stage DAG

En queries no recursivas, el stage graph deberá ser normalmente un DAG.

---

# 43. Recursión

Queries recursivas requerirán estructuras explícitas.

No se representarán como ciclos arbitrarios en un DAG común.

---

# 44. RecursiveExecutionRegion

Propuesta:

```php
final readonly class RecursiveExecutionRegion
{
    public function __construct(
        public RecursiveRegionId $id,
        public ExecutionStageId $anchor,
        public ExecutionStageId $recursiveMember,
        public RecursiveTerminationContract $termination,
        public RecursiveMaterializationContract $materialization,
    ) {}
}
```

---

# 45. Execution edge kinds

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

# 46. Data edge

Transporta datos lógicos/runtime entre unidades.

---

# 47. Control edge

Expresa orden requerido aunque no transporte filas.

---

# 48. Parameter edge

Expresa dependencia de valores runtime.

---

# 49. Correlation edge

Expresa que un subplan requiere valores producidos por un scope externo.

---

# 50. Correlation ≠ nested-loop implementation

La existencia de una correlación no obliga conceptualmente a un loop físico específico.

La estrategia ya habrá sido seleccionada por etapas anteriores.

---

# 51. Materialization edge

Representa dependencia sobre un resultado materializado.

---

# 52. Cleanup edge

Define relaciones de destrucción/liberación.

---

# 53. Cancellation edge

Define propagación de cancelación.

---

# 54. Execution topology

El sistema deberá poder representar:

```text
pipeline
fan-out
fan-in
materialization
repeated consumption
database delegation
framework processing
recursive execution
```

---

# 55. Pipeline

Ejemplo:

```text
DatabaseRead
     │
     ▼
FrameworkFilter
     │
     ▼
FrameworkProject
     │
     ▼
ResultForward
```

---

# 56. Fan-out

```text
MaterializedResult
      ├────► Consumer A
      └────► Consumer B
```

---

# 57. Fan-in

```text
Source A ─┐
          ├──► ResultMerge
Source B ─┘
```

---

# 58. DatabaseExecutionUnit

Representa trabajo delegado al servidor de base de datos.

```php
final readonly class DatabaseExecutionUnit implements ExecutionUnit
{
    public function __construct(
        public ExecutionUnitId $id,
        public PhysicalSubplanId $physicalSubplan,
        public DatabaseExecutionIntent $intent,
        public DatabaseResultContract $result,
        public DatabaseExecutionRequirementSet $requirements,
    ) {}
}
```

---

# 59. DatabaseExecutionIntent

Podrá indicar:

```text
READ
INSERT
UPDATE
DELETE
DDL
UTILITY
EXTENSION
```

aunque DDL tenga su propio pipeline especializado posteriormente.

---

# 60. DatabaseExecutionUnit ≠ SQL

La unidad contiene intención y referencias estructuradas.

El Compiler produce:

```text
CompiledDatabaseCommand
```

---

# 61. CompiledDatabaseCommand

Arquitectura:

```text
DatabaseExecutionUnit
        │
        ▼
SQL Compiler
        │
        ▼
CompiledDatabaseCommand
        │
        ▼
Execution Engine
```

---

# 62. Compiled command ≠ live statement

Incluso:

```text
CompiledDatabaseCommand
```

debe ser distinto de:

```text
PDOStatement
```

---

# 63. FrameworkExecutionUnit

Representa trabajo controlado por VoltStack.

```php
interface FrameworkExecutionUnit extends ExecutionUnit
{
    public function operatorKind(): FrameworkExecutionOperatorKind;
}
```

---

# 64. Framework-controlled examples

```text
FrameworkHashJoin
FrameworkMerge
FrameworkSort
FrameworkAggregate
FrameworkDistinct
FrameworkMaterialization
FrameworkShardMerge
FrameworkFederatedUnion
```

---

# 65. V1 philosophy

Para una única base SQL tradicional:

```text
prefer database delegation
```

cuando sea semánticamente seguro y razonable.

---

# 66. No unnecessary framework execution

VoltStack no deberá descargar millones de filas a PHP para ejecutar un hash join si la base puede realizar correctamente ese trabajo.

---

# 67. Future distributed scenarios

Framework execution adquiere mayor importancia para:

```text
sharding
federation
cross-database queries
cross-provider queries
distributed aggregation
distributed sorting
```

---

# 68. Hybrid execution

El plan podrá mezclar:

```text
DatabaseExecutionUnit
+
FrameworkExecutionUnit
```

---

# 69. Ejemplo híbrido

```text
Database A Query
       │
       ├─────────────┐
       │             │
       ▼             ▼
                 FrameworkMerge
       ▲             ▲
       │             │
Database B Query ────┘
```

---

# 70. ExecutionControlBoundary

Cada unidad deberá declarar:

```text
ExecutionControlLevel
```

heredando la distinción del Physical Plan:

```text
FRAMEWORK_CONTROLLED
COMPILER_INFLUENCED
DATABASE_DELEGATED
HYBRID
```

---

# 71. Control level ≠ trust level

No confundir:

```text
who executes
```

con:

```text
whether input is trusted
```

---

# 72. StatementExecutionUnit

Podrá especializar `DatabaseExecutionUnit`.

```php
final readonly class StatementExecutionUnit
{
    public function __construct(
        public ExecutionUnitId $id,
        public CompilableStatementReference $statement,
        public ExecutionParameterSlotSet $parameters,
        public StatementResultContract $result,
        public StatementExecutionRequirements $requirements,
    ) {}
}
```

---

# 73. Compilation timing

Dos estrategias podrán existir:

```text
plan-time compilation
execution-time compilation
```

dependiendo de cache/capabilities.

---

# 74. Preferred separation

Arquitectónicamente:

```text
ExecutionPlan
→
CompiledExecutionPlan
→
ExecutionInstance
```

podrá utilizarse cuando sea necesario.

---

# 75. ExecutionPlan vs CompiledExecutionPlan

```text
ExecutionPlan
=
platform-aware executable topology

CompiledExecutionPlan
=
target-dialect commands + execution topology
```

---

# 76. Compiler ownership

SQL generation sigue perteneciendo al bloque:

```text
66–75 SQL Compiler
```

No al documento actual.

---

# 77. Runtime parameter binding

El Execution Plan define:

```text
which unit consumes which parameter
```

pero no contiene necesariamente:

```text
actual parameter value
```

---

# 78. Parameter dependency graph

Ejemplo:

```text
Parameter(user_id)
       │
       ▼
ExecutionSlot P1
       │
       ▼
DatabaseExecutionUnit U4
```

---

# 79. Correlated parameter slot

Para subqueries correlacionadas:

```text
OuterRowSymbol
      │
      ▼
CorrelationSlot
      │
      ▼
CorrelatedExecutionUnit
```

---

# 80. CorrelationSlot ≠ query ParameterId

Un valor derivado de una fila externa no es automáticamente un parámetro declarado por el usuario.

---

# 81. RuntimeValueSource

Se modelarán orígenes:

```text
USER_BINDING
FRAMEWORK_BINDING
CORRELATION
GENERATED_VALUE
TRANSACTION_CONTEXT
EXECUTION_DERIVED
EXTENSION
```

---

# 82. Sensitive parameters

Cada slot conservará:

```text
ParameterSensitivity
```

para logging/telemetry.

---

# 83. No sensitive values in plan

El plan sólo contiene clasificación.

---

# 84. Resource planning

El Execution Plan debe especificar qué recursos runtime necesitará.

---

# 85. ExecutionResourcePlan

```php
final readonly class ExecutionResourcePlan
{
    public function __construct(
        public ResourceRequirementSet $requirements,
        public ResourceOwnershipGraph $ownership,
        public ResourceLifetimeTable $lifetimes,
    ) {}
}
```

---

# 86. Resource kinds

Podrán incluir:

```text
DATABASE_CONNECTION
STATEMENT
CURSOR
MEMORY_BUFFER
TEMPORARY_STORAGE
HASH_TABLE
SORT_BUFFER
SPOOL
RESULT_BUFFER
STREAM
EXTENSION_RESOURCE
```

---

# 87. Requirement ≠ resource

Siempre:

```text
ResourceRequirement
≠
LiveResource
```

---

# 88. Resource ownership

Todo recurso adquirido en runtime deberá tener propietario explícito.

---

# 89. Ownership principle

```text
Every acquired resource must have exactly one
well-defined cleanup responsibility.
```

---

# 90. Shared consumption

Un recurso puede tener múltiples consumidores.

Eso no implica múltiples propietarios.

---

# 91. CursorOwnership

Ejemplo:

```php
final readonly class CursorOwnership
{
    public function __construct(
        public ExecutionUnitId $owner,
        public ExecutionUnitIdSet $consumers,
        public CursorLifetime $lifetime,
    ) {}
}
```

---

# 92. Cursor lifetime

Podrá ser:

```text
UNIT
STAGE
PIPELINE
EXECUTION
STREAMING_RESULT
```

---

# 93. Streaming result

Cuando un cursor se entrega al consumidor final:

```text
cursor lifetime
```

puede superar la ejecución inicial del statement.

---

# 94. Streaming ownership transfer

Debe existir una transferencia explícita:

```text
Execution Engine
      │
      ▼
StreamingResult
```

---

# 95. No premature cleanup

El engine no podrá cerrar un cursor que ha transferido legítimamente al resultado streaming.

---

# 96. BufferOwnership

Similarmente:

```text
Buffer
```

deberá tener:

```text
owner
consumers
lifetime
release condition
```

---

# 97. Materialization runtime

`PhysicalMaterialize` podrá convertirse en:

```text
MaterializationExecutionUnit
```

---

# 98. Materialization medium

El plan puede expresar preferencia/requisito:

```text
MEMORY
TEMPORARY_STORAGE
AUTO
EXTENSION
```

---

# 99. AUTO

Permite que runtime seleccione dentro de restricciones declaradas.

---

# 100. Materialization ≠ cache

Sigue siendo estado temporal de una ejecución.

---

# 101. Spill planning

Operadores spillable deberán declarar:

```text
SpillContract
```

---

# 102. SpillContract

Puede indicar:

```text
spill allowed
spill required under pressure
temporary storage requirement
serialization format capability
cleanup responsibility
```

---

# 103. No hidden temp files

Todo temporary storage deberá estar representado en resource planning.

---

# 104. Execution scheduling

El plan deberá expresar dependencias de scheduling sin acoplarse a una implementación concreta de concurrencia.

---

# 105. Scheduling ≠ threads

No se modelará directamente:

```text
Thread #1
Thread #2
```

---

# 106. ExecutionSchedulingConstraint

Ejemplos:

```text
MUST_RUN_BEFORE
MAY_RUN_CONCURRENTLY
MUST_COMPLETE_BEFORE
STREAMS_INTO
REQUIRES_MATERIALIZATION
SERIALIZED_WITH
TRANSACTION_BOUND
```

---

# 107. Parallel eligibility

Una unidad puede declarar:

```text
PARALLELIZABLE
SERIAL_ONLY
PLATFORM_DELEGATED
UNKNOWN
```

---

# 108. Parallelizable ≠ parallel

La ejecución real dependerá del runtime/resource governor.

---

# 109. Persistent runtime compatibility

Esto permitirá que FrankenPHP/RoadRunner/OpenSwoole decidan scheduling sin modificar semántica del plan.

---

# 110. FrankenPHP

FrankenPHP seguirá siendo runtime predeterminado de VoltStack.

El Execution Plan no contendrá dependencias directas a sus clases.

---

# 111. Runtime adapter

La integración ocurrirá mediante:

```text
ExecutionRuntimeAdapter
```

---

# 112. ExecutionRuntimeAdapter

Contrato conceptual:

```php
interface ExecutionRuntimeAdapter
{
    public function capabilities(): ExecutionRuntimeCapabilities;

    public function scheduler(): ExecutionScheduler;

    public function resourceManager(): ExecutionResourceManager;
}
```

---

# 113. Runtime-specific implementation

Ejemplos futuros:

```text
FrankenPhpExecutionRuntimeAdapter
RoadRunnerExecutionRuntimeAdapter
OpenSwooleExecutionRuntimeAdapter
```

---

# 114. Execution ordering

El orden entre unidades deberá derivarse del grafo.

No del orden accidental de registro.

---

# 115. Stable ordering

Para nodos independientes donde sea necesario un orden determinista:

```text
ExecutionNodeId
```

o un ordering estable equivalente podrá utilizarse como tie-breaker.

---

# 116. Cancellation

La cancelación será una propiedad de primera clase.

---

# 117. ExecutionCancellationPlan

```php
final readonly class ExecutionCancellationPlan
{
    public function __construct(
        public CancellationNodeSet $nodes,
        public CancellationPropagationGraph $propagation,
        public CancellationPolicy $policy,
    ) {}
}
```

---

# 118. Cancellation source

Podrá originarse en:

```text
user request
HTTP disconnect
timeout
application cancellation token
worker shutdown
resource governor
database cancellation
extension
```

---

# 119. Cancellation propagation

Ejemplo:

```text
Result Consumer Cancelled
          │
          ▼
Framework Merge
          │
      ┌───┴───┐
      ▼       ▼
 DB Query A  DB Query B
```

ambas ramas deberán poder recibir cancelación cuando sea seguro.

---

# 120. Cancellation ≠ failure

Se distinguirán:

```text
SUCCESS
FAILURE
CANCELLED
TIMED_OUT
```

---

# 121. Timeout

El plan podrá contener:

```text
ExecutionTimeoutRequirement
```

pero el runtime mide/aplica el tiempo.

---

# 122. Timeout levels

Podrán existir:

```text
QUERY
STAGE
UNIT
DATABASE_STATEMENT
RESOURCE_WAIT
```

---

# 123. Timeout composition

El sistema deberá definir cómo interactúan múltiples límites.

---

# 124. Effective timeout

Conceptualmente:

```text
EffectiveTimeout
=
minimum applicable remaining deadline
```

cuando los contratos así lo establezcan.

---

# 125. Deadline preferred

Internamente será conveniente representar:

```text
deadline
```

en vez de reiniciar timers independientes que excedan el timeout global.

---

# 126. Execution failure

El plan debe permitir propagación estructurada de errores.

---

# 127. FailurePropagationGraph

Puede diferir parcialmente del data graph.

---

# 128. Failure policy

Ejemplos:

```text
FAIL_FAST
COLLECT_BRANCH_FAILURES
ROLLBACK_REQUIRED
CANCEL_DEPENDENTS
EXTENSION_DEFINED
```

---

# 129. Query execution default

Para una query ordinaria:

```text
FAIL_FAST
```

será normalmente apropiado.

---

# 130. Distributed future

Queries federadas podrán necesitar errores compuestos.

---

# 131. Failure ≠ cleanup

Un error inicia cleanup.

No lo sustituye.

---

# 132. Cleanup

El cleanup será explícito.

---

# 133. ExecutionCleanupPlan

```php
final readonly class ExecutionCleanupPlan
{
    public function __construct(
        public CleanupActionSet $actions,
        public CleanupDependencyGraph $dependencies,
        public CleanupPolicy $policy,
    ) {}
}
```

---

# 134. Cleanup resources

Puede incluir:

```text
close cursor
release statement
release connection lease
delete temporary file
release memory buffer
destroy hash table
release spool
extension cleanup
```

---

# 135. Connection release

El plan puede declarar:

```text
release connection lease
```

pero no ejecutarlo.

---

# 136. Cleanup order

Generalmente será inverso a ciertas relaciones de adquisición/dependencia.

---

# 137. Cleanup idempotency

Las acciones de cleanup deberían diseñarse para tolerar:

```text
partial execution
failure during cleanup
repeated defensive invocation
```

cuando sea técnicamente posible.

---

# 138. Cleanup failure

Un fallo de cleanup no deberá ocultar automáticamente el error primario.

---

# 139. Error aggregation

El runtime deberá poder conservar:

```text
PrimaryExecutionFailure
+
CleanupFailureSet
```

---

# 140. Cancellation cleanup

La cancelación también ejecuta cleanup.

---

# 141. Timeout cleanup

El timeout también ejecuta cleanup.

---

# 142. Partial startup cleanup

Si sólo se adquirió una parte de los recursos:

```text
cleanup only acquired resources
```

---

# 143. Resource acquisition plan

Podrá existir:

```php
final readonly class ResourceAcquisitionPlan
{
    public function __construct(
        public ResourceAcquisitionStepSet $steps,
        public ResourceDependencyGraph $dependencies,
    ) {}
}
```

---

# 144. Lazy acquisition

Preferir adquisición:

```text
as late as safely possible
```

cuando reduzca recursos retenidos.

---

# 145. Connection acquisition

No necesariamente debe ocurrir al inicio de toda la query.

---

# 146. Multiple connections

Una ejecución futura distribuida puede requerir:

```text
ConnectionLease A
ConnectionLease B
ConnectionLease C
```

con ownership explícito.

---

# 147. Connection lease ≠ connection object

El plan expresa requerimiento de lease.

El runtime obtiene el objeto real.

---

# 148. Transaction requirements

Un execution unit puede declarar:

```text
TransactionExecutionRequirement
```

---

# 149. TransactionExecutionRequirement

Puede indicar:

```text
REQUIRES_EXISTING
REQUIRES_TRANSACTION
MAY_RUN_WITHOUT
MUST_SHARE_TRANSACTION_WITH
MUST_NOT_CREATE_IMPLICITLY
```

según operación.

---

# 150. Execution planner no comienza transaction

El `TransactionManager` conserva esa responsabilidad.

---

# 151. Transaction grouping

Un conjunto de unidades puede requerir:

```text
same transaction context
```

---

# 152. Cross-database transaction

No se asumirá automáticamente que múltiples conexiones pueden compartir transacción atómica.

---

# 153. Distributed transaction

Será una capability separada.

---

# 154. DML execution

`INSERT`, `UPDATE` y `DELETE` deberán preservar:

```text
mutation set
transaction requirements
locking requirements
RETURNING contract
failure semantics
```

---

# 155. Mutation unit

Ejemplo:

```php
final readonly class DatabaseMutationExecutionUnit
{
    public function __construct(
        public ExecutionUnitId $id,
        public MutationKind $kind,
        public PhysicalSubplanId $physicalPlan,
        public MutationExecutionRequirementSet $requirements,
        public ?ReturningExecutionContract $returning,
    ) {}
}
```

---

# 156. RETURNING

Puede producir:

```text
cursor
buffered result
scalar result
row collection
```

según plataforma y higher-level contract.

---

# 157. Platform emulation

Si una plataforma no soporta `RETURNING` nativo, cualquier emulación deberá estar explícitamente planificada y demostrar semántica equivalente.

---

# 158. No unsafe emulation

No:

```text
UPDATE
then SELECT based on guessed criteria
```

si concurrencia puede producir resultados incorrectos.

---

# 159. Result contract

Cada root deberá definir:

```text
ExecutionResultContract
```

---

# 160. Result kinds

Podrán incluir:

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

# 161. Result contract ≠ hydration

`ROW_STREAM` todavía contiene datos de database/query layer.

No entidades ORM.

---

# 162. Result schema

Debe preservar:

```text
output symbols
types
nullability
ordering guarantees
cardinality guarantees where known
```

---

# 163. Result ordering

Sólo se declarará si el plan realmente garantiza ordering observable.

---

# 164. No accidental ordering

No se expondrá como garantía el orden accidental de:

```text
index scan
hash table
database engine
```

si el logical contract no lo garantiza.

---

# 165. Result buffering

El plan podrá seleccionar:

```text
STREAM
BUFFER
AUTO
```

según requirements.

---

# 166. Streaming

Favorece memoria baja.

---

# 167. Buffering

Puede ser necesario para:

```text
rewind
multiple consumers
sort
aggregate
window
result reuse
```

---

# 168. AUTO

Permite decisión runtime limitada por el contrato.

---

# 169. Backpressure

Para streaming, el execution model deberá soportar conceptualmente:

```text
producer
→ bounded channel
→ consumer
```

o mecanismo equivalente.

---

# 170. Backpressure ≠ buffering everything

El objetivo es impedir crecimiento ilimitado.

---

# 171. Backpressure contract

Podrá declarar:

```text
PULL
PUSH_BOUNDED
PLATFORM_CURSOR
EXTENSION
```

---

# 172. PHP V1

Para V1, el modelo podrá favorecer:

```text
pull-based cursor consumption
```

por simplicidad y compatibilidad.

---

# 173. Future async execution

La arquitectura deberá permitir:

```text
fibers
event loops
coroutines
async drivers
```

sin convertirlos en requisito del core.

---

# 174. Async ≠ concurrency

Un query async no implica automáticamente ejecución paralela.

---

# 175. Execution state machine

El runtime podrá manejar:

```text
CREATED
PREPARING
READY
RUNNING
DRAINING
COMPLETED
FAILED
CANCELLED
TIMED_OUT
CLEANING
CLOSED
```

---

# 176. State belongs to ExecutionInstance

No a `ExecutionPlan`.

---

# 177. ExecutionInstance

Modelo conceptual:

```php
final class ExecutionInstance
{
    private ExecutionState $state;

    public function __construct(
        public readonly ExecutionInstanceId $id,
        public readonly ExecutionPlan $plan,
        public readonly RuntimeBindingSet $bindings,
        public readonly ExecutionRuntimeContext $context,
    ) {}
}
```

---

# 178. ExecutionRuntimeContext

Aquí sí pueden existir referencias runtime controladas:

```text
connection manager
transaction context
cancellation token
deadline
resource manager
telemetry execution scope
```

pero esta clase pertenece al Execution Engine, no al plan artifact.

---

# 179. ExecutionPlanBuilder

El constructor principal:

```php
interface ExecutionPlanBuilder
{
    public function build(
        PhysicalQueryPlan $physicalPlan,
        ExecutionPlanningContext $context,
    ): ExecutionPlan;
}
```

---

# 180. Lowering

La transformación:

```text
PhysicalPlanNode
→
ExecutionNode(s)
```

se denominará:

```text
Execution Lowering
```

---

# 181. Lowering ≠ compilation

Lowering transforma estrategia física en topología ejecutable.

Compilation transforma operaciones delegadas a representación de plataforma.

---

# 182. ExecutionLoweringRule

```php
interface ExecutionLoweringRule
{
    public function supports(
        PhysicalPlanNode $node,
        ExecutionPlanningContext $context,
    ): bool;

    public function lower(
        PhysicalPlanNode $node,
        ExecutionLoweringContext $context,
    ): ExecutionLoweringResult;
}
```

---

# 183. Rule registry

Será:

```text
bootstrap mutable
→
validated
→
frozen
```

---

# 184. Deterministic lowering

La selección de lowering rule deberá ser determinista.

---

# 185. No last-one-wins

Dos reglas incompatibles deberán producir:

```text
bootstrap configuration error
```

o resolución explícita.

---

# 186. Lowering result

Podrá producir:

```text
one execution unit
multiple execution units
execution subgraph
database delegated unit
framework controlled region
```

---

# 187. Example scan lowering

Physical:

```text
DatabaseDelegatedScan(users)
```

Execution:

```text
DatabaseExecutionUnit(users scan)
```

---

# 188. Example framework hash join

Physical:

```text
FrameworkHashJoin
├── DatabaseSubplan A
└── DatabaseSubplan B
```

Execution:

```text
Stage A
└── DatabaseExecutionUnit A

Stage B
└── DatabaseExecutionUnit B

Stage C
├── HashBuildUnit
└── HashProbeUnit
```

---

# 189. Hash build state

El hash table runtime no existe en el plan.

El plan sólo contiene:

```text
HashTableResourceRequirement
```

---

# 190. Sort state

Igualmente:

```text
SortBufferRequirement
```

no un array PHP precargado.

---

# 191. Database command compilation reference

Cada delegable unit conservará suficiente estructura para que el compiler produzca:

```text
CompiledDatabaseCommand
```

sin reconstruir el Physical Plan.

---

# 192. No semantic reinterpretation

El Compiler no deberá decidir de nuevo qué significa el query.

---

# 193. No physical reinterpretation

El Execution Engine tampoco deberá volver a seleccionar joins/access paths.

---

# 194. Responsibility chain

```text
Semantic Engine
    determines meaning

Optimizer
    chooses equivalent logical form

Logical Planner
    constructs logical operations

Physical Planner
    chooses physical strategy

Execution Planner
    constructs executable topology

Compiler
    emits target representation

Execution Engine
    executes
```

---

# 195. Instrumentation

El plan podrá incluir:

```text
ExecutionInstrumentationPlan
```

---

# 196. Instrumentation points

Ejemplos:

```text
query start
query end
stage start
stage end
unit start
unit end
database command start
database command end
rows produced
rows consumed
buffer spill
cancellation
timeout
failure
cleanup
```

---

# 197. Instrumentation ≠ telemetry backend

El plan define puntos.

El Telemetry System decide exportación.

---

# 198. No OpenTelemetry dependency in core plan

El core no dependerá directamente de OpenTelemetry.

---

# 199. InstrumentationPointId

IDs estables dentro del plan podrán permitir correlación.

---

# 200. Row counters

Podrán habilitarse selectivamente.

No deben introducir overhead obligatorio excesivo.

---

# 201. Debug profile

Puede activar instrumentación más detallada.

---

# 202. Production profile

Puede mantener instrumentación mínima.

---

# 203. Sensitive values

Instrumentation nunca deberá requerir almacenar valores sensibles en el plan.

---

# 204. Execution dependencies

`ExecutionDependencySet` podrá incluir:

```text
physical plan fingerprint
compiled command dependencies
platform capabilities
execution extension versions
runtime capability requirements
resource profile assumptions
```

---

# 205. Dependency invalidation

Ejemplo:

```text
execution extension removed
```

puede invalidar el Execution Plan.

---

# 206. Runtime capability invalidation

Un plan que requiere:

```text
async cursor
```

no podrá ejecutarse en un runtime que carece de esa capability.

---

# 207. Hard compatibility check

Antes de crear una `ExecutionInstance`:

```text
ExecutionPlanCompatibilityChecker
```

deberá validar requirements.

---

# 208. ExecutionPlanFingerprint

Podrá incorporar:

```text
PhysicalPlanFingerprint
ExecutionTopology
ExecutionUnitKinds
StageStructure
ParameterLayout
ResourceRequirements
ControlBoundaries
LoweringRuleVersions
ExtensionVersions
ExecutionCapabilities
```

---

# 209. Runtime values excluded

Nunca por defecto:

```text
password
email
tenant identifier
search term
token
```

---

# 210. Plan caching

Podrá existir:

```text
PhysicalPlanFingerprint
       │
       ▼
ExecutionPlanCache
       │
       ▼
ExecutionPlan
```

---

# 211. Cache validation

Debe verificar:

```text
dependencies
capabilities
extensions
runtime compatibility
```

---

# 212. Execution plan reuse

Será particularmente importante en persistent workers.

---

# 213. Persistent worker safety

Un cached `ExecutionPlan` será reusable porque no contiene:

```text
current bindings
current cursor
current connection
current transaction
current buffers
current tenant runtime state
```

---

# 214. Tenant semantics

Las políticas tenant ya deberán estar representadas en etapas semánticas/lógicas/físicas.

---

# 215. Execution plan tenant state

No deberá guardar:

```text
current tenant singleton
```

---

# 216. Tenant runtime binding

Si un tenant identifier es un parámetro:

```text
ParameterId
→
ExecutionParameterSlot
→
RuntimeBinding
```

---

# 217. Plan reuse across tenants

Sólo será permitido cuando:

```text
semantic structure identical
physical dependencies compatible
execution requirements compatible
tenant-specific data represented as bindings/context
security policy permits reuse
```

---

# 218. Connection-per-tenant

Si diferentes tenants usan diferentes bases físicas:

```text
ConnectionResolutionRequirement
```

deberá permanecer dinámico cuando corresponda.

---

# 219. Connection identity in plan

No guardar:

```text
live PDO connection
```

---

# 220. ConnectionResolutionDescriptor

Puede guardar:

```text
logical connection role
routing requirement
tenant-aware resolution policy id
read/write intent
consistency requirement
```

---

# 221. Read/write routing

Un unit podrá declarar:

```text
READ
WRITE
PRIMARY_REQUIRED
REPLICA_ALLOWED
STICKY_REQUIRED
```

sin seleccionar todavía necesariamente una conexión viva.

---

# 222. Replica selection

Pertenece al Connection/ReadWrite routing runtime.

---

# 223. Failover

El Execution Plan puede declarar:

```text
failover permitted
```

pero la estrategia concreta pertenece a sistemas posteriores de resiliencia.

---

# 224. Retry

El plan podrá indicar:

```text
retry eligibility metadata
```

pero no ejecutar retries.

---

# 225. Retry eligibility ≠ retry policy

La política pertenece al:

```text
Execution Retry / Resilience System
```

---

# 226. Mutation retry

Será especialmente conservadora debido a:

```text
side effects
transaction state
idempotency
generated values
```

---

# 227. Idempotency

Un execution unit podrá declarar:

```text
IDEMPOTENT
CONDITIONALLY_IDEMPOTENT
NON_IDEMPOTENT
UNKNOWN
```

como evidencia para sistemas posteriores.

---

# 228. Retry cannot be guessed

Nunca:

```text
SELECT = always retryable
UPDATE = never retryable
```

como regla absoluta sin contexto.

---

# 229. Lock ownership

Lock requirements permanecen asociados a unidades/stages correspondientes.

---

# 230. Lock lifetime

Puede depender de:

```text
statement
transaction
cursor
platform semantics
```

---

# 231. Execution Plan cannot release DB locks itself

Sólo puede preservar sus requisitos.

---

# 232. Execution planning budget

Aunque menos combinatorio que physical planning, el lowering deberá ser bounded.

---

# 233. ExecutionPlanningBudget

Podrá limitar:

```text
execution nodes
stages
fusion attempts
materialization nodes
instrumentation points
extension expansions
recursive regions
```

---

# 234. Expansion attack resistance

Una extensión no podrá expandir:

```text
1 PhysicalNode
→
millions of ExecutionNodes
```

sin budget.

---

# 235. Budget exhaustion

Si no puede construirse un plan ejecutable completo:

```text
fail explicitly
```

---

# 236. No partial plan return

No se entregará un `ExecutionPlan` incompleto como válido.

---

# 237. ExecutionPlanValidator

Antes del freeze:

```php
interface ExecutionPlanValidator
{
    public function validate(
        ExecutionPlanDraft $plan,
        ExecutionPlanningContext $context,
    ): ExecutionPlanValidationResult;
}
```

---

# 238. Validation categories

```text
GRAPH
STAGES
PARAMETERS
RESOURCES
OWNERSHIP
CANCELLATION
CLEANUP
TRANSACTIONS
RESULT
CAPABILITIES
SECURITY
EXTENSIONS
MAPPING
```

---

# 239. Graph validation

Verifica:

```text
all referenced nodes exist
roots valid
dependencies valid
no illegal cycles
recursive cycles explicit
```

---

# 240. Parameter validation

Cada consumed slot deberá:

```text
exist
have compatible type
have valid source
have sensitivity metadata
```

---

# 241. Resource validation

Todo resource requirement deberá tener:

```text
acquisition strategy
owner
lifetime
cleanup responsibility
```

---

# 242. Cursor validation

Ningún cursor podrá quedar:

```text
unowned
```

---

# 243. Cleanup validation

Todo recurso adquirible deberá tener cleanup path.

---

# 244. Cancellation validation

Las ramas ejecutables deberán tener comportamiento definido ante cancelación.

---

# 245. Result validation

Cada root deberá producir exactamente el `ExecutionResultContract`.

---

# 246. Physical mapping validation

Todo execution node deberá poder rastrearse a:

```text
PhysicalPlanNode
```

o a infraestructura introducida explícitamente.

---

# 247. Infrastructure provenance

Ejemplos:

```text
cleanup node
buffer node
parameter binding node
instrumentation node
```

pueden no mapear 1:1 a physical nodes.

Deben marcar:

```text
ExecutionInfrastructureOrigin
```

---

# 248. PhysicalExecutionMapping

Modelo:

```php
final readonly class PhysicalExecutionMapping
{
    public function __construct(
        public PhysicalToExecutionMap $forward,
        public ExecutionToPhysicalOriginMap $reverse,
    ) {}
}
```

---

# 249. Mapping not 1:1

Siempre:

```text
PhysicalNode
→
0..N ExecutionNodes
```

y una unidad fusionada puede corresponder a:

```text
N PhysicalNodes
→
1 ExecutionUnit
```

---

# 250. Example database pushdown

Physical:

```text
PhysicalFilter
└── PhysicalProject
    └── DatabaseDelegatedScan
```

Execution:

```text
DatabaseExecutionUnit
```

Mapping:

```text
PhysicalFilter ──────┐
PhysicalProject ─────┼──► DatabaseExecutionUnit
DelegatedScan ───────┘
```

---

# 251. Example materialization

Physical:

```text
PhysicalMaterialize
```

Execution puede generar:

```text
MaterializationProducer
TemporaryStorageRequirement
MaterializationConsumer
CleanupAction
```

---

# 252. Explain execution plan

El sistema deberá poder representar:

```text
ExecutionPlan
├── Stage 1
│   └── DatabaseExecutionUnit #1
├── Stage 2
│   └── DatabaseExecutionUnit #2
├── Stage 3
│   ├── HashBuildUnit
│   └── HashProbeUnit
└── Stage 4
    └── ResultForwardUnit
```

---

# 253. Explain resource ownership

Ejemplo:

```text
HashTable H1
Owner: HashBuildUnit
Consumers: HashProbeUnit
Lifetime: Stage 3
Cleanup: ReleaseHashTable(H1)
```

---

# 254. Explain cancellation

```text
Cancel Root
   │
   ├── DatabaseUnit #1
   ├── DatabaseUnit #2
   └── FrameworkJoin #3
```

---

# 255. Explain parameter layout

Sólo:

```text
P1: Domain<UserId>
P2: DateTime
P3: Sensitive<String>
```

No valores.

---

# 256. Explain stages

Debe poder explicar por qué existe una frontera:

```text
Stage 2 created because:
- materialization required
- downstream input must be rewindable
```

---

# 257. Explain fusion

También:

```text
PhysicalFilter + PhysicalProject
fused into DatabaseExecutionUnit
because both operations are safely delegable.
```

---

# 258. Diagnostics

Cada lowering decision podrá generar:

```text
ExecutionPlanningTrace
```

---

# 259. Trace events

Ejemplos:

```text
NODE_LOWERED
NODES_FUSED
STAGE_CREATED
RESOURCE_REQUIRED
MATERIALIZATION_INSERTED
PARAMETER_SLOT_ALLOCATED
CLEANUP_REGISTERED
CANCELLATION_EDGE_ADDED
EXTENSION_LOWERED
VALIDATION_COMPLETED
```

---

# 260. Deterministic IDs

Los IDs generados durante planning deberán ser deterministas dentro de una ejecución de planning dada y no depender de globals.

---

# 261. No global counters

Evitar:

```php
static $nextExecutionNodeId;
```

---

# 262. Operation-scoped allocator

Usar:

```text
ExecutionPlanningIdAllocator
```

---

# 263. Security

El Execution Plan debe preservar security provenance.

---

# 264. Security policy execution

Si una política exige:

```text
database-side enforcement
```

el execution planner no podrá moverla a un framework post-filter.

---

# 265. Security boundary descriptor

Podrá existir:

```text
SecurityExecutionBoundary
```

---

# 266. SecurityExecutionBoundary

Puede declarar:

```text
MUST_EXECUTE_IN_DATABASE
MAY_EXECUTE_IN_FRAMEWORK
MUST_EXECUTE_BEFORE_EXPORT
EXTENSION_DEFINED
```

---

# 267. Authorization policy joins

No podrán desaparecer durante lowering.

---

# 268. Tenant predicates

Tampoco podrán quedar fuera de la región donde deben aplicarse.

---

# 269. Sensitive columns

El plan podrá conservar metadata:

```text
SensitiveOutputClassification
```

para telemetry/result handling.

---

# 270. Data minimization

Un execution unit no deberá solicitar columnas que el physical/logical dependency analysis ya eliminó.

---

# 271. Projection boundary

Particularmente en queries federadas:

```text
push minimal required projection
```

reduce exposición de datos.

---

# 272. ORM independence

El sistema no conoce:

```text
EntityManager
UnitOfWork
IdentityMap
Model
Repository
```

---

# 273. Hydration boundary

El resultado del Execution Engine será posteriormente consumido por:

```text
Result/Hydration layer
```

cuando corresponda.

---

# 274. Direct Query Builder

También puede consumir el resultado sin ORM.

---

# 275. Same engine

```text
ORM
Query Builder
Raw structured query
```

terminan utilizando el mismo Execution Plan System.

---

# 276. Raw expressions

Un raw expression que haya sobrevivido las etapas previas permanecerá bajo sus barriers/capabilities.

---

# 277. No raw reinterpretation

Execution Planner no analiza strings raw para "optimizar".

---

# 278. Extensions

`ExecutionExtensionRegistry` permitirá:

```text
custom lowering rules
custom execution units
custom resource types
custom result contracts
custom instrumentation
custom runtime requirements
```

---

# 279. Extension descriptor

```php
final readonly class ExecutionExtensionDescriptor
{
    public function __construct(
        public ExtensionId $id,
        public ExtensionVersion $version,
        public ExecutionCapabilityRequirementSet $capabilities,
        public ExecutionResourceDescriptorSet $resources,
        public ExecutionSecurityProfile $security,
    ) {}
}
```

---

# 280. Extension lifecycle

```text
discover
   ↓
validate
   ↓
resolve dependencies
   ↓
register
   ↓
freeze
```

---

# 281. Hot mutation

No se permitirá modificar el registry durante queries activas.

---

# 282. Extension cleanup

Toda extensión que adquiera recursos deberá registrar cleanup.

---

# 283. Extension cancellation

Toda extensión long-running deberá declarar comportamiento ante cancellation.

---

# 284. Extension failure

No podrá lanzar errores fuera del modelo de ejecución sin adaptación.

---

# 285. Exception model

Posibles excepciones:

```text
ExecutionPlanningException
ExecutionLoweringException
NoExecutablePlanException
ExecutionPlanValidationException
ExecutionGraphException
ExecutionStageException
ExecutionParameterLayoutException
ExecutionResourcePlanningException
ExecutionOwnershipException
ExecutionCancellationPlanningException
ExecutionCleanupPlanningException
ExecutionCapabilityException
ExecutionExtensionException
ExecutionPlanCompatibilityException
```

---

# 286. Planning error ≠ runtime error

Ejemplo:

```text
required execution capability unavailable
```

es planning/compatibility error.

---

# 287. Runtime error

Ejemplo:

```text
database disconnected while reading rows
```

pertenece al Execution Engine.

---

# 288. Testing strategy

El sistema requerirá:

```text
unit tests
lowering tests
graph tests
stage tests
parameter tests
resource tests
ownership tests
cleanup tests
cancellation tests
timeout tests
security tests
extension tests
persistent runtime tests
concurrency tests
property tests
integration tests
```

---

# 289. Lowering tests

Cada physical node deberá verificar:

```text
PhysicalNode
→
expected Execution topology
```

---

# 290. Equivalence tests

El Execution Plan deberá mantener:

```text
Semantics(ExecutionPlan)
=
Semantics(PhysicalQueryPlan)
```

---

# 291. Resource leak tests

Simular:

```text
success
failure before execution
failure during execution
cancellation
timeout
partial materialization
consumer abandonment
```

---

# 292. Cursor leak tests

Verificar que ningún camino deje cursor sin owner/cleanup.

---

# 293. Connection lease tests

Igualmente.

---

# 294. Temporary storage tests

Todo temporary storage deberá eliminarse.

---

# 295. Cancellation tests

Verificar propagación por:

```text
linear pipeline
fan-out
fan-in
database unit
framework unit
materialization
```

---

# 296. Timeout tests

Verificar deadlines anidados.

---

# 297. Security tests

Especialmente:

```text
tenant filtering
authorization filtering
database-side enforcement
sensitive projections
```

---

# 298. Persistent worker tests

Después de cada ejecución:

```text
no bindings leaked
no cursor leaked
no buffer leaked
no transaction leaked
no tenant state leaked
no cancellation state leaked
```

---

# 299. Concurrency tests

Dos `ExecutionInstance` del mismo plan deberán ser independientes.

---

# 300. Plan reuse test

```text
ExecutionPlan P
├── Instance A
└── Instance B
```

deberán aceptar bindings distintos sin interferencia.

---

# 301. Property-based testing

Generar grafos válidos y verificar invariantes de:

```text
ownership
cleanup
reachability
result production
parameter resolution
```

---

# 302. Failure injection

Introducir fallos en:

```text
resource acquisition
statement preparation
binding
execution
fetch
materialization
consumer processing
cleanup
```

para verificar lifecycle.

---

# 303. Namespace propuesto

```text
VoltStack/Quantum/Database/Query/Planner/Execution
├── Contract
│   ├── ExecutionPlanBuilder.php
│   ├── ExecutionNode.php
│   ├── ExecutionUnit.php
│   ├── ExecutionLoweringRule.php
│   └── ExecutionRuntimeAdapter.php
│
├── Plan
│   ├── ExecutionPlan.php
│   ├── ExecutionPlanId.php
│   ├── ExecutionPlanGraph.php
│   ├── ExecutionNodeId.php
│   ├── ExecutionPlanFingerprint.php
│   └── ExecutionPlanMetadata.php
│
├── Stage
│   ├── ExecutionStage.php
│   ├── ExecutionStageId.php
│   ├── ExecutionStageGraph.php
│   └── RecursiveExecutionRegion.php
│
├── Unit
│   ├── DatabaseExecutionUnit.php
│   ├── DatabaseMutationExecutionUnit.php
│   ├── FrameworkExecutionUnit.php
│   ├── StatementExecutionUnit.php
│   ├── MaterializationExecutionUnit.php
│   └── ResultForwardExecutionUnit.php
│
├── Edge
│   ├── ExecutionEdge.php
│   ├── DataExecutionEdge.php
│   ├── ControlExecutionEdge.php
│   ├── ParameterExecutionEdge.php
│   ├── CorrelationExecutionEdge.php
│   ├── CancellationExecutionEdge.php
│   └── CleanupExecutionEdge.php
│
├── Parameter
│   ├── ExecutionParameterSlot.php
│   ├── ExecutionParameterSlotId.php
│   ├── ExecutionParameterLayout.php
│   ├── CorrelationSlot.php
│   └── RuntimeValueSource.php
│
├── Resource
│   ├── ExecutionResourcePlan.php
│   ├── ResourceRequirement.php
│   ├── ResourceOwnershipGraph.php
│   ├── ResourceLifetime.php
│   ├── CursorOwnership.php
│   ├── BufferOwnership.php
│   ├── SpillContract.php
│   └── ResourceAcquisitionPlan.php
│
├── Scheduling
│   ├── ExecutionSchedulingConstraint.php
│   ├── ParallelEligibility.php
│   └── ExecutionSchedulingPlan.php
│
├── Cancellation
│   ├── ExecutionCancellationPlan.php
│   ├── CancellationPropagationGraph.php
│   └── CancellationPolicy.php
│
├── Timeout
│   ├── ExecutionTimeoutRequirement.php
│   └── ExecutionDeadline.php
│
├── Failure
│   ├── FailurePropagationGraph.php
│   └── FailurePolicy.php
│
├── Cleanup
│   ├── ExecutionCleanupPlan.php
│   ├── CleanupAction.php
│   └── CleanupDependencyGraph.php
│
├── Result
│   ├── ExecutionResultContract.php
│   ├── DatabaseResultContract.php
│   ├── ReturningExecutionContract.php
│   └── ResultBufferingMode.php
│
├── Transaction
│   └── TransactionExecutionRequirement.php
│
├── Connection
│   └── ConnectionResolutionDescriptor.php
│
├── Control
│   └── ExecutionControlBoundary.php
│
├── Security
│   ├── SecurityExecutionBoundary.php
│   └── SensitiveOutputClassification.php
│
├── Lowering
│   ├── ExecutionLoweringCoordinator.php
│   ├── ExecutionLoweringContext.php
│   ├── ExecutionLoweringResult.php
│   ├── ExecutionLoweringRegistry.php
│   └── ExecutionPlanningIdAllocator.php
│
├── Mapping
│   └── PhysicalExecutionMapping.php
│
├── Instrumentation
│   ├── ExecutionInstrumentationPlan.php
│   └── InstrumentationPoint.php
│
├── Dependency
│   └── ExecutionDependencySet.php
│
├── Extension
│   ├── ExecutionExtensionRegistry.php
│   └── ExecutionExtensionDescriptor.php
│
├── Validation
│   ├── ExecutionPlanValidator.php
│   └── ExecutionPlanCompatibilityChecker.php
│
├── Diagnostic
│   ├── ExecutionPlanPrinter.php
│   └── ExecutionPlanningTrace.php
│
└── Exception
```

---

# 304. Dependencias permitidas

```text
Execution Plan System
        │
        ├── Physical Query Plan
        ├── Platform Capabilities
        ├── Execution Capabilities
        ├── Runtime Resource Profile
        ├── Compiler Contracts
        ├── Transaction Contracts
        ├── Connection Resolution Contracts
        ├── Telemetry Contracts
        └── Extension Contracts
```

---

# 305. Dependencias prohibidas

```text
Execution Plan System
        ✗ ORM EntityManager
        ✗ UnitOfWork
        ✗ IdentityMap
        ✗ HTTP Controller
        ✗ Livewire component
        ✗ current PDO connection
        ✗ current PDOStatement
        ✗ current ResultCursor
        ✗ global tenant state
```

---

# 306. Invariantes

## DB-EPLAN-001

ExecutionPlan será distinto de PhysicalQueryPlan.

## DB-EPLAN-002

ExecutionPlan será distinto de ExecutionInstance.

## DB-EPLAN-003

ExecutionPlan será distinto de CompiledDatabaseCommand.

## DB-EPLAN-004

ExecutionPlan será distinto de SQL.

## DB-EPLAN-005

ExecutionPlan será distinto de live statement.

## DB-EPLAN-006

ExecutionPlan será distinto de query result.

## DB-EPLAN-007

ExecutionPlan no abrirá conexiones.

## DB-EPLAN-008

ExecutionPlan no ejecutará statements.

## DB-EPLAN-009

ExecutionPlan no contendrá runtime parameter values.

## DB-EPLAN-010

ExecutionPlan final será inmutable.

## DB-EPLAN-011

Runtime state pertenecerá a ExecutionInstance.

## DB-EPLAN-012

ExecutionPlanId será distinto de ExecutionInstanceId.

## DB-EPLAN-013

ParameterId será distinto de ExecutionParameterSlotId.

## DB-EPLAN-014

ExecutionParameterSlotId será distinto de SQLPlaceholder.

## DB-EPLAN-015

SQLPlaceholder será distinto de driver binding position.

## DB-EPLAN-016

ExecutionPlan será un grafo estructurado.

## DB-EPLAN-017

Execution cycles ordinarios estarán prohibidos.

## DB-EPLAN-018

Recursión será representada explícitamente.

## DB-EPLAN-019

ExecutionUnit representará trabajo ejecutable.

## DB-EPLAN-020

ExecutionNode no tendrá que equivaler 1:1 a ExecutionUnit.

## DB-EPLAN-021

Operator fusion requerirá preservación semántica.

## DB-EPLAN-022

ExecutionStage será distinto de transaction.

## DB-EPLAN-023

ExecutionStage será distinto de thread.

## DB-EPLAN-024

ExecutionStage será distinto de worker.

## DB-EPLAN-025

Data dependency será distinta de control dependency.

## DB-EPLAN-026

Correlation dependency será explícita.

## DB-EPLAN-027

Correlation no implicará automáticamente nested loop.

## DB-EPLAN-028

Materialization dependency será explícita.

## DB-EPLAN-029

Cancellation dependency será explícita.

## DB-EPLAN-030

Cleanup dependency será explícita.

## DB-EPLAN-031

DatabaseExecutionUnit será distinto de SQL string.

## DB-EPLAN-032

DatabaseExecutionUnit será compilado por SQL Compiler.

## DB-EPLAN-033

CompiledDatabaseCommand será distinto de live statement.

## DB-EPLAN-034

FrameworkExecutionUnit será explícito.

## DB-EPLAN-035

V1 favorecerá safe database delegation.

## DB-EPLAN-036

Framework execution no reemplazará innecesariamente native DB execution.

## DB-EPLAN-037

Hybrid execution será soportable.

## DB-EPLAN-038

ExecutionControlLevel será explícito.

## DB-EPLAN-039

Execution control será distinto de trust.

## DB-EPLAN-040

Execution lowering será distinto de SQL compilation.

## DB-EPLAN-041

Execution Engine no reinterpretará Physical Plan.

## DB-EPLAN-042

Compiler no reinterpretará semantic meaning.

## DB-EPLAN-043

Parameter consumption será explícito.

## DB-EPLAN-044

CorrelationSlot será distinto de user ParameterId.

## DB-EPLAN-045

Sensitive parameter classification será preservada.

## DB-EPLAN-046

Sensitive runtime values no estarán en el plan.

## DB-EPLAN-047

Resource requirement será distinto de live resource.

## DB-EPLAN-048

Todo resource runtime tendrá ownership definido.

## DB-EPLAN-049

Shared consumption no implicará shared ownership.

## DB-EPLAN-050

Cursor ownership será explícito.

## DB-EPLAN-051

Buffer ownership será explícito.

## DB-EPLAN-052

Resource lifetime será explícito.

## DB-EPLAN-053

Streaming cursor ownership podrá transferirse.

## DB-EPLAN-054

Transferred cursor no será limpiado prematuramente.

## DB-EPLAN-055

Materialization será distinta de cache.

## DB-EPLAN-056

Temporary storage será planificado explícitamente.

## DB-EPLAN-057

Spill contract será explícito.

## DB-EPLAN-058

Scheduling constraints serán runtime-agnostic.

## DB-EPLAN-059

Parallel eligibility será distinta de actual parallel execution.

## DB-EPLAN-060

FrankenPHP no aparecerá como dependencia en core execution nodes.

## DB-EPLAN-061

RoadRunner no aparecerá como dependencia en core execution nodes.

## DB-EPLAN-062

OpenSwoole no aparecerá como dependencia en core execution nodes.

## DB-EPLAN-063

Runtime integrations utilizarán adapters.

## DB-EPLAN-064

Execution ordering no dependerá del registration order.

## DB-EPLAN-065

Cancellation será first-class.

## DB-EPLAN-066

Cancellation será distinta de failure.

## DB-EPLAN-067

Timeout será distinto de cancellation aunque pueda producir cancelación operacional.

## DB-EPLAN-068

Timeout requirements serán explícitos.

## DB-EPLAN-069

Deadline composition será bounded.

## DB-EPLAN-070

Failure propagation será explícita.

## DB-EPLAN-071

Failure será distinta de cleanup.

## DB-EPLAN-072

Cleanup ocurrirá tras failure cuando existan recursos adquiridos.

## DB-EPLAN-073

Cleanup ocurrirá tras cancellation.

## DB-EPLAN-074

Cleanup ocurrirá tras timeout.

## DB-EPLAN-075

Cleanup responsibility será explícita.

## DB-EPLAN-076

Cleanup failure no ocultará automáticamente primary failure.

## DB-EPLAN-077

Partial startup tendrá partial cleanup.

## DB-EPLAN-078

Resource acquisition será distinta de planning.

## DB-EPLAN-079

Connection lease será distinto de connection object.

## DB-EPLAN-080

Execution Plan no almacenará live connections.

## DB-EPLAN-081

Execution Plan no iniciará transactions.

## DB-EPLAN-082

Transaction requirements serán explícitos.

## DB-EPLAN-083

Same-transaction requirements podrán abarcar múltiples units.

## DB-EPLAN-084

Cross-database atomicity no será asumida.

## DB-EPLAN-085

Mutation execution preservará mutation set.

## DB-EPLAN-086

Mutation execution preservará RETURNING semantics.

## DB-EPLAN-087

RETURNING emulation requerirá equivalencia demostrable.

## DB-EPLAN-088

Unsafe RETURNING emulation estará prohibida.

## DB-EPLAN-089

Todo plan tendrá result contract.

## DB-EPLAN-090

Result contract será distinto de ORM hydration.

## DB-EPLAN-091

Result ordering sólo se declarará cuando esté garantizado.

## DB-EPLAN-092

Accidental database ordering no será public contract.

## DB-EPLAN-093

Streaming y buffering serán estrategias distintas.

## DB-EPLAN-094

Backpressure será explícitamente soportable.

## DB-EPLAN-095

Backpressure no implicará buffering completo.

## DB-EPLAN-096

Async será distinto de parallelism.

## DB-EPLAN-097

Execution state pertenecerá a ExecutionInstance.

## DB-EPLAN-098

ExecutionPlanBuilder consumirá PhysicalQueryPlan.

## DB-EPLAN-099

ExecutionLoweringRule no ejecutará operaciones.

## DB-EPLAN-100

ExecutionLoweringRegistry será frozen después de bootstrap.

## DB-EPLAN-101

Lowering conflicts no serán last-one-wins.

## DB-EPLAN-102

Lowering será determinista.

## DB-EPLAN-103

One PhysicalNode podrá producir múltiples ExecutionNodes.

## DB-EPLAN-104

Multiple PhysicalNodes podrán fusionarse en un ExecutionUnit.

## DB-EPLAN-105

Runtime hash tables no existirán dentro del plan.

## DB-EPLAN-106

Runtime sort buffers no existirán dentro del plan.

## DB-EPLAN-107

Instrumentation points serán distintos del telemetry backend.

## DB-EPLAN-108

Core Execution Plan no dependerá directamente de OpenTelemetry.

## DB-EPLAN-109

Instrumentation no almacenará sensitive runtime values.

## DB-EPLAN-110

Execution dependencies serán explícitas.

## DB-EPLAN-111

Execution compatibility será validada antes de ejecución.

## DB-EPLAN-112

ExecutionPlanFingerprint excluirá runtime values por defecto.

## DB-EPLAN-113

Cached plans validarán capabilities.

## DB-EPLAN-114

Cached plans validarán extension versions.

## DB-EPLAN-115

Cached plans no contendrán mutable runtime state.

## DB-EPLAN-116

Persistent workers podrán reutilizar immutable ExecutionPlans.

## DB-EPLAN-117

Bindings no se compartirán entre ExecutionInstances.

## DB-EPLAN-118

Cursor state no se compartirá entre ExecutionInstances.

## DB-EPLAN-119

Transaction state no se compartirá entre ExecutionInstances.

## DB-EPLAN-120

Tenant runtime state no se almacenará globalmente.

## DB-EPLAN-121

Tenant bindings serán operation-scoped.

## DB-EPLAN-122

Connection resolution podrá permanecer dinámica.

## DB-EPLAN-123

Read/write intent será explícito.

## DB-EPLAN-124

Replica selection pertenecerá al runtime de routing.

## DB-EPLAN-125

Retry eligibility será distinta de retry policy.

## DB-EPLAN-126

Mutation retry no se asumirá seguro.

## DB-EPLAN-127

Idempotency será explícita cuando sea relevante.

## DB-EPLAN-128

Execution planning tendrá budget.

## DB-EPLAN-129

Extension expansion estará bounded.

## DB-EPLAN-130

Budget exhaustion no producirá plan parcial válido.

## DB-EPLAN-131

ExecutionPlanValidator será obligatorio antes de freeze.

## DB-EPLAN-132

Todo consumed parameter slot deberá existir.

## DB-EPLAN-133

Todo resource adquirible deberá tener owner.

## DB-EPLAN-134

Todo resource adquirible deberá tener cleanup path.

## DB-EPLAN-135

Todo execution root deberá producir result contract válido.

## DB-EPLAN-136

PhysicalExecutionMapping será explícito.

## DB-EPLAN-137

PhysicalExecutionMapping no será necesariamente 1:1.

## DB-EPLAN-138

Infrastructure nodes tendrán provenance explícita.

## DB-EPLAN-139

Explain no mostrará sensitive runtime values.

## DB-EPLAN-140

Explain podrá mostrar resource ownership.

## DB-EPLAN-141

Explain podrá mostrar cancellation propagation.

## DB-EPLAN-142

Explain podrá mostrar lowering decisions.

## DB-EPLAN-143

Execution IDs no dependerán de global counters.

## DB-EPLAN-144

Security enforcement boundaries serán preservadas.

## DB-EPLAN-145

Tenant isolation boundaries serán preservadas.

## DB-EPLAN-146

Authorization policy operations no serán eliminadas durante lowering.

## DB-EPLAN-147

Data minimization será preservada.

## DB-EPLAN-148

Execution Plan System será independiente del ORM.

## DB-EPLAN-149

Active Record no tendrá execution planner alternativo.

## DB-EPLAN-150

Raw expressions no serán reinterpretadas por Execution Planner.

## DB-EPLAN-151

Execution extensions deberán declarar resource lifecycle.

## DB-EPLAN-152

Execution extensions deberán declarar cancellation behavior.

## DB-EPLAN-153

Execution extension registry será immutable durante active queries.

## DB-EPLAN-154

Planning errors serán distintos de runtime execution errors.

## DB-EPLAN-155

ExecutionPlan será reusable sólo bajo compatibility validation.

## DB-EPLAN-156

`Semantics(ExecutionPlan) = Semantics(PhysicalQueryPlan)` será obligatorio.

## DB-EPLAN-157

Execution Engine ejecutará el plan; no lo rediseñará.

## DB-EPLAN-158

SQL Compiler compilará regiones delegadas; no rediseñará el plan.

## DB-EPLAN-159

Resource cleanup formará parte del contrato de ejecución.

## DB-EPLAN-160

Cancellation, failure y cleanup tendrán modelos explícitamente separados.

---

# 307. Anti-patrones

## 307.1 Guardar PDOStatement en el plan

Incorrecto:

```php
$plan->statement = $pdo->prepare($sql);
```

Mezcla planning con runtime.

---

## 307.2 Guardar bindings reales

Incorrecto:

```php
$plan->bindings = [
    'email' => 'john@example.com',
];
```

Debe guardar slots, no valores.

---

## 307.3 Abrir conexión durante lowering

Incorrecto:

```php
public function lower(...)
{
    $pdo = $connectionManager->connect();
}
```

---

## 307.4 Fusionar sin respetar seguridad

Incorrecto:

```text
Database Policy Filter
+
Framework Post Filter
→
remove database filter
```

porque "es más barato".

---

## 307.5 Cleanup implícito

Incorrecto:

```text
"PHP garbage collector eventually closes it."
```

No es un lifecycle contract aceptable.

---

## 307.6 Asumir thread por stage

Incorrecto:

```text
1 Stage = 1 Thread
```

---

## 307.7 Confundir cancellation con failure

Incorrecto:

```text
CancelledExecution
=
DatabaseException
```

---

## 307.8 Global execution state

Incorrecto:

```php
ExecutionManager::$currentBindings;
ExecutionManager::$currentCursor;
```

Especialmente peligroso con FrankenPHP.

---

## 307.9 Replanificar en Executor

Incorrecto:

```text
Executor sees HashJoin
→ decides MergeJoin instead
```

El executor ejecuta.

---

## 307.10 SQL como Execution Plan

Incorrecto:

```php
new ExecutionPlan(
    sql: 'SELECT ...'
);
```

---

# 308. Ejemplo completo: query delegada

Query:

```sql
SELECT id, email
FROM users
WHERE active = ?
ORDER BY created_at DESC
LIMIT 20;
```

Physical Plan:

```text
DatabaseDelegatedSubplan
├── Scan(users)
├── Filter(active = P1)
├── Project(id, email)
├── Sort(created_at DESC)
└── Limit(20)
```

Execution Plan:

```text
ExecutionPlan
│
├── ParameterLayout
│   └── Slot EP1
│       └── Parameter P1
│
├── Stage S1
│   └── DatabaseExecutionUnit U1
│       ├── Intent: READ
│       ├── ConnectionRole: READ
│       ├── Input: PhysicalSubplan PS1
│       ├── Parameters: [EP1]
│       └── Result: ROW_STREAM
│
├── ResourcePlan
│   ├── ConnectionLeaseRequirement
│   ├── StatementRequirement
│   └── CursorRequirement
│
├── CancellationPlan
│   └── Root → U1
│
└── CleanupPlan
    ├── CloseCursor
    ├── ReleaseStatement
    └── ReleaseConnectionLease
```

---

# 309. Runtime de ese ejemplo

Sólo durante ejecución:

```text
ExecutionInstance
│
├── EP1 = true
├── ConnectionLease #A
├── PreparedStatement #S
├── Cursor #C
└── CancellationToken #T
```

Nada de esto pertenece al `ExecutionPlan`.

---

# 310. Ejemplo híbrido

Supongamos una futura query federada:

```text
customers from database A
+
orders from database B
```

Physical Plan:

```text
FrameworkHashJoin
├── DatabaseDelegatedSubplan A
└── DatabaseDelegatedSubplan B
```

Execution Plan:

```text
Stage S1
└── DatabaseExecutionUnit A

Stage S2
└── DatabaseExecutionUnit B

Stage S3
├── HashBuildUnit
│   └── consumes S1
│
└── HashProbeUnit
    └── consumes S2

Stage S4
└── ResultForward
```

Resource Plan:

```text
ConnectionLease A
ConnectionLease B
HashTable H1
ResultStream R1
```

---

# 311. Failure en ejecución híbrida

Si `DatabaseExecutionUnit B` falla:

```text
B FAILED
   │
   ├── cancel HashProbe
   ├── cancel A if still active and no longer needed
   ├── destroy HashTable
   ├── close Cursor A
   ├── close Cursor B
   ├── release Connection A
   └── release Connection B
```

Esta topología debe poder derivarse del plan.

---

# 312. Ejemplo de streaming ownership

```text
DatabaseExecutionUnit
       │
       ▼
DatabaseCursor
       │
       ▼
StreamingResult
       │
       ▼
Application Consumer
```

Ownership:

```text
before transfer:
ExecutionInstance owns cursor

after transfer:
StreamingResult owns cursor
```

Al cerrar el resultado:

```text
StreamingResult
→
close cursor
→
release connection lease
```

---

# 313. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request A
│   └── ExecutionInstance A
│
├── Request B
│   └── ExecutionInstance B
│
└── Request C
    └── ExecutionInstance C
```

pueden compartir:

```text
ExecutionPlan
ExecutionLoweringRegistry
ExecutionExtensionRegistry
immutable descriptors
```

pero nunca:

```text
bindings
cursors
connection leases
transactions
buffers
deadlines
cancellation tokens
tenant runtime state
```

---

# 314. Modelo de lifecycle

```text
ExecutionPlan
     │
     ▼
Compatibility Check
     │
     ▼
ExecutionInstance Created
     │
     ▼
Runtime Bindings Attached
     │
     ▼
Resources Acquired
     │
     ▼
Commands Compiled/Prepared
     │
     ▼
Execution Started
     │
     ▼
Rows / Mutation Results
     │
     ▼
Result Finalization
     │
     ▼
Cleanup
     │
     ▼
ExecutionInstance Closed
```

---

# 315. Lifecycle ante error

```text
ExecutionInstance
     │
     ▼
Partial Resource Acquisition
     │
     ▼
Execution Failure
     │
     ▼
Failure Propagation
     │
     ▼
Cancellation of Dependents
     │
     ▼
Cleanup Acquired Resources
     │
     ▼
Failure Result
```

---

# 316. Lifecycle ante cancellation

```text
Running
   │
   ▼
Cancellation Requested
   │
   ▼
Propagate Cancellation
   │
   ▼
Stop New Work
   │
   ▼
Cancel Active Work
   │
   ▼
Drain/Abort as Contract Requires
   │
   ▼
Cleanup
   │
   ▼
CANCELLED
```

---

# 317. Lifecycle ante timeout

```text
Deadline Exceeded
      │
      ▼
TIMED_OUT
      │
      ▼
Cancellation Propagation
      │
      ▼
Cleanup
```

El estado final debe conservar que la causa fue:

```text
TIMEOUT
```

y no una cancelación genérica.

---

# 318. Arquitectura consolidada

```text
                   PhysicalQueryPlan
                           │
                           ▼
                ExecutionPlanBuilder
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
      Lowering         Parameters        Resources
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                 Execution Topology
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
        Stages        Dependencies       Ownership
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
          Cancellation / Cleanup Planning
                           │
                           ▼
                  Instrumentation
                           │
                           ▼
                 Contract Validation
                           │
                           ▼
                     ExecutionPlan
                           │
                           ▼
              ExecutionPlanCompatibility
                           │
                           ▼
                  ExecutionInstance
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
     DB Commands     Framework Ops      Resources
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                    Execution Engine
```

---

# 319. Fórmula de validez

```text
ValidExecutionPlan(E)
=
PhysicalSemanticsPreserved(E)
∧
AllInputsResolvable(E)
∧
AllParametersMapped(E)
∧
AllResourcesOwned(E)
∧
AllResourcesCleanable(E)
∧
AllDependenciesValid(E)
∧
CancellationDefined(E)
∧
FailurePropagationDefined(E)
∧
ResultContractSatisfied(E)
∧
SecurityBoundariesPreserved(E)
∧
CapabilitiesSatisfied(E)
```

---

# 320. Fórmula de seguridad de lifecycle

```text
SafeExecutionLifecycle
=
ExplicitAcquisition
+
ExplicitOwnership
+
ExplicitLifetime
+
ExplicitCancellation
+
ExplicitFailurePropagation
+
ExplicitCleanup
```

---

# 321. Fórmula para persistent runtimes

```text
PersistentSafeExecution
=
ImmutableExecutionPlan
+
OperationScopedExecutionInstance
+
OperationScopedBindings
+
OperationScopedResources
+
OperationScopedCancellation
+
DeterministicCleanup
```

---

# 322. Fórmula de separación

```text
Physical Plan
=
strategy

Execution Plan
=
orchestration

Compiled Command
=
platform representation

Execution Instance
=
runtime state

Execution Engine
=
runtime behavior
```

---

# 323. Invariante maestro

```text
Semantics(ExecutionPlan)
=
Semantics(PhysicalQueryPlan)
=
Semantics(LogicalQueryPlan)
```

---

# 324. Pipeline completo del Query Engine

```text
Developer Intent
      │
      ▼
Query Builder
      │
      ▼
Query Model / AST
      │
      ▼
Normalization
      │
      ▼
Validation
      │
      ▼
Semantic Analysis
      │
      ▼
SemanticQueryArtifact
      │
      ▼
Query Optimizer
      │
      ▼
OptimizedQueryArtifact
      │
      ▼
Logical Query Planner
      │
      ▼
LogicalQueryPlan
      │
      ▼
Physical Query Planner
      │
      ▼
PhysicalQueryPlan
      │
      ▼
Execution Plan System
      │
      ▼
ExecutionPlan
      │
      ├──────────────────────┐
      ▼                      ▼
SQL Compilation       Framework Operators
      │                      │
      ▼                      │
Compiled Commands            │
      └──────────┬───────────┘
                 ▼
          Execution Engine
                 │
                 ▼
       Connection / Driver
                 │
                 ▼
             Database
```

---

# 325. Distinción definitiva de artifacts

```text
QueryModel
=
structured developer query

ValidatedQueryArtifact
=
structurally valid query

SemanticQueryArtifact
=
authoritative query meaning

OptimizedQueryArtifact
=
preferred equivalent logical form

LogicalQueryPlan
=
required relational operations

PhysicalQueryPlan
=
selected physical strategies

ExecutionPlan
=
executable orchestration topology

CompiledDatabaseCommand
=
target-platform command representation

ExecutionInstance
=
operation-specific runtime state

Result
=
observable query output
```

---

# 326. Responsabilidades por capa

| Capa | Pregunta |
|---|---|
| Semantic Engine | ¿Qué significa la query? |
| Optimizer | ¿Qué forma equivalente conviene? |
| Logical Planner | ¿Qué operaciones lógicas se requieren? |
| Physical Planner | ¿Qué estrategias físicas las implementan? |
| Execution Planner | ¿Cómo se organizan como trabajo ejecutable? |
| SQL Compiler | ¿Cómo se representan para esta plataforma? |
| Execution Engine | ¿Cómo se ejecuta el trabajo? |
| Result System | ¿Cómo se expone el resultado? |
| Hydration | ¿Cómo se transforma a estructuras/entidades? |

---

# 327. Decisión arquitectónica principal

VoltStack adoptará:

```text
Plan Artifacts
≠
Runtime State
```

en todas las capas.

Por tanto:

```text
LogicalQueryPlan       immutable
PhysicalQueryPlan      immutable
ExecutionPlan          immutable
CompiledQuery          immutable/cacheable where possible

ExecutionInstance      mutable operation-scoped
ResultCursor           runtime-scoped
ConnectionLease        runtime-scoped
TransactionContext     runtime-scoped
```

---

# 328. Beneficio para FrankenPHP

Esta separación evita el problema clásico de persistent workers:

```text
Request A runtime state
        ↓
accidentally survives
        ↓
Request B
```

La arquitectura correcta será:

```text
Shared Worker State
├── immutable registries
├── immutable descriptors
├── immutable cached plans
└── immutable compiled artifacts

Request State
├── ExecutionInstance
├── bindings
├── connection leases
├── transaction context
├── cursors
├── buffers
├── cancellation
└── cleanup
```

---

# 329. Block 5 completado

Con este documento queda arquitectónicamente cerrado:

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

El flujo completo del bloque queda:

```text
SemanticQueryArtifact
        │
        ▼
Query Optimizer
        │
        ▼
OptimizedQueryArtifact
        │
        ▼
Logical Planner
        │
        ▼
LogicalQueryPlan
        │
        ▼
Physical Planner
        │
        ▼
PhysicalQueryPlan
        │
        ▼
Execution Planner
        │
        ▼
ExecutionPlan
```

---

# 330. Siguiente bloque

A partir del siguiente documento comienza:

```text
Block 6 — SQL Compiler
```

con:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
67_DATABASE_SQL_COMPILER_PIPELINE.md
68_DATABASE_SQL_GENERATION_SYSTEM.md
69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 331. Siguiente documento

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
```

deberá establecer la arquitectura mediante la cual VoltStack transforma regiones delegables del:

```text
ExecutionPlan
```

y sus referencias físicas estructuradas en:

```text
CompiledDatabaseCommand
```

para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin permitir que el Compiler vuelva a realizar:

```text
semantic analysis
query optimization
logical planning
physical planning
execution planning
```

Su invariante principal deberá ser:

```text
Compiler
=
representation transformation

Compiler
≠
semantic engine
≠
optimizer
≠
planner
≠
executor
```

---

# 332. Fórmula final

```text
Execution Plan System
=
Physical Strategy Lowering
+
Execution Topology
+
Stage Construction
+
Parameter Layout
+
Resource Requirements
+
Resource Ownership
+
Scheduling Dependencies
+
Cancellation Propagation
+
Failure Propagation
+
Cleanup Planning
+
Result Contract
+
Security Boundaries
+
Instrumentation Points
+
Runtime Compatibility
```

manteniendo siempre:

```text
No Runtime State In The Plan
+
No Hidden Resource Ownership
+
No Semantic Reinterpretation
+
No Physical Replanning
+
No SQL Generation
+
No Execution
```

---

# 333. Principio final

> **El `ExecutionPlan` de VoltStack no representa una query ejecutándose; representa todo lo que el runtime necesita saber para poder ejecutarla correctamente, sin tener que redescubrir su semántica, replantear su estrategia física ni improvisar el ciclo de vida de sus recursos.**

En forma compacta:

```text
PhysicalQueryPlan
        │
        │  "what physical strategy?"
        ▼
ExecutionPlan
        │
        │  "what executable work?"
        ▼
ExecutionInstance
        │
        │  "with which runtime state?"
        ▼
Execution Engine
        │
        │  "run it"
        ▼
Result
```