# 36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md

# VoltStack Quantum Database
## Semantic Analysis System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 36 — Semantic Analysis System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el sistema interno encargado de ejecutar el análisis semántico de consultas dentro de:

```text
VoltStack/Quantum/Database
```

El documento anterior:

```text
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
```

definió la arquitectura global del Semantic Query Engine.

Este documento define ahora:

```text
cómo se ejecuta realmente el análisis.
```

La transformación principal será:

```text
ValidatedQueryArtifact
        │
        ▼
SemanticAnalysisSystem
        │
        ├── Initialize
        ├── Discover
        ├── Build Scopes
        ├── Register Symbols
        ├── Resolve References
        ├── Resolve Schema
        ├── Collect Type Constraints
        ├── Solve Types
        ├── Resolve Functions
        ├── Resolve Operators
        ├── Analyze Relations
        ├── Analyze Joins
        ├── Analyze Aggregates
        ├── Analyze Correlations
        ├── Derive Constraints
        ├── Derive Dependencies
        ├── Derive Requirements
        ├── Validate Semantic Invariants
        └── Finalize
        │
        ▼
SemanticQueryArtifact
```

---

# 2. Regla maestra

> El Semantic Analysis System será un pipeline determinista, multipase, bounded, extensible y operation-scoped que transforma estructura validada en significado semántico resuelto sin modificar el AST original.

---

# 3. Responsabilidad

El sistema será responsable de coordinar:

- scopes;
- symbols;
- schema resolution;
- relation resolution;
- column resolution;
- parameter resolution;
- type inference;
- function resolution;
- operator resolution;
- nullability;
- aggregates;
- grouping;
- windows;
- joins;
- correlation;
- CTE dependencies;
- set operations;
- DML semantics;
- semantic constraints;
- lineage;
- dependencies;
- capability requirements;
- semantic diagnostics;
- artifact finalization.

---

# 4. No responsabilidades

No será responsable de:

- normalizar AST;
- validar estructura básica;
- optimizar consultas;
- seleccionar planes físicos;
- generar SQL;
- bindear runtime values;
- ejecutar statements;
- administrar conexiones;
- introspectar bases de datos implícitamente;
- hidratar resultados;
- administrar entidades;
- administrar UnitOfWork.

---

# 5. Entrada oficial

La entrada será:

```text
ValidatedQueryArtifact
```

junto con:

```text
SemanticQueryContext
```

Conceptualmente:

```php
$semanticArtifact = $semanticAnalyzer->analyze(
    query: $validatedQuery,
    context: $semanticContext,
);
```

---

# 6. Salida oficial

En caso exitoso:

```text
SemanticQueryArtifact
```

En caso fallido:

```text
SemanticAnalysisFailure
```

o una excepción de dominio apropiada según la API utilizada.

---

# 7. Modelo general

```text
ValidatedQueryArtifact
          │
          ▼
┌──────────────────────────────┐
│ SemanticAnalysisCoordinator  │
└──────────────┬───────────────┘
               │
               ▼
       AnalysisPlanBuilder
               │
               ▼
       SemanticAnalysisPlan
               │
               ▼
      SemanticPassExecutor
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
   Pass A   Pass B   Pass C
      │        │        │
      └────────┼────────┘
               ▼
      SemanticAnalysisState
               │
               ▼
       Finalization Pass
               │
               ▼
     SemanticQueryArtifact
```

---

# 8. SemanticAnalysisCoordinator

El punto principal de entrada será conceptualmente:

```text
SemanticAnalysisCoordinator
```

Su responsabilidad será:

1. recibir el artifact validado;
2. crear el estado operation-scoped;
3. resolver el plan de análisis;
4. ejecutar los passes;
5. coordinar iteraciones;
6. recolectar diagnostics;
7. verificar invariants;
8. congelar artifacts;
9. devolver el resultado.

No contendrá directamente toda la lógica semántica.

---

# 9. Interfaz conceptual

```php
interface SemanticQueryAnalyzerInterface
{
    public function analyze(
        ValidatedQueryArtifact $query,
        SemanticQueryContext $context,
    ): SemanticQueryArtifact;
}
```

Una API diagnóstica podrá utilizar:

```php
interface DiagnosticSemanticQueryAnalyzerInterface
{
    public function analyze(
        ValidatedQueryArtifact $query,
        SemanticQueryContext $context,
    ): SemanticAnalysisResult;
}
```

---

# 10. SemanticAnalysisResult

Conceptualmente:

```text
SemanticAnalysisResult
├── status
├── artifact?
├── diagnostics
├── statistics
└── failure?
```

Estados posibles:

```text
SUCCESS
SUCCESS_WITH_WARNINGS
FAILED
```

---

# 11. SemanticAnalysisPlan

El análisis no deberá depender de llamadas arbitrarias entre analyzers.

Se construirá conceptualmente un:

```text
SemanticAnalysisPlan
```

que determine:

```text
qué passes ejecutar
+
en qué orden
+
qué dependencias existen
+
qué passes pueden iterar
+
qué extensiones participan
```

---

# 12. Analysis pass

Una unidad especializada de análisis será:

```text
SemanticAnalysisPass
```

Interfaz conceptual:

```php
interface SemanticAnalysisPassInterface
{
    public function id(): SemanticPassId;

    public function analyze(
        SemanticAnalysisState $state,
        SemanticPassContext $context,
    ): SemanticPassResult;
}
```

---

# 13. PassId

Cada pass tendrá identidad estable.

Ejemplos:

```text
core.scope.discovery
core.symbol.registration
core.reference.resolution
core.schema.resolution
core.type.constraints
core.type.solve
core.function.resolution
core.operator.resolution
core.relation.analysis
core.join.analysis
core.aggregate.analysis
core.correlation.analysis
core.constraint.analysis
core.dependency.analysis
core.capability.derivation
core.final.validation
core.finalization
```

---

# 14. Pass metadata

Cada descriptor de pass podrá declarar:

```text
PassDescriptor
├── id
├── phase
├── requires
├── provides
├── before
├── after
├── repeatability
├── convergenceMode
├── priority
└── extensionOrigin
```

---

# 15. Dependencias explícitas

Ejemplo:

```text
ReferenceResolutionPass
requires:
    scopes
    registered_symbols
```

Mientras:

```text
TypeInferencePass
requires:
    resolved_references
    schema_types
    parameter_constraints
```

---

# 16. No orden implícito por registro

Nunca:

```text
el pass que fue registrado primero corre primero
```

El orden se calculará mediante dependencias declaradas.

---

# 17. Pass dependency graph

Conceptualmente:

```text
                 Scope Discovery
                       │
                       ▼
               Symbol Registration
                       │
                       ▼
              Reference Resolution
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
      Schema Resolution    Relation Resolution
             │                   │
             └─────────┬─────────┘
                       ▼
             Type Constraint Collection
                       │
                       ▼
                 Type Solving
                       │
              ┌────────┴────────┐
              ▼                 ▼
       Function Resolution  Operator Resolution
              │                 │
              └────────┬────────┘
                       ▼
               Expression Analysis
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
       Joins       Aggregates    Correlations
         │             │             │
         └─────────────┼─────────────┘
                       ▼
              Constraint Analysis
                       │
                       ▼
              Dependency Analysis
                       │
                       ▼
          Capability Requirement Derivation
                       │
                       ▼
             Semantic Validation
                       │
                       ▼
                  Finalization
```

---

# 18. Topological scheduling

El sistema podrá ordenar passes mediante:

```text
topological sort
```

sobre el dependency graph.

---

# 19. Cyclic pass dependency

Si:

```text
Pass A requires B
Pass B requires A
```

sin una estrategia iterativa explícita:

```text
SemanticPassDependencyCycleException
```

---

# 20. Cycles legítimos

Algunas relaciones semánticas sí pueden requerir iteración.

Ejemplo:

```text
type inference
↔
function overload resolution
```

Esto no deberá modelarse como dependencia circular accidental.

Se modelará mediante:

```text
AnalysisGroup
```

o:

```text
FixpointGroup
```

---

# 21. FixpointGroup

Conceptualmente:

```text
FixpointGroup
├── passes
├── convergenceDetector
├── maxIterations
└── failurePolicy
```

---

# 22. Ejemplo

```text
Type Resolution Fixpoint Group

1. Collect constraints
2. Infer known types
3. Resolve function candidates
4. Resolve operator candidates
5. Add derived constraints
6. Re-solve types
7. Check convergence
```

---

# 23. Convergencia

La convergencia deberá determinarse por estado semántico relevante.

No mediante:

```text
"ya ejecutamos suficientes veces"
```

como única condición.

---

# 24. SemanticRevision

El estado podrá mantener:

```text
SemanticRevision
```

incrementado cuando se produce conocimiento semántico nuevo.

Ejemplo:

```text
iteration 1 → revision 18
iteration 2 → revision 27
iteration 3 → revision 27
```

La tercera iteración indica potencial fixpoint.

---

# 25. Monotonic analysis

Siempre que sea posible, los passes iterativos deberán ser:

```text
monotonic
```

Es decir, avanzar de:

```text
less knowledge
→
more knowledge
```

sin alternar indefinidamente entre estados.

---

# 26. Ejemplo de lattice conceptual

Para type resolution:

```text
UNRESOLVED
    │
    ▼
CONSTRAINED
    │
    ▼
RESOLVED
```

o:

```text
UNRESOLVED
    │
    ├──► RESOLVED
    │
    └──► CONFLICT
```

---

# 27. No oscilación

Un pass no deberá producir:

```text
INTEGER
→ STRING
→ INTEGER
→ STRING
```

por decisiones heurísticas.

---

# 28. SemanticAnalysisState

Durante el análisis existirá:

```text
SemanticAnalysisState
```

Este objeto será:

```text
operation-scoped
```

y nunca compartido entre consultas concurrentes.

---

# 29. Contenido conceptual

```text
SemanticAnalysisState
├── validatedQuery
├── semanticContext
├── scopeGraphBuilder
├── symbolTableBuilder
├── relationTableBuilder
├── expressionSemanticBuilder
├── predicateSemanticBuilder
├── parameterSemanticBuilder
├── typeConstraintState
├── functionResolutionState
├── operatorResolutionState
├── correlationState
├── constraintSetBuilder
├── dependencySetBuilder
├── capabilityRequirementBuilder
├── semanticGraphBuilder
├── lineageBuilder
├── diagnosticCollector
├── budgetTracker
├── passState
├── revision
└── statistics
```

---

# 30. Estado mutable permitido

La mutabilidad es aceptable dentro de:

```text
SemanticAnalysisState
```

porque:

- tiene lifetime limitado;
- pertenece a una sola operación;
- no se comparte;
- se descarta al terminar;
- produce artifacts inmutables.

---

# 31. Regla de mutabilidad

```text
Mutable analysis workspace
        │
        ▼
freeze
        │
        ▼
Immutable semantic artifact
```

---

# 32. El AST permanece inmutable

Aunque `SemanticAnalysisState` sea mutable:

```text
Query AST
```

no lo será.

---

# 33. AnalysisState ≠ SemanticArtifact

No deberán confundirse.

```text
SemanticAnalysisState
= workspace temporal

SemanticQueryArtifact
= resultado inmutable
```

---

# 34. Lifecycle del análisis

```text
CREATED
   │
   ▼
INITIALIZED
   │
   ▼
DISCOVERING
   │
   ▼
RESOLVING
   │
   ▼
INFERRING
   │
   ▼
ANALYZING
   │
   ▼
DERIVING
   │
   ▼
VALIDATING
   │
   ▼
FINALIZING
   │
   ├──► COMPLETED
   │
   └──► FAILED
```

---

# 35. INITIALIZE phase

La primera fase deberá:

- validar compatibility del context;
- cargar frozen registries;
- inicializar builders;
- inicializar budgets;
- preparar source mapping;
- calcular analysis mode;
- preparar semantic policy;
- registrar root query scope.

---

# 36. No I/O durante initialize

No deberá:

- abrir conexiones;
- introspectar schema;
- cargar servicios remotos;
- resolver secretos.

---

# 37. DISCOVERY phase

La fase de discovery identifica la forma semántica general.

Podrá descubrir:

- query blocks;
- subqueries;
- CTEs;
- derived relations;
- projection scopes;
- aggregate scopes;
- window scopes;
- set-operation branches;
- DML targets.

---

# 38. Discovery ≠ resolution

Ejemplo:

```text
FROM users u
```

Discovery puede registrar:

```text
alias "u" exists
```

sin haber resuelto todavía la tabla física.

---

# 39. Scope construction

La construcción de scopes deberá ocurrir antes de resolver referencias.

Ejemplo:

```text
Query Scope Q1
├── Relation namespace
├── Projection namespace
├── Parameter namespace
├── CTE namespace
└── parent?
```

---

# 40. Namespace separation

No todos los symbols deberán vivir en el mismo namespace.

Podrán existir:

```text
RELATION namespace
COLUMN namespace
CTE namespace
PROJECTION_ALIAS namespace
WINDOW namespace
PARAMETER namespace
FUNCTION namespace
TYPE namespace
```

---

# 41. Shadowing

Las reglas de shadowing deberán ser explícitas.

Ejemplo:

```text
outer alias = u
inner alias = u
```

podrá ser válido si el scope interno permite shadowing.

---

# 42. Shadowing metadata

Cuando ocurra:

```text
inner u
shadows
outer u
```

deberá ser conocido por el Scope Graph.

---

# 43. Symbol registration

Después de scopes:

```text
definitions
```

serán registradas.

Ejemplos:

```text
table aliases
CTE names
projection aliases
window names
parameters
```

---

# 44. Definition before use

Cuando el lenguaje semántico lo requiera:

```text
definition
→ registration
→ reference resolution
```

---

# 45. Forward references

Algunas construcciones podrán permitir referencias adelantadas.

Esto deberá definirse explícitamente por namespace/kind.

---

# 46. CTE registration

Los nombres de CTE podrán necesitar registrarse antes de analizar completamente sus cuerpos, especialmente para recursive CTE.

---

# 47. Reference resolution

Una vez construidos scopes y symbols:

```text
ReferenceResolutionPass
```

resolverá:

- relation references;
- column references;
- aliases;
- CTE references;
- window references;
- outer references.

---

# 48. Resolution result

Una referencia deberá terminar como:

```text
RESOLVED
AMBIGUOUS
UNKNOWN
DEFERRED
ERROR
```

---

# 49. DEFERRED

`DEFERRED` sólo podrá utilizarse cuando otra fase legítimamente deba aportar información.

No será una forma de ignorar errores.

---

# 50. Deferred resolution tracking

Toda referencia diferida deberá registrarse.

Antes de finalization:

```text
required deferred references = 0
```

---

# 51. Schema resolution

Después o durante symbol resolution:

```text
SchemaAwareResolutionPass
```

resolverá información física/estructural mediante:

```text
SchemaView
```

---

# 52. Schema resolution examples

```text
users
→ physical relation descriptor

users.email
→ column descriptor

users.id
→ UUID / NOT NULL / PRIMARY KEY
```

---

# 53. Schema evidence

La información del SchemaView se tratará como:

```text
semantic evidence
```

No se copiará indiscriminadamente dentro del AST.

---

# 54. Schema lookup cache

El análisis podrá mantener:

```text
OperationSchemaResolutionCache
```

para evitar búsquedas repetidas.

---

# 55. Relation resolution

Se construirán semantic relations para:

```text
physical tables
derived tables
CTEs
VALUES
set operations
table functions
extension relations
```

---

# 56. Derived relation dependency

Una derived relation puede necesitar que su subquery esté suficientemente analizada antes de conocer:

```text
output columns
types
nullability
```

---

# 57. Dependency scheduling

Ejemplo:

```text
Subquery semantic output
        │
        ▼
Derived relation descriptor
        │
        ▼
Outer query column resolution
```

---

# 58. Deferred outer resolution

Por ello algunas resoluciones pueden requerir:

```text
deferred dependency
```

pero deberán resolverse mediante un plan explícito.

---

# 59. Type constraint collection

Antes de resolver todos los tipos, el sistema recolectará:

```text
TypeConstraintSet
```

---

# 60. Fuentes de constraints

```text
schema metadata
ORM mapping metadata
literal semantics
parameter declarations
function signatures
operator signatures
comparison operands
assignments
CASE branches
COALESCE
set operations
VALUES
casts
aggregate signatures
```

---

# 61. Ejemplo

Query:

```text
users.id = :id
```

Schema:

```text
users.id → UserId
```

Constraint:

```text
type(:id) compatible-with UserId
```

---

# 62. Type variable

Un valor no resuelto podrá representarse mediante:

```text
TypeVariable
```

Ejemplo:

```text
T(P1)
```

---

# 63. Constraint graph

Conceptualmente:

```text
TypeConstraintGraph

T(users.id) = UserId
T(P1) ~ T(users.id)
```

Entonces:

```text
T(P1) = UserId
```

---

# 64. Type solving

`QueryTypeConstraintSolver` resolverá el graph.

Resultado posible:

```text
RESOLVED
PARTIALLY_RESOLVED
CONFLICT
UNSATISFIABLE
```

---

# 65. Partial resolution

Puede aceptarse únicamente si el tipo desconocido no es requerido para fases posteriores.

---

# 66. Required type

Un tipo será requerido cuando afecte, por ejemplo:

- function overload;
- operator selection;
- binding conversion;
- set-operation compatibility;
- assignment compatibility;
- compiler semantics.

---

# 67. Function resolution

Una función podrá tener varias signatures.

Ejemplo:

```text
SUM(INTEGER) → BIG_INTEGER
SUM(DECIMAL) → DECIMAL
SUM(FLOAT)   → FLOAT
```

---

# 68. Candidate set

Inicialmente:

```text
FunctionCandidateSet
```

podrá contener varias firmas.

---

# 69. Candidate narrowing

Las constraints de tipos reducirán candidates.

---

# 70. Bidirectional inference

También puede ocurrir:

```text
known function signature
→ additional type constraints
```

Por ello function resolution puede participar en fixpoint.

---

# 71. Ambiguous overload

Si dos candidates igualmente válidos permanecen:

```text
AmbiguousFunctionOverloadException
```

---

# 72. Operator resolution

Utilizará el mismo principio.

Ejemplo:

```text
ADD(INTEGER, INTEGER)
ADD(DECIMAL, DECIMAL)
ADD(DATE, INTERVAL)
```

---

# 73. No SQL token matching

No resolver operadores mediante:

```text
string "+"
```

como única identidad.

Se utilizará:

```text
OperatorId
```

---

# 74. Expression analysis

Una vez resueltos symbols y tipos suficientes:

```text
ExpressionSemanticAnalyzer
```

derivará:

- resolved type;
- nullability;
- volatility;
- determinism;
- dependencies;
- lineage;
- constantness;
- capability requirements.

---

# 75. Bottom-up derivation

Muchas propiedades podrán derivarse:

```text
children
→ parent
```

Ejemplo:

```text
a + b
```

depende de las dependencies de:

```text
a ∪ b
```

---

# 76. Context-sensitive derivation

Otras propiedades requieren contexto.

Ejemplo:

```text
column nullability
```

puede depender del join environment.

---

# 77. Predicate analysis

`PredicateSemanticAnalyzer` derivará:

- truth domain;
- unknownability;
- dependencies;
- null rejection;
- constraints;
- volatility;
- portability.

---

# 78. Null rejection

Ejemplo:

```text
profile.id = 10
```

puede ser null-rejecting respecto a:

```text
profile.id
```

Esto importa para outer join optimization.

---

# 79. Aggregate analysis

El analyzer deberá identificar:

```text
aggregate scopes
grouping expressions
aggregate calls
non-aggregate dependencies
functional dependencies
```

---

# 80. Aggregate phase dependency

Debe ejecutarse después de:

```text
symbol resolution
+
type resolution suficiente
+
relation resolution
```

---

# 81. Grouping validation

Ejemplo:

```text
SELECT department, salary
GROUP BY department
```

El analyzer decidirá si `salary`:

- está agrupado;
- está agregado;
- es funcionalmente dependiente;
- o es inválido.

---

# 82. Functional dependency support

La arquitectura permitirá reglas avanzadas, pero deberán ser:

```text
conservative
+
platform-aware cuando corresponda
```

---

# 83. Window analysis

Deberá resolver:

```text
named windows
partition keys
ordering
frames
function eligibility
window inheritance
```

---

# 84. Window dependency graph

Named windows podrán formar dependencias.

Ejemplo conceptual:

```text
w2 extends w1
```

Estas dependencias deberán ser acíclicas salvo que la especificación futura indique lo contrario.

---

# 85. Join analysis

Después de relation y predicate resolution:

```text
JoinSemanticAnalyzer
```

podrá derivar:

```text
join dependencies
equality keys
null-producing sides
null-preserving sides
join constraints
relation sets
```

---

# 86. Join analysis does not choose algorithm

No decidirá:

```text
hash join
merge join
nested loop
```

Eso pertenece al Planner.

---

# 87. Correlation analysis

`CorrelationAnalyzer` identificará referencias que cruzan scope boundaries.

---

# 88. Correlation depth

Ejemplo:

```text
scope Q3
references symbol from Q1
```

podrá registrar:

```text
correlationDepth = 2
```

---

# 89. Correlation dependency

Una correlated subquery deberá registrar qué symbols externos necesita.

---

# 90. Decorrelation not here

No transformar:

```text
correlated EXISTS
```

en:

```text
semi join
```

durante Semantic Analysis.

---

# 91. CTE dependency analysis

Se construirá:

```text
CteDependencyGraph
```

---

# 92. CTE graph

Ejemplo:

```text
A → B
B → C
C
```

permite determinar orden semántico.

---

# 93. Recursive CTE

Un self-edge podrá ser válido sólo cuando:

```text
recursive semantics
```

lo permitan.

---

# 94. Mutual recursion

Deberá ser explícitamente soportada o rechazada.

Nunca aceptada accidentalmente.

---

# 95. Set-operation analysis

Para:

```text
UNION
INTERSECT
EXCEPT
```

deberá derivarse:

```text
branch output schemas
arity
common types
nullability
output names
lineage
```

---

# 96. Set operation common type

Ejemplo:

```text
INTEGER
UNION
BIG_INTEGER
```

podría derivar:

```text
BIG_INTEGER
```

si la regla de common type lo permite.

---

# 97. Incompatible set types

Ejemplo:

```text
UUID
UNION
DATE
```

sin coerción válida:

```text
SetOperationTypeMismatchException
```

---

# 98. DML semantic analysis

INSERT, UPDATE y DELETE tendrán analyzers especializados.

---

# 99. INSERT analyzer

Validará semánticamente:

```text
target
target columns
source arity
source types
required columns
generated columns
defaults
conflict semantics
returning
```

---

# 100. UPDATE analyzer

Validará:

```text
target
assignments
assignment compatibility
source visibility
predicates
returning
```

---

# 101. DELETE analyzer

Validará:

```text
target
visible relations
predicate references
join dependencies
returning
```

---

# 102. Constraint analysis

Después de resolver suficiente significado:

```text
QueryConstraintAnalyzer
```

derivará facts semánticos.

---

# 103. Constraint sources

```text
schema keys
schema uniqueness
schema nullability
foreign keys
equality predicates
range predicates
constant predicates
join conditions
grouping
set semantics
```

---

# 104. Constraint propagation

Ejemplo:

```text
a = b
b = 10
```

podrá derivar:

```text
a = 10
```

si es semánticamente seguro.

---

# 105. Three-valued logic

La propagación deberá respetar NULL.

No asumir:

```text
a = b
b = 10
→ a = 10
```

en cualquier contexto sin analizar nullability y truth semantics.

---

# 106. Constraint closure

Podrá utilizarse un proceso bounded para derivar closure parcial.

---

# 107. No theorem prover general

VoltStack no intentará resolver lógica arbitraria.

El constraint engine será:

```text
purpose-built
+
bounded
+
conservative
```

---

# 108. Dependency analysis

Se derivará:

```text
QueryDependencySet
```

---

# 109. Dependency categories

```text
SCHEMA_RELATION
SCHEMA_COLUMN
FUNCTION
OPERATOR
TYPE
CTE
SEQUENCE
EXTENSION
CAPABILITY
SEMANTIC_POLICY
```

---

# 110. Direct dependency

Ejemplo:

```text
SELECT users.email
```

depende directamente de:

```text
users.email
```

---

# 111. Transitive dependency

Si:

```text
CTE A
→ table users
```

y query:

```text
→ CTE A
```

puede existir:

```text
transitive dependency on users
```

---

# 112. Dependency classification

Podrá diferenciarse:

```text
DIRECT
TRANSITIVE
DERIVED
OPTIONAL
```

---

# 113. Capability derivation

El sistema producirá:

```text
CapabilityRequirementSet
```

---

# 114. Requirement source

Cada requirement deberá conservar provenance.

Ejemplo:

```text
Requirement:
DML.RETURNING

Origin:
ReturningClause Node #N84
```

---

# 115. Capability requirement severity

Podrá clasificarse:

```text
REQUIRED
OPTIONAL
PREFERRED
EMULATABLE
```

---

# 116. Semantic analysis vs capability validation

El analyzer puede decir:

```text
query requires RIGHT_JOIN
```

sin decidir todavía:

```text
target supports RIGHT_JOIN
```

---

# 117. Targeted semantic mode

Cuando exista target Platform, una fase adicional podrá detectar incompatibilidades semánticas conocidas.

---

# 118. Portable semantic mode

Cuando no exista target:

```text
requirements remain declarative
```

---

# 119. Semantic validation pass

Antes de finalization se ejecutará:

```text
SemanticInvariantValidationPass
```

---

# 120. Qué valida

Entre otros:

```text
all required symbols resolved
all required relations resolved
required types resolved
no type conflicts
valid aggregate scopes
valid correlations
valid CTE dependencies
valid output relation
valid DML assignments
valid semantic graph references
no dangling constraints
no unresolved fatal requirements
```

---

# 121. Validation ≠ documento 34

Debe distinguirse:

```text
34 Query Validation
```

de:

```text
Semantic Invariant Validation
```

---

# 122. Documento 34

Valida principalmente:

```text
estructura
forma
contexto sintáctico/AST
políticas estructurales
```

---

# 123. Documento 36

Valida:

```text
significado resuelto
```

---

# 124. Ejemplo

AST:

```text
SELECT foo
FROM users
```

puede ser estructuralmente válido.

Pero si `foo` no existe:

```text
semantic failure
```

---

# 125. Error recovery

En modo diagnóstico, un error no necesariamente detendrá inmediatamente todo el análisis.

---

# 126. Recovery placeholder

Podrán existir internamente:

```text
ErrorSymbol
ErrorType
ErrorRelation
ErrorExpressionSemanticInfo
```

---

# 127. Tainted semantic state

Cuando un error afecta resultados posteriores:

```text
SemanticTaint
```

podrá marcar información derivada.

---

# 128. Ejemplo

Si una columna no existe:

```text
UnknownColumn
```

su tipo derivado podrá ser:

```text
ErrorType
```

y evitar cascadas de mensajes irrelevantes.

---

# 129. Diagnostic suppression

Ejemplo:

No producir simultáneamente:

```text
Unknown column foo
Cannot infer type of foo
Cannot resolve operator for foo
Cannot determine nullability of foo
```

si todos son consecuencia del primer error.

---

# 130. Root-cause diagnostics

Preferir:

```text
1 root cause
+
useful context
```

sobre cascadas redundantes.

---

# 131. DiagnosticCollector

Será operation-scoped.

Conceptualmente:

```text
SemanticDiagnosticCollector
├── errors
├── warnings
├── notes
├── suppressed
└── limit
```

---

# 132. Diagnostic severity

```text
ERROR
WARNING
NOTICE
INFO
```

---

# 133. Fatality

Severity y fatality podrán ser conceptos distintos.

Por ejemplo:

```text
WARNING
```

no impide artifact.

Mientras:

```text
ERROR
```

normalmente sí.

---

# 134. Diagnostic path

Cada diagnostic deberá intentar incluir:

```text
query block
AST path
NodeId
source location
semantic scope
symbol/relation
diagnostic code
```

---

# 135. Sensitive data

Nunca incluir:

```text
runtime parameter values
credentials
tokens
passwords
```

en diagnostics.

---

# 136. Pass result

Cada pass podrá devolver:

```text
SemanticPassResult
├── status
├── changed
├── revisionDelta
├── diagnostics
├── deferredItems
└── statistics
```

---

# 137. Pass status

```text
SUCCESS
SUCCESS_WITH_CHANGES
NO_CHANGE
DEFERRED
FAILED
```

---

# 138. changed flag

Será importante para fixpoint processing.

---

# 139. Pass idempotence

Cuando sea posible:

```text
pass(state)
pass(state again without new evidence)
```

deberá producir:

```text
NO_CHANGE
```

---

# 140. Repeatability modes

Un pass podrá declarar:

```text
ONCE
UNTIL_STABLE
WHEN_INVALIDATED
FINAL_ONLY
```

---

# 141. Pass invalidation

Cuando nueva evidencia invalide un resultado previo, deberá hacerse explícitamente.

Ejemplo:

```text
new type information
→ function candidate set invalidated
```

---

# 142. Dependency invalidation graph

Podrá existir:

```text
SemanticPassInvalidationGraph
```

para evitar reejecutar todos los passes.

---

# 143. V1 simplification

La primera versión podrá utilizar:

```text
small fixpoint groups
```

en lugar de un sistema de invalidación extremadamente complejo.

---

# 144. V1 priority

```text
correctness
>
architectural clarity
>
incremental sophistication
```

---

# 145. Analysis groups

Propuesta inicial:

```text
Group A — Discovery
Group B — Name Resolution
Group C — Type Resolution
Group D — Semantic Properties
Group E — Relational Analysis
Group F — Constraint Derivation
Group G — Requirements
Group H — Finalization
```

---

# 146. Group A — Discovery

```text
QueryBlockDiscoveryPass
ScopeConstructionPass
DefinitionDiscoveryPass
CteDiscoveryPass
SubqueryDiscoveryPass
```

---

# 147. Group B — Name Resolution

```text
SymbolRegistrationPass
SchemaRelationResolutionPass
ReferenceResolutionPass
DerivedRelationResolutionPass
```

---

# 148. Group C — Type Resolution

```text
TypeConstraintCollectionPass
TypeInferencePass
FunctionResolutionPass
OperatorResolutionPass
TypeRefinementPass
```

Este grupo puede ser fixpoint.

---

# 149. Group D — Semantic Properties

```text
ExpressionSemanticPass
PredicateSemanticPass
NullabilityPass
VolatilityPass
LineagePass
```

---

# 150. Group E — Relational Analysis

```text
JoinAnalysisPass
AggregateAnalysisPass
WindowAnalysisPass
CorrelationAnalysisPass
SetOperationAnalysisPass
DmlSemanticAnalysisPass
```

---

# 151. Group F — Constraints

```text
ConstraintExtractionPass
ConstraintPropagationPass
FunctionalDependencyPass
ConstraintFinalizationPass
```

---

# 152. Group G — Requirements

```text
DependencyAnalysisPass
CapabilityRequirementPass
PortabilityAnalysisPass
```

---

# 153. Group H — Finalization

```text
SemanticInvariantValidationPass
SemanticGraphFinalizationPass
SemanticFingerprintPass
ArtifactFreezePass
```

---

# 154. Pass purity

Idealmente un pass deberá:

```text
read declared inputs
+
write declared outputs
```

---

# 155. Write ownership

Dos passes no deberán modificar arbitrariamente la misma estructura mutable.

---

# 156. Ejemplo

```text
TypeInferencePass
```

es propietario lógico de:

```text
resolved query types
```

Mientras:

```text
NullabilityPass
```

es propietario de:

```text
query nullability derivation
```

---

# 157. Shared facts

Cuando varias fases contribuyan a un fact set:

```text
builder API
```

deberá definir merge semantics explícitas.

---

# 158. Merge conflict

Ejemplo:

```text
Pass A → type INTEGER
Pass B → type UUID
```

No:

```text
last write wins
```

Debe convertirse en:

```text
semantic conflict
```

---

# 159. Fact provenance

Los facts importantes deberán conservar provenance.

Ejemplo:

```text
Parameter P1 = UUID

because:
Column users.id = UUID
and
predicate users.id = P1
```

---

# 160. SemanticFact

Conceptualmente:

```text
SemanticFact
├── kind
├── subject
├── value
├── provenance
├── confidence
└── sourcePass
```

---

# 161. Confidence

Para core semantic facts normalmente:

```text
PROVEN
DECLARED
DERIVED
UNKNOWN
```

No utilizar probabilidades heurísticas.

---

# 162. No fuzzy semantics

La capa semántica no deberá decidir significado por:

```text
"probably"
"likely"
"best guess"
```

---

# 163. Analysis budget

Cada análisis tendrá límites.

---

# 164. SemanticAnalysisBudget

Conceptualmente:

```php
final readonly class SemanticAnalysisBudget
{
    public function __construct(
        public int $maxPassExecutions,
        public int $maxFixpointIterations,
        public int $maxScopes,
        public int $maxSymbols,
        public int $maxRelations,
        public int $maxTypeVariables,
        public int $maxTypeConstraints,
        public int $maxConstraintFacts,
        public int $maxGraphNodes,
        public int $maxGraphEdges,
        public int $maxDiagnostics,
    ) {}
}
```

---

# 165. BudgetTracker

Cada operación utilizará:

```text
SemanticAnalysisBudgetTracker
```

---

# 166. Budget checks

Se realizarán:

- antes de allocations grandes;
- durante loops;
- durante graph expansion;
- durante fixpoint;
- durante extension execution.

---

# 167. Budget failure

Debe producir:

```text
SemanticAnalysisBudgetExceededException
```

con información segura como:

```text
limit category
configured limit
observed count
analysis phase
```

---

# 168. No partial valid artifact

Un budget failure no producirá un:

```text
valid SemanticQueryArtifact
```

---

# 169. Cancellation

El análisis podrá soportar:

```text
CancellationToken
```

o abstracción equivalente.

---

# 170. Cancellation points

Especialmente en:

```text
large graph traversal
fixpoint loops
CTE analysis
constraint propagation
extension passes
```

---

# 171. Cancellation ≠ timeout SQL

Esto es cancelación del análisis del framework.

No:

```text
database statement timeout
```

---

# 172. Deadline

`SemanticPassContext` podrá contener una deadline derivada del Query Context.

---

# 173. No wall-clock semantics

La deadline controla recursos.

No cambia el significado de la consulta.

---

# 174. Extension passes

Una extensión podrá registrar passes.

Pero deberá declarar:

```text
requires
provides
phase
repeatability
```

---

# 175. Extension isolation

Una extensión no deberá obtener acceso arbitrario al mutable state completo.

---

# 176. Restricted pass context

Preferir contracts especializados.

Ejemplo:

```text
SymbolResolutionExtensionContext
TypeInferenceExtensionContext
ConstraintExtensionContext
```

---

# 177. No mutable internals exposure

Evitar:

```php
$extension->analyze($entireInternalState);
```

si puede modificar cualquier cosa.

---

# 178. Extension output

Preferir:

```text
facts
constraints
diagnostics
requirements
semantic contributions
```

validados por el core.

---

# 179. Extension conformance

Toda extensión deberá pasar:

```text
SemanticExtensionConformanceSuite
```

---

# 180. Conformance checks

Entre otros:

- deterministic output;
- bounded execution;
- no hidden I/O;
- no AST mutation;
- valid dependencies;
- valid semantic IDs;
- valid fingerprint contribution;
- safe diagnostics;
- persistent-runtime safety.

---

# 181. Extension fingerprint

Si una extensión modifica significado semántico:

```text
extension identity/version
```

deberá contribuir al semantic fingerprint.

---

# 182. Observational extension

Una extensión puramente diagnóstica podrá declararse:

```text
NON_SEMANTIC
```

y no modificar fingerprint.

---

# 183. Pass categories

Cada pass deberá clasificarse:

```text
SEMANTIC
DERIVATIONAL
VALIDATION
OBSERVATIONAL
FINALIZATION
```

---

# 184. Fingerprint impact

Los passes deberán declarar si su output afecta:

```text
semantic fingerprint
plan fingerprint
diagnostic output only
```

---

# 185. Determinism

Misma entrada semántica deberá producir:

```text
same semantic artifact
```

independientemente de:

- process;
- worker;
- thread;
- coroutine;
- hash iteration order;
- service registration accident.

---

# 186. Deterministic ordering

Cuando se requiera ordenar:

```text
symbols
constraints
dependencies
diagnostics
graph nodes
```

deberá existir un criterio estable.

---

# 187. Stable ordering examples

```text
AST traversal order
NodeId
ScopeId
SymbolId
descriptor registration order after canonical sorting
```

---

# 188. Hash map warning

No depender de:

```text
PHP associative array incidental iteration
```

cuando afecte fingerprints o resultados.

---

# 189. Reentrancy

Shared analyzers deberán ser:

```text
reentrant
```

---

# 190. Ejemplo seguro

```text
Analyzer
├── frozen registries
└── no current-query mutable fields
```

---

# 191. Ejemplo inseguro

```php
final class SemanticAnalyzer
{
    private array $currentSymbols = [];
    private ?SchemaView $currentSchema = null;
}
```

si es singleton.

---

# 192. Persistent runtime cleanup

Después de cada análisis:

```text
AnalysisState
DiagnosticCollector
temporary caches
temporary graph builders
constraint workspace
```

deberán quedar sin referencias persistentes accidentales.

---

# 193. Memory retention

Especial cuidado con:

```text
closures capturing AST
diagnostic collectors
large schema snapshots
extension callbacks
graph builders
```

---

# 194. Weak references

No deberán utilizarse como sustituto de lifecycle correcto.

---

# 195. SchemaView lifetime

Puede ser:

```text
application-scoped
tenant-scoped
request-scoped
operation-scoped
```

dependiendo de su naturaleza.

El analyzer no asumirá lifetime universal.

---

# 196. Query Context integration

El documento:

```text
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
```

define contexto runtime/declarativo.

El Semantic System recibirá sólo la porción necesaria mediante:

```text
SemanticQueryContext
```

---

# 197. Context projection

Preferir:

```text
QueryContext
    │
    ▼
SemanticContextFactory
    │
    ▼
SemanticQueryContext
```

en lugar de entregar todo `QueryContext` indiscriminadamente.

---

# 198. Least-context principle

Cada fase debe recibir únicamente información relevante.

---

# 199. No security principal caching

Información request-specific como:

```text
current user
tenant
correlation ID
request ID
```

no deberá incorporarse a artifacts reutilizables salvo que explícitamente cambie la semántica y forme parte de su identidad.

---

# 200. Policy injection

Si una policy modifica la consulta antes del Semantic Engine:

```text
policy rewrite
→ normalization
→ validation
→ semantic analysis
```

---

# 201. Semantic policy

Si una policy depende de symbols resueltos:

```text
semantic policy pass
```

podrá ejecutarse en una fase declarada.

---

# 202. Policy result visibility

Toda modificación semánticamente relevante deberá ser visible.

No existirán filtros secretos aplicados durante ejecución.

---

# 203. Incremental analysis

La arquitectura deberá permitir en el futuro:

```text
incremental semantic analysis
```

aunque V1 pueda analizar la consulta completa.

---

# 204. Incremental candidate

Ejemplo:

```text
same query template
+
different execution metadata
```

no debería obligar a repetir análisis semántico.

---

# 205. Reusable semantic template

Podrá existir:

```text
SemanticQueryTemplate
```

para queries estructuralmente idénticas.

---

# 206. Runtime values excluded

Cambiar:

```text
:id = UUID A
```

por:

```text
:id = UUID B
```

no deberá repetir semantic analysis.

---

# 207. Shape changes

Cambiar cardinalidad de collection parameter tampoco debería modificar semantic meaning normalmente.

---

# 208. Exceptions

Sólo si una estrategia semántica explícita depende de shape declarada.

No de valores secretos.

---

# 209. Semantic cache key

Conceptualmente:

```text
SemanticCacheKey
=
NormalizedQueryFingerprint
+
SchemaSemanticFingerprint
+
SemanticPolicyFingerprint
+
TypeRegistryFingerprint
+
FunctionRegistryFingerprint
+
OperatorRegistryFingerprint
+
SemanticExtensionFingerprint
+
TargetSemanticProfile?
```

---

# 210. Query Context exclusion

Observational metadata como:

```text
trace ID
request ID
query label
```

no deberá entrar al semantic cache key.

---

# 211. Semantic-impacting metadata

Si metadata declarativa cambia significado:

```text
consistency semantics
tenant schema selection
platform semantic mode
```

deberá reflejarse en la identidad adecuada.

---

# 212. Cache validation

Un artifact recuperado de cache deberá comprobar:

```text
schema version compatibility
extension version compatibility
semantic policy compatibility
target profile compatibility
```

---

# 213. Semantic statistics

Cada análisis podrá producir:

```text
SemanticAnalysisStatistics
```

---

# 214. Campos posibles

```text
passesExecuted
fixpointIterations
scopesCreated
symbolsRegistered
relationsResolved
referencesResolved
typesResolved
constraintsCreated
graphNodes
graphEdges
diagnosticsGenerated
cacheHits
duration
```

---

# 215. Statistics ≠ semantic artifact

No deberán alterar semantic fingerprint.

---

# 216. Telemetry integration

Opcionalmente:

```text
SemanticAnalysisObserver
```

podrá recibir eventos.

---

# 217. Eventos conceptuales

```text
SemanticAnalysisStarted
SemanticPassStarted
SemanticPassCompleted
SemanticFixpointIterationCompleted
SemanticAnalysisCompleted
SemanticAnalysisFailed
```

---

# 218. Hot-path overhead

Por defecto la instrumentación deberá ser ligera.

---

# 219. No event per AST node

Evitar telemetría de altísima cardinalidad como:

```text
SemanticNodeResolved
```

para cada node en producción.

---

# 220. Debug mode

En desarrollo sí podrá generarse:

```text
SemanticAnalysisTrace
```

---

# 221. Analysis trace

Ejemplo:

```text
Pass 01 ScopeDiscovery          0.08 ms
Pass 02 SymbolRegistration      0.11 ms
Pass 03 ReferenceResolution     0.16 ms
Pass 04 TypeConstraints         0.09 ms
Pass 05 TypeSolve iteration 1   changed
Pass 06 FunctionResolution      changed
Pass 05 TypeSolve iteration 2   stable
...
```

---

# 222. Trace safety

Nunca registrar runtime values sensibles.

---

# 223. Debug artifact

Podrá visualizar:

```text
AST
Scopes
Symbols
Relations
Types
Constraints
Dependencies
Semantic Graph
```

---

# 224. Example complete flow

Consulta conceptual:

```php
DB::table('users as u')
    ->join('orders as o', 'o.user_id', '=', 'u.id')
    ->where('u.active', true)
    ->where('o.total', '>', $minimum)
    ->select('u.id', 'u.name')
    ->get();
```

---

# 225. Builder output

El Builder genera Query Model/AST.

Los runtime values se convierten en parameters.

Conceptualmente:

```text
SELECT
    u.id,
    u.name
FROM users u
JOIN orders o
    ON o.user_id = u.id
WHERE
    u.active = P1
    AND
    o.total > P2
```

---

# 226. Validation

Documento 34 garantiza:

```text
valid AST structure
valid clause placement
valid node categories
```

---

# 227. Discovery

Se descubren:

```text
QueryBlock Q1

Relations:
u
o

Parameters:
P1
P2
```

---

# 228. Scope registration

```text
Scope Q1
├── u
├── o
├── P1
└── P2
```

---

# 229. Schema resolution

```text
u → users
o → orders

u.id       → UserId / NOT NULL
u.name     → String / NOT NULL
u.active   → Boolean / NOT NULL
o.user_id  → UserId / NOT NULL
o.total    → Decimal(12,2) / NOT NULL
```

---

# 230. Symbol resolution

```text
u.id      → ColumnSymbol C1
u.name    → ColumnSymbol C2
u.active  → ColumnSymbol C3
o.user_id → ColumnSymbol C4
o.total   → ColumnSymbol C5
```

---

# 231. Type constraints

```text
C4 = C1
P1 compatible Boolean
P2 compatible Decimal(12,2)
```

---

# 232. Type solving

```text
P1 → Boolean
P2 → Decimal(12,2)
```

---

# 233. Operator resolution

```text
C4 EQUAL C1
→ UserId = UserId → Boolean

C3 EQUAL P1
→ Boolean = Boolean → Boolean

C5 GREATER_THAN P2
→ Decimal > Decimal → Boolean
```

---

# 234. Join analysis

```text
Join J1
├── left = users
├── right = orders
├── kind = INNER
├── key = users.id ↔ orders.user_id
└── predicate = resolved
```

---

# 235. Constraint analysis

Puede derivarse:

```text
users.id equivalent orders.user_id
users.active = TRUE
orders.total > P2
```

---

# 236. Dependency analysis

```text
Relations:
users
orders

Columns:
users.id
users.name
users.active
orders.user_id
orders.total
```

---

# 237. Output relation

```text
OutputRelation
├── id   → UserId / NOT NULL
└── name → String / NOT NULL
```

---

# 238. Final artifact

```text
SemanticQueryArtifact
├── AST
├── ScopeGraph
├── SymbolTable
├── RelationTable
├── TypeTable
├── PredicateTable
├── ParameterTable
├── ConstraintSet
├── DependencySet
├── OutputRelation
└── SemanticGraph
```

---

# 239. Error example

Consulta:

```text
SELECT u.email
FROM users x
```

---

# 240. Resolution

Scope contiene:

```text
x
```

pero no:

```text
u
```

Resultado:

```text
DB_SEM_UNKNOWN_RELATION
Unknown relation alias "u".
```

---

# 241. No cascade

No será necesario añadir:

```text
Cannot resolve column email
Cannot infer type
Cannot derive projection
```

si son consecuencias directas.

---

# 242. Type conflict example

```text
users.id = :value
AND
users.created_at > :value
```

Si:

```text
users.id → UUID
users.created_at → DateTime
```

entonces:

```text
P1 requires UUID
P1 requires DateTime
```

sin coerción válida:

```text
DB_SEM_PARAMETER_TYPE_CONFLICT
```

---

# 243. Fixpoint example

Supongamos:

```text
custom_function(:value)
```

con overloads:

```text
custom_function(UUID) → String
custom_function(Integer) → Integer
```

y posteriormente otra expresión impone:

```text
:value = users.id
users.id → UUID
```

Flujo:

```text
Iteration 1
P1 unknown
Function candidates = {UUID, Integer}

Type constraints discover:
P1 = UUID

Iteration 2
P1 resolved UUID
Function candidate = UUID

Iteration 3
no changes

FIXPOINT
```

---

# 244. Semantic pass executor

Conceptualmente:

```php
foreach ($plan->groups() as $group) {
    $executor->executeGroup(
        group: $group,
        state: $state,
        context: $context,
    );
}
```

---

# 245. Once group

Para passes no iterativos:

```text
execute once
```

---

# 246. Fixpoint group

Conceptualmente:

```php
for ($iteration = 1; $iteration <= $maxIterations; $iteration++) {
    $before = $state->revision();

    foreach ($group->passes() as $pass) {
        $executor->execute($pass, $state, $context);
    }

    if ($state->revision() === $before) {
        return;
    }
}

throw new SemanticFixpointNotReachedException();
```

---

# 247. Revision precision

No toda modificación temporal deberá incrementar revision.

Sólo cambios semánticamente relevantes.

---

# 248. Example non-semantic change

Incrementar:

```text
statistics.passCount
```

no deberá modificar `SemanticRevision`.

---

# 249. Example semantic change

Resolver:

```text
P1: UNKNOWN → UUID
```

sí deberá modificar revision.

---

# 250. Pass transactionality

Cuando sea viable, un pass deberá calcular resultados antes de publicarlos.

---

# 251. Atomic contribution

Preferir:

```text
analyze
→ validate contribution
→ commit contribution
```

sobre modificar state parcialmente y fallar a mitad.

---

# 252. SemanticContribution

Conceptualmente:

```text
SemanticContribution
├── facts
├── symbols
├── constraints
├── requirements
├── dependencies
├── diagnostics
└── invalidations
```

---

# 253. Contribution validation

El core podrá validar:

- IDs válidos;
- ownership;
- conflicts;
- budget;
- provenance;
- extension permissions.

---

# 254. Rollback interno

No es una transacción de base de datos.

Es simplemente evitar dejar:

```text
SemanticAnalysisState
```

en un estado incoherente después de un pass fallido.

---

# 255. Internal invariants

Después de cada pass deberán poder comprobarse invariants ligeros.

Ejemplos:

```text
no duplicate SymbolId
no dangling ScopeId
no relation without owning scope
no constraint referencing unknown semantic subject
```

---

# 256. Heavy validation

La validación completa ocurrirá al final.

---

# 257. Failure taxonomy

```text
SemanticAnalysisException
├── SemanticResolutionException
├── SemanticTypeException
├── SemanticRelationException
├── SemanticAggregateException
├── SemanticCorrelationException
├── SemanticConstraintException
├── SemanticExtensionException
├── SemanticBudgetException
├── SemanticPassException
└── SemanticInvariantViolationException
```

---

# 258. User error vs framework error

Distinguir:

```text
query semantic error
```

de:

```text
internal framework invariant violation
```

---

# 259. Query semantic error

Ejemplo:

```text
UnknownColumnException
```

---

# 260. Framework invariant violation

Ejemplo:

```text
SymbolTable contains duplicate SymbolId
```

Debe considerarse bug del framework/extensión.

---

# 261. Internal error wrapping

Errores internos podrán envolver:

```text
pass ID
phase
query fingerprint
semantic revision
```

sin incluir datos sensibles.

---

# 262. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Semantic\Analysis\
```

---

# 263. Estructura propuesta

```text
Query/
└── Semantic/
    └── Analysis/
        ├── Contract/
        │   ├── SemanticQueryAnalyzerInterface.php
        │   ├── SemanticAnalysisPassInterface.php
        │   ├── SemanticPassExecutorInterface.php
        │   ├── SemanticConvergenceDetectorInterface.php
        │   └── SemanticContributionInterface.php
        │
        ├── Coordinator/
        │   └── SemanticAnalysisCoordinator.php
        │
        ├── Plan/
        │   ├── SemanticAnalysisPlan.php
        │   ├── SemanticAnalysisPlanBuilder.php
        │   ├── SemanticPassDescriptor.php
        │   ├── SemanticPassDependencyGraph.php
        │   ├── SemanticAnalysisGroup.php
        │   └── SemanticFixpointGroup.php
        │
        ├── Execution/
        │   ├── SemanticPassExecutor.php
        │   ├── SemanticFixpointExecutor.php
        │   ├── SemanticPassResult.php
        │   └── SemanticPassStatus.php
        │
        ├── State/
        │   ├── SemanticAnalysisState.php
        │   ├── SemanticRevision.php
        │   ├── SemanticPassState.php
        │   └── SemanticTaint.php
        │
        ├── Context/
        │   ├── SemanticQueryContext.php
        │   ├── SemanticPassContext.php
        │   ├── SemanticAnalysisMode.php
        │   └── SemanticAnalysisOptions.php
        │
        ├── Contribution/
        │   ├── SemanticContribution.php
        │   ├── SemanticContributionValidator.php
        │   └── SemanticContributionCommitter.php
        │
        ├── Pass/
        │   ├── QueryBlockDiscoveryPass.php
        │   ├── ScopeConstructionPass.php
        │   ├── SymbolRegistrationPass.php
        │   ├── ReferenceResolutionPass.php
        │   ├── SchemaResolutionPass.php
        │   ├── RelationResolutionPass.php
        │   ├── TypeConstraintCollectionPass.php
        │   ├── TypeInferencePass.php
        │   ├── FunctionResolutionPass.php
        │   ├── OperatorResolutionPass.php
        │   ├── ExpressionSemanticPass.php
        │   ├── PredicateSemanticPass.php
        │   ├── NullabilityPass.php
        │   ├── JoinAnalysisPass.php
        │   ├── AggregateAnalysisPass.php
        │   ├── WindowAnalysisPass.php
        │   ├── CorrelationAnalysisPass.php
        │   ├── SetOperationAnalysisPass.php
        │   ├── DmlSemanticAnalysisPass.php
        │   ├── ConstraintExtractionPass.php
        │   ├── ConstraintPropagationPass.php
        │   ├── DependencyAnalysisPass.php
        │   ├── CapabilityRequirementPass.php
        │   ├── PortabilityAnalysisPass.php
        │   ├── SemanticInvariantValidationPass.php
        │   ├── SemanticFingerprintPass.php
        │   └── SemanticArtifactFinalizationPass.php
        │
        ├── Budget/
        │   ├── SemanticAnalysisBudget.php
        │   └── SemanticAnalysisBudgetTracker.php
        │
        ├── Diagnostic/
        │   ├── SemanticDiagnosticCollector.php
        │   ├── SemanticDiagnostic.php
        │   ├── SemanticDiagnosticSeverity.php
        │   └── SemanticDiagnosticSuppression.php
        │
        ├── Statistics/
        │   └── SemanticAnalysisStatistics.php
        │
        ├── Trace/
        │   └── SemanticAnalysisTrace.php
        │
        └── Exception/
            ├── SemanticAnalysisException.php
            ├── SemanticPassDependencyCycleException.php
            ├── SemanticFixpointNotReachedException.php
            ├── SemanticAnalysisBudgetExceededException.php
            ├── SemanticPassExecutionException.php
            └── SemanticInvariantViolationException.php
```

---

# 264. Architectural invariants

## SA-001

Todo análisis comenzará desde `ValidatedQueryArtifact`.

## SA-002

El AST de entrada será inmutable.

## SA-003

Todo mutable analysis state será operation-scoped.

## SA-004

El resultado exitoso será inmutable.

## SA-005

El orden de passes será explícito y determinista.

## SA-006

Las dependencias entre passes serán declaradas.

## SA-007

El orden accidental de registro no determinará semántica.

## SA-008

Los ciclos accidentales entre passes serán errores.

## SA-009

Los ciclos semánticos legítimos utilizarán Fixpoint Groups explícitos.

## SA-010

Todo Fixpoint Group tendrá límite de iteraciones.

## SA-011

Todo Fixpoint Group tendrá criterio de convergencia.

## SA-012

La convergencia ignorará cambios puramente observacionales.

## SA-013

La inferencia deberá ser monotónica siempre que sea posible.

## SA-014

No se permitirán oscilaciones heurísticas silenciosas.

## SA-015

Scopes deberán existir antes de resolver references dependientes de ellos.

## SA-016

Definitions deberán registrarse antes de sus usos cuando corresponda.

## SA-017

Forward references serán explícitamente soportadas o rechazadas.

## SA-018

Toda resolución diferida deberá ser rastreada.

## SA-019

No podrán existir referencias requeridas diferidas al finalizar.

## SA-020

Schema resolution utilizará `SchemaView`.

## SA-021

Semantic Analysis no abrirá conexiones.

## SA-022

Semantic Analysis no realizará introspection implícita.

## SA-023

Schema lookup podrá cachearse dentro de la operación.

## SA-024

Derived relations dependerán de output relations semánticas.

## SA-025

Type inference utilizará constraints explícitas.

## SA-026

No se utilizará first-type-wins.

## SA-027

Function overload resolution podrá aportar nuevas type constraints.

## SA-028

Operator resolution podrá aportar nuevas type constraints.

## SA-029

Type/function/operator resolution podrá utilizar fixpoint controlado.

## SA-030

Los runtime parameter values no formarán parte del análisis semántico normal.

## SA-031

Expression semantics se derivarán después de resolución suficiente.

## SA-032

Predicate semantics preservarán three-valued logic.

## SA-033

Nullability semántica podrá diferir de schema nullability.

## SA-034

Aggregate analysis será independiente del Compiler.

## SA-035

Join analysis no seleccionará physical join algorithms.

## SA-036

Correlation analysis no realizará decorrelation.

## SA-037

Constraint analysis será bounded.

## SA-038

Constraint analysis será conservador.

## SA-039

Dependency analysis preservará provenance cuando sea relevante.

## SA-040

Capability derivation no utilizará vendor conditionals dispersos.

## SA-041

Semantic validation será distinta de structural query validation.

## SA-042

Fatal semantic diagnostics impedirán finalization válida.

## SA-043

Error recovery podrá usar placeholders internos.

## SA-044

Error placeholders no escaparán en artifacts válidos.

## SA-045

Diagnostics secundarios podrán suprimirse cuando sean consecuencia directa de un root cause.

## SA-046

Los diagnostics tendrán códigos estables.

## SA-047

Los diagnostics no expondrán secretos.

## SA-048

Los passes deberán declarar ownership lógico de sus outputs.

## SA-049

Conflicting semantic facts no usarán last-write-wins.

## SA-050

Facts importantes deberán conservar provenance.

## SA-051

El sistema no utilizará fuzzy semantics para resolver ambigüedad.

## SA-052

Toda operación tendrá Analysis Budget.

## SA-053

Budget exhaustion producirá un error de dominio controlado.

## SA-054

Budget exhaustion no producirá artifact parcial válido.

## SA-055

Cancellation podrá ocurrir en puntos seguros.

## SA-056

Extension passes deberán declarar dependencias.

## SA-057

Extension passes no tendrán acceso irrestricto al mutable state salvo necesidad contractual explícita.

## SA-058

Contributions de extensiones serán validadas por core.

## SA-059

Extensiones semantic-impacting participarán en fingerprints.

## SA-060

Extensiones observational-only no modificarán significado.

## SA-061

El resultado será determinista.

## SA-062

Hash iteration accidental no cambiará resultados.

## SA-063

Service discovery accidental no cambiará resultados.

## SA-064

Shared analyzers serán reentrant.

## SA-065

No existirá `currentQuery` global.

## SA-066

No existirá `currentScope` global.

## SA-067

No existirá `currentSchema` global.

## SA-068

No existirá `currentTenant` global dentro del analyzer.

## SA-069

Temporary semantic state no sobrevivirá accidentalmente entre requests.

## SA-070

Semantic caches tendrán identidad suficiente para evitar contaminación entre schemas.

## SA-071

Semantic caches tendrán identidad suficiente para evitar contaminación entre semantic policies.

## SA-072

Semantic caches tendrán identidad suficiente para evitar contaminación entre extension sets.

## SA-073

Observational metadata no cambiará semantic cache identity.

## SA-074

Runtime values no cambiarán semantic cache identity.

## SA-075

Semantic-impacting context sí deberá reflejarse en la identidad apropiada.

## SA-076

Statistics no modificarán semantic fingerprint.

## SA-077

Telemetry no será dependencia obligatoria.

## SA-078

No se emitirán eventos por cada AST node en producción por defecto.

## SA-079

El debug trace será opcional.

## SA-080

El debug trace no contendrá valores sensibles.

## SA-081

Semantic Analysis podrá ejecutarse offline.

## SA-082

La mayoría de tests semánticos no requerirán una DB real.

## SA-083

Semantic Engine seguirá siendo independiente del ORM.

## SA-084

Semantic Engine seguirá siendo independiente del Executor.

## SA-085

Semantic Engine seguirá siendo independiente del Connection Manager.

## SA-086

Semantic Engine seguirá siendo independiente del Driver.

## SA-087

Semantic Engine seguirá siendo independiente del SQL Compiler.

## SA-088

El Optimizer consumirá el resultado del Semantic Engine.

## SA-089

El Planner consumirá semántica resuelta.

## SA-090

El Compiler no repetirá Symbol Resolution.

## SA-091

El Executor no repetirá Semantic Analysis.

## SA-092

Los passes deberán ser testables aisladamente.

## SA-093

Los Fixpoint Groups deberán tener tests de convergencia.

## SA-094

Los passes de extensiones deberán pasar conformance tests.

## SA-095

Semantic Analysis State deberá poder descartarse completamente después de finalization.

## SA-096

Un pass fallido no deberá dejar state parcialmente corrupto.

## SA-097

Semantic contributions deberán validarse antes de commit cuando sea necesario.

## SA-098

Las invariants internas podrán verificarse entre fases.

## SA-099

La validación completa se realizará antes de freeze.

## SA-100

`SemanticQueryArtifact` será la única salida válida para las fases posteriores cuando el análisis haya sido exitoso.

---

# 265. Anti-patterns

## 265.1 One giant analyzer

Evitar:

```text
SemanticAnalyzer
├── resolve columns
├── infer types
├── resolve functions
├── analyze joins
├── analyze grouping
├── build graph
├── optimize query
├── compile SQL
└── execute
```

---

## 265.2 Recursive analyzer calls without plan

Evitar:

```text
Analyzer A
→ Analyzer B
→ Analyzer C
→ Analyzer A
```

---

## 265.3 Arbitrary retry loops

Evitar:

```php
while (!$resolved) {
    analyzeAgain();
}
```

Utilizar:

```text
bounded FixpointGroup
```

---

## 265.4 Mutable AST annotations

Evitar:

```php
$node->type = $resolvedType;
```

---

## 265.5 Hidden schema introspection

Evitar:

```text
Semantic Analyzer
→ Connection
→ DESCRIBE
```

---

## 265.6 Runtime-value type inference as primary mechanism

Evitar:

```text
P1 runtime value is string
→ therefore semantic type = string
```

---

## 265.7 First candidate wins

Evitar:

```text
function overload candidates
→ choose first
```

---

## 265.8 Last fact wins

Evitar:

```text
UUID
then
DateTime
→ DateTime wins
```

Debe ser conflicto.

---

## 265.9 Extension mutates everything

Evitar entregar todo:

```text
SemanticAnalysisState
```

a cualquier extensión sin restricciones.

---

## 265.10 Unbounded constraint closure

Evitar convertir el Query Engine en un theorem prover.

---

# 266. Pipeline consolidado

```text
ValidatedQueryArtifact
        │
        ▼
┌───────────────────────────────┐
│  SemanticAnalysisCoordinator │
└───────────────┬───────────────┘
                │
                ▼
        Initialize State
                │
                ▼
        Discover Query Blocks
                │
                ▼
          Build Scopes
                │
                ▼
        Register Definitions
                │
                ▼
        Resolve References
                │
                ▼
          Resolve Schema
                │
                ▼
        Resolve Relations
                │
                ▼
      Collect Type Constraints
                │
                ▼
      ┌──────────────────────┐
      │ Type Fixpoint Group  │
      │                      │
      │ Type Inference       │
      │ Function Resolution  │
      │ Operator Resolution  │
      │ Type Refinement      │
      └──────────┬───────────┘
                 │
                 ▼
       Expression Semantics
                 │
                 ▼
        Predicate Semantics
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
     Joins   Aggregates  Correlation
       │         │          │
       └─────────┼──────────┘
                 ▼
        Constraint Analysis
                 │
                 ▼
        Dependency Analysis
                 │
                 ▼
      Capability Requirements
                 │
                 ▼
        Portability Analysis
                 │
                 ▼
      Semantic Invariant Check
                 │
                 ▼
       Semantic Graph Freeze
                 │
                 ▼
        Semantic Fingerprint
                 │
                 ▼
       SemanticQueryArtifact
```

---

# 267. Relación con documentos siguientes

Este documento define el motor que coordina las siguientes especializaciones.

```text
36 Semantic Analysis System
          │
          ├── 37 Symbol Resolution
          │
          ├── 38 Schema-Aware Resolution
          │
          ├── 39 Query Type Inference
          │
          ├── 40 Relation & Join Resolution
          │
          ├── 41 Query Constraint Analysis
          │
          └── 42 Semantic Query Graph
```

---

# 268. Fronteras documentales

## Documento 37

Profundizará en:

```text
Scope
Symbol
Namespace
Alias
Reference
Shadowing
Correlation lookup
Ambiguity
Symbol Table
```

## Documento 38

Profundizará en:

```text
SchemaView
physical relations
columns
keys
constraints
schema snapshots
schema versions
schema-aware resolution
```

## Documento 39

Profundizará en:

```text
TypeVariable
TypeConstraintGraph
inference
common types
coercion
function/operator typing
parameter typing
fixpoint
```

## Documento 40

Profundizará en:

```text
relations
derived relations
joins
outer joins
relation sets
join keys
nullability
join dependencies
```

## Documento 41

Profundizará en:

```text
semantic facts
constraints
equivalences
ranges
null constraints
functional dependencies
constraint propagation
```

## Documento 42

Profundizará en:

```text
SemanticQueryGraph
nodes
edges
lineage
dependencies
graph construction
graph freeze
graph fingerprints
```

---

# 269. Fórmulas del sistema

## Semantic analysis

```text
Semantic Analysis
=
Discovery
+
Resolution
+
Inference
+
Semantic Derivation
+
Constraint Derivation
+
Requirement Derivation
+
Invariant Validation
```

## Analysis plan

```text
SemanticAnalysisPlan
=
Core Passes
+
Extension Passes
+
Pass Dependencies
+
Fixpoint Groups
+
Execution Policies
```

## Analysis state

```text
SemanticAnalysisState
=
Validated Query
+
Temporary Semantic Facts
+
Builders
+
Resolution State
+
Constraint State
+
Diagnostics
+
Budget
+
Revision
```

## Convergencia

```text
Fixpoint Reached
⇔
SemanticRevision(before)
=
SemanticRevision(after)
```

para el conjunto relevante de passes.

## Final artifact

```text
SemanticQueryArtifact
=
freeze(
    Resolved Scopes
    +
    Resolved Symbols
    +
    Resolved Relations
    +
    Resolved Types
    +
    Semantic Properties
    +
    Constraints
    +
    Dependencies
    +
    Requirements
    +
    Semantic Graph
)
```

---

# 270. Resultado arquitectónico

El Semantic Analysis System convierte:

```text
"esta consulta tiene una estructura válida"
```

en:

```text
"VoltStack comprende formalmente qué significa esta consulta"
```

sin convertir todavía ese significado en una estrategia física o SQL.

La separación queda:

```text
Query AST
   │
   ▼
Normalization
   │
   ▼
Validation
   │
   ▼
════════════════════════════════
   SEMANTIC ANALYSIS SYSTEM
════════════════════════════════
   │
   ├── Discovery
   ├── Scope
   ├── Symbols
   ├── Schema
   ├── Types
   ├── Functions
   ├── Operators
   ├── Relations
   ├── Joins
   ├── Aggregates
   ├── Correlations
   ├── Constraints
   ├── Dependencies
   └── Requirements
   │
   ▼
SemanticQueryArtifact
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ▼
Compiler
```

---

# 271. Decisión final

VoltStack adoptará un **Semantic Analysis System multipase y orientado a facts**, con:

```text
explicit pass graph
+
operation-scoped mutable workspace
+
immutable final artifacts
+
bounded fixpoint processing
+
constraint-based inference
+
deterministic resolution
+
typed extension points
+
strict dependency boundaries
```

La arquitectura evitará deliberadamente:

```text
God Semantic Analyzer
mutable AST annotations
hidden database introspection
global analysis state
first-match resolution
first-type-wins
last-write-wins
unbounded fixpoint loops
vendor-specific conditionals
semantic logic inside Compiler
semantic logic inside Executor
```

---

# 272. Invariante central

> Una consulta sólo podrá abandonar el Semantic Analysis System cuando todos los elementos semánticos requeridos por las siguientes fases estén resueltos, sean internamente coherentes y formen un artifact inmutable y determinista.

---

# 273. Próximo documento

```text
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
```

definirá en profundidad:

```text
SemanticScope
Scope Graph
Symbol
SymbolId
Symbol Namespace
Symbol Table
Relation Symbols
Column Symbols
Alias Symbols
CTE Symbols
Projection Symbols
Window Symbols
Parameter Symbols
Definition Registration
Reference Resolution
Qualified Resolution
Unqualified Resolution
Shadowing
Outer Scope Lookup
Correlation
Ambiguity Detection
Unknown Symbol Diagnostics
Extension Symbols
Persistent Runtime Safety
```

La transformación principal será:

```text
AST References
      │
      ▼
Scope Resolution
      │
      ▼
Namespace Resolution
      │
      ▼
Symbol Lookup
      │
      ├── Unique ─────► ResolvedSymbol
      ├── Multiple ───► AmbiguousSymbol
      └── None ───────► UnknownSymbol
```