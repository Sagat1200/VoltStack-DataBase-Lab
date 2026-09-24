# 128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md

# VoltStack Quantum Database
## Database Persistence Planner System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 128 — Database Persistence Planner System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Persistence Planner System` define la arquitectura encargada de transformar un `PersistenceOperationGraph` válido en un `PersistencePlan` determinista, ordenado, verificable y posteriormente ejecutable.

El sistema recibe:

```text
PersistenceEngineResult
├── PersistenceGeneration
├── EntityChangeGraph
├── PersistenceOperationGraph
├── PersistenceRequirements
├── Diagnostics
└── PersistenceFingerprint
```

y produce:

```text
PersistencePlan
├── Ordered Phases
├── Persistence Steps
├── Dependency Barriers
├── Generated Value Barriers
├── Lifecycle Barriers
├── Reconciliation Boundaries
├── Transaction Requirements
├── Batch Candidates
├── Capability Strategies
└── Execution Metadata
```

Principio central:

> **El Persistence Planner decide cómo transformar un grafo válido de operaciones ORM en un plan de persistencia ordenado y ejecutable; planifica dependencias y estrategias, pero no ejecuta operaciones, no genera SQL y no altera el UnitOfWork.**

---

# 2. Posición arquitectónica

```text
Entity Mutations
      ↓
Change Tracking
      ↓
UnitOfWork
      ↓
PreparedUnitOfWork
      ↓
Persistence Engine
      ↓
PersistenceOperationGraph
      ↓
┌───────────────────────────────┐
│      Persistence Planner      │
└───────────────────────────────┘
      ↓
PersistencePlan
      ↓
Persistence Operation Translators
      ↓
Query Model / AST
      ↓
Semantic Query Engine
      ↓
Query Optimizer
      ↓
Query Planner
      ↓
SQL Compiler
      ↓
Execution Engine
      ↓
Connection
      ↓
Driver
      ↓
Database
```

---

# 3. Frontera fundamental

```text
Persistence Engine
≠
Persistence Planner
≠
Query Planner
≠
SQL Compiler
≠
Execution Engine
```

Los nombres `Persistence Planner` y `Query Planner` no representan el mismo componente.

---

# 4. Persistence Planner

Responde:

> ¿En qué orden semántico y bajo qué estrategia ORM deben realizarse las operaciones de persistencia?

Ejemplo:

```text
Insert Customer
      ↓
obtain customer.id
      ↓
Insert Order
      ↓
Insert OrderItems
```

---

# 5. Query Planner

Responde:

> ¿Cómo debe ejecutarse físicamente un Query Model concreto?

Ejemplo:

```text
Logical Query Plan
      ↓
Physical Query Plan
```

---

# 6. SQL Compiler

Responde:

> ¿Cómo se representa el Query Model mediante SQL para la plataforma actual?

---

# 7. Execution Engine

Responde:

> ¿Cómo se ejecutan los statements compilados y cómo se capturan sus resultados?

---

# 8. Regla de separación

Nunca:

```text
Persistence Planner
→ SQL string
```

Nunca:

```text
Persistence Planner
→ PDO
```

Nunca:

```text
Persistence Planner
→ execute()
```

---

# 9. Entrada principal

```php
final readonly class PersistencePlanningRequest
{
    public function __construct(
        public PersistenceEngineResult $persistence,
        public PersistencePlanningContext $context,
        public PersistencePlanningOptions $options,
    ) {}
}
```

---

# 10. Contrato principal

```php
interface PersistencePlanner
{
    public function plan(
        PersistencePlanningRequest $request,
    ): PersistencePlan;
}
```

La API deberá permanecer pequeña.

---

# 11. PersistencePlanningContext

```php
final readonly class PersistencePlanningContext
{
    public function __construct(
        public DatabaseContext $database,
        public PlatformCapabilitySnapshot $capabilities,
        public TransactionContextView $transaction,
        public PersistencePlanningPolicySet $policies,
        public PersistenceResourceBudget $resources,
    ) {}
}
```

---

# 12. Planning context ≠ runtime executor

El contexto no expondrá:

```text
PDO
Statement
DriverConnection
QueryExecutor
commit()
rollback()
```

---

# 13. Input stability

El Planner consumirá un:

```text
sealed PersistenceEngineResult
```

No un UnitOfWork mutable.

---

# 14. Regla

```text
Planner
→ reads PersistenceEngineResult

Planner
↛ mutates UnitOfWork
```

---

# 15. PersistencePlan

Se propone:

```php
final readonly class PersistencePlan
{
    public function __construct(
        public PersistencePlanId $id,
        public PersistenceGeneration $generation,
        public PersistenceFingerprint $sourceFingerprint,
        public PersistencePlanFingerprint $fingerprint,
        public PersistencePlanPhaseCollection $phases,
        public PersistencePlanGraph $graph,
        public PersistencePlanRequirementSet $requirements,
        public PersistencePlanDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 16. PersistencePlan ≠ PersistenceOperationGraph

El grafo de operaciones responde:

```text
WHAT must persist?
```

El plan responde:

```text
HOW should persistence proceed semantically?
```

---

# 17. PersistencePlan ≠ ExecutionPlan

El `ExecutionPlan` del Query Engine continúa siendo independiente.

---

# 18. PersistencePlan ≠ Transaction

Un plan puede ejecutarse:

```text
inside existing transaction
inside framework-managed transaction
without transaction where explicitly permitted
```

---

# 19. PersistencePlanId

```php
final readonly class PersistencePlanId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 20. Plan ID ≠ fingerprint

```text
PersistencePlanId
≠
PersistencePlanFingerprint
```

El ID identifica una instancia lógica del plan.

El fingerprint describe su contenido semántico.

---

# 21. Arquitectura interna

```text
PersistenceEngineResult
        │
        ▼
Input Validation
        │
        ▼
Dependency Analysis
        │
        ▼
Graph Classification
        │
        ▼
Cycle Analysis
        │
        ▼
Strategy Resolution
        │
        ▼
Constraint Analysis
        │
        ▼
Generated Value Analysis
        │
        ▼
Operation Ordering
        │
        ▼
Barrier Placement
        │
        ▼
Batch Candidate Analysis
        │
        ▼
Phase Construction
        │
        ▼
Plan Validation
        │
        ▼
Fingerprint
        │
        ▼
Seal
        │
        ▼
PersistencePlan
```

---

# 22. Planning pipeline

```php
enum PersistencePlanningPhase
{
    case VALIDATE_INPUT;
    case ANALYZE_DEPENDENCIES;
    case ANALYZE_CYCLES;
    case RESOLVE_STRATEGIES;
    case ANALYZE_CONSTRAINTS;
    case ANALYZE_GENERATED_VALUES;
    case ORDER_OPERATIONS;
    case PLACE_BARRIERS;
    case ANALYZE_BATCHING;
    case BUILD_PHASES;
    case VALIDATE_PLAN;
    case FINGERPRINT;
    case SEAL;
}
```

---

# 23. Dependency analysis

El Planner consume:

```text
PersistenceOperationGraph
```

con edges como:

```text
REQUIRES_BEFORE
REQUIRES_AFTER
REQUIRES_GENERATED_VALUE
REQUIRES_EXISTENCE
REQUIRES_DISSOCIATION
REQUIRES_COLLECTION_CHANGE
REQUIRES_VERSION_CHECK
CONFLICTS_WITH
```

---

# 24. Dependency ≠ order

Ejemplo:

```text
Order requires Customer identity
```

es una dependencia.

El Planner puede resolverla mediante diferentes estrategias.

---

# 25. Strategy example

### Estrategia A

```text
Insert Customer
↓
obtain generated ID
↓
Insert Order
```

### Estrategia B

Si IDs son application-generated:

```text
Customer ID already known

Insert Customer
Insert Order
```

pueden ser candidatos a otro orden/batch, sujeto a constraints.

---

# 26. Dependency resolver

```php
interface PersistenceDependencyResolver
{
    public function resolve(
        PersistenceOperationGraph $graph,
        PersistencePlanningContext $context,
    ): ResolvedPersistenceDependencyGraph;
}
```

---

# 27. Resolved dependency graph

No destruye la procedencia original.

```text
Original Operation Graph
        +
Resolved Strategy Metadata
        =
ResolvedPersistenceDependencyGraph
```

---

# 28. Topological ordering

Para grafos acíclicos:

```text
DAG
→ topological ordering
```

es la base natural.

---

# 29. Topological order ≠ unique order

Un grafo puede admitir:

```text
A → C
B → C
```

y tanto:

```text
A, B, C
```

como:

```text
B, A, C
```

son válidos.

VoltStack necesita orden determinista.

---

# 30. Deterministic tie-breaking

Cuando existan múltiples órdenes válidos:

```text
semantic dependency
→ operation category
→ entity mapping order
→ stable operation identity
```

podrán utilizarse como tie-breakers.

---

# 31. No memory-address ordering

Prohibido utilizar:

```text
spl_object_id()
memory address
hash iteration accident
```

como criterio final reproducible.

---

# 32. Stable ordering

El mismo input deberá producir:

```text
same semantic plan order
```

independientemente del orden accidental de estructuras internas.

---

# 33. Cycles

Los ciclos son first-class.

Ejemplo:

```text
A
↑ ↓
B
```

---

# 34. Cycle analysis

```php
interface PersistenceCycleAnalyzer
{
    public function analyze(
        ResolvedPersistenceDependencyGraph $graph,
        PersistencePlanningContext $context,
    ): PersistenceCycleAnalysis;
}
```

---

# 35. Strongly connected components

Una implementación puede utilizar:

```text
Tarjan
```

o:

```text
Kosaraju
```

para identificar `StronglyConnectedComponents`.

---

# 36. Algorithm ≠ public contract

VoltStack no acoplará la arquitectura a Tarjan.

El contrato será:

```text
SCC detection capability
```

---

# 37. Cycle classifications

```php
enum PersistenceCycleResolutionStatus
{
    case NOT_CYCLIC;
    case RESOLVABLE;
    case RESOLVABLE_WITH_LIMITATIONS;
    case REQUIRES_PLATFORM_CAPABILITY;
    case UNRESOLVABLE;
    case UNKNOWN;
}
```

---

# 38. UNKNOWN ≠ RESOLVABLE

Nunca asumir:

```text
UNKNOWN → safe
```

---

# 39. Cycle strategies

Posibles:

```php
enum PersistenceCycleStrategy
{
    case NATURAL_IDENTITY_ORDERING;
    case PREALLOCATED_IDENTIFIER;
    case NULLABLE_FOREIGN_KEY_TWO_PHASE;
    case DEFERRED_CONSTRAINT;
    case POST_INSERT_ASSOCIATION;
    case EXPLICIT_APPLICATION_BREAK;
    case EXTENSION_DEFINED;
}
```

---

# 40. Strategy ≠ capability

Ejemplo:

```text
DEFERRED_CONSTRAINT
```

es estrategia.

La plataforma deberá declarar si puede soportarla.

---

# 41. Nullable two-phase strategy

Ejemplo:

```text
A.b_id NOT required initially
B.a_id requires A.id
```

Plan:

```text
Insert A without b_id
↓
Insert B(a_id)
↓
Update A(b_id)
```

---

# 42. No hidden semantic degradation

El Planner no podrá establecer `NULL` temporalmente si:

```text
mapping says non-null
schema says non-null
domain policy forbids it
```

salvo estrategia explícitamente representable.

---

# 43. Deferred constraints

Si la plataforma soporta constraints deferibles:

```text
Begin Transaction
↓
Defer Constraint
↓
Insert A
↓
Insert B
↓
Constraint Validation
```

podría ser viable.

Pero el Planner no ejecuta estos pasos.

Solo los representa.

---

# 44. Generated value analysis

Una dependencia frecuente:

```text
Insert Parent
      ↓
Generated ID
      ↓
Insert Child
```

---

# 45. Generated value barrier

```php
final readonly class GeneratedValueBarrier implements PersistencePlanBarrier
{
    public function __construct(
        public PersistenceOperationId $producer,
        public PersistentPropertyId $property,
        public PersistenceOperationIdCollection $consumers,
    ) {}
}
```

---

# 46. Barrier semantics

```text
Producer executed
AND
required value reconciled/resolved
BEFORE
consumer becomes executable
```

---

# 47. GeneratedValueBarrier ≠ transaction barrier

Son conceptos distintos.

---

# 48. GeneratedValueBarrier ≠ query result cursor

El barrier es semántica del plan ORM.

---

# 49. Preallocated identifiers

Si:

```text
UUID assigned before persistence
```

no existe necesariamente generated-value barrier.

---

# 50. Assigned ID ≠ persisted row

Aunque el ID exista:

```text
Entity identity known
```

no significa:

```text
database row exists
```

Por tanto puede seguir existiendo:

```text
REQUIRES_EXISTENCE
```

---

# 51. Existence dependency

Ejemplo:

```text
Order.customer_id → Customer.id
```

Si FK inmediata exige existencia:

```text
Customer INSERT
→ Order INSERT
```

aunque ambos IDs ya sean conocidos.

---

# 52. Constraint-aware planning

El Planner deberá considerar información estructural relevante de:

```text
ORM Metadata
Schema Capability Metadata
Platform Capabilities
Persistence Requirements
```

---

# 53. Mapping ≠ live schema

El Planner no consultará la DB para descubrir constraints durante cada flush.

---

# 54. Schema metadata source

Puede consumir:

```text
compiled schema compatibility metadata
```

si está disponible.

---

# 55. Missing schema knowledge

Si una estrategia depende de un constraint desconocido:

```text
UNKNOWN
```

debe conservarse.

---

# 56. Constraint confidence

```php
enum PersistenceConstraintKnowledge
{
    case KNOWN;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 57. Not observed ≠ absent

Regla heredada del Schema System:

```text
NotObserved
≠
Absent
```

---

# 58. Insert ordering

Un orden típico:

```text
principal entities
↓
dependent entities
↓
join rows
```

pero no será una regla universal hardcoded.

---

# 59. Delete ordering

Frecuentemente:

```text
join rows
↓
dependent rows
↓
principal rows
```

pero dependerá de:

```text
ORM cascades
DB cascades
FK semantics
orphan rules
platform capabilities
```

---

# 60. Update ordering

Updates pueden depender de:

```text
generated values
version tokens
relationship reassignment
unique constraints
temporary value strategy
```

---

# 61. Unique constraint ordering

Ejemplo:

```text
User A username = "bob"
User B username = "alice"

swap:
A → alice
B → bob
```

Ejecutar:

```text
A first
```

puede violar UNIQUE.

---

# 62. Persistence conflict graph

El Planner podrá construir:

```text
PersistenceConflictGraph
```

para detectar operaciones que individualmente son válidas pero cuya secuencia intermedia viola restricciones.

---

# 63. Conflict edge

```php
final readonly class PersistenceConflictEdge
{
    public function __construct(
        public PersistenceOperationId $left,
        public PersistenceOperationId $right,
        public PersistenceConflictType $type,
    ) {}
}
```

---

# 64. Conflict types

```php
enum PersistenceConflictType
{
    case UNIQUE_VALUE_COLLISION;
    case FOREIGN_KEY_DEPENDENCY;
    case VERSION_DEPENDENCY;
    case GENERATED_VALUE_DEPENDENCY;
    case TEMPORARY_STATE_CONFLICT;
    case TARGET_CONFLICT;
    case EXTENSION_DEFINED;
}
```

---

# 65. Unique swap strategy

Posibles estrategias:

```text
deferred unique constraint
temporary sentinel value
multi-step update
reject
```

---

# 66. No unsafe sentinel by default

VoltStack no inventará:

```text
__TEMP_123__
```

como valor temporal sin estrategia explícita y segura.

---

# 67. Operation phases

El plan estará dividido en fases.

---

# 68. Proposed phases

```php
enum PersistencePlanPhaseType
{
    case PREPARATION;
    case PRE_INSERT;
    case INSERT;
    case GENERATED_VALUE_RESOLUTION;
    case UPDATE;
    case RELATIONSHIP_SYNCHRONIZATION;
    case COLLECTION_SYNCHRONIZATION;
    case DELETE;
    case POST_PERSISTENCE;
    case RECONCILIATION;
    case EXTENSION_DEFINED;
}
```

---

# 69. Phase ≠ mandatory SQL grouping

Una fase es semántica.

No significa necesariamente:

```text
one SQL statement
one DB roundtrip
one transaction
```

---

# 70. PersistencePlanPhase

```php
final readonly class PersistencePlanPhase
{
    public function __construct(
        public PersistencePlanPhaseId $id,
        public PersistencePlanPhaseType $type,
        public PersistencePlanStepCollection $steps,
        public PersistencePlanRequirementSet $requirements,
    ) {}
}
```

---

# 71. Plan step

```php
interface PersistencePlanStep
{
    public function id(): PersistencePlanStepId;

    public function type(): PersistencePlanStepType;

    public function dependencies(): PersistencePlanStepDependencySet;
}
```

---

# 72. Step types

```php
enum PersistencePlanStepType
{
    case OPERATION;
    case BARRIER;
    case GENERATED_VALUE_RESOLUTION;
    case LIFECYCLE;
    case RECONCILIATION;
    case TRANSACTION_REQUIREMENT;
    case CONSTRAINT_STRATEGY;
    case BATCH_GROUP;
    case EXTENSION_DEFINED;
}
```

---

# 73. Operation step

```php
final readonly class PersistenceOperationStep implements PersistencePlanStep
{
    public function __construct(
        public PersistencePlanStepId $id,
        public PersistenceOperation $operation,
        public PersistencePlanStepDependencySet $dependencies,
        public PersistenceExecutionRequirementSet $requirements,
    ) {}
}
```

---

# 74. Barrier architecture

Barriers representan puntos que no pueden atravesarse hasta satisfacer una condición.

---

# 75. Barrier types

```php
enum PersistenceBarrierType
{
    case GENERATED_VALUE;
    case LIFECYCLE;
    case RECONCILIATION;
    case CONSTRAINT;
    case TRANSACTION;
    case BATCH;
    case EXTENSION_DEFINED;
}
```

---

# 76. Barrier ≠ mutex

No es necesariamente un lock de concurrencia.

---

# 77. Reconciliation barrier

Ejemplo:

```text
Insert Customer
↓
DB returns ID
↓
Reconcile customer.id
↓
Child operations may continue
```

---

# 78. Why reconciliation matters

No basta con:

```text
statement succeeded
```

El valor generado debe incorporarse al estado ORM antes de consumidores dependientes.

---

# 79. Lifecycle barrier

Ejemplo:

```text
prePersist
↓
Insert operation
↓
postPersist
```

El Planner preservará el orden definido por el Lifecycle System.

---

# 80. Lifecycle mutation stabilization

La estabilización de `pre*` que puede alterar ChangeSets debe ocurrir antes de sellar el plan final.

---

# 81. Plan sealing rule

```text
Mutable lifecycle stabilization
↓
Prepared UnitOfWork stable
↓
Persistence Engine
↓
Persistence Planner
↓
SEALED PLAN
```

---

# 82. No plan mutation from postPersist

Un `postPersist` no deberá modificar retroactivamente el plan en ejecución.

Los cambios serán:

```text
future dirty state
```

---

# 83. Transaction requirements

El Planner agregará requirements.

```php
final readonly class PersistenceTransactionRequirements
{
    public function __construct(
        public TransactionRequirement $requirement,
        public bool $requiresSavepointSupport,
        public bool $requiresConstraintDeferral,
        public bool $requiresAtomicGeneratedValueChain,
    ) {}
}
```

---

# 84. Planner ≠ transaction owner

Nunca:

```php
$transaction->begin();
```

---

# 85. Transaction strategy

El plan podrá declarar:

```text
REQUIRES_TRANSACTION
```

y la capa coordinadora decidirá cómo satisfacerlo.

---

# 86. Existing transaction

Si existe una transacción:

```text
Plan Requirements
+
TransactionContext
→ compatibility validation
```

---

# 87. Isolation requirements

Algunas operaciones futuras podrán declarar:

```text
minimum isolation requirement
```

pero el Planner no modificará aislamiento arbitrariamente.

---

# 88. Savepoints

El Planner puede declarar:

```text
savepoint capability required
```

para una estrategia.

No crea el savepoint.

---

# 89. Flush ≠ commit

Se mantiene:

```text
flush()
≠
commit()
```

---

# 90. Plan completion ≠ transaction durability

Incluso si todas las operaciones del plan fueron ejecutadas:

```text
PersistencePlanExecuted
```

no implica:

```text
TransactionCommitted
```

---

# 91. Batch analysis

El Planner identificará oportunidades de batching.

---

# 92. Batch candidate

```php
final readonly class PersistenceBatchCandidate
{
    public function __construct(
        public PersistenceBatchKey $key,
        public PersistenceOperationIdCollection $operations,
        public PersistenceBatchRequirementSet $requirements,
    ) {}
}
```

---

# 93. Batch key

Puede considerar:

```text
operation type
entity mapping
target
column set
generated values
locking requirements
result shape
lifecycle boundaries
dependency barriers
```

---

# 94. Batch candidate ≠ batch execution

El plan puede indicar:

```text
these operations MAY batch
```

La capa 133 decidirá detalles.

---

# 95. Generated ID can break batching

Ejemplo:

```text
Parent1 INSERT → ID needed immediately
Parent2 INSERT → ID needed immediately
```

puede limitar batching dependiendo de platform capabilities.

---

# 96. Lifecycle can break batching

Si se requiere:

```text
postPersist(entity1)
before some dependent operation
```

puede existir un barrier.

---

# 97. Version result can break batching

Si cada UPDATE requiere:

```text
affected rows = 1
new version token
```

la estrategia de batch debe conservar correlación individual.

---

# 98. Batch semantics before performance

Nunca:

```text
performance optimization
>
correct persistence semantics
```

---

# 99. Query Model generation boundary

Después del plan:

```text
PersistencePlanStep
      ↓
Persistence Query Translator
      ↓
Query Model
```

---

# 100. Translator contract

```php
interface PersistenceQueryTranslator
{
    public function translate(
        PersistenceOperationStep $step,
        PersistenceQueryTranslationContext $context,
    ): PersistenceQueryBatch;
}
```

---

# 101. Planner does not call compiler

El Planner podrá seleccionar una estrategia abstracta, pero no deberá producir:

```text
INSERT INTO ...
UPDATE ...
DELETE ...
```

---

# 102. Query translation strategy

Ejemplo:

```text
InsertEntityOperation
+
GeneratedValueRequirement
+
PlatformCapabilitySnapshot
→
InsertQueryModel strategy
```

---

# 103. Returning capability

El Planner puede decidir:

```text
RETURNING strategy is available
```

mediante capability.

Pero el dialect/compiler decidirá su representación SQL.

---

# 104. Capability-based planning

Preferir:

```php
if ($capabilities->supportsInsertReturning()) {
    // select semantic strategy
}
```

No:

```php
if ($database === 'postgresql') {
}
```

---

# 105. Capability status

Una capability puede ser:

```text
SUPPORTED
WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 106. UNKNOWN handling

Si la estrategia depende de una capability `UNKNOWN`:

```text
do not assume support
```

---

# 107. Emulation

`REQUIRES_EMULATION` puede producir un plan de múltiples pasos.

---

# 108. Example generated-value emulation

```text
Insert
↓
Generated value retrieval
↓
Reconciliation
```

en vez de un único statement con returning.

---

# 109. Emulation ≠ hidden SQL

El plan representa:

```text
semantic steps
```

El Query Engine sigue generando queries.

---

# 110. Target validation

Todas las operaciones del plan deberán pertenecer a targets compatibles.

---

# 111. Cross-target

Default V1:

```text
atomic PersistencePlan
+
multiple incompatible targets
→ reject
```

---

# 112. Split plans

Una futura API podría producir:

```text
CompositePersistencePlan
├── Target A plan
└── Target B plan
```

pero:

```text
Composite
≠
Atomic
```

---

# 113. Distributed transaction

No deberá inferirse automáticamente.

---

# 114. Plan validation

Antes de seal:

```text
all operation nodes scheduled
all mandatory dependencies satisfied
all barriers satisfiable
all cycles resolved
all requirements representable
all targets compatible
all generated values have producers
all consumers have valid barriers
all lifecycle boundaries valid
all reconciliation paths valid
```

---

# 115. Missing operation detection

Si:

```text
OperationGraph has 20 nodes
Plan schedules 19
```

el plan es inválido salvo que el nodo haya sido eliminado mediante normalización explícita antes del Planner.

---

# 116. Duplicate scheduling

Una operación tampoco podrá aparecer dos veces accidentalmente.

---

# 117. Reachability validation

Todos los consumers deberán poder alcanzar sus prerequisites.

---

# 118. Barrier satisfiability

Ejemplo inválido:

```text
Operation B waits for generated User.id
```

pero ningún step produce `User.id`.

Debe fallar durante planning.

---

# 119. Plan fingerprint

```php
final readonly class PersistencePlanFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 120. Fingerprint input

Debe incluir semánticamente:

```text
source PersistenceFingerprint
operation ordering
selected strategies
barriers
phases
capability snapshot fingerprint
relevant policy fingerprint
target
extension contributions
```

---

# 121. Fingerprint should not include

Valores accidentales:

```text
memory addresses
random UUID generated solely for plan instance
timestamps
trace IDs
```

---

# 122. Plan determinism

Formalmente:

```text
Plan(
    SamePersistenceGraph,
    SameCapabilities,
    SamePolicies,
    SameMetadata,
    SameExtensions
)
=
SameSemanticPlan
```

---

# 123. Stale plan detection

Antes de ejecución deberá poder comprobarse:

```text
Plan.sourceFingerprint
==
current PersistenceEngineResult.fingerprint
```

---

# 124. Generation validation

También:

```text
Plan.persistenceGeneration
==
EntityManager expected generation
```

---

# 125. Stale plan

Si no:

```text
StalePersistencePlanException
```

---

# 126. No auto-patching

Un plan stale no deberá ser reparado parcialmente durante ejecución.

Preferir:

```text
abort
→ reprepare
→ rederive
→ replan
```

---

# 127. Plan immutability

Después de `seal()`:

```text
PersistencePlan
```

será immutable.

---

# 128. Sealed plan

No se permitirá:

```php
$plan->steps[] = $newStep;
```

---

# 129. Why immutable

Evita:

```text
execution/planning race
lifecycle mutation
extension mutation
debugger mutation
cross-worker contamination
```

---

# 130. Planning session

Estado mutable temporal:

```php
final class PersistencePlanningSession
{
    // temporary planning state
}
```

---

# 131. Session lifetime

```text
one planning operation
```

---

# 132. Planning session may contain

```text
dependency indexes
SCC analysis
candidate orders
selected strategies
barrier registry
batch candidates
diagnostics
resource counters
```

---

# 133. Session discarded after seal

No debe sobrevivir al plan.

---

# 134. Extension architecture

```php
interface PersistencePlanningExtension
{
    public function contribute(
        PersistencePlanningExtensionContext $context,
    ): PersistencePlanningContribution;
}
```

---

# 135. Extension phases

```php
enum PersistencePlanningExtensionPhase
{
    case DEPENDENCY_ANALYSIS;
    case CYCLE_ANALYSIS;
    case STRATEGY_RESOLUTION;
    case ORDERING;
    case BARRIER_PLACEMENT;
    case BATCH_ANALYSIS;
    case PLAN_VALIDATION;
}
```

---

# 136. Extension contribution

Podrá:

```text
add dependency
add requirement
provide cycle strategy
add barrier
restrict batching
add validation
add diagnostics
```

---

# 137. Extension cannot

```text
execute query
mutate entity
mutate UnitOfWork
commit transaction
generate SQL
silently remove mandatory operation
```

---

# 138. Extension conflicts

Dos extensiones pueden seleccionar estrategias incompatibles.

Debe producir:

```text
PersistencePlanningExtensionConflictException
```

---

# 139. No last-wins

Regla general VoltStack:

```text
incompatible semantic configuration
→ explicit error
```

---

# 140. Extension registry

En producción:

```text
compiled
ordered
validated
frozen
```

---

# 141. Explainability

Cada decisión importante deberá conservar razón.

---

# 142. PlanningDecision

```php
final readonly class PersistencePlanningDecision
{
    public function __construct(
        public PersistencePlanningDecisionId $id,
        public PersistencePlanningDecisionType $type,
        public string $reasonCode,
        public PersistencePlanningEvidence $evidence,
    ) {}
}
```

---

# 143. Example explanation

```text
Operation:
Insert Order#temp-4

Scheduled after:
Insert Customer#temp-1

Reason:
Order.customer requires Customer persistent identity.

Strategy:
GeneratedValueBarrier

Capability:
Insert generated identifier retrieval supported.

Barrier:
customer.id
```

---

# 144. Plan explain API

```php
interface PersistencePlanExplainer
{
    public function explain(
        PersistencePlan $plan,
    ): PersistencePlanExplanation;
}
```

---

# 145. Explain ≠ execute

Debe funcionar offline sobre el plan.

---

# 146. Diagnostics

Ejemplo:

```text
Persistence Plan
────────────────────────────────────

Generation: 42
Operations: 17
Steps: 24
Phases: 7

INSERT                6
UPDATE                4
RELATIONSHIP          2
COLLECTION            3
DELETE                2

Generated barriers    3
Lifecycle barriers    4
Reconciliation        3
Batch candidates      2

Cycles:
  detected            1
  resolved            1
  unresolved          0

Transaction:
  required

Target:
  database:default

Fingerprint:
  9c7f...
```

---

# 147. Plan visualization

Developer tooling podrá mostrar:

```text
[Insert Customer]
        │
        ▼
[Resolve customer.id]
        │
        ├──────────────┐
        ▼              ▼
[Insert Order A] [Insert Order B]
        │
        ▼
[Insert LineItems]
        │
        ▼
[Reconcile]
```

---

# 148. Resource governance

Planning también consume recursos.

---

# 149. Resource budget

```php
final readonly class PersistencePlanningResourceBudget
{
    public function __construct(
        public int $maxOperations,
        public int $maxDependencies,
        public int $maxCycles,
        public int $maxPlanSteps,
        public int $maxBarriers,
        public int $maxBatchCandidates,
        public int $maxPlanningPasses,
        public int $maxMemoryBytes,
    ) {}
}
```

---

# 150. Limit exceeded

Nunca truncar plan.

Debe fallar explícitamente.

---

# 151. Planning passes

Algunas estrategias pueden requerir múltiples pasadas:

```text
dependency analysis
→ cycle strategy
→ graph rewrite
→ dependency revalidation
```

---

# 152. Bounded stabilization

```text
while plan changes:
    revalidate

if passes > max:
    fail
```

---

# 153. Infinite planner rewrite

Debe producir:

```text
PersistencePlanningStabilizationException
```

---

# 154. Graph rewrite

El Planner sí puede transformar el **planning graph**, no el UnitOfWork.

---

# 155. Example graph rewrite

Una operación conceptual:

```text
Associate A ↔ B
```

puede resolverse en:

```text
Insert A
Insert B
Update A relationship
```

según estrategia.

---

# 156. Provenance preservation

Cada step derivado debe apuntar a:

```text
original PersistenceOperation
planning strategy
planning decision
```

---

# 157. Step provenance

```php
final readonly class PersistencePlanStepProvenance
{
    public function __construct(
        public PersistenceOperationIdCollection $operations,
        public PersistencePlanningDecisionIdCollection $decisions,
    ) {}
}
```

---

# 158. Provenance chain

Debe poder trazarse:

```text
Entity Property Mutation
      ↓
ChangeSet
      ↓
PersistenceOperation
      ↓
PersistencePlanStep
      ↓
Query Model
      ↓
Compiled Query
      ↓
Execution
```

---

# 159. Telemetry correlation

Esa cadena permitirá correlacionar:

```text
ORM
→ Query
→ SQL
→ DB timing
```

sin acoplar las capas.

---

# 160. Runtime isolation

El Planner será compatible con persistent runtimes.

---

# 161. Shared immutable

Puede compartirse:

```text
PersistencePlanner service
compiled planning rules
frozen extension registry
immutable policies
```

---

# 162. Scoped mutable

Nunca compartir:

```text
PlanningSession
current graph
current plan builder
current transaction
current tenant
current EntityManager
```

---

# 163. FrankenPHP

```text
Worker
├── Request A → PlanningSession A
└── Request B → PlanningSession B
```

---

# 164. RoadRunner

Mismo principio.

---

# 165. OpenSwoole

Mismo principio por logical coroutine scope.

---

# 166. Thread/coroutine safety

Un `PersistencePlanningSession` no será compartible concurrentemente.

---

# 167. Planner service

Puede ser singleton si:

```text
stateless
+
immutable dependencies
```

---

# 168. No static current plan

Prohibido:

```php
PersistencePlanner::$currentPlan
```

---

# 169. No static current tenant

También prohibido.

---

# 170. Cancellation

Planning de grafos enormes podría admitir:

```text
CancellationToken
```

---

# 171. Cancellation safety

Como todavía no existe DB execution:

```text
planning cancellation
→ no DB side effects
```

---

# 172. Timeout

Podrá existir:

```text
planning budget / timeout
```

separado del query timeout.

---

# 173. Planning timeout ≠ query timeout

Conceptos distintos.

---

# 174. Error hierarchy

```text
DatabaseOrmException
└── PersistenceException
    └── PersistencePlanningException
        ├── InvalidPersistencePlanningInputException
        ├── StalePersistenceGraphException
        ├── StalePersistencePlanException
        ├── PersistenceDependencyException
        ├── UnsatisfiedPersistenceDependencyException
        ├── PersistenceOrderingException
        ├── PersistenceCyclePlanningException
        ├── UnresolvablePersistenceCycleException
        ├── UnknownPersistenceCycleStrategyException
        ├── PersistenceConstraintPlanningException
        ├── PersistenceConstraintKnowledgeException
        ├── GeneratedValuePlanningException
        ├── MissingGeneratedValueProducerException
        ├── PersistenceBarrierException
        ├── UnsatisfiedPersistenceBarrierException
        ├── PersistenceTransactionRequirementException
        ├── PersistenceBatchPlanningException
        ├── PersistenceTargetPlanningException
        ├── CrossTargetPersistencePlanException
        ├── PersistenceCapabilityPlanningException
        ├── PersistencePlanValidationException
        ├── DuplicatePersistenceOperationSchedulingException
        ├── MissingPersistenceOperationSchedulingException
        ├── PersistencePlanFingerprintException
        ├── PersistencePlanningExtensionException
        ├── PersistencePlanningExtensionConflictException
        ├── PersistencePlanningResourceLimitException
        ├── PersistencePlanningStabilizationException
        ├── PersistencePlanningCancellationException
        ├── PersistencePlanningTimeoutException
        ├── PersistencePlanningRuntimeIsolationException
        └── PersistencePlanningInvariantException
```

---

# 175. Directory structure

```text
src/Quantum/Database/ORM/Persistence/Planner/
│
├── Contract/
│   ├── PersistencePlanner.php
│   ├── PersistenceDependencyResolver.php
│   ├── PersistenceCycleAnalyzer.php
│   ├── PersistencePlanValidator.php
│   └── PersistencePlanExplainer.php
│
├── Planning/
│   ├── DefaultPersistencePlanner.php
│   ├── PersistencePlanningRequest.php
│   ├── PersistencePlanningContext.php
│   ├── PersistencePlanningOptions.php
│   ├── PersistencePlanningSession.php
│   └── PersistencePlanningPhase.php
│
├── Plan/
│   ├── PersistencePlan.php
│   ├── PersistencePlanId.php
│   ├── PersistencePlanFingerprint.php
│   ├── PersistencePlanBuilder.php
│   └── PersistencePlanRequirementSet.php
│
├── Phase/
│   ├── PersistencePlanPhase.php
│   ├── PersistencePlanPhaseId.php
│   ├── PersistencePlanPhaseType.php
│   └── PersistencePlanPhaseCollection.php
│
├── Step/
│   ├── PersistencePlanStep.php
│   ├── PersistencePlanStepId.php
│   ├── PersistencePlanStepType.php
│   ├── PersistenceOperationStep.php
│   ├── GeneratedValueResolutionStep.php
│   ├── LifecycleStep.php
│   └── ReconciliationStep.php
│
├── Graph/
│   ├── PersistencePlanGraph.php
│   ├── ResolvedPersistenceDependencyGraph.php
│   ├── PersistenceConflictGraph.php
│   ├── PersistenceConflictEdge.php
│   └── PersistenceConflictType.php
│
├── Dependency/
│   ├── DefaultPersistenceDependencyResolver.php
│   ├── PersistenceDependencyAnalysis.php
│   └── PersistencePlanStepDependencySet.php
│
├── Ordering/
│   ├── PersistenceOperationOrderer.php
│   ├── DeterministicTopologicalOrderer.php
│   └── PersistenceOrderingPolicy.php
│
├── Cycle/
│   ├── PersistenceCycleAnalyzer.php
│   ├── PersistenceCycle.php
│   ├── PersistenceCycleAnalysis.php
│   ├── PersistenceCycleStrategy.php
│   ├── PersistenceCycleStrategyResolver.php
│   └── PersistenceCycleResolutionStatus.php
│
├── Barrier/
│   ├── PersistencePlanBarrier.php
│   ├── PersistenceBarrierType.php
│   ├── GeneratedValueBarrier.php
│   ├── LifecycleBarrier.php
│   ├── ReconciliationBarrier.php
│   └── ConstraintBarrier.php
│
├── Generated/
│   ├── GeneratedValueDependencyAnalyzer.php
│   ├── GeneratedValueResolutionStep.php
│   └── GeneratedValuePlanningStrategy.php
│
├── Constraint/
│   ├── PersistenceConstraintAnalyzer.php
│   ├── PersistenceConstraintKnowledge.php
│   └── PersistenceConstraintStrategy.php
│
├── Transaction/
│   ├── PersistenceTransactionRequirements.php
│   └── PersistenceTransactionRequirementResolver.php
│
├── Batch/
│   ├── PersistenceBatchCandidate.php
│   ├── PersistenceBatchKey.php
│   ├── PersistenceBatchAnalyzer.php
│   └── PersistenceBatchRequirementSet.php
│
├── Target/
│   ├── PersistenceTargetValidator.php
│   └── CrossTargetPersistencePolicy.php
│
├── Decision/
│   ├── PersistencePlanningDecision.php
│   ├── PersistencePlanningDecisionId.php
│   ├── PersistencePlanningDecisionType.php
│   └── PersistencePlanningEvidence.php
│
├── Provenance/
│   └── PersistencePlanStepProvenance.php
│
├── Validation/
│   ├── DefaultPersistencePlanValidator.php
│   ├── DependencyPlanValidator.php
│   ├── BarrierPlanValidator.php
│   ├── OperationCoverageValidator.php
│   └── PlanFingerprintValidator.php
│
├── Extension/
│   ├── PersistencePlanningExtension.php
│   ├── PersistencePlanningExtensionRegistry.php
│   ├── PersistencePlanningExtensionPhase.php
│   ├── PersistencePlanningContribution.php
│   └── PersistencePlanningExtensionContext.php
│
├── Resource/
│   ├── PersistencePlanningResourceBudget.php
│   └── PersistencePlanningResourceGuard.php
│
├── Telemetry/
│   ├── PersistencePlannerTelemetry.php
│   ├── PersistencePlanProfiler.php
│   └── PersistencePlanDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 176. Testing strategy

Debe cubrir como mínimo:

```text
dependency ordering
deterministic ordering
cycle detection
cycle resolution
generated IDs
constraints
relationships
deletes
batch candidates
transaction requirements
capability strategies
stale plans
extensions
resource governance
runtime isolation
```

---

# 177. Test — simple insert

```text
Insert User
```

produce un plan válido de una operación.

---

# 178. Test — independent inserts

```text
Insert A
Insert B
```

deberán utilizar tie-break determinista.

---

# 179. Test — parent/child

```text
Parent INSERT
→ Child INSERT
```

cuando FK requiere parent existence.

---

# 180. Test — generated parent ID

Debe introducir:

```text
GeneratedValueBarrier
```

---

# 181. Test — application-generated IDs

No debe crear generated-value barrier innecesario.

---

# 182. Test — known ID but existence dependency

Debe mantener ordering cuando FK lo requiere.

---

# 183. Test — simple cycle

Detectar SCC.

---

# 184. Test — resolvable nullable cycle

Producir estrategia two-phase.

---

# 185. Test — unresolvable cycle

Fallar antes de DB execution.

---

# 186. Test — unknown capability

No asumir constraint deferral.

---

# 187. Test — deferred constraint capability

Seleccionar estrategia cuando policy y schema lo permitan.

---

# 188. Test — insert order deterministic

Mismo graph → mismo order.

---

# 189. Test — delete order

Dependents antes de principal cuando sea requerido.

---

# 190. Test — database cascade

No generar operaciones redundantes si el contrato ORM/schema explícitamente delega la acción.

---

# 191. Test — ORM cascade

No confundir con DB cascade.

---

# 192. Test — many-to-many

Join operation correctamente posicionada.

---

# 193. Test — remove join before entity delete

Cuando FK lo requiera.

---

# 194. Test — optimistic locking

Update step conserva version requirement.

---

# 195. Test — unique value swap

Detecta conflicto intermedio.

---

# 196. Test — no unsafe sentinel

No inventa valor temporal.

---

# 197. Test — lifecycle barrier

Orden correcto.

---

# 198. Test — postPersist

No lo trata como commit.

---

# 199. Test — reconciliation barrier

Consumer no procede antes de generated-value reconciliation.

---

# 200. Test — transaction requirement

Plan declara requirement sin abrir transacción.

---

# 201. Test — existing transaction

Valida compatibilidad.

---

# 202. Test — batch candidate

Operaciones compatibles agrupables.

---

# 203. Test — generated result breaks batch

No agrupar incorrectamente.

---

# 204. Test — lifecycle breaks batch

Respeta barrier.

---

# 205. Test — target mismatch

Reject.

---

# 206. Test — no distributed atomicity assumption

Cross-target no se presenta como atomic.

---

# 207. Test — missing operation

Plan validator falla.

---

# 208. Test — duplicate operation

Plan validator falla.

---

# 209. Test — missing generated producer

Plan validator falla.

---

# 210. Test — stale fingerprint

Execution debe rechazar plan.

---

# 211. Test — stale generation

Reject.

---

# 212. Test — extension contribution

Se integra determinísticamente.

---

# 213. Test — extension conflict

No last-wins.

---

# 214. Test — planning pass limit

Fail.

---

# 215. Test — memory budget

Fail sin plan truncado.

---

# 216. Test — cancellation

No side effects.

---

# 217. Test — no DB connection

Planner funciona offline.

---

# 218. Test — no SQL compiler

Planner funciona sin compiler.

---

# 219. Test — no driver

Planner funciona sin driver.

---

# 220. Test — persistent runtime

Request A/B no comparten session.

---

# 221. Test — no current tenant static

Scope isolation.

---

# 222. Anti-pattern: Planner generates SQL

Prohibido.

---

# 223. Anti-pattern: Planner executes operations

Prohibido.

---

# 224. Anti-pattern: Planner mutates UnitOfWork

Prohibido.

---

# 225. Anti-pattern: Planner modifies entity

Prohibido.

---

# 226. Anti-pattern: Planner opens transaction

Prohibido.

---

# 227. Anti-pattern: graph array sorting only

Un simple:

```php
usort($operations);
```

no sustituye dependency planning.

---

# 228. Anti-pattern: hardcoded INSERT → UPDATE → DELETE

No existe un orden universal suficiente.

---

# 229. Anti-pattern: all inserts before all updates

Puede ser incorrecto por cycles/unique constraints/relationships.

---

# 230. Anti-pattern: all deletes last

También puede ser incorrecto.

---

# 231. Anti-pattern: generated IDs ignored

Rompería dependencias.

---

# 232. Anti-pattern: ID known means row exists

Incorrecto.

---

# 233. Anti-pattern: cycle means impossible

Incorrecto.

---

# 234. Anti-pattern: cycle always resolvable

También incorrecto.

---

# 235. Anti-pattern: UNKNOWN capability treated as support

Prohibido.

---

# 236. Anti-pattern: vendor-specific planning

Evitar:

```php
if ($driver === 'mysql')
```

como base arquitectónica.

---

# 237. Anti-pattern: schema not observed means constraint absent

Incorrecto.

---

# 238. Anti-pattern: temporary NULL without contract

Prohibido.

---

# 239. Anti-pattern: temporary sentinel without contract

Prohibido.

---

# 240. Anti-pattern: batching overrides lifecycle

Incorrecto.

---

# 241. Anti-pattern: batching overrides optimistic locking

Incorrecto.

---

# 242. Anti-pattern: flush means committed

Incorrecto.

---

# 243. Anti-pattern: plan executed means committed

Incorrecto.

---

# 244. Anti-pattern: postPersist means committed

Incorrecto.

---

# 245. Anti-pattern: planner owns retry

Incorrecto.

---

# 246. Anti-pattern: stale plan patched in place

Prohibido.

---

# 247. Anti-pattern: random ordering

Produce comportamiento no reproducible.

---

# 248. Anti-pattern: memory address tie-break

Prohibido.

---

# 249. Anti-pattern: extension last-wins

Prohibido.

---

# 250. Anti-pattern: static current plan

Incompatible con persistent runtime.

---

# 251. Architectural invariants

## DB-ORM-PERSISTENCE-PLANNER-001
Persistence Planner consumirá `PersistenceEngineResult` sellado.

## DB-ORM-PERSISTENCE-PLANNER-002
Persistence Planner será distinto de Persistence Engine.

## DB-ORM-PERSISTENCE-PLANNER-003
Persistence Planner será distinto de Query Planner.

## DB-ORM-PERSISTENCE-PLANNER-004
Persistence Planner será distinto de SQL Compiler.

## DB-ORM-PERSISTENCE-PLANNER-005
Persistence Planner será distinto de Execution Engine.

## DB-ORM-PERSISTENCE-PLANNER-006
Persistence Planner será distinto de Transaction Manager.

## DB-ORM-PERSISTENCE-PLANNER-007
Persistence Planner no generará SQL.

## DB-ORM-PERSISTENCE-PLANNER-008
Persistence Planner no ejecutará queries.

## DB-ORM-PERSISTENCE-PLANNER-009
Persistence Planner no utilizará PDO.

## DB-ORM-PERSISTENCE-PLANNER-010
Persistence Planner no alterará UnitOfWork.

## DB-ORM-PERSISTENCE-PLANNER-011
Persistence Planner no alterará entidades.

## DB-ORM-PERSISTENCE-PLANNER-012
Persistence Planner no alterará snapshots.

## DB-ORM-PERSISTENCE-PLANNER-013
Persistence Planner no abrirá transacciones.

## DB-ORM-PERSISTENCE-PLANNER-014
PersistencePlan será distinto de PersistenceOperationGraph.

## DB-ORM-PERSISTENCE-PLANNER-015
PersistencePlan será distinto de Query ExecutionPlan.

## DB-ORM-PERSISTENCE-PLANNER-016
PersistencePlan será distinto de transaction.

## DB-ORM-PERSISTENCE-PLANNER-017
PersistencePlan será immutable después de seal.

## DB-ORM-PERSISTENCE-PLANNER-018
PersistencePlanId será distinto de fingerprint.

## DB-ORM-PERSISTENCE-PLANNER-019
Source persistence fingerprint será preservado.

## DB-ORM-PERSISTENCE-PLANNER-020
Persistence generation será preservada.

## DB-ORM-PERSISTENCE-PLANNER-021
Dependencies serán analizadas antes del ordering final.

## DB-ORM-PERSISTENCE-PLANNER-022
Dependency será distinta de execution order.

## DB-ORM-PERSISTENCE-PLANNER-023
Topological ordering será usado únicamente cuando semánticamente aplicable.

## DB-ORM-PERSISTENCE-PLANNER-024
Topological order será determinista.

## DB-ORM-PERSISTENCE-PLANNER-025
Tie-breaking no dependerá de memory addresses.

## DB-ORM-PERSISTENCE-PLANNER-026
Tie-breaking no dependerá de iteration accident.

## DB-ORM-PERSISTENCE-PLANNER-027
Cycles serán first-class.

## DB-ORM-PERSISTENCE-PLANNER-028
Cycle no significará automáticamente error.

## DB-ORM-PERSISTENCE-PLANNER-029
Cycle no se considerará automáticamente resolvable.

## DB-ORM-PERSISTENCE-PLANNER-030
Cycle resolution strategy será explícita.

## DB-ORM-PERSISTENCE-PLANNER-031
UNKNOWN cycle resolution no será tratado como seguro.

## DB-ORM-PERSISTENCE-PLANNER-032
Nullable two-phase strategy requerirá semántica compatible.

## DB-ORM-PERSISTENCE-PLANNER-033
Constraint deferral requerirá capability.

## DB-ORM-PERSISTENCE-PLANNER-034
Planner no degradará constraints silenciosamente.

## DB-ORM-PERSISTENCE-PLANNER-035
Generated-value dependencies serán first-class.

## DB-ORM-PERSISTENCE-PLANNER-036
GeneratedValueBarrier será distinto de transaction barrier.

## DB-ORM-PERSISTENCE-PLANNER-037
GeneratedValueBarrier será distinto de DB result cursor.

## DB-ORM-PERSISTENCE-PLANNER-038
Consumer no precederá required generated-value producer.

## DB-ORM-PERSISTENCE-PLANNER-039
Generated value deberá reconciliarse cuando el consumer requiera el valor en entidad/contexto.

## DB-ORM-PERSISTENCE-PLANNER-040
Assigned identifier será distinto de persisted existence.

## DB-ORM-PERSISTENCE-PLANNER-041
Known identifier no eliminará existence dependencies automáticamente.

## DB-ORM-PERSISTENCE-PLANNER-042
Constraint-aware planning utilizará evidencia estructurada.

## DB-ORM-PERSISTENCE-PLANNER-043
Planner no realizará hidden schema introspection.

## DB-ORM-PERSISTENCE-PLANNER-044
NotObserved será distinto de Absent.

## DB-ORM-PERSISTENCE-PLANNER-045
UNKNOWN constraint knowledge será first-class.

## DB-ORM-PERSISTENCE-PLANNER-046
Insert ordering no será hardcoded universalmente.

## DB-ORM-PERSISTENCE-PLANNER-047
Update ordering no será hardcoded universalmente.

## DB-ORM-PERSISTENCE-PLANNER-048
Delete ordering no será hardcoded universalmente.

## DB-ORM-PERSISTENCE-PLANNER-049
Relationship dependencies participarán en ordering.

## DB-ORM-PERSISTENCE-PLANNER-050
Collection operations participarán en ordering.

## DB-ORM-PERSISTENCE-PLANNER-051
Unique conflicts podrán influir en planning.

## DB-ORM-PERSISTENCE-PLANNER-052
Temporary sentinel values no serán inventados automáticamente.

## DB-ORM-PERSISTENCE-PLANNER-053
PersistencePlan estará compuesto por fases explícitas.

## DB-ORM-PERSISTENCE-PLANNER-054
Plan phase será distinta de SQL batch.

## DB-ORM-PERSISTENCE-PLANNER-055
Plan phase será distinta de transaction.

## DB-ORM-PERSISTENCE-PLANNER-056
PersistencePlanStep será tipado.

## DB-ORM-PERSISTENCE-PLANNER-057
Barriers serán first-class.

## DB-ORM-PERSISTENCE-PLANNER-058
Barrier será distinto de mutex.

## DB-ORM-PERSISTENCE-PLANNER-059
Lifecycle barriers preservarán lifecycle ordering.

## DB-ORM-PERSISTENCE-PLANNER-060
Reconciliation barriers preservarán state dependencies.

## DB-ORM-PERSISTENCE-PLANNER-061
Mutable pre-lifecycle stabilization ocurrirá antes del plan sellado.

## DB-ORM-PERSISTENCE-PLANNER-062
Post lifecycle no reescribirá retroactivamente el plan actual.

## DB-ORM-PERSISTENCE-PLANNER-063
Transaction requirements serán declarativas.

## DB-ORM-PERSISTENCE-PLANNER-064
Planner no satisfará transaction requirements directamente.

## DB-ORM-PERSISTENCE-PLANNER-065
Existing TransactionContext será validado.

## DB-ORM-PERSISTENCE-PLANNER-066
Flush será distinto de commit.

## DB-ORM-PERSISTENCE-PLANNER-067
Plan execution será distinto de transaction commit.

## DB-ORM-PERSISTENCE-PLANNER-068
Savepoint requirement será distinto de savepoint execution.

## DB-ORM-PERSISTENCE-PLANNER-069
Batch candidates serán first-class.

## DB-ORM-PERSISTENCE-PLANNER-070
Batch candidate será distinto de physical batch.

## DB-ORM-PERSISTENCE-PLANNER-071
Batching no violará dependency barriers.

## DB-ORM-PERSISTENCE-PLANNER-072
Batching no violará lifecycle semantics.

## DB-ORM-PERSISTENCE-PLANNER-073
Batching no violará optimistic locking.

## DB-ORM-PERSISTENCE-PLANNER-074
Batching no perderá result correlation.

## DB-ORM-PERSISTENCE-PLANNER-075
Correctness tendrá prioridad sobre batching.

## DB-ORM-PERSISTENCE-PLANNER-076
Persistence operation será traducida a Query Model después del planning.

## DB-ORM-PERSISTENCE-PLANNER-077
Planner no almacenará SQL en plan steps.

## DB-ORM-PERSISTENCE-PLANNER-078
Query Engine conservará autoridad sobre Query Model semantics.

## DB-ORM-PERSISTENCE-PLANNER-079
SQL Compiler conservará autoridad sobre dialect representation.

## DB-ORM-PERSISTENCE-PLANNER-080
Capabilities determinarán estrategias, no vendor conditionals.

## DB-ORM-PERSISTENCE-PLANNER-081
Version será distinta de capability.

## DB-ORM-PERSISTENCE-PLANNER-082
MariaDB será first-class.

## DB-ORM-PERSISTENCE-PLANNER-083
UNKNOWN capability no será asumida como supported.

## DB-ORM-PERSISTENCE-PLANNER-084
REQUIRES_EMULATION podrá generar múltiples semantic steps.

## DB-ORM-PERSISTENCE-PLANNER-085
Emulation no significará hidden SQL.

## DB-ORM-PERSISTENCE-PLANNER-086
Targets serán validados.

## DB-ORM-PERSISTENCE-PLANNER-087
Cross-target atomicity no será inferida.

## DB-ORM-PERSISTENCE-PLANNER-088
Distributed transaction no será creada implícitamente.

## DB-ORM-PERSISTENCE-PLANNER-089
Todas las mandatory operations deberán estar cubiertas por el plan.

## DB-ORM-PERSISTENCE-PLANNER-090
Ninguna operación se programará dos veces accidentalmente.

## DB-ORM-PERSISTENCE-PLANNER-091
Mandatory dependencies deberán ser satisfacibles.

## DB-ORM-PERSISTENCE-PLANNER-092
Generated-value consumers deberán tener producer válido.

## DB-ORM-PERSISTENCE-PLANNER-093
Mandatory barriers deberán ser satisfacibles.

## DB-ORM-PERSISTENCE-PLANNER-094
Plan validation ocurrirá antes de seal.

## DB-ORM-PERSISTENCE-PLANNER-095
Plan fingerprint será determinista.

## DB-ORM-PERSISTENCE-PLANNER-096
Fingerprint no dependerá de timestamps.

## DB-ORM-PERSISTENCE-PLANNER-097
Fingerprint no dependerá de trace IDs.

## DB-ORM-PERSISTENCE-PLANNER-098
Fingerprint no dependerá de memory addresses.

## DB-ORM-PERSISTENCE-PLANNER-099
Stale plan será detectable.

## DB-ORM-PERSISTENCE-PLANNER-100
Stale generation será detectable.

## DB-ORM-PERSISTENCE-PLANNER-101
Stale plan no será patched durante execution.

## DB-ORM-PERSISTENCE-PLANNER-102
Stale plan requerirá reprepare/rederive/replan.

## DB-ORM-PERSISTENCE-PLANNER-103
Planning mutable state será session-scoped.

## DB-ORM-PERSISTENCE-PLANNER-104
PlanningSession no sobrevivirá al planning operation.

## DB-ORM-PERSISTENCE-PLANNER-105
PlanningSession no será process-global.

## DB-ORM-PERSISTENCE-PLANNER-106
Planner service podrá compartirse únicamente si es stateless/immutable.

## DB-ORM-PERSISTENCE-PLANNER-107
Extensions contribuirán de forma estructurada.

## DB-ORM-PERSISTENCE-PLANNER-108
Extensions no ejecutarán queries.

## DB-ORM-PERSISTENCE-PLANNER-109
Extensions no alterarán UnitOfWork.

## DB-ORM-PERSISTENCE-PLANNER-110
Extensions no alterarán entidades.

## DB-ORM-PERSISTENCE-PLANNER-111
Extensions no harán commit/rollback.

## DB-ORM-PERSISTENCE-PLANNER-112
Extension ordering será determinista.

## DB-ORM-PERSISTENCE-PLANNER-113
Extension conflicts serán errores explícitos.

## DB-ORM-PERSISTENCE-PLANNER-114
Extension registry estará frozen en production runtime.

## DB-ORM-PERSISTENCE-PLANNER-115
Planning decisions serán explicables.

## DB-ORM-PERSISTENCE-PLANNER-116
Plan steps conservarán provenance.

## DB-ORM-PERSISTENCE-PLANNER-117
Provenance conectará operation con planning decisions.

## DB-ORM-PERSISTENCE-PLANNER-118
Planning provenance será distinta de business audit.

## DB-ORM-PERSISTENCE-PLANNER-119
Telemetry no cambiará plan semantics.

## DB-ORM-PERSISTENCE-PLANNER-120
Plan explainability no requerirá DB execution.

## DB-ORM-PERSISTENCE-PLANNER-121
Resource budgets serán explícitos.

## DB-ORM-PERSISTENCE-PLANNER-122
Resource limit no truncará el plan.

## DB-ORM-PERSISTENCE-PLANNER-123
Planning stabilization será bounded.

## DB-ORM-PERSISTENCE-PLANNER-124
Infinite rewrite será error.

## DB-ORM-PERSISTENCE-PLANNER-125
Planner podrá reescribir planning graph.

## DB-ORM-PERSISTENCE-PLANNER-126
Planner no podrá reescribir UnitOfWork.

## DB-ORM-PERSISTENCE-PLANNER-127
Derived steps conservarán source provenance.

## DB-ORM-PERSISTENCE-PLANNER-128
ORM→Query→SQL correlation será posible sin acoplar capas.

## DB-ORM-PERSISTENCE-PLANNER-129
FrankenPHP requests no compartirán PlanningSession.

## DB-ORM-PERSISTENCE-PLANNER-130
RoadRunner requests no compartirán PlanningSession.

## DB-ORM-PERSISTENCE-PLANNER-131
OpenSwoole logical scopes no compartirán PlanningSession.

## DB-ORM-PERSISTENCE-PLANNER-132
No existirá static current plan.

## DB-ORM-PERSISTENCE-PLANNER-133
No existirá static current tenant.

## DB-ORM-PERSISTENCE-PLANNER-134
Planning cancellation no producirá DB side effects.

## DB-ORM-PERSISTENCE-PLANNER-135
Planning timeout será distinto de query timeout.

## DB-ORM-PERSISTENCE-PLANNER-136
Planning failure ocurrirá antes de persistence execution.

## DB-ORM-PERSISTENCE-PLANNER-137
Planning failure no implicará DB mutation.

## DB-ORM-PERSISTENCE-PLANNER-138
Plan deberá ser testeable sin DB connection.

## DB-ORM-PERSISTENCE-PLANNER-139
Plan deberá ser testeable sin SQL Compiler.

## DB-ORM-PERSISTENCE-PLANNER-140
Plan deberá ser testeable sin Driver.

## DB-ORM-PERSISTENCE-PLANNER-141
Deterministic plan será requisito arquitectónico.

## DB-ORM-PERSISTENCE-PLANNER-142
Semantic dependency tendrá prioridad sobre optimization.

## DB-ORM-PERSISTENCE-PLANNER-143
Constraint safety tendrá prioridad sobre batching.

## DB-ORM-PERSISTENCE-PLANNER-144
Lifecycle semantics tendrán prioridad sobre batching.

## DB-ORM-PERSISTENCE-PLANNER-145
Outcome reconciliation requirements tendrán prioridad sobre batching.

## DB-ORM-PERSISTENCE-PLANNER-146
Transaction durability no será inferida desde plan completion.

## DB-ORM-PERSISTENCE-PLANNER-147
Generated identity no será inferida antes del producer result.

## DB-ORM-PERSISTENCE-PLANNER-148
Known identity no será equivalente a known database existence.

## DB-ORM-PERSISTENCE-PLANNER-149
Constraint knowledge incompleto preservará incertidumbre.

## DB-ORM-PERSISTENCE-PLANNER-150
Planner no realizará hidden I/O para resolver incertidumbre.

## DB-ORM-PERSISTENCE-PLANNER-151
Planning strategies serán tipadas.

## DB-ORM-PERSISTENCE-PLANNER-152
Planning strategies serán inspeccionables.

## DB-ORM-PERSISTENCE-PLANNER-153
Planning strategies serán validables.

## DB-ORM-PERSISTENCE-PLANNER-154
Planning strategies podrán depender de capabilities.

## DB-ORM-PERSISTENCE-PLANNER-155
Planning strategies no dependerán de vendor string checks como diseño base.

## DB-ORM-PERSISTENCE-PLANNER-156
Plan phases preservarán semantic boundaries.

## DB-ORM-PERSISTENCE-PLANNER-157
Plan barriers preservarán execution prerequisites.

## DB-ORM-PERSISTENCE-PLANNER-158
Plan graph preservará dependencies aun cuando exista una ordered view.

## DB-ORM-PERSISTENCE-PLANNER-159
Ordered view no reemplazará el graph.

## DB-ORM-PERSISTENCE-PLANNER-160
Persistence Planner convertirá un grafo ORM válido en un plan determinista, seguro, explicable y ejecutable sin ejecutar todavía ninguna operación.

---

# 252. Fórmulas fundamentales

## 252.1 Planning

```text
PersistencePlan
=
Plan(
    PersistenceOperationGraph,
    Capabilities,
    Constraints,
    Policies,
    TransactionContext,
    Extensions
)
```

---

# 253. Dependency ordering

Para operaciones `a` y `b`:

```text
RequiresBefore(a, b)
⇒
Order(a) < Order(b)
```

salvo que una estrategia explícita transforme el grafo preservando la misma semántica.

---

# 254. Canonical planning

```text
SameSemanticInputs
⇒
SameSemanticPersistencePlan
```

---

# 255. Safe operation scheduling

```text
Schedulable(op)
=
AllMandatoryDependenciesResolvable(op)
∧
AllRequiredCapabilitiesAvailable(op)
∧
AllRequiredBarriersConstructible(op)
∧
TargetCompatible(op)
```

---

# 256. Safe generated-value consumer

```text
SafeConsumer(C, V)
=
Producer(V) Exists
∧
ProducerScheduledBefore(C)
∧
ResolutionBarrier(V) Exists
∧
ReconciliationAvailable(V)
```

---

# 257. Cycle resolution

```text
ResolvableCycle(C)
=
Exists Strategy S
such that
PreservesPersistenceSemantics(S, C)
∧
CapabilitiesSupport(S)
∧
ConstraintsPermit(S)
∧
PoliciesPermit(S)
```

---

# 258. Unknown capability

```text
CapabilityRequired(S)
∧
CapabilityStatus = UNKNOWN
⇒
¬ SelectStrategy(S)
```

salvo política explícita que cambie el nivel de seguridad.

---

# 259. Batch safety

```text
Batchable(A, B)
=
CompatibleOperationShape
∧
CompatibleTarget
∧
NoDependencyBarrierBetween(A, B)
∧
CompatibleGeneratedValueRequirements
∧
CompatibleLifecycleSemantics
∧
CompatibleConcurrencySemantics
∧
ResultCorrelationPreserved
```

---

# 260. Plan validity

```text
ValidPersistencePlan
=
CompleteOperationCoverage
∧
NoAccidentalDuplicates
∧
DependenciesSatisfied
∧
CyclesResolved
∧
BarriersSatisfiable
∧
TargetsCompatible
∧
RequirementsRepresentable
∧
GeneratedValuesResolvable
∧
LifecycleOrderValid
∧
ReconciliationPathsValid
```

---

# 261. Staleness

```text
PlanIsCurrent
=
Plan.SourceFingerprint
=
CurrentPersistenceFingerprint
∧
Plan.Generation
=
CurrentPersistenceGeneration
```

---

# 262. Safe execution prerequisite

```text
MayExecutePersistencePlan
=
PlanIsCurrent
∧
PlanIsSealed
∧
PlanIsValid
∧
TransactionRequirementsSatisfiable
∧
RuntimeScopeValid
```

---

# 263. Transaction distinction

```text
PersistencePlanCompleted
⇏
TransactionCommitted
```

---

# 264. Generated value distinction

```text
IdentifierAssigned
⇏
DatabaseRowDurable
```

---

# 265. Persistence planning safety

```text
SafePersistencePlanning
=
StableSourceGraph
∧
DeterministicDependencyAnalysis
∧
ExplicitCycleResolution
∧
CapabilityAwareStrategies
∧
ConstraintAwareOrdering
∧
GeneratedValueBarriers
∧
LifecycleBarriers
∧
ReconciliationBarriers
∧
ValidatedTransactionRequirements
∧
CompleteOperationCoverage
∧
ImmutableSealedPlan
```

---

# 266. Persistent runtime safety

```text
SafePlannerRuntime
=
ImmutableSharedPlanner
∧
FrozenPlanningRules
∧
ScopedPlanningSession
∧
NoStaticCurrentPlan
∧
NoStaticCurrentTenant
∧
DeterministicCleanup
∧
NoCrossRequestState
```

---

# 267. Arquitectura consolidada

```text
                         ORM
                          │
                          ▼
                    EntityManager
                          │
                          ▼
                     UnitOfWork
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        IdentityMap   Snapshots    ChangeSets
             │            │            │
             └────────────┼────────────┘
                          ▼
                 PreparedUnitOfWork
                          │
                          ▼
                 Persistence Engine
                          │
                          ▼
              PersistenceOperationGraph
                          │
                          ▼
              ┌─────────────────────┐
              │ Persistence Planner │
              └─────────────────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
       Dependency     Strategy      Barrier
        Analysis      Resolution    Placement
             │            │            │
             └────────────┼────────────┘
                          ▼
                   PersistencePlan
                          │
                          ▼
            Persistence Query Translator
                          │
                          ▼
                    Query Model
                          │
                          ▼
                Semantic Query Engine
                          │
                          ▼
                     Optimizer
                          │
                          ▼
                   Query Planner
                          │
                          ▼
                   SQL Compiler
                          │
                          ▼
                 Execution Engine
                          │
                          ▼
                      Database
                          │
                          ▼
                 Execution Results
                          │
                          ▼
             Persistence Reconciliation
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        IdentityMap   Snapshots     UoW State
```

---

# 268. Master Formula

```text
Database Persistence Planner System
=
Persistence Engine Result Validation
+
Persistence Operation Graph Analysis
+
Dependency Resolution
+
Deterministic Topological Ordering
+
Cycle Detection
+
Strongly Connected Component Analysis
+
Cycle Strategy Resolution
+
Constraint-Aware Planning
+
Generated Value Dependency Planning
+
Generated Value Barriers
+
Relationship Ordering
+
Collection Ordering
+
Insert Ordering
+
Update Ordering
+
Delete Ordering
+
Conflict Graph Analysis
+
Capability-Aware Strategy Resolution
+
Persistence Plan Phases
+
Typed Persistence Plan Steps
+
Lifecycle Barriers
+
Reconciliation Barriers
+
Transaction Requirements
+
Savepoint Requirements
+
Batch Candidate Analysis
+
Logical Target Validation
+
Plan Validation
+
Complete Operation Coverage
+
Plan Fingerprinting
+
Stale Plan Detection
+
Immutable Plan Sealing
+
Planning Decision Provenance
+
Explainability
+
Extension Governance
+
Resource Governance
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Failure Modeling
```

---

# 269. Master Rule

> **En VoltStack, el Persistence Planner es el puente entre la intención ORM y la futura ejecución: recibe un grafo estable de operaciones de persistencia y construye un plan determinista que conserva dependencias, ciclos, constraints, valores generados, lifecycle boundaries, reconciliación, requisitos transaccionales y oportunidades de batching. El Planner puede decidir el orden y la estrategia semántica, pero nunca debe ejecutar la persistencia, generar SQL, modificar entidades ni alterar el UnitOfWork.**

---

# 270. Resultado del bloque hasta este punto

Con:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
```

VoltStack posee ahora:

```text
Entity
  │
  ▼
Identity Map
  │
  ▼
UnitOfWork
  │
  ├── Entity State
  ├── Snapshot
  └── ChangeSet
  │
  ▼
PreparedUnitOfWork
  │
  ▼
Persistence Engine
  │
  ▼
PersistenceOperationGraph
  │
  ▼
Persistence Planner
  │
  ▼
PersistencePlan
```

La arquitectura ya puede responder dos preguntas diferentes:

```text
Persistence Engine
"What must be persisted?"

Persistence Planner
"In what safe semantic order and strategy?"
```

La siguiente etapa comienza a especializar la persistencia por operación.

---

# 271. Siguiente documento

```text
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
```

Deberá formalizar específicamente el flujo:

```text
NEW Entity
   ↓
Insert Persistence Analysis
   ↓
Insertable Property Resolution
   ↓
Identifier Strategy
   ↓
Generated Value Requirements
   ↓
Relationship Dependencies
   ↓
InsertEntityOperation
   ↓
Persistence Planner
   ↓
Insert Persistence Plan Step
   ↓
Insert Query Model
   ↓
Execution
   ↓
Generated Value Capture
   ↓
Identity Establishment
   ↓
IdentityMap Registration
   ↓
Snapshot Establishment
   ↓
Entity State Reconciliation
```

incluyendo:

```text
assigned identifiers
database-generated identifiers
UUID/ULID identifiers
sequence strategies
identity columns
composite identifiers
generated/default columns
insertable fields
non-insertable fields
database defaults
nullable/default semantics
omitted value vs explicit NULL
relationship foreign keys
deferred foreign keys
insert dependencies
cascade persist integration
generated-value barriers
RETURNING capabilities
generated-key fallback strategies
insert batching constraints
optimistic version initialization
lifecycle prePersist/postPersist
insert outcome certainty
zero-row/one-row semantics
unknown outcomes
retry/replay safety
duplicate-key failures
IdentityMap promotion
snapshot creation
UnitOfWork reconciliation
transaction rollback implications
persistent runtime isolation
telemetry
diagnostics
testing
extension points
```

con la regla central:

> **Una entidad `NEW` no se convierte en `MANAGED` simplemente porque exista una intención de INSERT; VoltStack solo podrá establecer su baseline persistente cuando la operación de inserción haya alcanzado un resultado suficientemente cierto y cualquier identidad o valor generado necesario haya sido reconciliado correctamente.**