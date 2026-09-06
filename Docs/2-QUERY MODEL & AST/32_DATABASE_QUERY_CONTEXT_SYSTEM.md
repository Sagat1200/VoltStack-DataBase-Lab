# 32_DATABASE_QUERY_CONTEXT_SYSTEM.md

# VoltStack Quantum Database
## Query Context System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 32 — Query Context System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Processing Context  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del:

```text
VoltStack/Quantum/Database Query Context System
```

`QueryContext` representará el **estado temporal, explícito y operation-scoped** necesario para procesar una consulta a través del Query Engine.

Su propósito principal será soportar etapas como:

```text
Query Model
    │
    ▼
AST Construction
    │
    ▼
Normalization
    │
    ▼
Structural Validation
    │
    ▼
Semantic Analysis
    │
    ▼
Optimization
    │
    ▼
Planning
    │
    ▼
Compilation
```

sin introducir:

- estado global;
- Service Locator;
- conexiones físicas;
- transacciones activas;
- EntityManager;
- UnitOfWork;
- Request HTTP;
- current user;
- current tenant mutable;
- recursos de ejecución.

La regla fundamental será:

```text
QueryMetadata
≠
QueryContext
≠
ExecutionContext
≠
DatabaseContext
≠
TransactionContext
```

---

# 2. Problema arquitectónico

El Query Engine necesita información temporal mientras procesa una consulta.

Por ejemplo, durante semantic analysis puede necesitar:

```text
current semantic phase
symbol scopes
resolved symbols
temporary type constraints
schema metadata view
capability snapshot
diagnostic collector
extension state
normalization state
analysis memoization
```

Durante planning podría necesitar:

```text
logical planning state
strategy candidates
capability requirements
cost hints
planning diagnostics
```

Y durante compilation:

```text
parameter allocator
alias allocator
dialect
platform snapshot
capability snapshot
compiler-local state
```

El error sería colocar todo esto dentro de:

```text
Query
AST
QueryMetadata
DatabaseContext
global static variables
```

---

# 3. Solución

VoltStack utilizará contextos temporales explícitos.

Conceptualmente:

```text
Query Artifact
     │
     ▼
Query Processing Session
     │
     ├── QueryContext
     │
     ├── NormalizationContext
     │
     ├── ValidationContext
     │
     ├── SemanticContext
     │
     ├── OptimizationContext
     │
     ├── PlanningContext
     │
     └── CompilationContext
     │
     ▼
CompiledQuery
```

---

# 4. Regla maestra

> `QueryContext` será un contexto temporal de procesamiento, nunca un contenedor universal de servicios o recursos runtime.

---

# 5. QueryContext como operation scope

El lifetime normal será:

```text
BEGIN QUERY PROCESSING
        │
        ▼
 Create QueryContext
        │
        ▼
 Process Query
        │
        ▼
 Produce Artifact
        │
        ▼
 Dispose QueryContext
        │
        ▼
END QUERY PROCESSING
```

Por tanto:

```text
QueryContext Lifetime
=
Query Processing Operation
```

---

# 6. QueryContext ≠ Request Scope

Una request puede procesar muchas consultas:

```text
HTTP Request
│
├── QueryContext A
├── QueryContext B
├── QueryContext C
└── QueryContext D
```

Por tanto:

```text
Request Scope
>
Query Processing Scope
```

---

# 7. QueryContext ≠ DatabaseContext

`DatabaseContext` representa el contexto Database de una ejecución del framework.

Por ejemplo:

```text
ExecutionScope
    │
    ▼
DatabaseContext
    │
    ├── EntityManager
    ├── UnitOfWork
    ├── IdentityMap
    ├── TransactionContext
    ├── scoped connection coordination
    └── lifecycle state
```

Mientras:

```text
QueryContext
    │
    ├── processing identity
    ├── target information
    ├── capabilities
    ├── schema view
    ├── temporary analysis state
    └── diagnostics
```

---

# 8. QueryContext ≠ ExecutionContext

`ExecutionContext` existe cuando una consulta va a ejecutarse realmente.

```text
Query Processing
      │
      ▼
CompiledQuery
      │
      ▼
ExecutionContext
      │
      ├── ConnectionLease
      ├── TransactionContext
      ├── deadline
      ├── cancellation
      ├── execution telemetry
      └── cleanup ownership
```

`QueryContext` no deberá poseer esos recursos.

---

# 9. QueryContext ≠ TransactionContext

Incorrecto:

```text
QueryContext
└── activeTransaction
```

Correcto:

```text
QueryContext
└── TransactionRequirement

ExecutionContext
└── TransactionContext
```

---

# 10. QueryContext ≠ QueryMetadata

`QueryMetadata` es declarativa y potencialmente reusable.

`QueryContext` es temporal.

```text
QueryMetadata
=
what accompanies the query

QueryContext
=
temporary environment used to process the query
```

---

# 11. QueryContext ≠ Query

El Query artifact podrá sobrevivir al contexto:

```text
Reusable Query
      │
      ├── Processing Context A
      ├── Processing Context B
      └── Processing Context C
```

---

# 12. Objetivos

El Query Context System deberá proporcionar:

1. aislamiento por operación;
2. procesamiento determinista;
3. información explícita del target;
4. capability snapshots;
5. schema metadata access controlado;
6. contextos especializados por fase;
7. diagnostics temporales;
8. memoization local;
9. extension state temporal;
10. protección contra state leakage;
11. soporte para persistent workers;
12. soporte para fibers/coroutines;
13. cancellation cooperativa de procesamiento cuando aplique;
14. budgets y límites;
15. ownership explícito;
16. lifecycle determinista.

---

# 13. No objetivos

No deberá:

- ejecutar SQL;
- abrir conexiones automáticamente;
- almacenar PDO;
- poseer ConnectionLease;
- manejar resultados;
- hidratar entidades;
- almacenar UnitOfWork;
- almacenar IdentityMap;
- resolver current user globalmente;
- resolver current tenant globalmente;
- consultar Service Container arbitrariamente;
- funcionar como application context.

---

# 14. Arquitectura general

```text
                     Query Artifact
                          │
                          ▼
                  QueryContextFactory
                          │
                          ▼
                     QueryContext
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
 Target Snapshot    Schema Snapshot   Capability Snapshot
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
                Phase Context Factory
                          │
       ┌──────────────────┼───────────────────┐
       ▼                  ▼                   ▼
Normalization       SemanticContext      PlanningContext
Context                  │                   │
       │                  ▼                   ▼
       │           OptimizationContext  CompilationContext
       │
       ▼
ValidationContext
```

---

# 15. Query processing session

Podrá conceptualizarse una:

```text
QueryProcessingSession
```

que delimite una ejecución completa del pipeline.

---

# 16. QueryProcessingSession

Conceptualmente:

```text
QueryProcessingSession
├── ProcessingId
├── QueryContext
├── phase lifecycle
├── diagnostics
├── budgets
└── temporary resources
```

No necesariamente deberá existir como clase pública.

---

# 17. ProcessingId

Cada procesamiento podrá tener:

```text
QueryProcessingId
```

distinto de:

```text
QueryInstanceId
ExecutionId
TraceId
```

---

# 18. Identidades

```text
QueryInstanceId
=
logical query artifact

QueryProcessingId
=
one pass through query pipeline

ExecutionId
=
one actual execution
```

---

# 19. Ejemplo

Una Query reusable:

```text
QueryInstance:
Q-100
```

puede producir:

```text
Processing:
P-1 → PostgreSQL compile

Processing:
P-2 → MySQL compile

Processing:
P-3 → PostgreSQL validation
```

Y posteriormente:

```text
Execution:
E-1
E-2
E-3
```

---

# 20. QueryContext mínimo

El contexto base deberá mantenerse pequeño.

Conceptualmente:

```text
QueryContext
├── processingId
├── processingMode
├── target
├── capabilities
├── schemaView
├── metadata
├── policySnapshot
├── diagnostics
└── budgets
```

No todo será obligatorio en todas las fases.

---

# 21. ProcessingMode

Valores conceptuales:

```text
VALIDATE
ANALYZE
OPTIMIZE
PLAN
COMPILE
EXECUTE_PREPARATION
EXPLAIN
```

---

# 22. EXECUTE_PREPARATION

No significa ejecutar SQL.

Significa preparar artifacts necesarios para una ejecución posterior.

---

# 23. Target

El contexto podrá trabajar con:

```text
QueryTarget
```

---

# 24. QueryTarget

Conceptualmente:

```text
QueryTarget
├── logicalConnection?
├── platform
├── dialect
├── capabilitySnapshot
├── bindingProfile?
└── targetFingerprint
```

---

# 25. QueryTarget ≠ physical connection

No contendrá:

```text
PDO
NativeConnection
ConnectionLease
socket
```

---

# 26. Offline target

VoltStack deberá soportar:

```text
OfflineQueryTarget
```

Ejemplo:

```text
Platform:
PostgreSQL

Version:
18

Dialect:
postgresql

Capabilities:
precomputed
```

sin conexión real.

---

# 27. Online target

Un target podrá haber sido derivado previamente de información descubierta online.

Pero QueryContext recibe:

```text
ResolvedTargetSnapshot
```

no la conexión utilizada para descubrirlo.

---

# 28. Snapshot semantics

Información target deberá preferentemente ser:

- immutable;
- versioned;
- fingerprinted;
- stable durante la operación.

---

# 29. CapabilitySnapshot

El contexto utilizará:

```text
CapabilitySnapshot
```

del sistema definido en:

```text
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
```

---

# 30. Snapshot consistency

Una misma operación no deberá observar capacidades cambiantes a mitad del pipeline.

---

# 31. Regla

```text
One Query Processing Operation
→ One Stable Capability Snapshot
```

salvo una transición explícita de fase que invalide y reinicie el procesamiento.

---

# 32. Platform snapshot

El mismo principio aplicará a:

```text
ResolvedPlatform
DialectFingerprint
BindingProfile
SchemaSnapshot
PolicySnapshot
```

---

# 33. No live mutable target

Incorrecto:

```text
QueryContext
└── mutable connection target whose platform changes
```

---

# 34. Schema access

Semantic Analysis necesita información del schema.

No deberá consultar la base de datos arbitrariamente durante cualquier visitor.

---

# 35. SchemaView

Se introducirá conceptualmente:

```text
QuerySchemaView
```

---

# 36. QuerySchemaView

Podrá exponer:

```text
table metadata
column metadata
relation metadata
type metadata
constraint metadata
schema namespaces
```

---

# 37. SchemaView ≠ SchemaManager

Preferir:

```text
immutable/read-only view
```

sobre un servicio capaz de realizar introspection arbitraria.

---

# 38. Offline semantic analysis

Deberá ser posible utilizar:

```text
CompiledSchemaMetadata
```

sin conexión.

---

# 39. Online schema discovery

Si se requiere introspection online, deberá ocurrir antes o mediante un mecanismo explícito controlado.

No como efecto oculto de:

```text
resolveColumn()
```

---

# 40. No hidden I/O

Regla crítica:

> Las operaciones del Query Engine que se declaren puras u offline no realizarán I/O oculto mediante QueryContext.

---

# 41. SchemaSnapshot

Conceptualmente:

```text
SchemaSnapshot
├── schemaFingerprint
├── metadata
├── source
└── generation
```

---

# 42. Schema generation

Permite invalidar:

- semantic cache;
- plan cache;
- compiled artifacts;

cuando el schema cambia.

---

# 43. PolicySnapshot

El QueryContext podrá recibir políticas ya resueltas.

---

# 44. PolicySnapshot

Puede contener:

```text
query safety policy
security policy
compilation policy
optimization policy
portability policy
diagnostic policy
```

---

# 45. PolicySnapshot ≠ Policy Service Locator

No deberá contener:

```text
AuthorizationManager
CurrentUserResolver
TenantManager
Container
```

---

# 46. Context inputs

Los inputs del QueryContext deberán ser explícitos.

Ejemplo conceptual:

```php
final readonly class QueryContextInput
{
    public function __construct(
        public QueryProcessingMode $mode,
        public QueryTarget $target,
        public QuerySchemaView $schema,
        public QueryPolicySnapshot $policies,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 47. QueryContextFactory

Su responsabilidad será crear contextos válidos.

---

# 48. Factory responsibilities

Puede:

- validar inputs;
- asignar ProcessingId;
- crear diagnostics collector;
- establecer budgets;
- crear memoization stores;
- construir phase context factories.

---

# 49. Factory non-responsibilities

No deberá:

- abrir conexión;
- iniciar transacción;
- ejecutar introspection implícita;
- resolver current tenant global;
- resolver current user global.

---

# 50. Context specialization

No se recomienda pasar un gigantesco `QueryContext` a todas las fases.

---

# 51. Phase-specific contexts

Se preferirá:

```text
QueryContext
      │
      ├── NormalizationContext
      ├── ValidationContext
      ├── SemanticAnalysisContext
      ├── OptimizationContext
      ├── PlanningContext
      └── CompilationContext
```

---

# 52. Regla de mínimo privilegio

Cada fase deberá recibir sólo aquello que necesita.

---

# 53. Ejemplo

`Normalizer` probablemente no necesita:

```text
Connection topology
TransactionContext
Driver
Physical connection
```

---

# 54. SemanticAnalysisContext

Podrá necesitar:

```text
QuerySchemaView
QueryTypeRegistry
SemanticFunctionRegistry
CapabilitySnapshot
PolicySnapshot
DiagnosticSink
AnalysisMemo
```

---

# 55. PlanningContext

Podrá necesitar:

```text
SemanticGraph
CapabilitySnapshot
PlatformSnapshot
PlanningPolicy
CostHints
StrategyRegistry
DiagnosticSink
```

---

# 56. CompilationContext

Podrá necesitar:

```text
ExecutionPlan
Dialect
PlatformSnapshot
CapabilitySnapshot
BindingProfile
ParameterAllocator
AliasAllocator
CompilerExtensionRegistry
```

---

# 57. CompilationContext state

Algunas estructuras son mutable operation-local por naturaleza.

Ejemplo:

```text
ParameterAllocator
AliasAllocator
SqlBuffer
CompilationMemo
```

---

# 58. Mutable local state is allowed

La arquitectura no prohíbe toda mutabilidad.

Prohíbe:

```text
unowned
shared
cross-request
global
hidden
mutable state
```

---

# 59. Regla

```text
Local Explicit Mutation
>
Global Hidden Mutation
```

---

# 60. QueryContext immutability

El contexto base debería ser principalmente immutable.

---

# 61. Mutable phase state

El estado mutable deberá vivir en objetos especializados.

Ejemplo:

```text
SemanticAnalysisState
PlanningState
CompilationState
```

---

# 62. Context ≠ State

Preferentemente:

```text
Context
=
stable inputs/environment

State
=
temporary mutable progress
```

---

# 63. Ejemplo

```text
SemanticAnalysisContext
├── schema
├── capabilities
├── type registry
└── policies

SemanticAnalysisState
├── symbol scopes
├── constraints
├── inferred types
└── annotations
```

---

# 64. Ventaja

Esto evita convertir:

```text
SemanticContext
```

en un objeto mutable imposible de razonar.

---

# 65. Context/state pair

Patrón recomendado:

```text
PhaseContext
+
PhaseState
→
PhaseArtifact
```

---

# 66. Pipeline conceptual

```text
Query AST
   │
   ▼
SemanticAnalysisContext
+
SemanticAnalysisState
   │
   ▼
SemanticArtifact
```

---

# 67. NormalizationContext

Responsabilidades:

- normalization rules;
- normalization policy;
- extension normalizers;
- diagnostic sink;
- normalization budget.

---

# 68. NormalizationState

Puede contener:

```text
rewrite count
visited nodes
memoization
cycle detection
change flags
```

---

# 69. ValidationContext

Puede contener:

```text
validation rules
validation mode
capability snapshot
schema view when needed
policy snapshot
diagnostics
```

---

# 70. ValidationState

Puede contener:

```text
errors
warnings
visited paths
rule execution state
```

---

# 71. SemanticAnalysisContext

Será central para documentos posteriores.

---

# 72. Semantic context inputs

```text
AST
SchemaView
Type System
Symbol Rules
Function Registry
Operator Registry
Capability Snapshot
Semantic Policy
Extension Registry
```

---

# 73. SemanticAnalysisState

Puede contener:

```text
ScopeStack
SymbolTableBuilder
TypeConstraintGraph
RelationResolutionState
SemanticAnnotationBuilder
CapabilityRequirementBuilder
```

---

# 74. Semantic output

Produce:

```text
SemanticQueryArtifact
├── AST
├── SemanticGraph
├── SymbolTable
├── TypeTable
├── RelationTable
├── ConstraintSet
├── CapabilityRequirements
└── SemanticAnnotations
```

---

# 75. OptimizationContext

Puede contener:

```text
optimization rules
optimization level
semantic artifact
capabilities
policy
cost hints
diagnostic sink
budget
```

---

# 76. OptimizationState

Puede contener:

```text
iteration
rewrite count
rule matches
memoization
equivalence data
cost observations
```

---

# 77. Optimizer budget

Debe existir protección contra:

```text
infinite rewrite loops
combinatorial explosion
extension bugs
excessive memory use
```

---

# 78. OptimizationBudget

Podrá definir:

```text
maxIterations
maxRewrites
maxNodes
maxTime
maxMemoryEstimate
```

---

# 79. Deterministic budget

Los límites principales deberán ser deterministas cuando sea posible.

---

# 80. Wall-clock budget

Puede existir como protección adicional, pero no debería ser la única forma de limitar procesamiento.

---

# 81. PlanningContext

Recibe:

```text
SemanticArtifact
OptimizedArtifact
CapabilitySnapshot
PlatformSnapshot
PlanningPolicy
StrategyRegistry
```

---

# 82. PlanningState

Puede contener:

```text
candidate strategies
logical plan builder
physical plan builder
cost data
rejected strategies
requirement resolution
```

---

# 83. Planning output

```text
LogicalPlan
PhysicalPlan
ExecutionPlan
```

según la fase.

---

# 84. CompilationContext

Debe ser independiente de:

```text
EntityManager
UnitOfWork
HTTP
current user
current tenant object
```

---

# 85. Compiler input

Idealmente:

```text
ExecutionPlan
+
CompilationContext
```

---

# 86. Compiler output

```text
CompiledQuery
```

---

# 87. ParameterAllocator

Es operation-local.

Nunca:

```text
singleton
static
worker-global mutable counter
```

---

# 88. AliasAllocator

Misma regla.

---

# 89. Example

Incorrecto:

```php
final class PostgreSqlDialect
{
    private int $parameter = 0;
}
```

si el Dialect es compartido.

Correcto:

```text
PostgreSqlDialect
=
stateless

CompilationContext
└── ParameterAllocator
```

---

# 90. Reentrancy

Stateless services compartidos deberán ser reentrant.

---

# 91. Persistent runtime

Esto es crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 92. FrankenPHP lifecycle

```text
Worker Boot
    │
    ├── immutable registries
    ├── compiler services
    ├── dialects
    └── platform descriptors
    │
    ▼
Request A
    │
    ├── QueryContext A1
    ├── QueryContext A2
    └── QueryContext A3
    │
    ▼
Request Cleanup
    │
    ▼
Request B
    │
    ├── QueryContext B1
    └── QueryContext B2
```

---

# 93. Invariante

Ningún `QueryContext A*` podrá ser accesible desde Request B.

---

# 94. RoadRunner

Mismo modelo:

```text
persistent worker
+
execution-scoped DatabaseContext
+
operation-scoped QueryContext
```

---

# 95. OpenSwoole

La concurrencia puede ocurrir dentro del mismo worker.

```text
Worker
├── Coroutine A
│   ├── QueryContext A
│   └── CompilationState A
│
└── Coroutine B
    ├── QueryContext B
    └── CompilationState B
```

---

# 96. Prohibición

Nunca:

```text
static QueryContext $current
```

---

# 97. Fiber-local context

VoltStack podrá utilizar internamente mecanismos de propagación scoped.

Pero los componentes Query Engine no deberán depender directamente de globals fiber-local.

---

# 98. Preferencia

Pasar contexto explícitamente:

```php
$semanticAnalyzer->analyze(
    $ast,
    $semanticContext,
);
```

---

# 99. Context propagation

La propagación explícita será la estrategia principal.

---

# 100. ContextAccessor

Un accessor scoped podrá existir en capas de integración si fuera necesario.

Pero no deberá convertirse en dependencia central del Query Engine.

---

# 101. No Service Locator

Incorrecto:

```php
$context = QueryContext::current();

$schema = $context->container()->get(...);
```

---

# 102. Correcto

```php
$analyzer->analyze(
    ast: $ast,
    context: $context,
);
```

---

# 103. Context capabilities

Cada phase context podrá exponer typed views.

Ejemplo:

```text
SemanticCapabilityView
PlanningCapabilityView
CompilationCapabilityView
```

---

# 104. Why typed views

Reduce dependencia de un gigantesco:

```text
CapabilitySnapshot
```

cuando una fase sólo necesita algunas categorías.

---

# 105. QueryContext composition

Conceptualmente:

```text
QueryContext
├── Identity
├── Target
├── Schema
├── Metadata
├── Policies
├── Capabilities
├── Diagnostics
├── Budgets
└── Extensions
```

---

# 106. Identity section

```text
QueryProcessingId
QueryInstanceId?
ProcessingMode
```

---

# 107. Target section

```text
ResolvedPlatformSnapshot
Dialect
CapabilitySnapshot
BindingProfile
TargetFingerprint
```

---

# 108. Schema section

```text
QuerySchemaView
SchemaFingerprint
SchemaGeneration
```

---

# 109. Metadata section

```text
DeclaredQueryMetadata
EffectiveQueryMetadata?
```

según la fase.

---

# 110. Policy section

```text
QueryPolicySnapshot
SecurityPolicySnapshot
OptimizationPolicy
CompilationPolicy
```

---

# 111. Diagnostics section

```text
DiagnosticSink
DiagnosticPolicy
TracePolicy
```

---

# 112. Budgets section

```text
QueryProcessingBudget
NormalizationBudget
SemanticBudget
OptimizationBudget
PlanningBudget
CompilationBudget
```

---

# 113. Extension section

No será:

```text
array<string, mixed>
```

---

# 114. ExtensionContextRegistry

Podrá registrar factories/descriptors de context extensions durante bootstrap.

---

# 115. Extension context data

Debe ser:

- typed;
- scoped;
- explicit;
- lifecycle-aware;
- resource-safe.

---

# 116. Extension state

Una extensión puede necesitar state temporal.

Ejemplo:

```text
CustomSemanticExtensionState
```

---

# 117. Extension state lifetime

Como máximo:

```text
QueryProcessingOperation
```

salvo que sea immutable shared configuration.

---

# 118. Extension state registration

Durante bootstrap:

```text
Extension
   │
   ▼
Context Extension Descriptor
   │
   ▼
Registry
   │
   ▼
Freeze
```

---

# 119. Runtime extension values

Durante query processing:

```text
Descriptor
   │
   ▼
Scoped Extension Context Factory
   │
   ▼
Operation-local Extension Context
```

---

# 120. No arbitrary extension objects

El core no deberá aceptar cualquier object sin descriptor/lifetime.

---

# 121. Context ownership

Cada context tendrá owner explícito.

---

# 122. Ownership table

| Context | Owner |
|---|---|
| DatabaseContext | ExecutionScope |
| QueryContext | Query Processing Session |
| NormalizationContext | Normalization operation |
| ValidationContext | Validation operation |
| SemanticContext | Semantic analysis |
| OptimizationContext | Optimizer |
| PlanningContext | Planner |
| CompilationContext | Compiler pipeline |
| ExecutionContext | Query Executor |
| TransactionContext | Transaction Manager |

---

# 123. Disposal

`QueryContext` puede requerir disposal aunque no posea conexiones.

---

# 124. Why disposal

Puede poseer:

- temporary buffers;
- diagnostic collectors;
- memoization maps;
- extension state;
- large graphs;
- cancellation registrations.

---

# 125. QueryContext lifecycle

```text
CREATED
   │
   ▼
ACTIVE
   │
   ▼
CLOSING
   │
   ▼
CLOSED
```

---

# 126. Invalid use

Después de `CLOSED`, cualquier operación mutable deberá fallar.

---

# 127. Immutable snapshots

Los snapshots extraídos pueden sobrevivir si están diseñados para ello.

---

# 128. Cleanup

Debe:

- release temporary references;
- clear large memoization stores;
- dispose extension state;
- detach diagnostics;
- invalidate mutable phase state.

---

# 129. No database rollback

QueryContext cleanup no realiza rollback de transacciones.

Eso pertenece a:

```text
TransactionContext / DatabaseContext cleanup
```

---

# 130. No connection release

Tampoco libera ConnectionLease si no la posee.

---

# 131. Failure cleanup

Si semantic analysis falla:

```text
QueryContext
→ CLOSING
→ cleanup
→ CLOSED
```

---

# 132. Exceptions

El lifecycle debe funcionar incluso cuando ocurre:

```text
NormalizationException
ValidationException
SemanticException
OptimizationException
PlanningException
CompilationException
ExtensionException
```

---

# 133. Try/finally model

Conceptualmente:

```php
$context = $factory->create($input);

try {
    return $pipeline->process($query, $context);
} finally {
    $context->close();
}
```

---

# 134. Idempotent close

`close()` deberá ser idempotente.

---

# 135. QueryContext resource policy

Por defecto no deberá poseer recursos externos.

---

# 136. Temporary memory resources

Sí puede poseer memoria operation-local.

---

# 137. Memory budget

Consultas extremadamente complejas pueden consumir memoria durante:

- AST processing;
- semantic graph construction;
- optimization;
- planning.

---

# 138. QueryProcessingBudget

Podrá incluir:

```text
maxAstNodes
maxExpressionDepth
maxPredicateDepth
maxSubqueryDepth
maxCteCount
maxJoinCount
maxParameters
maxSemanticSymbols
maxRewriteCount
maxPlanningCandidates
```

---

# 139. Security benefit

Budgets ayudan a proteger contra:

```text
pathological query construction
extension bugs
resource exhaustion
denial-of-service patterns
```

---

# 140. Budget ≠ business limit

No confundir con:

```text
LIMIT 100
```

del SQL.

---

# 141. Structural depth limits

AST traversal deberá protegerse contra profundidad extrema.

---

# 142. Iterative traversal

Donde sea conveniente, podrá preferirse traversal iterativo para evitar stack overflow.

---

# 143. Budget exceeded

Deberá producir errores específicos:

```text
QueryComplexityLimitExceededException
QueryProcessingBudgetExceededException
```

---

# 144. Time budget

Puede existir:

```text
ProcessingDeadline
```

pero debe distinguirse de:

```text
StatementTimeout
```

---

# 145. Processing deadline

Limita tiempo del Query Engine.

---

# 146. Statement timeout

Limita ejecución en DB.

---

# 147. Diferencia

```text
Compilation Timeout
≠
SQL Execution Timeout
```

---

# 148. Cancellation

Query processing puede soportar cancelación cooperativa.

---

# 149. ProcessingCancellation

Puede ser un abstraction operation-local.

---

# 150. No execution token reuse

No asumir que el mismo cancellation token de ejecución debe utilizarse automáticamente durante compilation.

---

# 151. Cancellation checkpoints

Podrán existir en:

```text
large AST traversal
semantic graph construction
optimizer iterations
large planning search
```

---

# 152. Determinism caution

La cancelación interrumpe el proceso, pero no deberá producir artifacts parciales considerados válidos.

---

# 153. Partial artifacts

Artifacts incompletos deberán ser:

- internos;
- marcados incomplete;
- no cacheables;
- no ejecutables.

---

# 154. Diagnostics

El contexto podrá tener:

```text
DiagnosticSink
```

---

# 155. DiagnosticSink

Recibe eventos estructurados.

Ejemplos:

```text
normalization warning
semantic warning
capability fallback
optimization note
planning decision
compilation warning
```

---

# 156. DiagnosticSink ≠ logger

El Query Engine no debería depender directamente de un logger.

---

# 157. Diagnostic adapters

Posteriormente:

```text
DiagnosticSink
   │
   ├── Debug Toolbar Adapter
   ├── Telemetry Adapter
   ├── CLI Adapter
   └── Test Collector
```

---

# 158. NullDiagnosticSink

Cuando diagnostics estén desactivados:

```text
NullDiagnosticSink
```

evita condicionales dispersos.

---

# 159. Diagnostic levels

Conceptualmente:

```text
TRACE
INFO
NOTICE
WARNING
ERROR
```

---

# 160. Diagnostic categories

```text
NORMALIZATION
VALIDATION
SEMANTIC
CAPABILITY
OPTIMIZATION
PLANNING
COMPILATION
PORTABILITY
SECURITY
EXTENSION
PERFORMANCE
```

---

# 161. Diagnostic redaction

Nunca deberá exponer:

- parameter values sensibles;
- credentials;
- connection secrets;
- private tenant data.

---

# 162. Diagnostics and source map

Podrán referenciar:

```text
AstNodeId
ExpressionId
PredicateId
ParameterId
SourceLocation
```

---

# 163. Diagnostics side table

No será necesario mutar AST nodes para adjuntar warnings.

---

# 164. Memoization

El Query Engine podrá usar caches temporales.

---

# 165. Operation-local memoization

Ejemplos:

```text
ExpressionTypeMemo
SymbolResolutionMemo
PredicateNormalizationMemo
CapabilityRequirementMemo
CompilerFragmentMemo
```

---

# 166. Memoization ≠ global cache

Operation-local memoization se destruye al cerrar QueryContext.

---

# 167. Shared cache

Caches persistentes requerirán:

- deterministic key;
- fingerprint;
- versioning;
- no scoped state.

---

# 168. Promotion

Un resultado local sólo podrá promoverse a cache compartido mediante un componente explícito.

---

# 169. No accidental promotion

Nunca:

```text
QueryContext.memo
→ static cache automatically
```

---

# 170. Context fingerprints

El contexto podrá tener:

```text
QueryProcessingFingerprint
```

---

# 171. Processing fingerprint inputs

Puede incluir:

```text
QueryShapeFingerprint
SchemaFingerprint
CapabilityFingerprint
PolicyFingerprint
ExtensionGraphFingerprint
ProcessingMode
EngineVersion
```

---

# 172. No runtime resource identity

No incluir:

```text
PDO object ID
ConnectionLease ID
Request object hash
Fiber ID
```

para caches semánticos.

---

# 173. Context generation

Podrán existir generations para:

```text
schema
configuration
capabilities
extensions
compiler
policies
```

---

# 174. Cache invalidation

Cuando una generación relevante cambia:

```text
old artifact
→ no longer reusable
```

---

# 175. QueryContextFactory inputs

Idealmente serán snapshots ya preparados.

---

# 176. Avoid hidden resolution

No:

```text
QueryContextFactory
→ inspect database
→ load tenant
→ open connection
→ read current user
```

---

# 177. Composition Root

Las dependencias se preparan desde capas superiores.

---

# 178. Context assembly

Conceptualmente:

```text
Database Services
      │
      ├── Config Snapshot
      ├── Platform Resolution
      ├── Schema Metadata
      ├── Policy Snapshot
      └── Extension Snapshot
      │
      ▼
QueryContextFactory
      │
      ▼
QueryContext
```

---

# 179. Connection resolution timing

No todas las queries necesitan conexión antes de compilation.

---

# 180. Offline compilation

Debe ser posible:

```text
Query
+
OfflineTarget
+
CompiledSchemaMetadata
→
CompiledQuery
```

---

# 181. Online execution

Posteriormente:

```text
CompiledQuery
+
ExecutionContext
→
Result
```

---

# 182. Runtime target validation

Antes de ejecutar un `CompiledQuery`, el Executor deberá validar compatibilidad suficiente con el target real.

---

# 183. QueryContext does not guarantee runtime availability

Un query puede compilar correctamente y fallar al ejecutar porque:

- DB unavailable;
- connection timeout;
- replica unavailable;
- credential expired.

Eso no invalida QueryContext architecture.

---

# 184. Capability snapshot provenance

Debe conocerse si fue:

```text
CONFIGURED
OFFLINE
DISCOVERED
CACHED
RESOLVED
```

---

# 185. Certainty

Puede existir:

```text
EXACT
ASSUMED
CONSERVATIVE
UNKNOWN
```

---

# 186. Planning under uncertainty

Si una capability crítica es UNKNOWN:

```text
Planner
```

deberá:

- elegir estrategia conservadora;
- requerir runtime validation;
- o rechazar.

No asumir soporte.

---

# 187. Context and portability

`QueryContext` podrá incluir:

```text
PortabilityPolicy
```

---

# 188. Portability policies

Ejemplos:

```text
STRICT_PORTABLE
ALLOW_CAPABILITY_DEPENDENT
ALLOW_PLATFORM_SPECIFIC
ALLOW_DIALECT_SPECIFIC
ALLOW_RAW
```

---

# 189. Compiler enforcement

Una query platform-specific bajo:

```text
STRICT_PORTABLE
```

deberá producir diagnóstico/error apropiado.

---

# 190. Context and security

Podrá incluir:

```text
QuerySecurityPolicySnapshot
```

---

# 191. Security snapshot

Puede definir:

```text
raw SQL policy
identifier policy
parameter count limits
sensitive diagnostics policy
unsafe hint policy
extension policy
```

---

# 192. Security snapshot immutable

No deberá cambiar durante procesamiento.

---

# 193. Security policy precedence

Una extension no podrá debilitarla.

---

# 194. Context and query metadata

`QueryContext` puede leer:

```text
QueryMetadata
```

pero no deberá mutarla.

---

# 195. Effective metadata

Puede derivarse durante processing:

```text
DeclaredMetadata
+
SemanticRequirements
+
PolicySnapshot
+
CapabilitySnapshot
→
EffectiveQueryMetadata
```

---

# 196. Ownership

`EffectiveQueryMetadata` será un artifact producido, no mutable state global.

---

# 197. Context and semantic graph

`SemanticGraph` será output de Semantic Analysis.

No parte del QueryContext base.

---

# 198. Optimization context receives semantic graph

```text
SemanticArtifact
+
OptimizationContext
→
OptimizedSemanticArtifact
```

---

# 199. Planning receives optimized artifact

```text
OptimizedArtifact
+
PlanningContext
→
QueryPlan
```

---

# 200. Compilation receives plan

```text
ExecutionPlan
+
CompilationContext
→
CompiledQuery
```

---

# 201. No backwards dependency

Compiler no deberá volver al:

```text
EntityManager
Repository
Query Builder
HTTP Request
```

para obtener información faltante.

---

# 202. Context completeness

Cada phase context debe estar completo antes de iniciar su fase.

---

# 203. Fail early

Si falta:

```text
Dialect
CapabilitySnapshot
SchemaView
required extension
```

deberá fallar al crear/validar el contexto correspondiente.

---

# 204. Context validation

Podrá existir:

```text
QueryContextValidator
```

---

# 205. Validation checks

Ejemplos:

```text
target/dialect compatibility
platform/capability compatibility
schema generation validity
policy compatibility
extension compatibility
processing mode requirements
```

---

# 206. Context mode requirements

Ejemplo:

```text
COMPILE
```

requiere:

```text
Dialect
Platform Snapshot
Capability Snapshot
Binding Profile
```

---

# 207. VALIDATE mode

Podría requerir menos.

---

# 208. ANALYZE mode

Requiere SchemaView si la query depende de schema resolution.

---

# 209. Context profiles

Podrán existir:

```text
ValidationContextProfile
SemanticContextProfile
CompilationContextProfile
```

---

# 210. Avoid nullable everything

No diseñar:

```php
final class QueryContext
{
    ?Dialect $dialect;
    ?Schema $schema;
    ?Planner $planner;
    ?Connection $connection;
    ?Transaction $transaction;
    // 50 nullable properties
}
```

---

# 211. Better approach

Typed phase contexts con invariantes de construcción.

---

# 212. Example

```php
final readonly class CompilationContext
{
    public function __construct(
        public ResolvedPlatform $platform,
        public DialectInterface $dialect,
        public CapabilitySnapshot $capabilities,
        public BindingProfile $bindingProfile,
        public CompilationPolicy $policy,
    ) {}
}
```

---

# 213. Context objects vs services

Un context puede referenciar servicios stateless requeridos por la fase.

Pero deberá hacerse con moderación.

---

# 214. Example acceptable

```text
SemanticContext
└── TypeResolverInterface
```

si el resolver es una dependencia explícita de semantic analysis.

---

# 215. Prefer direct DI when stable

Si todos los SemanticAnalyzers necesitan siempre `TypeResolver`, puede ser mejor:

```text
SemanticAnalyzer constructor
→ TypeResolver
```

en vez de colocarlo en context.

---

# 216. Context inclusion rule

Colocar en Context principalmente aquello que:

- cambia por operación;
- cambia por target;
- cambia por mode;
- cambia por policy snapshot;
- representa state/snapshot de la operación.

---

# 217. Constructor DI rule

Colocar en el servicio aquello que:

- es estable;
- stateless;
- application-scoped;
- forma parte intrínseca de la implementación.

---

# 218. Important distinction

```text
Stable Dependency
→ Constructor Injection

Operation-specific Dependency
→ Context
```

---

# 219. Example

```text
PostgreSqlCompiler
```

puede recibir sus handlers estables por constructor.

`CompilationContext` recibe:

```text
target capabilities
parameter allocator
operation policy
```

---

# 220. This prevents God Context

La regla anterior será normativa.

---

# 221. QueryContext and extensions

Las extensiones no deberán usar QueryContext como service locator.

---

# 222. Extension API

Preferir interfaces especializadas:

```text
SemanticExtensionContext
OptimizationExtensionContext
CompilerExtensionContext
```

---

# 223. Extension context visibility

Una compiler extension no necesita acceder a:

```text
SemanticAnalysisState
```

salvo artifact explícito pasado por el plan.

---

# 224. Phase boundaries

Cada fase deberá publicar artifacts estables.

---

# 225. Phase artifact rule

```text
Phase N State
```

no será leído directamente por:

```text
Phase N+2
```

---

# 226. Correct

```text
SemanticAnalysisState
→ SemanticArtifact
→ Planner
```

---

# 227. Incorrect

```text
Planner
→ reach into SemanticAnalysisState
```

---

# 228. Benefit

Esto permite:

- testing independiente;
- caching;
- replay;
- debugging;
- alternative implementations;
- parallelism futuro.

---

# 229. Context replay

Con snapshots deterministas podrá ser posible reproducir:

```text
Query + Context Fingerprints
```

para diagnostics.

---

# 230. Replay security

No deberán almacenarse secrets/binding values innecesariamente.

---

# 231. Query Context diagnostics snapshot

Puede generarse:

```text
QueryContextSummary
```

---

# 232. Summary example

```text
Processing ID: P-1042
Mode: COMPILE
Platform: PostgreSQL 18
Dialect: postgresql
Schema generation: 42
Capability fingerprint: cap:...
Policy fingerprint: pol:...
Extensions: 7
Diagnostics: enabled
```

---

# 233. No credential information

Nunca:

```text
username
password
DSN with secrets
token
certificate private key
```

---

# 234. QueryContext debug toolbar integration

Debug Toolbar puede mostrar el summary mediante adapter.

Database no dependerá de Debug Toolbar.

---

# 235. Telemetry

Puede emitirse:

```text
query.processing.started
query.processing.completed
query.processing.failed
```

---

# 236. Telemetry dimensions

Ejemplos seguros:

```text
processing mode
platform id
query origin
query intent
success/failure
phase
```

---

# 237. Avoid high-cardinality dimensions

No utilizar:

```text
raw SQL
query values
tenant ID
user ID
file path
```

como metric labels por defecto.

---

# 238. Processing timing

Podrán medirse:

```text
normalization.duration
validation.duration
semantic.duration
optimization.duration
planning.duration
compilation.duration
```

---

# 239. Telemetry state ownership

Timers/spans activos pertenecen a telemetry adapter/processing instrumentation, no al immutable Query artifact.

---

# 240. Context hooks

Evitar lifecycle hooks arbitrarios dentro de QueryContext.

---

# 241. Better

El pipeline orchestrator emite:

```text
phase started
phase completed
phase failed
```

a ports especializados.

---

# 242. Context does not orchestrate itself

`QueryContext` no será el Query Engine.

---

# 243. QueryEngine orchestration

```text
QueryEngine
   │
   ├── Context Factory
   ├── Normalizer
   ├── Validator
   ├── Semantic Analyzer
   ├── Optimizer
   ├── Planner
   └── Compiler
```

---

# 244. QueryEngine ≠ QueryContext

QueryEngine coordina.

QueryContext transporta información operation-specific.

---

# 245. No methods like

Evitar:

```php
$context->normalize();
$context->analyze();
$context->optimize();
$context->compile();
```

---

# 246. Correct direction

```php
$normalizer->normalize($query, $context);
```

---

# 247. Context API should be boring

Idealmente QueryContext será:

- pequeño;
- predecible;
- mostly readonly;
- typed;
- resource-safe.

---

# 248. Context inheritance

No se recomienda una jerarquía profunda:

```text
BaseContext
→ QueryContext
→ SemanticContext
→ PostgreSqlSemanticContext
→ ...
```

---

# 249. Prefer composition

```text
SemanticContext
├── QueryProcessingIdentity
├── QueryTargetView
├── QuerySchemaView
├── SemanticPolicy
└── DiagnosticSink
```

---

# 250. Vendor-specific context

Debe evitarse salvo extension explícita.

---

# 251. Platform-specific data

Debe viajar mediante:

```text
PlatformSnapshot
CapabilitySnapshot
PlatformExtensionData
```

tipados.

---

# 252. No vendor conditionals

Upper Query Engine no deberá hacer:

```php
if ($context->platform()->name() === 'mysql') {
    // ...
}
```

---

# 253. Correct

```text
Capability Requirement
+
Planner Strategy
+
Dialect/Platform service
```

---

# 254. Context and raw SQL

Raw SQL processing podrá usar un contexto reducido.

---

# 255. Raw path

```text
RawSqlQuery
    │
    ├── Raw validation
    ├── binding compilation
    ├── metadata resolution
    └── execution preparation
```

No requiere AST/Semantic Graph completo necesariamente.

---

# 256. Raw SQL still gets context isolation

Aunque salte etapas:

```text
Raw SQL
≠
global execution
```

---

# 257. Raw context

Puede contener:

```text
target
binding profile
security policy
metadata
capabilities
diagnostics
```

---

# 258. QueryContext and ORM

ORM puede crear Query Model y metadata.

Pero QueryContext no contiene:

```text
EntityManager
UnitOfWork
IdentityMap
Entity
Repository
```

---

# 259. ORM boundary

```text
ORM
 │
 ▼
Query Model
 │
 ▼
Query Engine
 │
 ▼
QueryContext
```

---

# 260. ORM result handling

Después de Execution:

```text
Result
→ Hydration
→ EntityManager/UoW
```

Eso ocurre fuera de QueryContext.

---

# 261. Schema system boundary

Schema queries pueden utilizar Query Engine para introspection normal.

Pero Schema AST tiene su propio context/planner/compiler donde corresponda.

---

# 262. Avoid universal context

No crear:

```text
DatabaseProcessingContext
```

que sirva para:

- Query;
- Schema;
- Migration;
- ORM;
- Transactions;
- Backup;
- Admin.

---

# 263. Context per bounded domain

Cada dominio tendrá contextos específicos.

---

# 264. Shared snapshots

Pueden compartir value objects como:

```text
CapabilitySnapshot
ResolvedPlatform
PolicySnapshot
```

sin compartir mutable processing state.

---

# 265. QueryContext and transactions

Durante compile puede conocerse:

```text
TransactionRequirement
```

pero no necesariamente si una transacción real existirá.

---

# 266. Runtime transaction validation

Executor/TransactionManager verificará:

```text
REQUIRES_EXISTING
FORBIDS_TRANSACTION
REQUIRED
```

contra estado real.

---

# 267. QueryContext and connection affinity

Puede conocer:

```text
ConnectionRequirement
```

pero no la `ConnectionLease`.

---

# 268. Transaction affinity

Se resolverá en execution.

---

# 269. QueryContext and topology

Puede contener topology-neutral requirements.

---

# 270. Avoid dynamic topology state in compile cache

Replica health cambia rápidamente y no debería invalidar compiled SQL.

---

# 271. Planning distinction

```text
Query Planning
```

no deberá confundirse con:

```text
Infrastructure Routing
```

---

# 272. Query planner

Decide estrategia de consulta.

---

# 273. Topology resolver

Decide dónde ejecutar.

---

# 274. QueryContext may express requirement

Ejemplo:

```text
requires primary
```

pero no:

```text
use host 10.0.0.17
```

---

# 275. Context and sharding

Future sharding may derive:

```text
ShardRoutingRequirement
```

from semantic information.

---

# 276. Shard routing decision

Será responsabilidad del distributed/topology layer.

---

# 277. No implicit fan-out

QueryContext no deberá disparar queries a múltiples shards.

---

# 278. QueryContext and tenant routing

Optional Multitenancy adapter puede producir:

```text
ConnectionDefinitionReference
SchemaScopeRequirement
PartitionScopeRequirement
```

---

# 279. No tenant singleton

Nunca:

```text
Tenant::current()
```

desde Query Engine.

---

# 280. Context policy assembly

La integración superior puede construir:

```text
QueryPolicySnapshot
```

usando tenant/auth/security context antes de entrar al Query Engine.

---

# 281. This preserves dependency direction

```text
Application/Integration
        │
        ▼
Policy Snapshot
        │
        ▼
Query Engine
```

No:

```text
Query Engine
        │
        ▼
Application Services
```

---

# 282. Context cloning

No deberá depender de `clone` arbitrario para cambiar fase.

---

# 283. Phase factory

Preferir:

```text
QueryContext
    │
    ▼
SemanticContextFactory
    │
    ▼
SemanticContext
```

---

# 284. Derived contexts

Deben copiar/referenciar sólo immutable snapshots necesarios.

---

# 285. Memory retention

Cuidado con contexts que mantengan referencia al AST completo cuando ya no sea necesario.

---

# 286. Release strategy

Después de cada fase se podrán liberar:

```text
temporary builders
memoization
large intermediate graphs
diagnostic traces
```

cuando no sean necesarios.

---

# 287. Streaming pipeline internally

En futuras optimizaciones, algunas fases podrían producir artifacts sin duplicar todo el árbol.

La arquitectura no deberá impedirlo.

---

# 288. But correctness first

La V1 priorizará:

```text
clear immutable artifacts
+
explicit phase state
```

sobre micro-optimizaciones.

---

# 289. Context pooling

No se recomienda reutilizar objetos `QueryContext` mutables mediante object pooling inicialmente.

---

# 290. Reason

El riesgo de state leakage supera el ahorro probable.

---

# 291. Safe reusable pieces

Sí pueden reutilizarse:

```text
immutable descriptors
frozen registries
stateless visitors
compiled rules
immutable policies
dialects
platform descriptors
```

---

# 292. Unsafe reusable pieces

No:

```text
scope stack
parameter allocator
alias allocator
diagnostic collector
semantic state
optimization memo
planning candidates
```

---

# 293. Testing strategy

El sistema requerirá:

```text
Unit Tests
Lifecycle Tests
Isolation Tests
Persistent Runtime Tests
Concurrency Tests
Budget Tests
Extension Tests
Context Validation Tests
Architecture Tests
Determinism Tests
Memory Retention Tests
```

---

# 294. Unit tests

Cada context podrá construirse sin DB real cuando sea posible.

---

# 295. Offline tests

Ejemplo:

```text
Query
+
FakeSchemaView
+
FakeCapabilitySnapshot
+
FakePolicySnapshot
→
Semantic Analysis
```

---

# 296. Determinism tests

Mismos inputs:

```text
Query
SchemaSnapshot
Capabilities
Policies
Extensions
```

deben producir mismo artifact.

---

# 297. Isolation test

```text
Context A
Context B
```

no deberán compartir:

```text
memo
diagnostics
allocators
state
```

---

# 298. Persistent worker test

Procesar:

```text
10000 query operations
```

en un mismo worker y comprobar ausencia de contaminación.

---

# 299. Concurrency test

Ejecutar múltiples contexts simultáneamente en fibers/coroutines.

---

# 300. Budget test

Crear query patológica y verificar:

```text
QueryProcessingBudgetExceededException
```

sin agotar el proceso.

---

# 301. Extension test

Una extension con state temporal deberá recibir un state diferente por QueryProcessingSession.

---

# 302. Architecture tests

QueryContext core no deberá importar:

```text
PDO
NativeConnection
ConnectionLease
EntityManager
UnitOfWork
IdentityMap
HTTP Request
AuthenticatedUser
Tenant Entity
Service Container
OpenTelemetry Span
```

---

# 303. Query context exceptions

Jerarquía conceptual:

```text
QueryContextException
├── InvalidQueryContextException
├── QueryContextClosedException
├── MissingQueryContextDependencyException
├── QueryContextCompatibilityException
├── QueryProcessingBudgetExceededException
├── QueryProcessingCancelledException
├── QueryContextExtensionException
└── QueryContextLifecycleException
```

---

# 304. Failures

Un error de QueryContext deberá indicar:

- processing phase;
- missing/incompatible requirement;
- safe diagnostic context;
- no secrets.

---

# 305. Example diagnostic

```text
Compilation context invalid.

Platform: PostgreSQL
Dialect: mysql
Reason: dialect/platform incompatibility.
```

---

# 306. No connection credentials in exception

Nunca:

```text
postgres://user:password@host/...
```

---

# 307. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Context\
```

---

# 308. Proposed structure

```text
Query/
└── Context/
    ├── Contract/
    │   ├── QueryContextInterface.php
    │   ├── QueryContextFactoryInterface.php
    │   ├── QueryContextValidatorInterface.php
    │   └── QueryContextLifecycleInterface.php
    │
    ├── Core/
    │   ├── QueryContext.php
    │   ├── QueryContextFactory.php
    │   ├── QueryContextInput.php
    │   ├── QueryProcessingId.php
    │   ├── QueryProcessingMode.php
    │   └── QueryProcessingSession.php
    │
    ├── Target/
    │   ├── QueryTarget.php
    │   ├── OfflineQueryTarget.php
    │   ├── ResolvedQueryTarget.php
    │   ├── TargetFingerprint.php
    │   └── TargetCertainty.php
    │
    ├── Schema/
    │   ├── QuerySchemaView.php
    │   ├── SchemaSnapshot.php
    │   └── SchemaGeneration.php
    │
    ├── Policy/
    │   ├── QueryPolicySnapshot.php
    │   ├── QuerySecurityPolicySnapshot.php
    │   ├── PortabilityPolicy.php
    │   └── PolicyFingerprint.php
    │
    ├── Budget/
    │   ├── QueryProcessingBudget.php
    │   ├── NormalizationBudget.php
    │   ├── SemanticBudget.php
    │   ├── OptimizationBudget.php
    │   ├── PlanningBudget.php
    │   └── CompilationBudget.php
    │
    ├── Diagnostic/
    │   ├── DiagnosticSink.php
    │   ├── NullDiagnosticSink.php
    │   ├── QueryDiagnostic.php
    │   ├── DiagnosticLevel.php
    │   └── DiagnosticCategory.php
    │
    ├── Cancellation/
    │   ├── QueryProcessingCancellation.php
    │   └── QueryProcessingDeadline.php
    │
    ├── Memo/
    │   ├── QueryProcessingMemo.php
    │   └── MemoKey.php
    │
    ├── Phase/
    │   ├── NormalizationContext.php
    │   ├── ValidationContext.php
    │   ├── SemanticAnalysisContext.php
    │   ├── OptimizationContext.php
    │   ├── PlanningContext.php
    │   └── CompilationContext.php
    │
    ├── State/
    │   ├── NormalizationState.php
    │   ├── ValidationState.php
    │   ├── SemanticAnalysisState.php
    │   ├── OptimizationState.php
    │   ├── PlanningState.php
    │   └── CompilationState.php
    │
    ├── Extension/
    │   ├── QueryContextExtensionDescriptor.php
    │   ├── QueryContextExtensionRegistry.php
    │   ├── QueryContextExtensionFactory.php
    │   └── QueryContextExtensionState.php
    │
    ├── Lifecycle/
    │   ├── QueryContextLifecycle.php
    │   ├── QueryContextState.php
    │   └── QueryContextCleanup.php
    │
    ├── Fingerprint/
    │   ├── QueryProcessingFingerprint.php
    │   └── QueryProcessingFingerprintFactory.php
    │
    ├── Diagnostics/
    │   ├── QueryContextSummary.php
    │   └── QueryContextDiagnosticRenderer.php
    │
    └── Exception/
        ├── QueryContextException.php
        ├── InvalidQueryContextException.php
        ├── QueryContextClosedException.php
        ├── QueryContextCompatibilityException.php
        ├── QueryProcessingBudgetExceededException.php
        └── QueryProcessingCancelledException.php
```

---

# 309. Dependency direction

```text
Query Model / AST
        │
        ▼
Query Context
        │
        ├── Metadata Snapshot
        ├── Schema View
        ├── Capability Snapshot
        └── Policy Snapshot
        │
        ▼
Phase Contexts
        │
        ▼
Query Engine Stages
        │
        ▼
Artifacts
```

---

# 310. Forbidden dependency direction

Nunca:

```text
QueryContext
→ EntityManager
→ Query Builder
```

---

# 311. Forbidden runtime dependency

Nunca:

```text
QueryContext
→ ConnectionLease
→ NativeConnection
```

---

# 312. Allowed target dependency

Sí:

```text
QueryContext
→ ResolvedPlatform
→ CapabilitySnapshot
→ Dialect descriptor/service
```

porque son parte del entorno de procesamiento.

---

# 313. Dialect caution

Si Dialect es un servicio stateless compartido, puede referenciarse.

Nunca deberá contener state de compilation.

---

# 314. Context vs registry

Frozen registries pueden ser dependencies de servicios.

No es obligatorio poner todos los registries dentro de QueryContext.

---

# 315. Registry rule

```text
Stable registry
→ service constructor

operation-specific registry view
→ context
```

---

# 316. Context vs configuration

No almacenar el árbol completo de configuración de Database.

---

# 317. Policy compilation

Transformar configuración relevante en:

```text
typed policy snapshots
```

antes de QueryContext.

---

# 318. Context minimalism

El objetivo no es que QueryContext tenga toda la información disponible.

El objetivo es que tenga la información **necesaria**.

---

# 319. God Context anti-pattern

Incorrecto:

```text
QueryContext
├── Container
├── Config
├── Logger
├── Cache
├── Events
├── HTTP Request
├── User
├── Tenant
├── EntityManager
├── UnitOfWork
├── Connection
├── Transaction
├── Dialect
├── Platform
├── Driver
├── Query
└── Result
```

---

# 320. Correct model

```text
QueryContext
├── Processing Identity
├── Target Snapshot
├── Schema View
├── Metadata
├── Policy Snapshot
├── Capability Snapshot
├── Budgets
└── Diagnostics
```

---

# 321. Context minimality test

Para cada propiedad candidata preguntar:

```text
Does this vary per query-processing operation?
```

Si no:

```text
probably constructor dependency
```

Después preguntar:

```text
Is this a runtime resource?
```

Si sí:

```text
probably ExecutionContext / DatabaseContext
```

---

# 322. Third question

```text
Is this derived output of a phase?
```

Si sí:

```text
probably PhaseArtifact
```

no QueryContext.

---

# 323. Fourth question

```text
Is this mutable progress inside a phase?
```

Si sí:

```text
PhaseState
```

---

# 324. Decision matrix

| Información | Ubicación |
|---|---|
| Dialect service | Compiler dependency / target |
| Capability snapshot | Query/Phase Context |
| Current parameter index | CompilationState |
| Current alias index | CompilationState |
| Resolved symbols | SemanticArtifact |
| Symbol builder | SemanticState |
| Schema view | SemanticContext |
| Current connection | ExecutionContext |
| Transaction requirement | Metadata/Plan |
| Active transaction | TransactionContext |
| Query label | QueryMetadata |
| Current telemetry span | Processing instrumentation |
| Type registry | Analyzer dependency |
| Effective type table | SemanticArtifact |
| Optimizer rule registry | Optimizer dependency |
| Optimizer iteration | OptimizationState |
| Planner candidates | PlanningState |
| Compiled SQL | CompiledQuery |

---

# 325. QueryContext public exposure

El contexto completo será principalmente Internal/Extension API.

---

# 326. Public application API

Las aplicaciones normalmente no deberían crear `QueryContext` manualmente.

---

# 327. Public Query API

El developer utiliza:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

---

# 328. Framework internals

VoltStack crea:

```text
Query Model
AST
QueryContext
SemanticArtifact
Plan
CompiledQuery
ExecutionContext
```

automáticamente.

---

# 329. Advanced extension API

Extensiones podrán recibir context views especializados.

---

# 330. No direct context mutation

Extension API no deberá permitir:

```php
$context->setPlatform(...);
$context->setSchema(...);
$context->setCapabilities(...);
```

---

# 331. Extension outputs

Las extensiones deberán retornar:

- transformation;
- annotation;
- requirement;
- diagnostic;
- strategy candidate;

mediante contratos explícitos.

---

# 332. Context security

QueryContext será tratado como infraestructura sensible.

---

# 333. Information minimization

Una extension sólo deberá recibir la vista mínima necesaria.

---

# 334. Example

Compiler extension no necesita:

```text
SourceLocation
```

si no realiza diagnostics.

---

# 335. Capability access

Preferir typed capability checks.

---

# 336. Example

```php
if (!$context->capabilities()->supports($requirement)) {
    // fail / fallback
}
```

No:

```php
if ($context->platform()->name() === 'postgresql') {
}
```

---

# 337. Platform-specific extension

Si realmente es PostgreSQL-specific:

```text
PostgreSqlCompilerExtension
```

se registra mediante compatibility descriptor.

---

# 338. Context compatibility

La extension podrá declarar:

```text
required platform
required capability
required dialect
required compiler version
```

---

# 339. Fail before execution

Incompatibilidades deberán detectarse durante context/extension validation.

---

# 340. Query context lifecycle events

Internamente podrán existir:

```text
QueryProcessingStarted
QueryPhaseStarted
QueryPhaseCompleted
QueryPhaseFailed
QueryProcessingCompleted
QueryProcessingFailed
```

---

# 341. Events optional

No depender obligatoriamente de:

```text
Quantum/EventSystem
```

---

# 342. Internal hooks

Puede utilizarse un port ligero:

```text
QueryProcessingObserver
```

---

# 343. Observer constraints

Observers:

- no mutan QueryContext;
- no cambian query semantics;
- no bloquean cleanup;
- respetan redaction.

---

# 344. Security observers

Políticas que sí alteran/rechazan queries no son simples observers.

Son:

```text
Query Policies / Security Policies
```

---

# 345. Context lifecycle and exceptions

Si un observer falla:

```text
cleanup must still run
```

---

# 346. Failure policy

Observability failures normalmente no deberán impedir procesamiento salvo configuración explícita.

Security-critical policy failures sí pueden hacerlo.

---

# 347. Context and cache

QueryContext puede consultar un cache semántico mediante servicios externos del pipeline, pero no deberá convertirse en CacheManager.

---

# 348. Better architecture

```text
SemanticAnalyzer
├── SemanticCache
└── analyze(query, context)
```

en vez de:

```text
context->cache()->get(...)
```

---

# 349. Same for telemetry

Preferir:

```text
InstrumentedSemanticAnalyzer
```

o orchestrator instrumentation.

No:

```text
context->telemetry()->startSpan(...)
```

como dependencia universal.

---

# 350. Same for events

No:

```text
context->events()->dispatch(...)
```

desde cualquier node visitor.

---

# 351. Same for container

Nunca:

```text
context->container()
```

---

# 352. Context as data, not infrastructure gateway

Éste será un principio normativo.

---

# 353. DB-QCTX-001

`QueryContext` será operation-scoped.

---

# 354. DB-QCTX-002

`QueryContext` no será request-scoped por definición.

---

# 355. DB-QCTX-003

Una request podrá crear múltiples QueryContexts.

---

# 356. DB-QCTX-004

`QueryContext` será distinto de `DatabaseContext`.

---

# 357. DB-QCTX-005

`QueryContext` será distinto de `ExecutionContext`.

---

# 358. DB-QCTX-006

`QueryContext` será distinto de `TransactionContext`.

---

# 359. DB-QCTX-007

`QueryContext` será distinto de `QueryMetadata`.

---

# 360. DB-QCTX-008

`QueryContext` no contendrá PDO.

---

# 361. DB-QCTX-009

`QueryContext` no contendrá NativeConnection.

---

# 362. DB-QCTX-010

`QueryContext` no contendrá ConnectionLease.

---

# 363. DB-QCTX-011

`QueryContext` no contendrá ResultCursor.

---

# 364. DB-QCTX-012

`QueryContext` no contendrá EntityManager.

---

# 365. DB-QCTX-013

`QueryContext` no contendrá UnitOfWork.

---

# 366. DB-QCTX-014

`QueryContext` no contendrá IdentityMap.

---

# 367. DB-QCTX-015

`QueryContext` no contendrá HTTP Request.

---

# 368. DB-QCTX-016

`QueryContext` no contendrá current user object.

---

# 369. DB-QCTX-017

`QueryContext` no contendrá Tenant entity.

---

# 370. DB-QCTX-018

`QueryContext` no expondrá Service Container.

---

# 371. DB-QCTX-019

No existirá un `QueryContext::$current` global mutable.

---

# 372. DB-QCTX-020

Context propagation será explícita por defecto.

---

# 373. DB-QCTX-021

El contexto base será principalmente immutable.

---

# 374. DB-QCTX-022

Mutable processing progress vivirá en PhaseState.

---

# 375. DB-QCTX-023

Stable environment vivirá en PhaseContext.

---

# 376. DB-QCTX-024

PhaseState no sobrevivirá accidentalmente a su fase.

---

# 377. DB-QCTX-025

Phase outputs serán artifacts explícitos.

---

# 378. DB-QCTX-026

Una fase posterior no accederá directamente al mutable state de una fase anterior.

---

# 379. DB-QCTX-027

CapabilitySnapshot será estable durante una operación.

---

# 380. DB-QCTX-028

SchemaSnapshot será estable durante una operación.

---

# 381. DB-QCTX-029

PolicySnapshot será estable durante una operación.

---

# 382. DB-QCTX-030

El Query Engine no realizará I/O oculto mediante QueryContext.

---

# 383. DB-QCTX-031

Offline compilation será una capacidad arquitectónica.

---

# 384. DB-QCTX-032

Offline semantic analysis será posible cuando exista schema metadata suficiente.

---

# 385. DB-QCTX-033

QueryTarget no será physical connection.

---

# 386. DB-QCTX-034

QueryTarget será representado mediante snapshots/descriptors.

---

# 387. DB-QCTX-035

Target-specific artifacts tendrán fingerprints apropiados.

---

# 388. DB-QCTX-036

Runtime topology health no contaminará compiled query fingerprints.

---

# 389. DB-QCTX-037

QueryProcessingId será distinto de QueryInstanceId.

---

# 390. DB-QCTX-038

QueryProcessingId será distinto de ExecutionId.

---

# 391. DB-QCTX-039

ParameterAllocator será operation-local.

---

# 392. DB-QCTX-040

AliasAllocator será operation-local.

---

# 393. DB-QCTX-041

Shared Dialects serán stateless.

---

# 394. DB-QCTX-042

Shared Platform services serán stateless/immutable.

---

# 395. DB-QCTX-043

Shared registries estarán frozen.

---

# 396. DB-QCTX-044

QueryContext extension state será scoped.

---

# 397. DB-QCTX-045

Extension context descriptors se registrarán durante bootstrap.

---

# 398. DB-QCTX-046

Runtime descriptor registration no será soportado por defecto.

---

# 399. DB-QCTX-047

Extension state no será arbitrary untyped object storage.

---

# 400. DB-QCTX-048

QueryContext tendrá lifecycle explícito.

---

# 401. DB-QCTX-049

Cleanup será determinista.

---

# 402. DB-QCTX-050

Cleanup será idempotente.

---

# 403. DB-QCTX-051

Failure durante processing no evitará cleanup.

---

# 404. DB-QCTX-052

QueryContext cleanup no realizará transaction rollback salvo que explícitamente posea un recurso transaccional, lo cual el diseño base prohíbe.

---

# 405. DB-QCTX-053

QueryContext cleanup no liberará ConnectionLease porque no la posee.

---

# 406. DB-QCTX-054

Processing budgets serán explícitos.

---

# 407. DB-QCTX-055

Budgets protegerán contra complejidad patológica.

---

# 408. DB-QCTX-056

Processing timeout será distinto de SQL statement timeout.

---

# 409. DB-QCTX-057

Cancellation no producirá artifacts incompletos considerados válidos.

---

# 410. DB-QCTX-058

Diagnostic state será operation-local.

---

# 411. DB-QCTX-059

Diagnostics podrán desactivarse con overhead mínimo.

---

# 412. DB-QCTX-060

Diagnostic output estará redacted por defecto.

---

# 413. DB-QCTX-061

Memoization local no será cache global.

---

# 414. DB-QCTX-062

Memoization local será liberada al finalizar el contexto.

---

# 415. DB-QCTX-063

Promoción a cache compartido requerirá un componente explícito.

---

# 416. DB-QCTX-064

QueryContext no será QueryEngine.

---

# 417. DB-QCTX-065

QueryContext no orquestará las fases mediante métodos de negocio.

---

# 418. DB-QCTX-066

QueryEngine será responsable de la orquestación.

---

# 419. DB-QCTX-067

Context objects deberán ser pequeños y typed.

---

# 420. DB-QCTX-068

Se evitarán contextos con grandes cantidades de propiedades nullable.

---

# 421. DB-QCTX-069

Se preferirá composición sobre herencia profunda.

---

# 422. DB-QCTX-070

No existirán contextos específicos por vendor salvo extensión explícita justificada.

---

# 423. DB-QCTX-071

Upper Query Engine consultará capabilities, no nombres de vendor.

---

# 424. DB-QCTX-072

QueryContext no decidirá routing físico.

---

# 425. DB-QCTX-073

QueryContext no seleccionará replica concreta.

---

# 426. DB-QCTX-074

QueryContext no iniciará transacciones.

---

# 427. DB-QCTX-075

QueryContext no ejecutará queries.

---

# 428. DB-QCTX-076

QueryContext no hidratará entities.

---

# 429. DB-QCTX-077

QueryContext no resolverá application services.

---

# 430. DB-QCTX-078

QueryContext no leerá environment variables.

---

# 431. DB-QCTX-079

QueryContext no accederá a globals de configuración.

---

# 432. DB-QCTX-080

QueryContext será seguro bajo persistent workers.

---

# 433. DB-QCTX-081

QueryContext será seguro bajo fibers/coroutines.

---

# 434. DB-QCTX-082

Dos operaciones concurrentes no compartirán mutable phase state.

---

# 435. DB-QCTX-083

Una operación no observará state temporal de otra.

---

# 436. DB-QCTX-084

QueryContext deberá ser testeable sin servidor DB cuando el modo lo permita.

---

# 437. DB-QCTX-085

Context validation deberá fallar antes de iniciar una fase incompatible.

---

# 438. DB-QCTX-086

Missing required target information producirá error explícito.

---

# 439. DB-QCTX-087

Platform/Dialect incompatibility producirá error explícito.

---

# 440. DB-QCTX-088

UNKNOWN capabilities se tratarán conservadoramente.

---

# 441. DB-QCTX-089

Security policies no podrán ser debilitadas por extensions.

---

# 442. DB-QCTX-090

Extension context access seguirá least privilege.

---

# 443. DB-QCTX-091

QueryContext no será usado como event bus.

---

# 444. DB-QCTX-092

QueryContext no será usado como telemetry manager.

---

# 445. DB-QCTX-093

QueryContext no será usado como cache manager.

---

# 446. DB-QCTX-094

QueryContext no será usado como logger.

---

# 447. DB-QCTX-095

Stable dependencies irán por constructor injection.

---

# 448. DB-QCTX-096

Operation-specific dependencies irán por context.

---

# 449. DB-QCTX-097

Derived outputs serán artifacts, no context properties mutadas globalmente.

---

# 450. DB-QCTX-098

Mutable phase progress será explícitamente owned.

---

# 451. DB-QCTX-099

El common path evitará reflection y container lookup.

---

# 452. DB-QCTX-100

El diseño deberá priorizar correctness e isolation sobre object pooling prematuro.

---

# 453. Anti-pattern — God Context

```text
QueryContext
├── everything
└── everyone depends on it
```

Esto equivale a un Service Locator.

---

# 454. Anti-pattern — global current context

```php
QueryContext::current();
```

como dependencia principal del Query Engine.

---

# 455. Anti-pattern — runtime resources

```text
QueryContext
├── PDO
├── Transaction
└── Result
```

---

# 456. Anti-pattern — mutable dialect

```text
Dialect
└── current parameter index
```

---

# 457. Anti-pattern — mutable platform

```text
Platform
└── current schema/current tenant/current query
```

---

# 458. Anti-pattern — hidden schema I/O

```text
SemanticAnalyzer
→ context
→ silently queries information_schema
```

---

# 459. Anti-pattern — hidden application dependency

```text
QueryContext
→ Auth::user()
```

---

# 460. Anti-pattern — hidden tenant dependency

```text
QueryContext
→ Tenant::current()
```

---

# 461. Anti-pattern — context inheritance explosion

```text
BaseContext
  → SqlContext
    → QueryContext
      → SemanticContext
        → PostgreSqlSemanticContext
```

---

# 462. Anti-pattern — context as mutable result bag

```text
$context->setAst(...)
$context->setSemanticGraph(...)
$context->setPlan(...)
$context->setSql(...)
$context->setResult(...)
```

---

# 463. Correct model

```text
Input Artifact
+
Phase Context
+
Phase State
→
Output Artifact
```

---

# 464. Query processing formula

```text
Query Processing
=
Immutable Input Artifacts
+
Stable Operation Context
+
Explicit Temporary Phase State
+
Deterministic Transformations
+
Explicit Output Artifacts
```

---

# 465. QueryContext formula

```text
QueryContext
=
Processing Identity
+
Target Snapshot
+
Capability Snapshot
+
Schema View
+
Policy Snapshot
+
Query Metadata
+
Budgets
+
Diagnostics
+
Scoped Extension Data
```

---

# 466. QueryContext exclusion formula

```text
QueryContext
≠
Connection
+
Transaction
+
EntityManager
+
UnitOfWork
+
Request
+
User
+
Tenant Entity
+
Service Container
+
Result
```

---

# 467. Phase formula

```text
Phase Artifact
=
Input Artifact
+
Phase Context
+
Phase State
+
Phase Algorithm
```

---

# 468. Persistent runtime formula

```text
Persistent-Safe Query Processing
=
Immutable Shared Services
+
Frozen Registries
+
Operation-Scoped QueryContext
+
Phase-Scoped Mutable State
+
Explicit Ownership
+
Deterministic Cleanup
+
No Global Current State
```

---

# 469. Offline compilation formula

```text
CompiledQuery
=
Query Artifact
+
Resolved Offline Target
+
Schema Snapshot
+
Capability Snapshot
+
Policy Snapshot
+
Compiler
```

No requiere:

```text
Physical Connection
```

---

# 470. Runtime execution formula

Posteriormente:

```text
Result
=
CompiledQuery
+
ExecutionContext
+
ConnectionLease
+
Driver
```

---

# 471. Context boundary

La arquitectura completa queda:

```text
Application
    │
    ▼
Query Builder
    │
    ▼
Query Model
    │
    ▼
Query AST
    │
    ▼
┌─────────────────────────────┐
│ Query Processing Boundary   │
│                             │
│ QueryContext                │
│   │                         │
│   ├── NormalizationContext  │
│   ├── ValidationContext     │
│   ├── SemanticContext       │
│   ├── OptimizationContext   │
│   ├── PlanningContext       │
│   └── CompilationContext    │
└──────────────┬──────────────┘
               │
               ▼
         CompiledQuery
               │
               ▼
┌─────────────────────────────┐
│ Execution Boundary          │
│                             │
│ ExecutionContext            │
│ TransactionContext          │
│ ConnectionLease             │
└──────────────┬──────────────┘
               │
               ▼
             Driver
               │
               ▼
            Database
```

---

# 472. Relationship with previous documents

```text
23_DATABASE_QUERY_ARCHITECTURE.md
        │
        ▼
24_DATABASE_QUERY_MODEL.md
        │
        ▼
25_DATABASE_QUERY_AST_SYSTEM.md
        │
        ▼
26_DATABASE_QUERY_AST_NODE_MODEL.md
        │
        ├── 27 Expressions
        ├── 28 Predicates
        ├── 29 Parameters / Bindings
        ├── 30 Query Types
        ├── 31 Query Metadata
        └── 32 Query Context
```

Con esto queda definida la infraestructura fundamental necesaria para entrar a:

```text
Normalization
Validation
Semantic Analysis
```

---

# 473. Architectural checkpoint

Hasta este punto VoltStack distingue formalmente:

```text
Query Model
=
developer-oriented structured representation

Query AST
=
canonical structural representation

Query Expressions
=
value-producing semantic structures

Query Predicates
=
truth-producing semantic structures

Query Parameters
=
runtime value references

Query Types
=
semantic value/type model

Query Metadata
=
declarative requirements/provenance/policies

Query Context
=
temporary processing environment
```

---

# 474. Master invariant

La siguiente separación será permanente:

```text
Query Artifact
      │
      ├── Structure
      └── Metadata
      │
      ▼
QueryContext
      │
      ├── Processing Environment
      └── Temporary Phase State
      │
      ▼
Semantic / Plan / Compiled Artifacts
      │
      ▼
ExecutionContext
      │
      └── Runtime Resources
```

---

# 475. Master rule

> `QueryContext` deberá proporcionar a cada fase exactamente el contexto temporal que necesita para procesar una consulta, sin convertirse en propietario de recursos de ejecución ni en puerta de acceso universal al framework.

---

# 476. Regla de diseño

Ante cualquier nueva propiedad propuesta para `QueryContext`, deberán formularse cuatro preguntas:

```text
1. ¿Es estable y compartida?
   → Constructor dependency.

2. ¿Varía por operación de procesamiento?
   → Context.

3. ¿Es progreso mutable de una fase?
   → PhaseState.

4. ¿Es un resultado derivado?
   → PhaseArtifact.
```

Y una quinta:

```text
5. ¿Es un recurso runtime?
   → ExecutionContext / DatabaseContext / TransactionContext.
```

---

# 477. Resultado arquitectónico

VoltStack obtiene un modelo donde:

```text
shared services
```

pueden sobrevivir durante todo el worker;

```text
DatabaseContext
```

vive durante una ejecución del framework;

```text
QueryContext
```

vive durante el procesamiento de una consulta;

```text
PhaseState
```

vive durante una fase;

y:

```text
ExecutionContext
```

posee los recursos utilizados para ejecutar el `CompiledQuery`.

Esto permite que el Query Engine sea:

- determinista;
- testeable;
- portable;
- reentrant;
- concurrent-safe;
- compatible con persistent workers;
- extensible;
- observable;
- optimizable;
- independiente del runtime HTTP.

---

# 478. Próximo documento

El siguiente documento será:

```text
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
```

y deberá formalizar la transformación:

```text
Query Model / AST
        │
        ▼
Non-Canonical Query
        │
        ▼
Normalization Rules
        │
        ▼
Canonical Query Representation
```

incluyendo:

```text
canonical forms
expression normalization
predicate normalization
parameter normalization
identifier normalization
boolean normalization
comparison normalization
join normalization
subquery normalization
deterministic ordering
rewrite convergence
normalization budgets
extension rules
fingerprinting
idempotence
```

La propiedad fundamental será:

```text
normalize(normalize(Q))
=
normalize(Q)
```

es decir, la normalización deberá ser **determinista, convergente e idempotente**, sin modificar el significado semántico de la consulta.