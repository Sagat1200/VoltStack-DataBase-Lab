# 67_DATABASE_SQL_COMPILER_PIPELINE.md

# VoltStack Quantum Database
## SQL Compiler Pipeline

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 67 — SQL Compiler Pipeline  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`SQL Compiler Pipeline` define el proceso completo, ordenado, determinista y verificable mediante el cual VoltStack transforma una:

```text
CompilableDatabaseOperation
```

en:

```text
CompiledDatabaseCommand
```

El documento anterior definió la arquitectura general:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
```

Este documento define ahora:

- fases;
- passes;
- artifacts intermedios;
- contratos entre fases;
- validaciones;
- resolución de compiler;
- SQL lowering;
- adaptación de dialecto;
- asignación de aliases;
- planificación de placeholders;
- generación SQL;
- binding layout;
- result contract;
- dependencies;
- fingerprints;
- source maps;
- diagnostics;
- budgets;
- extensiones;
- failure semantics;
- aislamiento para persistent runtimes.

El principio maestro será:

```text
Compilation Pipeline
=
Ordered Transformation Pipeline
```

y no:

```text
Compilation Pipeline
=
Collection of callbacks mutating SQL strings
```

---

# 2. Objetivo

El pipeline deberá garantizar que:

```text
Valid Structured Operation
        │
        ▼
Deterministic Compilation
        │
        ▼
Valid Target-Specific Representation
```

manteniendo:

```text
semantic preservation
security preservation
parameter safety
capability compliance
result-shape preservation
determinism
traceability
bounded resource usage
```

---

# 3. Fórmula principal

```text
CompiledDatabaseCommand
=
Pipeline(
    CompilableDatabaseOperation,
    SqlCompilationContext
)
```

donde:

```text
Pipeline
=
P0 Input Acceptance
+
P1 Pre-Compilation Validation
+
P2 Compiler Resolution
+
P3 Structural SQL Lowering
+
P4 Dialect Adaptation
+
P5 Representation Validation
+
P6 Alias Planning
+
P7 Placeholder Planning
+
P8 SQL Generation
+
P9 Binding Layout Compilation
+
P10 Result Contract Compilation
+
P11 Dependency Collection
+
P12 Fingerprint Construction
+
P13 Post-Compilation Validation
+
P14 Artifact Finalization
```

---

# 4. Pipeline completo

```text
CompilableDatabaseOperation
            │
            ▼
┌─────────────────────────────┐
│ P0 Input Acceptance         │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P1 Pre-Compilation          │
│    Validation               │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P2 Compiler Resolution      │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P3 Structural SQL Lowering  │
└──────────────┬──────────────┘
               ▼
       SqlEmissionTree
               │
               ▼
┌─────────────────────────────┐
│ P4 Dialect Adaptation       │
└──────────────┬──────────────┘
               ▼
       AdaptedSqlTree
               │
               ▼
┌─────────────────────────────┐
│ P5 Representation           │
│    Validation               │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P6 Alias Planning           │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P7 Placeholder Planning     │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P8 SQL Generation           │
└──────────────┬──────────────┘
               ▼
          RenderedSql
               │
        ┌──────┴───────┐
        ▼              ▼
     SQL text      SqlSourceMap
        │              │
        └──────┬───────┘
               ▼
┌─────────────────────────────┐
│ P9 Binding Layout           │
│    Compilation              │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P10 Result Contract         │
│     Compilation             │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P11 Dependency Collection   │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P12 Fingerprint             │
│     Construction            │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P13 Post-Compilation        │
│     Validation              │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ P14 Artifact Finalization   │
└──────────────┬──────────────┘
               ▼
   CompiledDatabaseCommand
```

---

# 5. Pipeline ≠ Query lifecycle

No debe confundirse con el lifecycle completo:

```text
Build
→ Normalize
→ Validate
→ Semantic Analysis
→ Optimize
→ Logical Plan
→ Physical Plan
→ Execution Plan
→ Compile
→ Execute
```

Este documento cubre exclusivamente:

```text
Compile
```

---

# 6. Pipeline boundary

La frontera de entrada es:

```text
CompilableDatabaseOperation
```

La frontera de salida es:

```text
CompiledDatabaseCommand
```

---

# 7. Precondición

El pipeline asume que la operación ya tiene:

```text
valid semantics
resolved symbols
resolved types
resolved relations
validated scopes
selected logical strategy
selected physical strategy
execution requirements
```

según corresponda.

---

# 8. Lo que el pipeline no hace

Está prohibido que el pipeline vuelva a ejecutar:

```text
query normalization
semantic resolution
type inference
constraint discovery
predicate optimization
join optimization
cardinality estimation
logical planning
physical planning
execution scheduling
query execution
```

---

# 9. Modelo de fases

Cada fase deberá definirse mediante:

```text
Input Artifact
+
Context
+
Phase Contract
+
Output Artifact
+
Diagnostics
+
Budget Accounting
```

---

# 10. Phase contract

Contrato conceptual:

```php
interface SqlCompilationPhase
{
    public function id(): SqlCompilationPhaseId;

    public function execute(
        SqlCompilationArtifact $input,
        SqlCompilationSession $session,
    ): SqlCompilationArtifact;
}
```

No todas las fases necesariamente compartirán literalmente esta interfaz, pero deberán respetar el mismo modelo arquitectónico.

---

# 11. PhaseId

Ejemplos:

```text
input_acceptance
pre_validation
compiler_resolution
structural_lowering
dialect_adaptation
representation_validation
alias_planning
placeholder_planning
sql_generation
binding_layout
result_contract
dependency_collection
fingerprint
post_validation
finalization
```

---

# 12. Pipeline Coordinator

El componente central será:

```php
interface SqlCompilerPipeline
{
    public function compile(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): CompiledDatabaseCommand;
}
```

---

# 13. Coordinator responsibility

Debe:

```text
create compilation session
execute phases
enforce ordering
track budget
collect diagnostics
manage artifacts
handle failure
finalize output
```

No deberá implementar directamente toda la compilación.

---

# 14. SqlCompilationSession

```php
final class SqlCompilationSession
{
    public function __construct(
        public readonly SqlCompilationContext $context,
        public readonly SqlCompilationId $id,
        public readonly SqlCompilationBudgetTracker $budget,
        public readonly SqlCompilationDiagnosticCollector $diagnostics,
        public readonly SqlCompilationTraceCollector $trace,
        public readonly SqlAliasAllocator $aliases,
        public readonly SqlPlaceholderAllocator $placeholders,
        public readonly SqlSourceMapBuilder $sourceMap,
        public readonly CompilationDependencyCollector $dependencies,
    ) {}
}
```

---

# 15. Session lifecycle

```text
create
  │
  ▼
compile exactly one operation
  │
  ▼
finalize
  │
  ▼
discard
```

---

# 16. Session isolation

Nunca:

```text
Session A
   ↓
mutable state
   ↓
Session B
```

---

# 17. Artifact progression

Se recomienda que el pipeline no pase un único objeto mutable.

Preferir artifacts explícitos:

```text
AcceptedCompilationInput
        ↓
ValidatedCompilationInput
        ↓
ResolvedCompilerArtifact
        ↓
LoweredSqlArtifact
        ↓
DialectAdaptedSqlArtifact
        ↓
RepresentationValidatedArtifact
        ↓
AliasPlannedSqlArtifact
        ↓
PlaceholderPlannedSqlArtifact
        ↓
RenderedSqlArtifact
        ↓
BindingCompiledArtifact
        ↓
ResultContractCompiledArtifact
        ↓
DependencyResolvedArtifact
        ↓
FingerprintResolvedArtifact
        ↓
ValidatedCompiledArtifact
        ↓
CompiledDatabaseCommand
```

---

# 18. Por qué artifacts explícitos

Evita estados ambiguos como:

```php
$context->sqlMaybeGenerated = true;
$context->bindingsMaybeReady = false;
```

---

# 19. Artifact invariants

Cada artifact certifica:

```text
all previous phases completed successfully
```

---

# 20. Phase ordering

El orden será declarado.

No dependerá de:

```text
service registration order
array insertion order
plugin load accident
```

---

# 21. Pipeline graph

Aunque V1 será principalmente lineal, arquitectónicamente podrá modelarse como DAG.

```text
             ┌───────────────┐
             │ Input         │
             └───────┬───────┘
                     ▼
              Pre-Validation
                     │
                     ▼
             Compiler Resolve
                     │
                     ▼
                 Lowering
                     │
                     ▼
            Dialect Adaptation
                     │
                     ▼
          Representation Validate
                     │
              ┌──────┴──────┐
              ▼             ▼
          Alias Plan    Dependency
              │          Discovery
              ▼             │
       Placeholder Plan     │
              │             │
              ▼             │
           Rendering        │
              │             │
       ┌──────┴──────┐      │
       ▼             ▼      │
    Bindings       Result    │
       │          Contract   │
       └──────┬──────┘      │
              ▼             │
          Dependencies ◄────┘
              │
              ▼
          Fingerprint
              │
              ▼
        Post-Validation
              │
              ▼
          Finalization
```

---

# 22. P0 — Input Acceptance

Primera fase:

```text
P0_INPUT_ACCEPTANCE
```

---

# 23. Responsabilidad

Verificar que la entrada pertenece al dominio del Compiler.

---

# 24. Checks

```text
operation is not null
operation kind known
operation artifact finalized
required IDs available
operation immutable/frozen
execution requirements readable
parameter definitions available
```

---

# 25. Input acceptance ≠ query validation

No valida si:

```text
users.email exists
SUM() is legal
JOIN aliases resolve
```

---

# 26. AcceptedCompilationInput

```php
final readonly class AcceptedCompilationInput
{
    public function __construct(
        public CompilableDatabaseOperation $operation,
        public SqlCompilationContext $context,
    ) {}
}
```

---

# 27. Invalid lifecycle state

Si llega una operación no finalizada:

```text
UnfinalizedCompilableOperationException
```

---

# 28. P1 — Pre-Compilation Validation

```text
P1_PRE_COMPILATION_VALIDATION
```

---

# 29. Objetivo

Determinar:

> ¿La operación contiene toda la información requerida para iniciar compilación contra este target?

---

# 30. Validaciones

Puede comprobar:

```text
target platform defined
dialect defined
capability snapshot present
operation kind supported
required physical properties present
parameter model valid
required compiler extensions available
required capabilities declared
security requirements represented
```

---

# 31. Capability validation

Ejemplo:

```text
Operation:
DELETE + RETURNING

Requires:
DELETE_RETURNING
```

Context:

```text
DELETE_RETURNING = false
```

Resultado:

```text
CompilationCapabilityException
```

si no existe estrategia previamente planificada.

---

# 32. Capability validation ≠ fallback planning

Nunca:

```text
unsupported feature
→ compiler invents another strategy
```

---

# 33. ValidatedCompilationInput

```php
final readonly class ValidatedCompilationInput
{
    public function __construct(
        public CompilableDatabaseOperation $operation,
        public CompilationRequirementSet $requirements,
        public ValidatedCapabilitySet $capabilities,
    ) {}
}
```

---

# 34. P2 — Compiler Resolution

```text
P2_COMPILER_RESOLUTION
```

---

# 35. Propósito

Seleccionar los componentes correctos.

---

# 36. Ejemplo

```text
SELECT operation
      │
      ▼
SelectSqlCompiler
```

---

# 37. Resolver

```php
interface SqlStatementCompilerResolver
{
    public function resolve(
        CompilableDatabaseOperation $operation,
        SqlCompilationContext $context,
    ): QueryStatementCompiler;
}
```

---

# 38. Selection inputs

Puede considerar:

```text
operation kind
semantic construct kind
platform capabilities
dialect
registered extensions
compiler version
```

---

# 39. No registration-order selection

Si dos compilers reclaman el mismo constructo sin prioridad/regla explícita:

```text
AmbiguousSqlCompilerException
```

---

# 40. ResolvedCompilerArtifact

```php
final readonly class ResolvedCompilerArtifact
{
    public function __construct(
        public ValidatedCompilationInput $input,
        public QueryStatementCompiler $statementCompiler,
        public SqlCompilerComponentSet $components,
    ) {}
}
```

---

# 41. Component set

Puede incluir:

```text
expression compiler
predicate compiler
relation compiler
join compiler
CTE compiler
aggregate compiler
window compiler
set-operation compiler
identifier compiler
function compiler registry
operator compiler registry
extension compiler registry
```

---

# 42. P3 — Structural SQL Lowering

```text
P3_STRUCTURAL_SQL_LOWERING
```

---

# 43. Propósito

Transformar:

```text
CompilableDatabaseOperation
```

en:

```text
SqlEmissionTree
```

---

# 44. Lowering formula

```text
Lower(
    CompilableDatabaseOperation
)
=
SqlEmissionTree
```

---

# 45. Lowering conserva significado

```text
Meaning(Operation)
=
Meaning(SqlEmissionTree)
```

dentro de la representación SQL objetivo.

---

# 46. Lowering example

Entrada:

```text
SelectOperation
├── projection
├── relation
├── predicate
├── ordering
└── limit
```

Salida:

```text
SqlSelectStatement
├── SqlProjectionClause
├── SqlFromClause
├── SqlWhereClause
├── SqlOrderByClause
└── SqlLimitClause
```

---

# 47. Lowering no produce necesariamente texto

No:

```text
Lower()
→ "SELECT ..."
```

Preferir:

```text
Lower()
→ SqlEmissionTree
```

---

# 48. SqlEmissionTree

```php
final readonly class SqlEmissionTree
{
    public function __construct(
        public SqlStatementNode $root,
        public SqlEmissionMetadata $metadata,
    ) {}
}
```

---

# 49. Recursive lowering

Ejemplo:

```text
Select
  ↓
Projection
  ↓
Expressions
```

Cada componente puede delegar:

```text
Statement Compiler
      │
      ├── Relation Compiler
      ├── Expression Compiler
      ├── Predicate Compiler
      └── Window Compiler
```

---

# 50. Lowering context

```php
final readonly class SqlLoweringContext
{
    public function __construct(
        public SqlCompilationContext $compilation,
        public SqlCompilerComponentSet $components,
        public SqlCompilationBudgetTracker $budget,
    ) {}
}
```

---

# 51. Lowering no resuelve symbols

Recibe:

```text
ResolvedSymbolId
ResolvedRelationId
ResolvedFunctionId
ResolvedType
```

cuando sean necesarios.

---

# 52. No semantic fallback

Si falta información semántica obligatoria:

```text
IncompleteCompilationArtifactException
```

No:

```text
guess from column name
```

---

# 53. LoweredSqlArtifact

```php
final readonly class LoweredSqlArtifact
{
    public function __construct(
        public SqlEmissionTree $tree,
        public LoweringMetadata $metadata,
    ) {}
}
```

---

# 54. P4 — Dialect Adaptation

```text
P4_DIALECT_ADAPTATION
```

---

# 55. Propósito

Transformar una representación SQL estructurada común en una forma representable por el dialecto concreto.

---

# 56. Formula

```text
DialectAdapt(
    SqlEmissionTree,
    Dialect,
    Capabilities
)
→
DialectAdaptedSqlTree
```

---

# 57. Dialect adaptation examples

```text
identifier quoting strategy
boolean literal representation
pagination representation
upsert syntax representation
RETURNING syntax
locking syntax
JSON operator syntax
function spelling
cast syntax
```

---

# 58. Adaptation ≠ optimization

No:

```text
LEFT JOIN → INNER JOIN
```

Sí:

```text
same semantic function
→ dialect-specific spelling
```

---

# 59. Adaptation ≠ physical planning

No:

```text
choose index
choose hash join
choose nested loop
```

---

# 60. Representational rewrite

Sólo se permiten transformaciones que satisfagan:

```text
same semantics
+
same already-selected strategy
+
target-specific representation
```

---

# 61. RepresentationalRewriteRule

```php
interface SqlRepresentationalRewriteRule
{
    public function supports(
        SqlEmissionNode $node,
        SqlDialect $dialect,
    ): bool;

    public function rewrite(
        SqlEmissionNode $node,
        SqlDialectAdaptationContext $context,
    ): SqlEmissionNode;
}
```

---

# 62. Rules registry

Debe ser:

```text
typed
versioned
validated
frozen
deterministic
```

---

# 63. Rewrite safety

Cada rule deberá declarar:

```text
target node kinds
required capabilities
supported dialects
semantic preservation contract
extension ownership
```

---

# 64. Adapted artifact

```php
final readonly class DialectAdaptedSqlArtifact
{
    public function __construct(
        public SqlEmissionTree $tree,
        public DialectAdaptationMetadata $metadata,
    ) {}
}
```

---

# 65. P5 — Representation Validation

```text
P5_REPRESENTATION_VALIDATION
```

---

# 66. Propósito

Comprobar que el árbol adaptado ya puede representarse válidamente en el target.

---

# 67. Checks

```text
no unsupported SQL emission nodes
all functions compilable
all operators compilable
all required clauses supported
all extensions have compiler
no unresolved representation placeholders
identifier forms valid
dialect requirements satisfied
```

---

# 68. Why validate here

Porque:

```text
lowering
+
dialect adaptation
```

pueden revelar requisitos que no eran visibles en P1.

---

# 69. No partial fallback

Si falla:

```text
abort compilation
```

No continuar con SQL incompleto.

---

# 70. RepresentationValidatedArtifact

```php
final readonly class RepresentationValidatedArtifact
{
    public function __construct(
        public SqlEmissionTree $tree,
        public RepresentationValidationReport $report,
    ) {}
}
```

---

# 71. P6 — Alias Planning

```text
P6_ALIAS_PLANNING
```

---

# 72. Propósito

Resolver nombres SQL internos necesarios para representar:

```text
relations
derived tables
subqueries
CTEs
output columns
internal projections
```

---

# 73. Alias planning ≠ symbol resolution

Symbol resolution ya ocurrió.

Alias planning responde:

> ¿Qué nombre textual seguro y único tendrá esta identidad semántica dentro del SQL generado?

---

# 74. AliasMap

```php
final readonly class SqlAliasMap
{
    public function __construct(
        public array $relationAliases,
        public array $columnAliases,
        public array $generatedAliases,
    ) {}
}
```

---

# 75. Alias identity

```text
SemanticRelationId
      │
      ▼
SqlAliasMap
      │
      ▼
SqlAlias
```

---

# 76. User aliases

Cuando el usuario especificó un alias válido y observable:

```text
preserve if possible
```

---

# 77. Generated aliases

Si se necesita uno:

```text
deterministic allocator
```

---

# 78. Collision detection

Debe considerar:

```text
user aliases
generated aliases
CTE names
derived relation names
reserved identifiers where applicable
```

---

# 79. Alias allocator

No utilizar:

```text
random()
uniqid()
object hash
global counter
```

---

# 80. Deterministic example

```text
Relation R14
→ __vs_r1

Derived relation R19
→ __vs_r2
```

para una traversal estable.

---

# 81. Alias scope

Aliases pueden ser scoped.

Un alias válido dentro de subquery no necesariamente pertenece al outer query.

---

# 82. AliasPlanningContext

Debe conocer scopes SQL de representación, no volver a crear scopes semánticos.

---

# 83. Alias planned artifact

```php
final readonly class AliasPlannedSqlArtifact
{
    public function __construct(
        public SqlEmissionTree $tree,
        public SqlAliasMap $aliases,
    ) {}
}
```

---

# 84. P7 — Placeholder Planning

```text
P7_PLACEHOLDER_PLANNING
```

---

# 85. Propósito

Mapear:

```text
ParameterOccurrence
```

a:

```text
SqlPlaceholder
```

---

# 86. Distinción

```text
ParameterId
≠
ParameterOccurrenceId
≠
SqlPlaceholderId
≠
RenderedPlaceholder
```

---

# 87. Parameter occurrence discovery

El pipeline recorrerá la representación en orden determinista.

Ejemplo:

```text
WHERE a = P1
  AND b = P2
  OR c = P1
```

Occurrences:

```text
PO1 → P1
PO2 → P2
PO3 → P1
```

---

# 88. Placeholder style

Dependiendo del target:

```text
?
$1
$2
:name
```

---

# 89. Example numbered

```text
PO1 → $1 → P1
PO2 → $2 → P2
PO3 → $3 → P1
```

---

# 90. Example reusable named

Si el driver lo permite:

```text
PO1 → :p1 → P1
PO2 → :p2 → P2
PO3 → :p1 → P1
```

---

# 91. Reuse capability

Nunca asumir que un placeholder puede reutilizarse.

Será capability/driver-contract driven.

---

# 92. PlaceholderPlan

```php
final readonly class SqlPlaceholderPlan
{
    public function __construct(
        public PlaceholderStyle $style,
        public ParameterOccurrenceList $occurrences,
        public PlaceholderAssignmentMap $assignments,
    ) {}
}
```

---

# 93. Collection parameters

Una collection parameter puede necesitar expansión:

```text
P1 = [10, 20, 30]
```

pero el Compiler no deberá inspeccionar runtime values ordinarios para decidir arbitrariamente.

---

# 94. Collection strategy

Debe haber sido determinada mediante:

```text
physical/execution strategy
```

o mediante especialización explícita.

---

# 95. Example expansion plan

```text
CollectionParameter P1
Cardinality specialization = 3

→ P1.1
→ P1.2
→ P1.3
```

---

# 96. Specialization requirement

Si el número de placeholders depende del tamaño runtime:

```text
compile-time specialization required
```

o debe usarse otra estrategia.

---

# 97. Empty collections

No deberán resolverse mediante SQL string hacks.

La semántica deberá estar ya definida estructuralmente.

---

# 98. Placeholder limits

La fase comprobará límites de plataforma/driver cuando estén disponibles.

---

# 99. Placeholder planned artifact

```php
final readonly class PlaceholderPlannedSqlArtifact
{
    public function __construct(
        public SqlEmissionTree $tree,
        public SqlAliasMap $aliases,
        public SqlPlaceholderPlan $placeholders,
    ) {}
}
```

---

# 100. P8 — SQL Generation

```text
P8_SQL_GENERATION
```

---

# 101. Propósito

Convertir:

```text
structured SQL representation
+
aliases
+
placeholder plan
```

en:

```text
SQL text
+
source map
```

---

# 102. SQL generation formula

```text
Render(
    SqlEmissionTree,
    SqlAliasMap,
    SqlPlaceholderPlan,
    SqlDialect
)
→
RenderedSql
```

---

# 103. SQL Generation ≠ lowering

Aquí ya no deberían tomarse decisiones semánticas.

---

# 104. SqlRenderer

```php
interface SqlRenderer
{
    public function render(
        PlaceholderPlannedSqlArtifact $artifact,
        SqlRenderingContext $context,
    ): RenderedSqlArtifact;
}
```

---

# 105. Rendering traversal

Debe ser:

```text
stable
deterministic
bounded
```

---

# 106. SQL text builder

El renderer puede utilizar:

```text
SqlTextBuilder
```

para evitar concatenación ineficiente.

---

# 107. Rendering modes

Posibles:

```text
CANONICAL
PRETTY
DEBUG
```

---

# 108. Canonical mode

Será la forma recomendada para:

```text
prepared statement caching
compiled query caching
fingerprints based on rendered representation
deterministic tests
```

---

# 109. Pretty mode

Para:

```text
debugging
developer tools
logs
explain output
```

---

# 110. Debug mode

Puede incluir comentarios internos sólo si:

```text
explicitly enabled
safe
non-sensitive
not used as canonical production command
```

---

# 111. Source map generation

Mientras se emite SQL:

```text
sourceMap.begin(node)
append(...)
sourceMap.end(node)
```

conceptualmente.

---

# 112. Source map levels

Configurables:

```text
NONE
STATEMENT
CLAUSE
NODE
DETAILED
```

---

# 113. Production overhead

En producción puede utilizarse:

```text
STATEMENT
```

o:

```text
CLAUSE
```

según configuración.

---

# 114. RenderedSqlArtifact

```php
final readonly class RenderedSqlArtifact
{
    public function __construct(
        public string $sql,
        public SqlSourceMap $sourceMap,
        public SqlRenderingMetadata $metadata,
    ) {}
}
```

---

# 115. SQL length accounting

Cada append puede contabilizar:

```text
rendered bytes
rendered characters
nodes rendered
nesting depth
```

---

# 116. Oversized SQL

Debe abortar con:

```text
SqlCompilationBudgetExceededException
```

antes de consumir memoria ilimitada.

---

# 117. P9 — Binding Layout Compilation

```text
P9_BINDING_LAYOUT_COMPILATION
```

---

# 118. Propósito

Transformar el placeholder plan en un contrato ejecutable para binding.

---

# 119. Input

```text
Parameter Definitions
+
Parameter Occurrences
+
Placeholder Assignments
+
Query Types
+
Platform Binding Rules
```

---

# 120. Output

```text
CompiledBindingLayout
```

---

# 121. CompiledBindingSlot

```php
final readonly class CompiledBindingSlot
{
    public function __construct(
        public BindingSlotId $id,
        public ParameterId $parameter,
        public SqlPlaceholder $placeholder,
        public DriverBindingType $bindingType,
        public BindingConversionPlan $conversion,
        public SensitivityClassification $sensitivity,
    ) {}
}
```

---

# 122. Binding layout no contiene valor

No:

```php
new CompiledBindingSlot(
    value: 'john@example.com'
);
```

---

# 123. Runtime binding

Ocurrirá después:

```text
CompiledBindingLayout
+
BindingSet
        │
        ▼
Runtime Binder
```

---

# 124. Type mapping

Ejemplo:

```text
QueryType:
Domain<UserId>

Platform representation:
BIGINT

Driver binding:
INTEGER / STRING according to safe range/platform
```

---

# 125. Typed NULL

NULL deberá conservar información suficiente cuando el driver/plataforma lo requiera.

---

# 126. Binding conversion

Ejemplos:

```text
DateTimeImmutable
→ platform datetime representation

UUID
→ textual/binary representation

Enum
→ mapped scalar

JSON value
→ encoded representation
```

---

# 127. Encoding errors

No deben aparecer durante SQL generation.

Deben manejarse en binding/conversion runtime cuando dependan del valor.

---

# 128. BindingCompiledArtifact

```php
final readonly class BindingCompiledArtifact
{
    public function __construct(
        public RenderedSqlArtifact $sql,
        public CompiledBindingLayout $bindings,
    ) {}
}
```

---

# 129. P10 — Result Contract Compilation

```text
P10_RESULT_CONTRACT_COMPILATION
```

---

# 130. Propósito

Describir el resultado esperado del comando SQL.

---

# 131. SELECT result

Puede describir:

```text
column count
column identities
driver labels
semantic output IDs
database representation types
nullability expectations
```

---

# 132. DML result

Puede describir:

```text
affected row count
RETURNING shape
generated values
command status
```

---

# 133. CompiledResultContract

```php
final readonly class CompiledResultContract
{
    public function __construct(
        public ResultKind $kind,
        public CompiledResultColumnList $columns,
        public AffectedRowsContract $affectedRows,
        public ReturningContract $returning,
    ) {}
}
```

---

# 134. Result contract ≠ ORM hydration

No contiene:

```text
User entity constructor
EntityManager
IdentityMap
UnitOfWork
```

---

# 135. Output aliases

El result contract deberá mapear:

```text
SemanticOutputId
↔
InternalSqlAlias
↔
DriverColumnLabel
```

---

# 136. Duplicate names

Ejemplo:

```sql
SELECT users.id, orders.id
```

puede necesitar:

```text
users.id  → __vs_c1
orders.id → __vs_c2
```

internamente.

---

# 137. User-visible metadata

El mapping deberá permitir recuperar nombres semánticos originales.

---

# 138. ResultContractCompiledArtifact

```php
final readonly class ResultContractCompiledArtifact
{
    public function __construct(
        public BindingCompiledArtifact $compiled,
        public CompiledResultContract $resultContract,
    ) {}
}
```

---

# 139. P11 — Dependency Collection

```text
P11_DEPENDENCY_COLLECTION
```

---

# 140. Propósito

Registrar todo aquello cuya modificación podría invalidar el compiled artifact.

---

# 141. Dependency categories

```text
platform
platform version where relevant
dialect
capabilities
compiler version
compiler configuration
extension versions
schema objects
semantic functions
custom types
custom operators
physical strategy
execution strategy
specialization inputs
```

---

# 142. CompilationDependency

```php
interface CompilationDependency
{
    public function type(): CompilationDependencyType;

    public function fingerprint(): DependencyFingerprint;
}
```

---

# 143. Schema dependency precision

Preferir:

```text
Table(users)
Column(users.id)
Column(users.email)
```

sobre:

```text
EntireDatabaseSchema
```

cuando sea seguro.

---

# 144. Capability dependency

Ejemplo:

```text
supportsReturning = true
```

puede participar si determinó la representación.

---

# 145. Compiler version

El cambio de reglas de rendering puede invalidar compiled cache.

---

# 146. Extension version

También.

---

# 147. Configuration dependency

Ejemplos:

```text
placeholder style
canonical rendering version
identifier normalization policy
compiler feature flags
```

---

# 148. Pretty mode dependency

Si Pretty SQL no se usa como cache artifact, puede excluirse del fingerprint canónico.

---

# 149. DependencyResolvedArtifact

```php
final readonly class DependencyResolvedArtifact
{
    public function __construct(
        public ResultContractCompiledArtifact $compiled,
        public CompilationDependencySet $dependencies,
    ) {}
}
```

---

# 150. P12 — Fingerprint Construction

```text
P12_FINGERPRINT_CONSTRUCTION
```

---

# 151. Propósito

Construir una identidad determinista del artifact compilado.

---

# 152. Fingerprint inputs

```text
operation fingerprint
target platform fingerprint
dialect fingerprint
capability fingerprint
compiler version
compiler configuration
extension versions
representation dependencies
specialization fingerprint
```

---

# 153. Formula

```text
CompiledQueryFingerprint
=
Hash(
    DomainTag
    + OperationFingerprint
    + TargetFingerprint
    + CompilerFingerprint
    + DependencyFingerprint
    + SpecializationFingerprint
)
```

---

# 154. Domain separation

No reutilizar un hash ambiguo.

Ejemplo conceptual:

```text
voltstack.database.compiled-query.v1
```

---

# 155. Runtime values

No forman parte por defecto.

---

# 156. Example

Estas dos ejecuciones:

```text
email = alice@example.com
email = bob@example.com
```

deberían poder compartir el mismo compiled artifact si sólo cambia `BindingSet`.

---

# 157. Specialization exception

Si:

```text
IN collection
```

fue especializada a 5 placeholders:

```text
specialization cardinality = 5
```

deberá formar parte del fingerprint.

---

# 158. Fingerprint ≠ SQL hash

Aunque SQL text pueda participar, no deberá ser necesariamente:

```text
hash($sql)
```

la única identidad.

---

# 159. Why

Dos artifacts con SQL textual similar pueden depender de:

```text
different binding types
different result contracts
different capabilities
different extensions
```

---

# 160. FingerprintResolvedArtifact

```php
final readonly class FingerprintResolvedArtifact
{
    public function __construct(
        public DependencyResolvedArtifact $compiled,
        public CompiledQueryFingerprint $fingerprint,
    ) {}
}
```

---

# 161. P13 — Post-Compilation Validation

```text
P13_POST_COMPILATION_VALIDATION
```

---

# 162. Propósito

Verificar invariantes finales antes de entregar el comando al Execution Engine.

---

# 163. Final checks

```text
SQL not empty
all placeholders mapped
all binding slots valid
no orphan bindings
all identifiers rendered
no unresolved SQL nodes
result contract complete
dependency set complete
fingerprint present
security requirements preserved
budget not exceeded
```

---

# 164. Placeholder completeness

Debe cumplirse:

```text
∀ PlaceholderOccurrence
∃ BindingSlot
```

cuando corresponda a runtime binding.

---

# 165. Binding completeness

Y:

```text
∀ BindingSlot
∃ ValidParameterDefinition
```

---

# 166. Orphan parameter

Un parameter definition no utilizado puede permitirse o diagnosticarse según origen, pero:

```text
orphan placeholder
```

no.

---

# 167. Security verification

Si la operación declaraba:

```text
RequiredSecurityPolicySet
```

el compiled artifact deberá demostrar preservación.

---

# 168. Security proof token

Puede utilizarse:

```text
CompilationSecurityPreservationRecord
```

para registrar qué estructuras de seguridad llegaron a SQL.

---

# 169. No SQL reparsing

Post-validation no deberá parsear nuevamente el SQL generado para reconstruir semántica.

Debe usar:

```text
source maps
emission metadata
binding maps
compilation records
```

---

# 170. Optional SQL parser testing

Un parser SQL puede utilizarse en:

```text
tests
debug verification
```

pero no será requisito arquitectónico del production pipeline.

---

# 171. ValidatedCompiledArtifact

```php
final readonly class ValidatedCompiledArtifact
{
    public function __construct(
        public FingerprintResolvedArtifact $artifact,
        public PostCompilationValidationReport $report,
    ) {}
}
```

---

# 172. P14 — Artifact Finalization

```text
P14_ARTIFACT_FINALIZATION
```

---

# 173. Propósito

Construir:

```text
CompiledDatabaseCommand
```

como artifact final inmutable.

---

# 174. Final artifact

```php
final readonly class CompiledDatabaseCommand
{
    public function __construct(
        public CompiledDatabaseCommandId $id,
        public DatabaseOperationKind $kind,
        public RenderedSql $sql,
        public CompiledBindingLayout $bindings,
        public CompiledResultContract $result,
        public CompiledCommandRequirementSet $requirements,
        public CompilationDependencySet $dependencies,
        public CompiledQueryFingerprint $fingerprint,
        public CompilationMetadata $metadata,
    ) {}
}
```

---

# 175. Finalization

Una vez finalizado:

```text
no mutation
```

---

# 176. Builders discarded

Objetos como:

```text
SqlTextBuilder
SqlSourceMapBuilder
DependencyCollector
DiagnosticCollector
PlaceholderAllocator
AliasAllocator
```

no forman parte mutable del artifact final.

---

# 177. Pipeline success

Resultado:

```text
CompilationSuccess
    │
    ▼
CompiledDatabaseCommand
```

---

# 178. Pipeline failure

Resultado:

```text
CompilationFailure
```

sin comando parcialmente ejecutable.

---

# 179. Atomic compilation

Regla:

```text
Compilation
=
all-or-nothing
```

---

# 180. No partial command

Nunca devolver:

```text
SQL partially generated
+
bindings incomplete
+
warning
```

como comando ejecutable.

---

# 181. Failure model

Errores pueden ocurrir en:

```text
input acceptance
pre-validation
compiler resolution
lowering
dialect adaptation
representation validation
alias planning
placeholder planning
rendering
binding compilation
result contract
dependency collection
fingerprint
post-validation
finalization
```

---

# 182. Phase-specific exceptions

Ejemplos:

```text
SqlCompilationInputException
SqlCompilationValidationException
SqlCompilerResolutionException
SqlLoweringException
SqlDialectAdaptationException
SqlRepresentationValidationException
SqlAliasPlanningException
SqlPlaceholderPlanningException
SqlRenderingException
SqlBindingCompilationException
SqlResultContractCompilationException
SqlDependencyResolutionException
SqlFingerprintException
SqlPostCompilationValidationException
```

---

# 183. Unified root

Todas derivarán de:

```text
SqlCompilationException
```

---

# 184. Failure record

```php
final readonly class SqlCompilationFailure
{
    public function __construct(
        public SqlCompilationPhaseId $phase,
        public SqlCompilationDiagnosticList $diagnostics,
        public ?Throwable $cause,
    ) {}
}
```

---

# 185. Exception vs result

La API pública puede utilizar exceptions.

Internamente, fases pueden utilizar:

```text
PhaseResult
```

para diagnostics más ricos.

---

# 186. No swallowed failure

Una extensión no deberá hacer:

```php
try {
    ...
} catch (\Throwable) {
    return '';
}
```

---

# 187. Recoverable diagnostics

Algunas situaciones pueden ser:

```text
INFO
WARNING
```

si no afectan correctness.

---

# 188. Fatal diagnostics

Ejemplos:

```text
unsupported capability
unresolved extension
invalid placeholder plan
security requirement lost
budget exceeded
```

---

# 189. Severity

```php
enum CompilationDiagnosticSeverity
{
    case INFO;
    case WARNING;
    case ERROR;
    case FATAL;
}
```

---

# 190. Pipeline trace

Cada fase podrá producir:

```text
CompilationPhaseTrace
```

---

# 191. Trace record

```php
final readonly class CompilationPhaseTrace
{
    public function __construct(
        public SqlCompilationPhaseId $phase,
        public int $inputNodeCount,
        public int $outputNodeCount,
        public CompilationWorkUnits $work,
        public DiagnosticCount $diagnostics,
    ) {}
}
```

---

# 192. Time telemetry

Duración puede medirse externamente.

No deberá formar parte del deterministic artifact fingerprint.

---

# 193. Trace data

Puede incluir:

```text
compiler selected
rules applied
capabilities consulted
extensions used
aliases generated
placeholders generated
SQL bytes emitted
dependencies collected
```

---

# 194. No sensitive trace

No incluir:

```text
password
token
secret
PII runtime values
```

---

# 195. Budget architecture

El budget será transversal.

---

# 196. SqlCompilationBudget

```php
final readonly class SqlCompilationBudget
{
    public function __construct(
        public int $maxVisitedNodes,
        public int $maxEmissionNodes,
        public int $maxNestingDepth,
        public int $maxAliases,
        public int $maxPlaceholders,
        public int $maxSqlBytes,
        public int $maxSourceMapEntries,
        public int $maxExtensionExpansions,
        public int $maxRepresentationalRewrites,
    ) {}
}
```

---

# 197. Work units

Preferir unidades deterministas:

```text
nodes visited
nodes emitted
rewrites applied
placeholders allocated
bytes rendered
```

sobre:

```text
milliseconds
```

como límite principal.

---

# 198. Why

Tiempo depende de:

```text
CPU
machine load
runtime
environment
```

y reduce determinismo.

---

# 199. Time limits

Pueden existir como defensa externa adicional, pero no sustituyen work budgets.

---

# 200. Budget checking

Cada fase consume del mismo budget global y, opcionalmente, de budgets por fase.

---

# 201. Example

```text
Global budget: 100,000 work units

Lowering:       18,000
Adaptation:      4,000
Alias planning:  2,000
Placeholder:     3,000
Rendering:      25,000
...
```

---

# 202. Budget exhaustion semantics

```text
budget exhausted
→ fail compilation
```

No:

```text
skip clause
```

---

# 203. Extension budget

Las extensiones estarán sujetas al mismo budget.

---

# 204. No unlimited extension expansion

Una extensión no puede expandir:

```text
1 node
→ 10 million nodes
```

sin límites.

---

# 205. Extension participation

Las extensiones podrán participar sólo en phases declaradas.

---

# 206. ExtensionPhase

```text
PRE_VALIDATION
LOWERING
DIALECT_ADAPTATION
REPRESENTATION_VALIDATION
RENDERING
BINDING
RESULT_CONTRACT
DEPENDENCY_COLLECTION
POST_VALIDATION
```

---

# 207. No arbitrary hooks

Evitar:

```php
beforeEverything(callable $callback)
afterAnything(callable $callback)
```

---

# 208. Typed extension points

Preferir:

```text
SqlExpressionCompilerExtension
SqlPredicateCompilerExtension
SqlFunctionCompilerExtension
SqlDialectAdaptationExtension
SqlBindingCompilerExtension
SqlResultContractExtension
```

---

# 209. Extension ordering

Debe estar determinado por:

```text
dependencies
phase
priority where explicitly allowed
stable ExtensionId tie-break
```

---

# 210. Extension cycles

Si existen:

```text
A before B
B before A
```

bootstrap deberá fallar.

---

# 211. Extension registration lifecycle

```text
discover
→ validate
→ resolve dependencies
→ build phase graph
→ freeze
```

---

# 212. No hot registration

En persistent workers:

```text
active compilation
+
registry mutation
```

estará prohibido.

---

# 213. Phase dependencies

Ejemplo:

```text
PlaceholderPlanning
requires
AliasPlanning
```

si placeholders dependen de traversal final.

---

# 214. Explicit phase DAG

```php
interface SqlCompilationPhaseDescriptor
{
    public function id(): SqlCompilationPhaseId;

    public function requires(): array;

    public function runsBefore(): array;

    public function runsAfter(): array;
}
```

---

# 215. Core phases fixed

Las extensiones no podrán reordenar arbitrariamente invariantes core.

---

# 216. Example forbidden extension

```text
Render SQL
before
Dialect Adaptation
```

no deberá permitirse.

---

# 217. Pipeline version

El pipeline tendrá versión:

```text
SqlCompilationPipelineVersion
```

---

# 218. Why

Cambios en:

```text
phase ordering
canonical rendering
placeholder allocation
dialect adaptation
```

pueden afectar artifacts compilados.

---

# 219. Pipeline version fingerprint

Participará cuando sea relevante en:

```text
CompiledQueryFingerprint
```

---

# 220. Pipeline profiles

Posibles perfiles:

```text
PRODUCTION
DEBUG
VALIDATION_HEAVY
```

---

# 221. Profiles cannot change semantics

Sólo podrán alterar:

```text
diagnostic detail
trace detail
source-map detail
additional invariant checks
```

---

# 222. Validation-heavy

Puede habilitar:

```text
additional internal assertions
artifact consistency checks
extension contract checks
```

---

# 223. Production

Puede reducir:

```text
trace
source-map granularity
debug metadata
```

sin omitir validaciones de correctness.

---

# 224. Compiler context immutability

El `SqlCompilationContext` deberá ser:

```text
immutable snapshot
```

---

# 225. Context contents

Puede incluir:

```text
platform
dialect
capability snapshot
compiler configuration
pipeline profile
extension set
compilation budget
platform type mappings
rendering policy
```

---

# 226. Context must not contain

```text
live PDO
current ResultSet
EntityManager
current HTTP Request
mutable tenant global
mutable transaction
```

---

# 227. Context fingerprint

Las partes relevantes pueden producir:

```text
SqlCompilationContextFingerprint
```

---

# 228. Snapshot consistency

Una compilación no deberá comenzar con:

```text
capabilities version A
```

y terminar con:

```text
capabilities version B
```

---

# 229. Target consistency

```text
Platform
Dialect
Capabilities
Driver contract
```

deberán ser compatibles.

---

# 230. Driver contract

Aunque Compiler no depende de driver runtime, puede necesitar un:

```text
DriverCompilationContract
```

inmutable.

---

# 231. Example driver compilation information

```text
placeholder style
supports repeated named placeholders
maximum parameter count
binding type capabilities
prepared statement restrictions
```

---

# 232. DriverCompilationContract ≠ Driver

No contiene:

```text
connect()
execute()
fetch()
```

---

# 233. Cross-driver differences

Una misma plataforma podría tener drivers con distintas restricciones de binding.

El Compiler puede recibir esas restricciones como contrato.

---

# 234. Pipeline and Prepared Statements

El pipeline produce información suficiente para:

```text
prepare
+
bind
```

pero no realiza ninguno.

---

# 235. Pipeline and query cache

No ejecuta result cache.

---

# 236. Pipeline and compiled cache

Antes de compilar, un nivel superior podría consultar:

```text
Compiled Query Cache
```

y evitar ejecutar el pipeline completo.

---

# 237. Cache flow

```text
Execution Unit
     │
     ▼
Compilation Cache Key Candidate
     │
     ▼
Compiled Query Cache
     │
 ┌───┴────┐
 │        │
hit      miss
 │        │
 ▼        ▼
reuse   Compiler Pipeline
           │
           ▼
        store
```

El sistema completo se define en documento 75.

---

# 238. Cache lookup circularity

El cache key preliminar no puede depender únicamente del fingerprint final si éste sólo existe después de compilar.

---

# 239. Two-level identity

Podrá existir:

```text
CompilationLookupKey
```

antes del pipeline y:

```text
CompiledQueryFingerprint
```

después.

---

# 240. CompilationLookupKey

Puede basarse en:

```text
operation fingerprint
platform
dialect
capabilities
compiler version
configuration
extensions
specialization
```

---

# 241. Final fingerprint

Además podrá incorporar dependencies descubiertas durante compilation.

---

# 242. Pipeline and telemetry

Eventos conceptuales:

```text
SqlCompilationStarted
SqlCompilationPhaseStarted
SqlCompilationPhaseCompleted
SqlCompilationCompleted
SqlCompilationFailed
```

---

# 243. Core independence

El Compiler no dependerá directamente del Telemetry package.

---

# 244. Event sink abstraction

Puede usar:

```text
CompilationObserver
```

o integración externa.

---

# 245. No observer mutation

Observers no deberán poder modificar artifacts.

---

# 246. Observer failure

Un observer de telemetría no debería invalidar una compilación correcta salvo configuración explícita de infraestructura crítica.

---

# 247. Pipeline and security

Security se preserva mediante estructuras ya presentes.

---

# 248. Security checkpoints

Recomendados:

```text
P1  verify required security metadata exists
P3  preserve security-tagged nodes
P4  preserve during adaptation
P8  map security nodes in source map
P13 verify required security constructs emitted
```

---

# 249. Security preservation ledger

Puede existir:

```php
final class SecurityCompilationLedger
{
    public function recordInput(...): void;

    public function recordLowered(...): void;

    public function recordRendered(...): void;

    public function verify(): SecurityPreservationReport;
}
```

---

# 250. Ledger is operation-scoped

Nunca global.

---

# 251. Security policy identity

Debe preservarse por:

```text
SecurityPolicyId
```

no por comparar SQL strings.

---

# 252. Multitenancy

Igualmente:

```text
TenantPolicyId
```

puede conservar provenance sin que Compiler dependa del paquete Multitenancy.

---

# 253. Pipeline and raw SQL

Raw nodes pasarán por:

```text
RawSqlCompilationPolicy
```

---

# 254. Raw validation

Verificará:

```text
allowed context
trust classification
parameter model
security restrictions
platform restriction
```

---

# 255. Raw does not bypass budget

Raw SQL largo cuenta contra:

```text
maxSqlBytes
```

---

# 256. Raw does not bypass source map

Al menos deberá mapearse como:

```text
RawExpressionNode
```

---

# 257. Raw does not bypass fingerprint

Su contenido estructural deberá participar en fingerprints relevantes.

---

# 258. Raw SQL and secrets

Raw fragments con secretos embebidos deberán ser rechazados o marcados como no seguros según policy; la API nunca debe promover ese patrón.

---

# 259. Pipeline and DML

INSERT, UPDATE y DELETE recorren el mismo pipeline general.

---

# 260. No second DML compiler pipeline

No:

```text
SelectPipeline
InsertPipeline
UpdatePipeline
DeletePipeline
```

completamente independientes.

---

# 261. Shared pipeline

```text
SQL Compiler Pipeline
        │
        ├── Select statement compiler
        ├── Insert statement compiler
        ├── Update statement compiler
        └── Delete statement compiler
```

---

# 262. Statement-specific phases

Una fase podrá delegar según statement kind.

---

# 263. DDL future

Schema/DDL podrá utilizar:

```text
same compiler infrastructure
```

con artifacts especializados.

---

# 264. DDL semantic differences

No significa que Query AST y Schema AST sean iguales.

Sólo comparten:

```text
compiler pipeline infrastructure
rendering
dialect
identifiers
capabilities
diagnostics
```

---

# 265. Pipeline and batches

Un `ExecutionPlan` puede contener múltiples operaciones compilables.

---

# 266. Compilation unit

Cada:

```text
CompilableDatabaseOperation
```

se compila de forma independiente salvo que exista un artifact batch explícito.

---

# 267. Multi-statement SQL

No será el default.

---

# 268. Why

Multi-statement strings complican:

```text
security
binding
driver portability
result contracts
error mapping
prepared statements
```

---

# 269. Explicit batch

Si se soporta:

```text
CompiledCommandBatch
```

será un concepto explícito.

---

# 270. Batch ≠ SQL concatenation

No:

```php
$sql = $sql1 . ';' . $sql2;
```

como arquitectura general.

---

# 271. Transaction semantics

El Compiler no inserta:

```sql
BEGIN;
COMMIT;
```

alrededor de comandos por iniciativa propia.

---

# 272. Transaction orchestration

Pertenece a:

```text
Transaction Manager
+
Execution Plan
+
Execution Engine
```

---

# 273. Savepoints

Misma regla.

---

# 274. Compilation concurrency

El mismo pipeline service podrá utilizarse concurrentemente si:

```text
shared services immutable
session state isolated
```

---

# 275. Reentrancy

Los compiler components deberán ser:

```text
stateless
```

o utilizar exclusivamente session/context explícito.

---

# 276. No mutable component fields

Evitar:

```php
final class SelectSqlCompiler
{
    private int $currentParameter = 0;
}
```

si la instancia se comparte.

---

# 277. Correct model

```php
final class SelectSqlCompiler
{
    public function compile(
        SelectOperation $operation,
        SqlCompilationSession $session,
    ): SqlEmissionNode {
        // operation-scoped state comes from session
    }
}
```

---

# 278. Persistent runtime

FrankenPHP:

```text
Worker starts
   │
   ▼
Compiler Registry built
   │
   ▼
Registry frozen
   │
   ├── Request A → Session A → destroy
   ├── Request B → Session B → destroy
   ├── Job C     → Session C → destroy
   └── Request D → Session D → destroy
```

---

# 279. RoadRunner/OpenSwoole

La misma arquitectura será válida.

---

# 280. No request assumptions

El pipeline puede ejecutarse desde:

```text
HTTP
CLI
Queue Worker
Scheduler
Test
Migration
Background Task
```

---

# 281. Memory management

Artifacts intermedios grandes deberán liberarse tan pronto como ya no sean necesarios.

---

# 282. Artifact retention profile

Puede existir:

```text
MINIMAL
DIAGNOSTIC
DEBUG
```

---

# 283. Production retention

No es necesario conservar todos los artifacts intermedios después de producir el comando.

---

# 284. Debug retention

Puede conservar references/summaries para explain tooling.

---

# 285. Avoid artifact duplication

Cuando los artifacts sean inmutables podrán compartir subestructuras.

---

# 286. Persistent data structures

Son opcionales, no requisito de V1.

---

# 287. Memory budget

Además de work budget podrá existir:

```text
approximate structural memory budget
```

si se puede medir de forma determinista razonable.

---

# 288. Phase metrics

Por fase:

```text
input nodes
output nodes
work units
diagnostics
extensions invoked
rewrites
```

---

# 289. Compiler diagnostics example

```text
Compilation ID:
C-0187

Platform:
PostgreSQL

Pipeline:
v1

P0 Input Acceptance
OK

P1 Pre-Validation
OK

P2 Compiler Resolution
SelectSqlCompiler

P3 Structural Lowering
38 nodes → 41 SQL nodes

P4 Dialect Adaptation
2 representational adaptations

P5 Representation Validation
OK

P6 Alias Planning
3 aliases

P7 Placeholder Planning
4 occurrences / 3 logical parameters

P8 SQL Generation
324 bytes

P9 Binding Layout
4 slots

P10 Result Contract
5 columns

P11 Dependencies
12 dependencies

P12 Fingerprint
vsdb:compiled:v1:...

P13 Post Validation
OK

P14 Finalization
SUCCESS
```

---

# 290. Explain pipeline

Developer tools podrán mostrar:

```text
Query
 ↓
Semantic
 ↓
Physical Plan
 ↓
Execution Plan
 ↓
Compiler Pipeline
 ├─ Lowering
 ├─ Adaptation
 ├─ Aliases
 ├─ Placeholders
 ├─ Rendering
 ├─ Bindings
 └─ Result Contract
 ↓
SQL
```

---

# 291. Example end-to-end

Operación:

```text
SELECT
    users.id,
    users.email
FROM users
WHERE
    users.active = P1
    AND users.created_at >= P2
ORDER BY
    users.created_at DESC
LIMIT 20
```

---

# 292. P0

Acepta:

```text
SelectCompilableOperation
```

---

# 293. P1

Comprueba:

```text
SELECT supported
ORDER BY supported
LIMIT supported
parameters valid
```

---

# 294. P2

Selecciona:

```text
SelectSqlCompiler
```

---

# 295. P3

Produce:

```text
SqlSelectStatement
├── Projection
│   ├── Column(users.id)
│   └── Column(users.email)
├── From
│   └── Relation(users)
├── Where
│   └── And
│       ├── Eq(users.active, P1)
│       └── Gte(users.created_at, P2)
├── OrderBy
│   └── users.created_at DESC
└── Limit
    └── 20
```

---

# 296. P4 PostgreSQL

Adapta a convenciones PostgreSQL.

---

# 297. P5

Verifica representabilidad.

---

# 298. P6

Alias:

```text
users → u
```

si el plan de rendering lo requiere.

---

# 299. P7

Placeholders:

```text
P1 occurrence → $1
P2 occurrence → $2
```

---

# 300. P8

SQL:

```sql
SELECT "u"."id", "u"."email"
FROM "users" AS "u"
WHERE "u"."active" = $1
  AND "u"."created_at" >= $2
ORDER BY "u"."created_at" DESC
LIMIT 20
```

---

# 301. P9

Bindings:

```text
$1 → P1 → Boolean
$2 → P2 → DateTime
```

---

# 302. P10

Result:

```text
column 0 → users.id
column 1 → users.email
```

---

# 303. P11

Dependencies:

```text
PostgreSQL dialect
compiler v1
users.id
users.email
users.active
users.created_at
Boolean binding
DateTime binding
```

---

# 304. P12

Fingerprint:

```text
CompiledQueryFingerprint(...)
```

---

# 305. P13

Verifica:

```text
2 placeholders
2 binding slots
2 result columns
all capabilities satisfied
```

---

# 306. P14

Produce:

```text
CompiledDatabaseCommand
```

---

# 307. Pipeline architecture

```text
                     ┌────────────────────┐
                     │ Compilation Input  │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │ Input Validation   │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │ Compiler Resolve   │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │ SQL Lowering       │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │ Dialect Adaptation │
                     └─────────┬──────────┘
                               │
                               ▼
                     ┌────────────────────┐
                     │ Representation     │
                     │ Validation         │
                     └─────────┬──────────┘
                               │
                      ┌────────┴─────────┐
                      ▼                  ▼
                Alias Planning      Dependencies
                      │                  │
                      ▼                  │
             Placeholder Planning       │
                      │                  │
                      ▼                  │
                 SQL Rendering           │
                      │                  │
              ┌───────┴────────┐         │
              ▼                ▼         │
          Bindings          Results      │
              │                │         │
              └───────┬────────┘         │
                      ▼                  │
                Dependencies ◄───────────┘
                      │
                      ▼
                 Fingerprint
                      │
                      ▼
                Final Validation
                      │
                      ▼
                  Finalization
                      │
                      ▼
             Compiled DB Command
```

---

# 308. Phase invariant table

| Phase | May change representation | May change semantics | May execute DB | May inspect runtime bindings |
|---|---:|---:|---:|---:|
| Input Acceptance | No | No | No | No |
| Pre-Validation | No | No | No | No |
| Compiler Resolution | No | No | No | No |
| SQL Lowering | Yes | No | No | No |
| Dialect Adaptation | Yes | No | No | No |
| Representation Validation | No | No | No | No |
| Alias Planning | Yes | No | No | No |
| Placeholder Planning | Yes | No | No | No* |
| SQL Generation | Yes | No | No | No |
| Binding Layout | Yes | No | No | No |
| Result Contract | No | No | No | No |
| Dependency Collection | No | No | No | No |
| Fingerprint | No | No | No | No |
| Post-Validation | No | No | No | No |
| Finalization | No | No | No | No |

`*` Salvo especialización explícita previamente autorizada y representada como input de compilación.

---

# 309. Pipeline invariants

## DB-SQLPIPE-001

El pipeline tendrá entrada y salida explícitas.

## DB-SQLPIPE-002

La entrada será `CompilableDatabaseOperation`.

## DB-SQLPIPE-003

La salida será `CompiledDatabaseCommand`.

## DB-SQLPIPE-004

El pipeline no ejecutará queries.

## DB-SQLPIPE-005

El pipeline no abrirá conexiones.

## DB-SQLPIPE-006

El pipeline no hará semantic analysis.

## DB-SQLPIPE-007

El pipeline no hará query optimization.

## DB-SQLPIPE-008

El pipeline no hará logical planning.

## DB-SQLPIPE-009

El pipeline no hará physical planning.

## DB-SQLPIPE-010

El pipeline no hará execution scheduling.

## DB-SQLPIPE-011

Las fases tendrán orden explícito.

## DB-SQLPIPE-012

El orden no dependerá de registration order.

## DB-SQLPIPE-013

Las fases core serán versionadas.

## DB-SQLPIPE-014

Los artifacts intermedios tendrán estados explícitos.

## DB-SQLPIPE-015

No se utilizará un God Context mutable como artifact.

## DB-SQLPIPE-016

Cada artifact certificará fases anteriores.

## DB-SQLPIPE-017

Input Acceptance no repetirá semantic validation.

## DB-SQLPIPE-018

Pre-Compilation Validation comprobará representabilidad preliminar.

## DB-SQLPIPE-019

Capability failures no generarán fallback implícito.

## DB-SQLPIPE-020

Compiler Resolution será determinista.

## DB-SQLPIPE-021

Ambiguous compiler resolution producirá error.

## DB-SQLPIPE-022

Structural Lowering producirá representación estructurada.

## DB-SQLPIPE-023

Structural Lowering no necesitará producir SQL text.

## DB-SQLPIPE-024

Lowering preservará semántica.

## DB-SQLPIPE-025

Lowering no resolverá symbols.

## DB-SQLPIPE-026

Lowering no inferirá tipos.

## DB-SQLPIPE-027

Dialect Adaptation no optimizará queries.

## DB-SQLPIPE-028

Dialect Adaptation no seleccionará physical strategies.

## DB-SQLPIPE-029

Representational rewrites preservarán semántica.

## DB-SQLPIPE-030

Representational rewrites preservarán physical intent.

## DB-SQLPIPE-031

Representation Validation ocurrirá antes de rendering final.

## DB-SQLPIPE-032

Unsupported emission nodes producirán error.

## DB-SQLPIPE-033

Alias Planning será distinto de Symbol Resolution.

## DB-SQLPIPE-034

Aliases generados serán deterministas.

## DB-SQLPIPE-035

Aliases generados serán scope-aware.

## DB-SQLPIPE-036

Alias allocator será operation-scoped.

## DB-SQLPIPE-037

Placeholder Planning será distinto de Parameter Definition.

## DB-SQLPIPE-038

ParameterId será distinto de ParameterOccurrenceId.

## DB-SQLPIPE-039

ParameterOccurrenceId será distinto de SqlPlaceholderId.

## DB-SQLPIPE-040

Placeholder allocation será determinista.

## DB-SQLPIPE-041

Placeholder reuse dependerá de capability/driver contract.

## DB-SQLPIPE-042

Runtime values no serán inspeccionados por defecto.

## DB-SQLPIPE-043

Collection specialization será explícita.

## DB-SQLPIPE-044

Placeholder limits serán validados cuando sean conocidos.

## DB-SQLPIPE-045

SQL Generation no cambiará query semantics.

## DB-SQLPIPE-046

SQL Generation será determinista.

## DB-SQLPIPE-047

SQL Generation utilizará structured emission.

## DB-SQLPIPE-048

Canonical rendering será estable.

## DB-SQLPIPE-049

Pretty rendering no cambiará semántica.

## DB-SQLPIPE-050

Source maps podrán generarse durante rendering.

## DB-SQLPIPE-051

Source maps no requerirán reparsing SQL.

## DB-SQLPIPE-052

SQL size estará sujeto a budget.

## DB-SQLPIPE-053

Binding Layout será explícito.

## DB-SQLPIPE-054

Binding Layout no contendrá runtime values.

## DB-SQLPIPE-055

Todos los placeholders runtime tendrán binding slots.

## DB-SQLPIPE-056

Todos los binding slots referenciarán parámetros válidos.

## DB-SQLPIPE-057

Typed NULL será preservable.

## DB-SQLPIPE-058

Binding conversion será estructurada.

## DB-SQLPIPE-059

Result Contract será explícito.

## DB-SQLPIPE-060

Result Contract será distinto de ORM hydration.

## DB-SQLPIPE-061

Semantic output identity será preservada.

## DB-SQLPIPE-062

Internal alias será distinto de semantic output name.

## DB-SQLPIPE-063

Compilation dependencies serán explícitas.

## DB-SQLPIPE-064

Dependency collection será granular cuando sea seguro.

## DB-SQLPIPE-065

Compiler version podrá participar en dependencies.

## DB-SQLPIPE-066

Extension versions podrán participar en dependencies.

## DB-SQLPIPE-067

Fingerprint será determinista.

## DB-SQLPIPE-068

Fingerprint tendrá domain separation.

## DB-SQLPIPE-069

Runtime binding values serán excluidos por defecto.

## DB-SQLPIPE-070

Specialization data relevante participará en fingerprint.

## DB-SQLPIPE-071

Compiled fingerprint será distinto de SQL text hash.

## DB-SQLPIPE-072

Post-validation verificará artifact completeness.

## DB-SQLPIPE-073

Post-validation no reconstruirá semántica parseando SQL.

## DB-SQLPIPE-074

Security preservation será verificable.

## DB-SQLPIPE-075

Security predicates no podrán desaparecer silenciosamente.

## DB-SQLPIPE-076

Tenant policy provenance podrá preservarse sin core dependency.

## DB-SQLPIPE-077

Finalization producirá artifact inmutable.

## DB-SQLPIPE-078

Compilation será atómica.

## DB-SQLPIPE-079

Failure no devolverá comando parcialmente ejecutable.

## DB-SQLPIPE-080

Todas las phase exceptions derivarán de root común.

## DB-SQLPIPE-081

Diagnostics no expondrán runtime secrets.

## DB-SQLPIPE-082

Trace no participará en semantic correctness.

## DB-SQLPIPE-083

Timing telemetry no participará en deterministic fingerprint.

## DB-SQLPIPE-084

Compilation budget será transversal.

## DB-SQLPIPE-085

Budgets favorecerán deterministic work units.

## DB-SQLPIPE-086

Budget exhaustion fallará compilación.

## DB-SQLPIPE-087

Budget exhaustion no truncará SQL.

## DB-SQLPIPE-088

Extensions estarán sujetas a budgets.

## DB-SQLPIPE-089

Extension points serán typed.

## DB-SQLPIPE-090

Extensions declararán phases.

## DB-SQLPIPE-091

Extensions no reordenarán arbitrariamente core phases.

## DB-SQLPIPE-092

Extension dependencies serán validadas en bootstrap.

## DB-SQLPIPE-093

Extension cycles serán rechazados.

## DB-SQLPIPE-094

Extension registry será frozen.

## DB-SQLPIPE-095

No habrá hot registration durante compilaciones activas.

## DB-SQLPIPE-096

Pipeline version podrá participar en cache/fingerprint.

## DB-SQLPIPE-097

Pipeline profiles no cambiarán semantics.

## DB-SQLPIPE-098

Production profile no omitirá correctness validation.

## DB-SQLPIPE-099

CompilationContext será immutable snapshot.

## DB-SQLPIPE-100

CompilationContext no contendrá live connection.

## DB-SQLPIPE-101

CompilationContext no contendrá EntityManager.

## DB-SQLPIPE-102

CompilationContext no contendrá HTTP Request.

## DB-SQLPIPE-103

Capabilities serán snapshot-consistent.

## DB-SQLPIPE-104

Platform/Dialect/DriverCompilationContract deberán ser compatibles.

## DB-SQLPIPE-105

DriverCompilationContract será distinto de runtime Driver.

## DB-SQLPIPE-106

Compiler no hará prepare().

## DB-SQLPIPE-107

Compiler no hará bind().

## DB-SQLPIPE-108

Compiler no hará execute().

## DB-SQLPIPE-109

Compiled cache será distinto del pipeline.

## DB-SQLPIPE-110

CompilationLookupKey podrá ser distinto de final fingerprint.

## DB-SQLPIPE-111

Observers no modificarán artifacts.

## DB-SQLPIPE-112

Telemetry integration no será dependencia obligatoria.

## DB-SQLPIPE-113

Raw SQL no bypassará validation.

## DB-SQLPIPE-114

Raw SQL no bypassará budget.

## DB-SQLPIPE-115

Raw SQL no bypassará fingerprinting.

## DB-SQLPIPE-116

Raw SQL no bypassará security policy.

## DB-SQLPIPE-117

DML utilizará el mismo pipeline base.

## DB-SQLPIPE-118

No existirán pipelines completamente duplicados por statement kind.

## DB-SQLPIPE-119

DDL podrá reutilizar infraestructura sin compartir necesariamente Query AST.

## DB-SQLPIPE-120

Multi-statement SQL no será default.

## DB-SQLPIPE-121

Batch execution será explícita.

## DB-SQLPIPE-122

Compiler no insertará transaction boundaries por iniciativa propia.

## DB-SQLPIPE-123

Compiler components compartidos serán stateless o immutable.

## DB-SQLPIPE-124

Operation state residirá en SqlCompilationSession.

## DB-SQLPIPE-125

SqlCompilationSession será descartada después de compilation.

## DB-SQLPIPE-126

No habrá state leakage entre compilaciones.

## DB-SQLPIPE-127

Concurrent compilation será segura.

## DB-SQLPIPE-128

Pipeline no asumirá HTTP lifecycle.

## DB-SQLPIPE-129

Pipeline funcionará en CLI/workers/tests.

## DB-SQLPIPE-130

Persistent workers reutilizarán sólo estado seguro.

## DB-SQLPIPE-131

Intermediate artifacts podrán liberarse anticipadamente.

## DB-SQLPIPE-132

Debug retention será configurable.

## DB-SQLPIPE-133

Artifact retention no cambiará output semantics.

## DB-SQLPIPE-134

Generated SQL deberá ser completamente explicable por inputs/context.

## DB-SQLPIPE-135

Compiler selection deberá ser completamente explicable.

## DB-SQLPIPE-136

Every representational rewrite deberá ser trazable.

## DB-SQLPIPE-137

Every extension invocation deberá ser identificable.

## DB-SQLPIPE-138

Every dependency affecting representation deberá ser trackable.

## DB-SQLPIPE-139

Every specialization affecting SQL deberá ser trackable.

## DB-SQLPIPE-140

Every final artifact deberá haber pasado post-validation.

## DB-SQLPIPE-141

No final artifact podrá contener unresolved emission nodes.

## DB-SQLPIPE-142

No final artifact podrá contener unresolved aliases.

## DB-SQLPIPE-143

No final artifact podrá contener unresolved placeholders.

## DB-SQLPIPE-144

No final artifact podrá contener orphan placeholders.

## DB-SQLPIPE-145

No final artifact podrá carecer de target platform identity.

## DB-SQLPIPE-146

No final artifact podrá carecer de compiler identity/version.

## DB-SQLPIPE-147

Compilation correctness tendrá prioridad sobre SQL compactness.

## DB-SQLPIPE-148

Compilation correctness tendrá prioridad sobre aggressive portability fallback.

## DB-SQLPIPE-149

Unsupported exact semantics deberán fallar.

## DB-SQLPIPE-150

El pipeline nunca cambiará el significado observable de la operación compilable.

---

# 310. Anti-patrones

## 310.1 SQL string desde la primera fase

Incorrecto:

```text
Operation
→ concatenate SQL immediately
→ patch string later
```

Correcto:

```text
Operation
→ structured lowering
→ adaptation
→ rendering
```

---

# 311. God Compilation Context

Incorrecto:

```php
$context->sql
$context->bindings
$context->currentNode
$context->connection
$context->entityManager
$context->tenant
$context->request
$context->everything
```

---

# 312. Callbacks sin tipado

Incorrecto:

```php
$compiler->hook('before', fn (&$sql) => ...);
```

---

# 313. Last-write-wins extension

Incorrecto:

```text
extension B silently replaces extension A
```

---

# 314. Runtime COUNT durante compilation

Incorrecto:

```sql
SELECT COUNT(*) FROM users;
```

para decidir cómo compilar.

Eso sería hidden I/O/planning.

---

# 315. Runtime EXPLAIN

Igualmente incorrecto.

---

# 316. SQL reparsing

Incorrecto como pipeline normal:

```text
generate SQL
→ parse SQL again
→ discover what compiler generated
```

La información debe mantenerse estructuralmente.

---

# 317. Placeholder regex

Incorrecto:

```php
preg_match_all('/\?/', $sql)
```

para descubrir bindings.

Los placeholders se conocen antes de rendering.

---

# 318. Alias regex

Incorrecto:

```text
generate SQL
→ regex aliases
→ resolve collisions
```

Aliases se planifican estructuralmente.

---

# 319. Security verification por string

Incorrecto:

```php
str_contains($sql, 'tenant_id')
```

Correcto:

```text
SecurityPolicyId
→ emitted node provenance
→ security preservation ledger
```

---

# 320. Cache por SQL solamente

Incorrecto:

```text
cache key = hash(SQL)
```

si ignora:

```text
binding layout
result contract
platform
capabilities
compiler version
extensions
```

---

# 321. Mutar registry por request

Incorrecto en FrankenPHP:

```text
request
→ register compiler
→ compile
→ maybe unregister
```

---

# 322. Pipeline target

La arquitectura resultante deberá permitir:

```text
one semantic/query engine
+
one compiler pipeline architecture
+
multiple SQL targets
```

---

# 323. Flujo por plataformas

```text
                    Compilable Operation
                            │
                            ▼
                    Common Pipeline
                            │
                   Structural Lowering
                            │
                            ▼
                     SQL Emission Tree
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
       MySQL             PostgreSQL          SQLite
     Adaptation          Adaptation         Adaptation
          │                 │                  │
          ▼                 ▼                  ▼
       Render            Render             Render
          │                 │                  │
          ▼                 ▼                  ▼
    CompiledCommand   CompiledCommand    CompiledCommand
```

MariaDB tendrá su propio target junto a ellos.

---

# 324. Arquitectura propuesta de directorios

```text
VoltStack/Quantum/Database/Query/Compiler/Pipeline
├── Contract
│   ├── SqlCompilerPipeline.php
│   ├── SqlCompilationPhase.php
│   └── SqlCompilationPhaseDescriptor.php
│
├── Context
│   ├── SqlCompilationContext.php
│   ├── SqlCompilationSession.php
│   └── DriverCompilationContract.php
│
├── Artifact
│   ├── AcceptedCompilationInput.php
│   ├── ValidatedCompilationInput.php
│   ├── ResolvedCompilerArtifact.php
│   ├── LoweredSqlArtifact.php
│   ├── DialectAdaptedSqlArtifact.php
│   ├── RepresentationValidatedArtifact.php
│   ├── AliasPlannedSqlArtifact.php
│   ├── PlaceholderPlannedSqlArtifact.php
│   ├── RenderedSqlArtifact.php
│   ├── BindingCompiledArtifact.php
│   ├── ResultContractCompiledArtifact.php
│   ├── DependencyResolvedArtifact.php
│   ├── FingerprintResolvedArtifact.php
│   └── ValidatedCompiledArtifact.php
│
├── Phase
│   ├── InputAcceptancePhase.php
│   ├── PreCompilationValidationPhase.php
│   ├── CompilerResolutionPhase.php
│   ├── StructuralSqlLoweringPhase.php
│   ├── DialectAdaptationPhase.php
│   ├── RepresentationValidationPhase.php
│   ├── AliasPlanningPhase.php
│   ├── PlaceholderPlanningPhase.php
│   ├── SqlGenerationPhase.php
│   ├── BindingLayoutCompilationPhase.php
│   ├── ResultContractCompilationPhase.php
│   ├── DependencyCollectionPhase.php
│   ├── FingerprintConstructionPhase.php
│   ├── PostCompilationValidationPhase.php
│   └── ArtifactFinalizationPhase.php
│
├── Alias
│   ├── SqlAliasAllocator.php
│   ├── SqlAliasMap.php
│   └── SqlAliasPlanningContext.php
│
├── Placeholder
│   ├── SqlPlaceholderAllocator.php
│   ├── SqlPlaceholderPlan.php
│   └── PlaceholderAssignmentMap.php
│
├── Validation
│   ├── PreCompilationValidator.php
│   ├── RepresentationValidator.php
│   └── PostCompilationValidator.php
│
├── Dependency
│   ├── CompilationDependencyCollector.php
│   └── CompilationDependencySet.php
│
├── Fingerprint
│   ├── CompilationLookupKey.php
│   ├── CompiledQueryFingerprint.php
│   └── CompiledQueryFingerprintBuilder.php
│
├── Budget
│   ├── SqlCompilationBudget.php
│   ├── SqlCompilationBudgetTracker.php
│   └── CompilationWorkUnits.php
│
├── Diagnostic
│   ├── SqlCompilationDiagnostic.php
│   ├── SqlCompilationDiagnosticCollector.php
│   ├── CompilationPhaseTrace.php
│   └── SqlCompilationTraceCollector.php
│
├── Security
│   ├── SecurityCompilationLedger.php
│   └── SecurityPreservationReport.php
│
├── Extension
│   ├── SqlCompilationExtensionPhase.php
│   ├── SqlCompilationExtensionDescriptor.php
│   └── SqlCompilationExtensionRegistry.php
│
└── Exception
    ├── SqlCompilationInputException.php
    ├── SqlCompilationValidationException.php
    ├── SqlCompilerResolutionException.php
    ├── SqlLoweringException.php
    ├── SqlDialectAdaptationException.php
    ├── SqlRepresentationValidationException.php
    ├── SqlAliasPlanningException.php
    ├── SqlPlaceholderPlanningException.php
    ├── SqlRenderingException.php
    ├── SqlBindingCompilationException.php
    ├── SqlResultContractCompilationException.php
    ├── SqlDependencyResolutionException.php
    ├── SqlFingerprintException.php
    └── SqlPostCompilationValidationException.php
```

---

# 325. Relación con documento 66

El documento 66 responde:

> ¿Qué es el SQL Compiler y cuáles son sus fronteras?

Este documento responde:

> ¿En qué orden y mediante qué artifacts ocurre la compilación?

---

# 326. Relación con documento 68

El siguiente documento:

```text
68_DATABASE_SQL_GENERATION_SYSTEM.md
```

profundizará específicamente:

```text
SqlEmissionTree
        │
        ▼
SQL Generation System
        │
        ├── statement rendering
        ├── clause rendering
        ├── identifier rendering
        ├── expression rendering
        ├── predicate rendering
        ├── parenthesis/precedence
        ├── literal rendering
        ├── alias rendering
        ├── placeholder rendering
        ├── whitespace/canonical formatting
        ├── source maps
        └── output validation
        │
        ▼
RenderedSql
```

---

# 327. Block 6 status

```text
Block 6 — SQL Compiler

66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
   ✓

67_DATABASE_SQL_COMPILER_PIPELINE.md
   ✓

68_DATABASE_SQL_GENERATION_SYSTEM.md
   next

69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 328. Principio final

La regla fundamental del pipeline será:

```text
Every compilation decision
must belong to a known phase.
```

No deberán existir decisiones ocultas entre componentes.

Por tanto:

```text
Input
  ↓
Validate
  ↓
Resolve Compiler
  ↓
Lower
  ↓
Adapt
  ↓
Validate Representation
  ↓
Plan Aliases
  ↓
Plan Placeholders
  ↓
Render
  ↓
Compile Bindings
  ↓
Compile Result Contract
  ↓
Collect Dependencies
  ↓
Fingerprint
  ↓
Validate
  ↓
Finalize
```

y en ningún punto:

```text
Guess Semantics
Optimize Again
Plan Again
Open Connection
Inspect Database
Interpolate Runtime Values
Execute SQL
```

---

# 329. Fórmula final

```text
SafeSqlCompilationPipeline
=
ExplicitPhases
+
ImmutableArtifacts
+
DeterministicOrdering
+
SemanticPreservation
+
CapabilityValidation
+
StructuredLowering
+
SafeParameterization
+
DeterministicRendering
+
DependencyTracking
+
FinalInvariantValidation
+
BoundedResourceUsage
```

La arquitectura completa puede resumirse como:

```text
CompilableDatabaseOperation
        │
        │ already semantically valid
        │ already optimized
        │ already physically planned
        │ already execution-planned
        ▼
SQL Compiler Pipeline
        │
        │ representation transformation only
        ▼
CompiledDatabaseCommand
        │
        │ immutable
        │ parameterized
        │ target-specific
        │ dependency-aware
        │ fingerprinted
        │ validated
        ▼
Execution Engine
```

> El pipeline de compilación SQL de VoltStack no será una secuencia de concatenaciones de strings, sino una cadena formal de transformaciones tipadas, verificables, deterministas y aisladas que convierte una operación ya planificada en un comando SQL seguro y ejecutable sin alterar su significado.