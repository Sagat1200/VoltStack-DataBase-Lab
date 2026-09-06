# 35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md

# VoltStack Quantum Database
## Semantic Query Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 35 — Semantic Query Architecture  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del:

```text
VoltStack/Quantum/Database Semantic Query Engine
```

El Semantic Query Engine será responsable de transformar una consulta que ya es:

```text
estructuralmente válida
+
normalizada
+
contextualmente coherente
```

en una representación donde VoltStack comprenda formalmente:

```text
qué significa cada símbolo,
qué representa cada expresión,
qué tipos intervienen,
qué relaciones existen,
qué scopes aplican,
qué dependencias existen,
qué constraints pueden deducirse,
y qué capacidades requerirá posteriormente la consulta.
```

La transformación fundamental será:

```text
ValidatedQueryArtifact
        │
        ▼
Semantic Query Engine
        │
        ├── Scope Analysis
        ├── Symbol Resolution
        ├── Schema Resolution
        ├── Type Inference
        ├── Function Resolution
        ├── Operator Resolution
        ├── Relation Resolution
        ├── Join Analysis
        ├── Aggregate Analysis
        ├── Correlation Analysis
        ├── Constraint Analysis
        └── Requirement Derivation
        │
        ▼
SemanticQueryArtifact
        │
        ▼
Semantic Query Graph
```

---

# 2. Regla maestra

> El AST describe la estructura de una consulta; el Semantic Query Engine determina qué significa esa estructura dentro de un contexto de resolución concreto.

---

# 3. AST ≠ Semantic Model

Una expresión:

```text
users.email
```

dentro del AST representa estructuralmente:

```text
ColumnReferenceExpression
├── qualifier = users
└── name = email
```

Pero todavía no necesariamente sabemos:

```text
qué source representa "users",
qué tabla real representa ese source,
si "email" existe,
qué tipo posee,
si acepta NULL,
qué collation utiliza,
qué lineage tiene,
o qué symbol ID le corresponde.
```

La capa semántica resolverá esa información.

---

# 4. Ejemplo

AST validado:

```text
ComparisonPredicate
├── left
│   └── ColumnReference(users.age)
├── operator = GREATER_THAN
└── right
    └── Parameter(P1)
```

Después del análisis semántico:

```text
ComparisonPredicate
│
├── left
│   └── ColumnSymbol #C17
│       ├── relation = #R3
│       ├── physicalColumn = age
│       ├── type = INTEGER
│       └── nullable = false
│
├── operator
│   └── ResolvedComparisonOperator
│       └── INTEGER > INTEGER → BOOLEAN
│
└── right
    └── ParameterSymbol #P1
        ├── inferredType = INTEGER
        └── nullable = unknown/false
```

---

# 5. Objetivos

El Semantic Query Engine deberá proporcionar:

1. resolución de scopes;
2. resolución de symbols;
3. resolución schema-aware;
4. resolución de relations;
5. resolución de columns;
6. resolución de aliases;
7. resolución de CTEs;
8. resolución de correlated references;
9. inferencia de Query Types;
10. resolución de parámetros;
11. resolución de funciones;
12. resolución de operadores;
13. análisis de aggregates;
14. análisis de grouping;
15. análisis de joins;
16. análisis de set operations;
17. análisis de nullability;
18. análisis de constraints;
19. lineage semántico;
20. dependency analysis;
21. derivación de capability requirements;
22. generación del Semantic Query Graph;
23. diagnostics semánticos;
24. extensibilidad;
25. procesamiento determinista;
26. compatibilidad con persistent runtimes.

---

# 6. No objetivos

El Semantic Query Engine no deberá:

- generar SQL;
- seleccionar sintaxis SQL;
- quote identifiers;
- ejecutar consultas;
- abrir conexiones;
- seleccionar índices físicos;
- calcular planes físicos;
- hidratar entidades;
- modificar UnitOfWork;
- iniciar transacciones;
- adquirir ConnectionLease;
- hacer I/O arbitrario;
- convertirse en ORM;
- convertirse en SQL Compiler.

---

# 7. Posición en el Query Pipeline

```text
Developer API
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
Normalization
     │
     ▼
Validation
     │
     ▼
ValidatedQueryArtifact
     │
     ▼
┌─────────────────────────────┐
│   Semantic Query Engine     │
└─────────────────────────────┘
     │
     ▼
SemanticQueryArtifact
     │
     ▼
Query Optimizer
     │
     ▼
Query Planner
     │
     ▼
SQL Compiler
     │
     ▼
Executor
```

---

# 8. Frontera de entrada

El Semantic Query Engine recibirá:

```text
ValidatedQueryArtifact
```

No deberá aceptar normalmente:

```text
raw Builder state
mutable Query Builder
unvalidated Query AST
SQL strings
PDO statements
```

---

# 9. Frontera de salida

La salida principal será:

```text
SemanticQueryArtifact
```

que contendrá o referenciará:

```text
Validated AST
Semantic Query Graph
Semantic Symbol Table
Semantic Scope Graph
Expression Semantic Table
Predicate Semantic Table
Parameter Semantic Table
Relation Semantic Table
Constraint Set
Capability Requirements
Dependency Information
Semantic Diagnostics
```

---

# 10. Principio de inmutabilidad

El análisis semántico no deberá modificar el AST original.

No:

```php
$columnNode->resolvedType = $type;
$columnNode->table = $table;
```

Preferir:

```text
Immutable AST
     │
     ├───────────────┐
     │               │
     ▼               ▼
NodeId           Semantic Tables
                     │
                     └── semantic information
```

---

# 11. Side-table architecture

La información semántica se asociará mediante identidades estables.

Ejemplo:

```text
NodeId #N42
   │
   ▼
ExpressionSemanticTable
   │
   └── ExpressionSemanticInfo
```

Esto evita contaminar el AST con estado de resolución.

---

# 12. Beneficios

La arquitectura side-table permite:

- AST inmutable;
- análisis repetible;
- múltiples contextos semánticos;
- thread/coroutine safety;
- caching controlado;
- optimizaciones posteriores;
- persistent workers seguros;
- separación entre syntax structure y meaning.

---

# 13. Arquitectura general

```text
                     ValidatedQueryArtifact
                              │
                              ▼
                  SemanticAnalysisCoordinator
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
   Scope Builder        Symbol Resolver      Schema Resolver
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                    Relation Resolver
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
          Type Engine     Function       Operator
                          Resolver       Resolver
                │             │             │
                └─────────────┼─────────────┘
                              ▼
                     Semantic Analyzer
                              │
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
      Join Analysis     Aggregate Analysis   Correlation
           │                  │               Analysis
           └──────────────────┼──────────────────┘
                              ▼
                    Constraint Analyzer
                              │
                              ▼
                 Requirement Derivation
                              │
                              ▼
                  Semantic Query Graph
                              │
                              ▼
                 SemanticQueryArtifact
```

---

# 14. SemanticAnalysisCoordinator

El componente coordinador será conceptualmente:

```text
SemanticAnalysisCoordinator
```

Responsabilidades:

- organizar las fases;
- mantener orden determinista;
- proporcionar contextos;
- coordinar analyzers;
- recolectar diagnostics;
- construir artifact final.

No deberá implementar toda la lógica semántica directamente.

---

# 15. No God Semantic Analyzer

Evitar:

```text
SemanticAnalyzer.php
20,000+ lines
```

La lógica deberá distribuirse entre componentes especializados.

---

# 16. Fases semánticas

La arquitectura inicial utilizará conceptualmente:

```text
Phase 1  → Scope Construction
Phase 2  → Source Registration
Phase 3  → Symbol Resolution
Phase 4  → Schema-Aware Resolution
Phase 5  → Relation Resolution
Phase 6  → Type Constraint Collection
Phase 7  → Type Inference
Phase 8  → Function Resolution
Phase 9  → Operator Resolution
Phase 10 → Aggregate / Group Analysis
Phase 11 → Join Analysis
Phase 12 → Correlation Analysis
Phase 13 → Constraint Analysis
Phase 14 → Capability Requirement Derivation
Phase 15 → Semantic Graph Finalization
```

La implementación podrá combinar fases cuando sea seguro, pero las responsabilidades conceptuales permanecerán separadas.

---

# 17. ¿Por qué varias fases?

Porque existen dependencias.

Ejemplo:

```text
LOWER(users.email)
```

Para resolver `LOWER` puede ser necesario conocer el tipo de:

```text
users.email
```

Para conocerlo primero hay que resolver:

```text
users
```

y después:

```text
email
```

Por tanto:

```text
Scope
  ↓
Symbol
  ↓
Schema
  ↓
Type
  ↓
Function Overload
```

---

# 18. Resolución iterativa

Algunos casos requerirán resolución bidireccional.

Ejemplo:

```text
users.id = :id
```

Podemos inferir:

```text
users.id → UUID
```

y por contexto:

```text
:id → UUID
```

---

# 19. Constraint-based semantics

Por ello el Type Engine no dependerá únicamente de un recorrido:

```text
bottom-up
```

Podrá construir:

```text
Type Constraints
```

y resolverlos posteriormente.

---

# 20. SemanticContext

Cada análisis utilizará un contexto explícito.

Conceptualmente:

```php
final readonly class SemanticQueryContext
{
    public function __construct(
        public SchemaView $schema,
        public QuerySemanticPolicy $policy,
        public SemanticExtensionSet $extensions,
        public SemanticAnalysisBudget $budget,
        public ?SemanticDiagnosticSink $diagnostics = null,
    ) {}
}
```

---

# 21. No globals

Prohibido resolver:

```text
current schema
current tenant
current connection
current EntityManager
```

mediante variables globales.

---

# 22. SchemaView

La capa semántica no deberá depender directamente de una conexión viva.

Utilizará:

```text
SchemaView
```

como representación abstracta de la información schema necesaria.

---

# 23. SchemaView puede provenir de

```text
compiled schema metadata
schema cache
migration model
schema introspection snapshot
application metadata
tenant-resolved schema view
```

---

# 24. Semantic Analysis offline

Idealmente podrá ejecutarse:

```text
sin conexión física
```

si existe suficiente metadata.

Esto habilita:

- static analysis;
- testing;
- IDE tooling;
- query compilation;
- warmup;
- precompilation.

---

# 25. SchemaView ≠ Schema Manager

`SchemaView` será lectura semántica.

No deberá:

- crear tablas;
- alterar columnas;
- ejecutar introspection automáticamente;
- adquirir conexiones.

---

# 26. Scope

Un `SemanticScope` representa el espacio donde determinados symbols pueden resolverse.

---

# 27. Ejemplo

```sql
SELECT u.name
FROM users u
```

El scope contiene:

```text
u → RelationSymbol(users)
```

---

# 28. Nested scopes

```sql
SELECT u.name
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

produce conceptualmente:

```text
Scope S1
└── u

Scope S2
├── o
└── parent → S1
```

---

# 29. Scope Graph

Los scopes formarán:

```text
SemanticScopeGraph
```

No dependerán de mutable parent pointers dentro del AST.

---

# 30. Scope kinds

Inicialmente:

```text
QUERY
SUBQUERY
CTE
DERIVED_TABLE
JOIN
PROJECTION
GROUP
WINDOW
SET_OPERATION
EXPRESSION
EXTENSION
```

---

# 31. Symbol

Un symbol es una identidad semántica resoluble.

---

# 32. Symbol kinds

```text
RELATION
COLUMN
ALIAS
CTE
PARAMETER
FUNCTION
OPERATOR
WINDOW
PROJECTION
TYPE
EXTENSION
```

---

# 33. SymbolId

Cada symbol tendrá identidad independiente de su nombre textual.

```text
SymbolId
```

---

# 34. Por qué SymbolId

Porque:

```text
users
```

puede aparecer varias veces.

Ejemplo:

```sql
FROM users manager
JOIN users employee
```

Ambos representan la misma tabla física pero distintos relation symbols.

---

# 35. Symbol table

Conceptualmente:

```text
SemanticSymbolTable
├── SymbolId
├── Kind
├── Name
├── ScopeId
├── Definition
├── ResolvedTarget
└── SemanticProperties
```

---

# 36. Name ≠ identity

Regla:

```text
Symbol Name
≠
Symbol Identity
```

---

# 37. Relation Symbol

Representa una relation visible dentro de la query.

Puede originarse en:

```text
physical table
CTE
subquery
VALUES relation
table function
extension source
```

---

# 38. Relation Descriptor

Conceptualmente:

```text
RelationSemanticInfo
├── relationSymbolId
├── origin
├── outputColumns
├── keys
├── nullability
├── lineage
└── scope
```

---

# 39. Physical table relation

Ejemplo:

```text
users u
```

podrá producir:

```text
RelationSymbol #R1
├── alias = u
├── source = PhysicalTable(users)
└── columns
    ├── id
    ├── email
    └── name
```

---

# 40. Derived relation

Ejemplo:

```sql
FROM (
    SELECT id, name
    FROM users
) u
```

La relation `u` no será tratada como tabla física.

Será:

```text
DerivedRelation
```

con output schema derivado de la subquery.

---

# 41. CTE relation

Igualmente:

```text
WITH active_users AS (...)
SELECT ...
FROM active_users
```

creará:

```text
CteRelationSymbol
```

---

# 42. Column Symbol

Una referencia:

```text
u.email
```

se resolverá a:

```text
ColumnSymbol
```

con identidad estable.

---

# 43. Column semantic information

Podrá contener:

```text
symbolId
relationId
logicalName
physicalOrigin?
queryType
nullable
lineage
visibility
```

---

# 44. Unqualified column

Ejemplo:

```sql
SELECT id
FROM users
```

podrá resolverse si existe exactamente una opción válida.

---

# 45. Ambiguous column

```sql
SELECT id
FROM users u
JOIN orders o ON ...
```

si ambos contienen `id`:

```text
AmbiguousColumnReferenceException
```

---

# 46. Unknown column

```text
UnknownColumnReferenceException
```

---

# 47. Alias resolution

Los aliases deberán respetar:

- scope;
- visibility;
- query phase;
- semantic rules.

No se asumirá que un projection alias es visible universalmente en todas las clauses.

---

# 48. Platform-specific alias rules

Cuando existan diferencias de DB, el core semantic model deberá representar la intención de manera portable siempre que sea posible.

Las reglas específicas se resolverán mediante:

```text
Semantic Policy
+
Capability Requirements
+
Compiler/Planner rules
```

sin llenar el AST de vendor checks.

---

# 49. Query Type System

Se reutilizará:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

La capa semántica resolverá tipos concretos para:

```text
expressions
parameters
predicates
function results
operators
projection items
relations
```

---

# 50. Type stages

Debe mantenerse:

```text
PHP Type
    ≠
Query Semantic Type
    ≠
Database Native Type
    ≠
Driver Binding Type
```

---

# 51. Semantic type inference

Ejemplo:

```text
users.age + :increment
```

Si:

```text
users.age → INTEGER
```

podemos derivar constraints:

```text
P1 compatible with numeric
result compatible with numeric
```

---

# 52. Parameter inference

Ejemplo:

```text
users.uuid = :id
```

podrá inferir:

```text
:id → UUID
```

sin inspeccionar el valor runtime.

---

# 53. Conflicting inference

```text
:id compared with UUID
:id used as DATE
```

sin conversión válida:

```text
ConflictingParameterTypeException
```

---

# 54. Nullability

El Semantic Query Engine deberá modelar explícitamente:

```text
nullable
non-nullable
unknown
```

---

# 55. Nullability changes

Un outer join puede modificar nullability semántica.

Ejemplo:

```sql
users
LEFT JOIN profiles
```

Aunque:

```text
profiles.name
```

sea `NOT NULL` físicamente, dentro del resultado del LEFT JOIN podrá ser:

```text
nullable
```

---

# 56. Important distinction

```text
Schema Nullability
≠
Query Result Nullability
```

---

# 57. Function resolution

Una función AST será semántica.

Ejemplo:

```text
FunctionExpression(
    function = LOWER,
    args = [...]
)
```

No:

```text
sqlName = "LOWER"
```

---

# 58. FunctionRegistry

El Semantic Engine utilizará descriptors registrados.

```text
FunctionDescriptor
├── FunctionId
├── signatures
├── argument rules
├── return type rule
├── volatility
├── determinism
├── aggregate behavior
├── window behavior
└── capability requirements
```

---

# 59. Overload resolution

Ejemplo:

```text
ABS(INTEGER) → INTEGER
ABS(DECIMAL) → DECIMAL
ABS(FLOAT)   → FLOAT
```

El Semantic Engine seleccionará la firma apropiada.

---

# 60. Unknown function

Si no existe descriptor:

```text
UnknownQueryFunctionException
```

salvo escape hatch explícito.

---

# 61. Operator resolution

Los operadores también serán semánticos.

Ejemplo:

```text
ADD
EQUAL
GREATER_THAN
CONCAT
```

---

# 62. OperatorDescriptor

Conceptualmente:

```text
OperatorDescriptor
├── OperatorId
├── operand constraints
├── result type rule
├── null semantics
├── volatility
└── capability requirements
```

---

# 63. Operator syntax

El Semantic Engine no decidirá:

```text
||
+
CONCAT()
IS DISTINCT FROM
```

como representación SQL.

Eso pertenece al Dialect/Compiler.

---

# 64. Predicate semantics

Cada PredicateNode deberá producir:

```text
PredicateSemanticInfo
```

---

# 65. Predicate semantic information

Podrá incluir:

```text
truthDomain
unknownability
dependencies
volatility
determinism
constraints
nullRejectionProperties
capabilityRequirements
```

---

# 66. SQL three-valued logic

El Semantic Engine deberá preservar:

```text
TRUE
FALSE
UNKNOWN
```

---

# 67. No PHP boolean semantics

Nunca analizar:

```text
SQL predicate
```

como si fuese simplemente:

```php
(bool) $value
```

---

# 68. ExpressionSemanticInfo

Cada expresión podrá poseer:

```text
resolvedType
nullability
volatility
determinism
symbolDependencies
relationDependencies
lineage
constantness
capabilityRequirements
portability
```

---

# 69. Volatility

Categorías iniciales:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

---

# 70. Why volatility matters

El optimizer no podrá asumir:

```text
f() = f()
```

si `f()` es volatile.

---

# 71. Determinism

Puede modelarse separadamente cuando sea necesario.

Ejemplo:

```text
random()
```

no es determinista.

---

# 72. Constantness

Una expresión puede clasificarse como:

```text
COMPILE_TIME_CONSTANT
EXECUTION_CONSTANT
ROW_DEPENDENT
VOLATILE
UNKNOWN
```

---

# 73. Aggregate analysis

El Semantic Engine identificará:

```text
aggregate expressions
aggregate scopes
grouping keys
aggregate dependencies
```

---

# 74. Ejemplo inválido

```sql
SELECT department, salary
FROM employees
GROUP BY department
```

Si `salary` no está agregado ni funcionalmente determinado según las reglas aplicables, deberá producir error semántico.

---

# 75. Aggregate analysis ≠ compiler

No deberá simplemente esperar que el servidor DB rechace la consulta.

---

# 76. Grouping semantic model

Podrá contener:

```text
GroupingSemanticInfo
├── groupingExpressions
├── aggregateExpressions
├── groupedSymbols
├── functionalDependencies
└── violations
```

---

# 77. Aggregate function

Un FunctionDescriptor podrá declarar:

```text
aggregate = true
```

pero el análisis de dónde puede utilizarse pertenece al Aggregate Analyzer.

---

# 78. Nested aggregates

Ejemplo:

```text
SUM(AVG(price))
```

podrá ser inválido dentro del mismo aggregate scope.

---

# 79. Window functions

Se distinguirán de aggregates normales.

```text
Aggregate Function
≠
Windowed Aggregate
≠
Window Function
```

---

# 80. Window semantic analysis

Resolverá:

- named windows;
- partition expressions;
- ordering expressions;
- frame semantics;
- window function eligibility;
- window references.

---

# 81. Join analysis

Se integra con:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

---

# 82. Join semantic information

Podrá incluir:

```text
left relation set
right relation set
join kind
join predicate
dependencies
equi-join keys
null-preserving side
null-supplying side
correlation
constraints
```

---

# 83. Outer join semantics

Crítico para optimizer.

Ejemplo:

```text
LEFT JOIN
```

no puede tratarse como:

```text
INNER JOIN
```

al mover predicates.

---

# 84. Null-rejecting predicates

El Semantic Engine podrá identificar predicates que rechazan NULL respecto a determinados symbols.

Esto permitirá optimizaciones posteriores seguras.

---

# 85. Join equivalence

Ejemplo:

```text
users.id = orders.user_id
```

podrá generar:

```text
EquivalenceConstraint(
    users.id,
    orders.user_id
)
```

---

# 86. Relation resolution

El Semantic Engine distinguirá:

```text
PhysicalRelation
DerivedRelation
CteRelation
ValuesRelation
FunctionRelation
ExtensionRelation
```

---

# 87. Relation output schema

Toda relation deberá exponer conceptualmente:

```text
ordered output columns
types
nullability
symbol IDs
lineage
```

---

# 88. Projection semantics

La projection transformará input relations en:

```text
OutputRelation
```

---

# 89. Output column

Conceptualmente:

```text
OutputColumn
├── outputSymbolId
├── name
├── expression
├── type
├── nullable
└── lineage
```

---

# 90. Wildcard expansion

La expansión semántica de:

```text
*
```

podrá ocurrir aquí cuando exista schema information suficiente.

---

# 91. Wildcard normalization distinction

Normalization no deberá convertir:

```text
*
```

en columnas físicas sin conocer schema.

Semantic Analysis sí puede hacerlo.

---

# 92. Qualified wildcard

```text
u.*
```

se expandirá utilizando la relation resuelta.

---

# 93. Projection order

La expansión deberá preservar orden semántico definido por la relation.

---

# 94. CTE analysis

Los CTEs requieren:

```text
name resolution
scope creation
dependency analysis
output schema derivation
recursive analysis
```

---

# 95. CTE dependency graph

Conceptualmente:

```text
CteDependencyGraph
```

---

# 96. Non-recursive cycle

Un ciclo no autorizado deberá rechazarse.

---

# 97. Recursive CTE

Un recursive CTE podrá requerir:

```text
anchor query
recursive query
column compatibility
self-reference rules
termination-related metadata
capability requirements
```

---

# 98. Semantic recursion ≠ AST cycle

Se mantiene la invariant del documento 34:

```text
AST object graph cycle
→ invalid

semantic recursive relation reference
→ potentially valid
```

---

# 99. Correlation

Una subquery puede referenciar symbols del scope externo.

---

# 100. Correlated reference

Ejemplo:

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

`u.id` dentro de la subquery será:

```text
CorrelatedSymbolReference
```

---

# 101. Correlation metadata

Podrá registrar:

```text
innerScope
outerScope
symbol
correlationDepth
usage
```

---

# 102. Why correlation matters

El Planner podrá decidir posteriormente:

```text
nested execution
semi join
anti join
decorrelation
other strategy
```

---

# 103. Semantic Engine does not decorrelate

Detecta y describe la correlation.

La transformación pertenece al Optimizer/Planner.

---

# 104. Set operations

El Semantic Engine analizará:

```text
UNION
INTERSECT
EXCEPT
```

---

# 105. Set operation requirements

Branches deberán producir output relations compatibles.

---

# 106. Arity

Después de wildcard expansion/resolution deberá conocerse la arity real.

---

# 107. Type compatibility

Por posición:

```text
left.column[i]
↔
right.column[i]
```

deberá encontrarse un tipo semántico compatible.

---

# 108. Output type

Podrá derivarse mediante:

```text
least common compatible type
```

según Query Type System.

---

# 109. Output nullability

Será derivada de ambos branches.

---

# 110. INSERT semantics

El Semantic Engine resolverá:

```text
target relation
target columns
source values/query
assignment compatibility
defaults
generated columns
conflict target
returning output
```

---

# 111. INSERT column mapping

Ejemplo:

```text
INSERT users(name, age)
VALUES (:name, :age)
```

produce:

```text
P1 → users.name type constraint
P2 → users.age type constraint
```

---

# 112. Generated columns

El SchemaView podrá indicar:

```text
generated
identity
defaulted
required
nullable
```

---

# 113. Missing required columns

Podrá detectarse semánticamente.

---

# 114. UPDATE semantics

Resolverá:

```text
target relation
assignment targets
assignment types
predicate symbols
joins
returning
```

---

# 115. Assignment compatibility

```text
column type
↔
expression type
```

deberá ser compatible.

---

# 116. DELETE semantics

Resolverá:

```text
target relation
visible sources
join dependencies
predicate
returning
```

---

# 117. Mutation safety

La policy estructural ya pudo aplicarse en Validation.

Semantic Analysis podrá añadir reglas basadas en conocimiento real del target cuando sea necesario.

---

# 118. Constraints

El Semantic Engine derivará:

```text
SemanticConstraintSet
```

---

# 119. Constraint examples

```text
EqualityConstraint
RangeConstraint
NullConstraint
NotNullConstraint
TypeConstraint
EquivalenceConstraint
UniquenessConstraint
FunctionalDependency
JoinConstraint
ConstantConstraint
ParameterConstraint
```

---

# 120. Constraint ≠ database constraint

No confundir:

```text
Semantic Query Constraint
```

con:

```text
FOREIGN KEY
UNIQUE
CHECK
PRIMARY KEY
```

aunque estos últimos puedan contribuir información.

---

# 121. Schema constraints as evidence

Ejemplo:

```text
PRIMARY KEY(users.id)
```

puede aportar:

```text
users.id → unique
users.id → not null
```

---

# 122. Foreign key evidence

Puede aportar relación potencial:

```text
orders.user_id → users.id
```

pero no implica automáticamente que todo join sea correcto o necesario.

---

# 123. Functional dependencies

Podrán derivarse de:

- primary keys;
- unique constraints;
- grouping;
- equality predicates;
- relation semantics.

---

# 124. Why constraints matter

El Optimizer podrá utilizarlos para:

- predicate simplification;
- join elimination;
- constant propagation;
- redundant condition detection;
- cardinality reasoning;
- grouping reasoning.

---

# 125. Constraint analysis must be conservative

Nunca inferir:

```text
constraint
```

si no puede demostrarse.

---

# 126. Semantic Query Graph

La representación final central será:

```text
SemanticQueryGraph
```

---

# 127. AST tree vs Semantic graph

El AST suele ser:

```text
tree-oriented
```

La semántica naturalmente forma un:

```text
graph
```

porque múltiples nodes pueden depender del mismo symbol/relation/parameter.

---

# 128. Ejemplo

```text
            Relation users
             /      \
            /        \
       users.id    users.id
          │            │
       predicate    projection
```

Ambas referencias apuntan al mismo `ColumnSymbol`.

---

# 129. Semantic graph nodes

Podrán representar:

```text
Query
Scope
Relation
Column
Expression
Predicate
Parameter
Function
Operator
Aggregate
Window
Constraint
Output
```

---

# 130. Semantic graph edges

Ejemplos:

```text
CONTAINS
REFERENCES
DEPENDS_ON
DERIVES_FROM
JOINS
CORRELATES_WITH
CONSTRAINS
OUTPUTS
GROUPS_BY
ORDERS_BY
EQUIVALENT_TO
```

---

# 131. Semantic Graph ≠ Execution Plan

No contendrá todavía:

```text
index scan
hash join
nested loop
physical operator
connection endpoint
prepared statement
```

---

# 132. Semantic Graph ≠ AST replacement

El AST continuará existiendo.

La relación será:

```text
AST
+
Semantic Graph
+
Semantic Tables
=
SemanticQueryArtifact
```

---

# 133. Node lineage

Cada semantic element deberá poder rastrearse hacia:

```text
AST NodeId
```

cuando corresponda.

---

# 134. Data lineage

Además podrá existir:

```text
DataLineage
```

Ejemplo:

```text
output.email_lower
    ↓
LOWER(users.email)
    ↓
users.email
```

---

# 135. Why lineage

Útil para:

- optimizer;
- diagnostics;
- authorization integrations;
- telemetry;
- result metadata;
- debugging;
- future static analysis.

---

# 136. Query dependencies

El Semantic Engine producirá:

```text
QueryDependencySet
```

---

# 137. Dependency kinds

```text
TABLE
COLUMN
CTE
FUNCTION
TYPE
SEQUENCE
CONSTRAINT
EXTENSION
CAPABILITY
```

---

# 138. Dependency fingerprint

Podrá utilizarse posteriormente para:

- cache invalidation;
- compiled query invalidation;
- schema change detection;
- warmup.

---

# 139. Capability requirements

La capa semántica no necesariamente decidirá si una capability está disponible.

Pero sí podrá derivar:

```text
CapabilityRequirementSet
```

---

# 140. Ejemplo

Una query que utiliza:

```text
RIGHT JOIN
```

podrá declarar:

```text
requires JOIN.RIGHT
```

---

# 141. Otro ejemplo

```text
RETURNING
```

podrá declarar:

```text
requires DML.RETURNING
```

---

# 142. Capability requirement ≠ capability resolution

Separación:

```text
Semantic Engine
→ "This query requires X"

Capability Resolver
→ "Target supports X"

Planner
→ "Use X or valid alternative"
```

---

# 143. Emulation

Si una capability puede emularse, la capa semántica no deberá realizar la emulación.

Podrá declarar:

```text
Requirement
```

y permitir que Planner/Compiler seleccione estrategia.

---

# 144. Portability

Cada semantic feature podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 145. Portability aggregation

La query completa podrá obtener:

```text
QueryPortabilityProfile
```

derivado de sus elementos.

---

# 146. Semantic diagnostics

Ejemplos:

```text
Unknown relation "u"
Unknown column "email"
Ambiguous column "id"
Parameter :id has incompatible inferred types
Function LOWER cannot accept JSON
UNION branches have incompatible arity
Aggregate expression violates grouping rules
Recursive CTE reference is invalid
```

---

# 147. Stable diagnostic codes

Ejemplos:

```text
DB_SEM_UNKNOWN_RELATION
DB_SEM_UNKNOWN_COLUMN
DB_SEM_AMBIGUOUS_COLUMN
DB_SEM_TYPE_MISMATCH
DB_SEM_PARAMETER_TYPE_CONFLICT
DB_SEM_UNKNOWN_FUNCTION
DB_SEM_INVALID_OPERATOR
DB_SEM_INVALID_GROUPING
DB_SEM_INVALID_CORRELATION
DB_SEM_SET_ARITY_MISMATCH
```

---

# 148. Diagnostics and source mapping

Se reutilizará:

```text
NormalizationSourceMap
+
Validation paths
+
AST NodeId
```

para producir mensajes precisos.

---

# 149. Example

```text
Unknown column "emali" in relation "users".
Did you mean "email"?
```

---

# 150. Suggestions

Las sugerencias deberán ser:

- bounded;
- deterministic;
- no destructivas;
- opcionales.

---

# 151. Symbol similarity

Podrá utilizarse sólo para diagnostics.

Nunca para resolver silenciosamente:

```text
emali
→ email
```

---

# 152. Semantic ambiguity

VoltStack preferirá:

```text
explicit error
```

sobre una resolución arbitraria.

---

# 153. Example

```sql
SELECT id
FROM users
JOIN orders ...
```

No:

```text
"users.id seems more likely"
```

Debe ser:

```text
AmbiguousColumnReference
```

---

# 154. QuerySemanticPolicy

Algunas reglas semánticas podrán configurarse.

Ejemplos:

```text
strictGrouping
allowImplicitNumericWidening
allowUnknownFunctions
allowVendorSemanticExtensions
strictAliasVisibility
```

---

# 155. Invariant ≠ policy

Como en Validation:

```text
semantic invariant
≠
configurable semantic policy
```

---

# 156. Example invariant

Una referencia debe resolver inequívocamente.

No deberá poder desactivarse.

---

# 157. Example policy

Permitir determinado tipo de coerción implícita puede ser configurable.

---

# 158. Coercion

Toda coerción semántica deberá ser explícitamente modelada.

---

# 159. No hidden PHP coercion

Nunca:

```php
"10" == 10
```

como fundamento de compatibilidad SQL.

---

# 160. Coercion graph

El Query Type System podrá proporcionar:

```text
QueryTypeCoercionGraph
```

---

# 161. Coercion categories

```text
EXACT
SAFE_IMPLICIT
CONTEXTUAL
EXPLICIT_ONLY
UNSAFE
FORBIDDEN
```

---

# 162. Semantic cast insertion

Si el sistema necesita representar una coerción implícita, no deberá mutar el AST original.

Podrá representarla mediante:

```text
SemanticCoercion
```

asociada al node/edge.

---

# 163. Optimizer visibility

Las coerciones deberán ser visibles al Optimizer y Compiler.

---

# 164. Semantic rewrite?

La capa semántica deberá evitar convertirse en un segundo Normalizer.

Sin embargo, puede producir:

```text
resolved semantic representation
```

que haga explícitas decisiones semánticas sin modificar el AST.

---

# 165. Example

AST:

```text
P1
```

Semantic model:

```text
P1
└── inferredType = UUID
```

No necesita reescribir el ParameterNode.

---

# 166. Semantic IDs

Se utilizarán identidades especializadas:

```text
ScopeId
SymbolId
RelationId
SemanticExpressionId
ConstraintId
OutputColumnId
```

cuando aporten claridad.

---

# 167. IDs deterministic?

Cuando sea útil para fingerprints/cache, podrán derivarse determinísticamente del traversal/canonical structure.

No deberán depender de:

```text
random UUID
process ID
memory address
```

---

# 168. Semantic fingerprint

Podrá existir:

```text
SemanticQueryFingerprint
```

---

# 169. Semantic fingerprint inputs

Conceptualmente:

```text
Normalized Query Fingerprint
+
Resolved Symbol Structure
+
Resolved Query Types
+
Schema Semantic Version
+
Semantic Extension Set
+
Semantic Policy
```

---

# 170. No runtime values

No deberá incluir normalmente:

```text
parameter values
credentials
connection handles
```

---

# 171. Schema semantic version

`SchemaView` deberá proporcionar identidad/versionado suficiente para determinar cuándo un artifact semántico deja de ser válido.

---

# 172. Schema changes

Ejemplo:

```text
users.age INTEGER
→
users.age VARCHAR
```

deberá invalidar artifacts semánticos dependientes.

---

# 173. Dependency-aware invalidation

Idealmente no será necesario invalidar todo el universo.

Podrá utilizarse:

```text
QueryDependencySet
```

para invalidación selectiva futura.

---

# 174. Caching

Semantic artifacts podrán ser cacheables cuando:

```text
query shape
schema view
semantic policy
extension set
```

sean estables.

---

# 175. Cache boundary

El Semantic Engine no deberá depender obligatoriamente de:

```text
Quantum/Cache
```

---

# 176. Optional cache port

Podrá existir:

```text
SemanticArtifactCachePort
```

---

# 177. Cache safety

Nunca cachear:

- request-scoped context;
- tenant context no fingerprinted;
- mutable SchemaView;
- runtime parameter values;
- connection state.

---

# 178. Multitenancy

Database core no dependerá de Multitenancy.

---

# 179. Tenant integration

Un paquete Multitenancy podrá proporcionar:

```text
TenantAwareSchemaView
```

o resolver el SchemaView antes de entrar al Semantic Engine.

---

# 180. Critical rule

No:

```php
Tenant::current()
```

dentro del Semantic Analyzer.

---

# 181. Authorization integration

Authorization tampoco será dependencia core.

---

# 182. Column/resource policy

Si Authorization necesita conocer semantic lineage, podrá consumir:

```text
SemanticQueryArtifact
```

mediante integration layer.

---

# 183. Security policies before/after semantics

Algunas políticas pueden ejecutarse:

```text
before semantic analysis
```

y otras requieren:

```text
resolved semantic symbols
```

---

# 184. Example

Antes:

```text
raw SQL forbidden
```

Después:

```text
query accesses protected column X
```

---

# 185. No circular dependency

Evitar:

```text
Database Semantic Engine
→ Authorization
→ Database ORM
→ Semantic Engine
```

---

# 186. Integration ports

Preferir:

```text
SemanticQueryPolicyPort
```

cuando realmente se requiera integración.

---

# 187. ORM integration

ORM podrá producir Query AST utilizando entity metadata.

Pero una vez generado:

```text
ORM
  ↓
Query AST
  ↓
Semantic Query Engine
```

la capa semántica no deberá depender del UnitOfWork.

---

# 188. Entity metadata

Si es necesario mapear entity fields a database columns, esa transformación deberá ocurrir en una frontera definida antes o mediante un neutral mapping view.

---

# 189. Query Engine remains ORM-independent

Debe poder ejecutar:

```text
DB::table(...)
```

sin instalar ORM completo.

---

# 190. Persistent runtime

La arquitectura deberá ser segura para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 191. Shared semantic services

Podrán ser app/worker-scoped:

```text
SemanticAnalysisCoordinator
FrozenFunctionRegistry
FrozenOperatorRegistry
FrozenSemanticExtensionRegistry
QueryTypeRules
```

si son:

```text
immutable
stateless
reentrant
```

---

# 192. Operation-scoped semantic state

Debe incluir:

```text
SemanticAnalysisState
ScopeBuilderState
SymbolResolutionState
TypeConstraintState
DiagnosticCollector
ConstraintBuilder
SemanticGraphBuilder
BudgetTracker
```

---

# 193. No global current scope

Prohibido:

```php
SemanticAnalyzer::$currentScope;
```

---

# 194. No global current schema

Prohibido.

---

# 195. No global symbol table mutation

Cada analysis operation tendrá su propio semantic artifact.

---

# 196. FrankenPHP model

```text
Worker
│
├── Frozen Semantic Services
│
├── Request A
│   └── SemanticAnalysisState A
│
├── Request B
│   └── SemanticAnalysisState B
│
└── Request C
    └── SemanticAnalysisState C
```

---

# 197. OpenSwoole model

```text
Coroutine A ── SemanticState A
Coroutine B ── SemanticState B
Coroutine C ── SemanticState C
```

---

# 198. Analysis budget

Existirá:

```text
SemanticAnalysisBudget
```

---

# 199. Budget examples

```text
maxScopes
maxSymbols
maxRelations
maxTypeConstraints
maxConstraintIterations
maxFunctionCandidates
maxOperatorCandidates
maxCorrelationDepth
maxCteDependencies
maxSemanticGraphNodes
maxSemanticGraphEdges
maxDiagnostics
```

---

# 200. Why bounded analysis

Queries generadas dinámicamente podrían provocar:

- huge symbol graphs;
- pathological type inference;
- recursive CTE analysis explosion;
- extension resolution explosion;
- excessive memory.

---

# 201. Semantic budget exceeded

Debe producir un error de dominio:

```text
SemanticAnalysisBudgetExceededException
```

no worker failure.

---

# 202. Fixpoint analysis

Algunas inferencias podrán requerir iteración hasta:

```text
fixpoint
```

---

# 203. Example

Constraints:

```text
P1 = C1
C1 compatible C2
C2 = INTEGER
```

pueden propagar:

```text
P1 → INTEGER
```

---

# 204. Iteration bound

Todo fixpoint algorithm deberá tener:

```text
bounded iterations
```

o criterio formal de convergencia.

---

# 205. Non-convergence

Debe ser error interno/semantic complexity error, nunca loop infinito.

---

# 206. Extension architecture

Extensiones podrán aportar:

```text
SemanticNodeAnalyzer
FunctionDescriptor
OperatorDescriptor
TypeInferenceRule
RelationResolver
ConstraintRule
CapabilityRequirementProvider
SemanticGraphContributor
```

---

# 207. Extension completeness

Un extension node deberá proporcionar todos los handlers requeridos por las fases que atraviese.

---

# 208. Registration

Ocurrirá durante bootstrap.

---

# 209. Frozen registries

Antes de runtime:

```text
FunctionRegistry
OperatorRegistry
SemanticRuleRegistry
RelationResolverRegistry
SemanticExtensionRegistry
```

deberán quedar frozen.

---

# 210. Conflict handling

No:

```text
last registered wins
```

---

# 211. Function conflicts

Dos extensiones registrando la misma firma deberán:

```text
fail bootstrap
```

o utilizar un mecanismo explícito de override autorizado.

---

# 212. Semantic extension priority

Si existe prioridad, deberá ser:

```text
explicit
deterministic
diagnosable
```

---

# 213. Capability-driven architecture

No usar:

```php
if ($database === 'postgres') {
}
```

dentro del semantic core.

---

# 214. Semantic feature requirements

Preferir:

```text
feature → capability requirement
```

---

# 215. Platform-aware semantics

Existen casos donde la semántica real depende del target Platform.

Estos deberán modelarse mediante:

```text
PlatformSemanticProfile
```

o contracts equivalentes, no mediante conditionals dispersos.

---

# 216. Example

Identifier case-folding puede variar.

El resolver podrá utilizar:

```text
IdentifierSemanticPolicy
```

derivada del Platform profile.

---

# 217. Important separation

```text
Platform Semantic Rules
≠
SQL Dialect Syntax
```

---

# 218. Semantic profile

Puede incluir:

```text
identifier semantics
type compatibility extensions
function semantics
operator semantics
null behavior extensions
collation semantics
platform capabilities
```

---

# 219. Dialect not needed for meaning

Idealmente Semantic Analysis no necesita el Dialect completo.

Sólo un perfil semántico neutral cuando las diferencias del target sean relevantes.

---

# 220. Query portability

Si no existe target Platform todavía, el análisis podrá operar en:

```text
PORTABLE_SEMANTIC_MODE
```

---

# 221. Portable mode

Podrá producir:

```text
unresolved platform requirements
```

sin rechazar la query prematuramente.

---

# 222. Targeted mode

Cuando exista target:

```text
TARGETED_SEMANTIC_MODE
```

podrá aplicar reglas semánticas específicas del Platform.

---

# 223. Two-stage semantic analysis

La arquitectura deberá permitir conceptualmente:

```text
Portable Semantic Analysis
        │
        ▼
Target Semantic Refinement
```

cuando resulte necesario.

---

# 224. Avoid duplicate analysis

El segundo stage deberá refinar, no rehacer arbitrariamente todo el trabajo.

---

# 225. SemanticQueryArtifact

Estructura conceptual:

```text
SemanticQueryArtifact
├── validatedQuery
├── semanticGraph
├── scopeGraph
├── symbolTable
├── relationTable
├── expressionSemanticTable
├── predicateSemanticTable
├── parameterSemanticTable
├── outputRelation
├── constraintSet
├── dependencySet
├── capabilityRequirements
├── portabilityProfile
├── semanticFingerprint
└── diagnosticsSummary
```

---

# 226. Artifact immutable

Una vez finalizado:

```text
SemanticQueryArtifact
```

será inmutable.

---

# 227. Semantic graph builder

Durante análisis podrá existir:

```text
Mutable SemanticGraphBuilder
```

operation-scoped.

Después:

```text
freeze()
```

producirá:

```text
Immutable SemanticQueryGraph
```

---

# 228. Same pattern for tables

```text
SymbolTableBuilder
→ freeze
→ SemanticSymbolTable
```

---

# 229. Same for constraints

```text
ConstraintSetBuilder
→ freeze
→ SemanticConstraintSet
```

---

# 230. Error strategy

Un error semántico deberá detener la producción del artifact final válido.

---

# 231. Collect mode

Desarrollo podrá recolectar varios errores cuando uno no invalide la posibilidad de continuar el análisis local.

---

# 232. Error recovery

El analyzer podrá introducir internamente:

```text
ErrorSemanticType
UnresolvedSymbol
ErrorRelation
```

para continuar diagnostics.

---

# 233. Error placeholders

No deberán escapar a:

```text
VALID SemanticQueryArtifact
```

---

# 234. Finalization invariant

Antes de `freeze()`:

```text
no unresolved required symbols
no unresolved required types
no unresolved required relations
no fatal semantic diagnostics
```

---

# 235. Unknown ≠ unresolved

Debe distinguirse.

Ejemplo:

```text
UnknownQueryType
```

puede ser una representación legítima en determinadas fases.

Mientras:

```text
UnresolvedRequiredColumnSymbol
```

es failure.

---

# 236. Semantic state lifecycle

```text
CREATED
   │
   ▼
SCOPES_BUILT
   │
   ▼
SYMBOLS_REGISTERED
   │
   ▼
REFERENCES_RESOLVED
   │
   ▼
TYPES_RESOLVED
   │
   ▼
RELATIONS_ANALYZED
   │
   ▼
CONSTRAINTS_DERIVED
   │
   ▼
GRAPH_FINALIZED
   │
   ▼
FROZEN
```

---

# 237. Failure state

Cualquier fase podrá producir:

```text
FAILED
```

con cleanup determinista.

---

# 238. Performance priorities

```text
Correctness
>
Semantic determinism
>
Bounded complexity
>
Reusable metadata
>
Low allocation
>
Cacheability
```

---

# 239. Avoid repeated schema lookup

Resolver:

```text
users.email
```

cientos de veces no deberá provocar cientos de lookups costosos.

---

# 240. Operation-local lookup cache

Puede existir:

```text
SemanticResolutionCache
```

para:

```text
relation lookup
column lookup
function candidate lookup
type compatibility
```

---

# 241. No stale global cache

Un cache global deberá incluir schema/version/profile en su identidad.

---

# 242. Precompiled metadata

En producción podrán precompilarse:

```text
function descriptors
operator descriptors
type rules
schema descriptors
extension dispatch tables
```

---

# 243. No reflection hot path

Reflection y discovery deberán ocurrir durante bootstrap/warmup cuando sea posible.

---

# 244. Observability

Métricas posibles:

```text
database.semantic.duration
database.semantic.scopes
database.semantic.symbols
database.semantic.relations
database.semantic.constraints
database.semantic.failures
database.semantic.cache_hits
```

---

# 245. Low cardinality

No usar como labels:

```text
table names
column names
tenant IDs
query fingerprints
parameter values
```

por defecto.

---

# 246. Tracing

Span opcional:

```text
database.query.semantic_analysis
```

---

# 247. Phase spans

Development/profiling puede exponer:

```text
semantic.scope
semantic.symbol_resolution
semantic.type_inference
semantic.relation_resolution
semantic.constraint_analysis
semantic.graph_finalize
```

---

# 248. No mandatory Telemetry

La integración será mediante port opcional.

---

# 249. Testing

El bloque requerirá:

```text
Semantic Unit Tests
Scope Resolution Tests
Symbol Resolution Tests
Schema Resolution Tests
Type Inference Tests
Function Resolution Tests
Operator Resolution Tests
Relation Tests
Join Tests
Aggregate Tests
Correlation Tests
Constraint Tests
CTE Tests
Set Operation Tests
Persistent Runtime Tests
Concurrency Tests
Extension Conformance Tests
Cross-Platform Semantic Tests
```

---

# 250. Property-based testing

Especialmente útil para:

```text
scope nesting
symbol shadowing
type constraint propagation
nullability propagation
constraint closure
```

---

# 251. Semantic fixture

Podrá existir:

```text
SemanticTestSchema
```

in-memory para tests.

---

# 252. No real database required

La mayoría de semantic tests deberán ejecutarse sin MySQL/PostgreSQL/SQLite.

---

# 253. Driver conformance not here

Los tests de Driver pertenecen al Driver subsystem.

---

# 254. Platform semantic conformance

Sí podrán existir suites para comprobar que los semantic profiles de cada Platform respeten los contracts.

---

# 255. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Semantic\
```

---

# 256. Estructura propuesta

```text
Query/
└── Semantic/
    ├── Contract/
    │   ├── SemanticQueryAnalyzerInterface.php
    │   ├── SemanticNodeAnalyzerInterface.php
    │   ├── SymbolResolverInterface.php
    │   ├── RelationResolverInterface.php
    │   ├── FunctionResolverInterface.php
    │   ├── OperatorResolverInterface.php
    │   └── SemanticDiagnosticSinkInterface.php
    │
    ├── Core/
    │   ├── SemanticAnalysisCoordinator.php
    │   ├── SemanticQueryArtifact.php
    │   ├── SemanticAnalysisState.php
    │   └── SemanticAnalysisResult.php
    │
    ├── Context/
    │   ├── SemanticQueryContext.php
    │   ├── QuerySemanticPolicy.php
    │   ├── SemanticAnalysisMode.php
    │   └── PlatformSemanticProfile.php
    │
    ├── Scope/
    │   ├── ScopeId.php
    │   ├── SemanticScope.php
    │   ├── SemanticScopeGraph.php
    │   ├── SemanticScopeBuilder.php
    │   └── ScopeResolver.php
    │
    ├── Symbol/
    │   ├── SymbolId.php
    │   ├── SemanticSymbol.php
    │   ├── SemanticSymbolTable.php
    │   ├── SymbolTableBuilder.php
    │   └── SymbolResolver.php
    │
    ├── Relation/
    │   ├── RelationId.php
    │   ├── RelationSymbol.php
    │   ├── RelationSemanticInfo.php
    │   ├── RelationSemanticTable.php
    │   ├── OutputRelation.php
    │   └── OutputColumn.php
    │
    ├── Schema/
    │   ├── SchemaView.php
    │   ├── SchemaRelationView.php
    │   ├── SchemaColumnView.php
    │   └── SchemaSemanticVersion.php
    │
    ├── Expression/
    │   ├── ExpressionSemanticInfo.php
    │   ├── ExpressionSemanticTable.php
    │   └── ExpressionSemanticAnalyzer.php
    │
    ├── Predicate/
    │   ├── PredicateSemanticInfo.php
    │   ├── PredicateSemanticTable.php
    │   └── PredicateSemanticAnalyzer.php
    │
    ├── Parameter/
    │   ├── ParameterSemanticInfo.php
    │   └── ParameterSemanticTable.php
    │
    ├── Type/
    │   ├── QueryTypeInferenceEngine.php
    │   ├── TypeConstraint.php
    │   ├── TypeConstraintSet.php
    │   ├── QueryTypeCoercionGraph.php
    │   └── SemanticCoercion.php
    │
    ├── Function/
    │   ├── FunctionDescriptor.php
    │   ├── FunctionSignature.php
    │   ├── FunctionRegistry.php
    │   └── FunctionResolver.php
    │
    ├── Operator/
    │   ├── OperatorDescriptor.php
    │   ├── OperatorRegistry.php
    │   └── OperatorResolver.php
    │
    ├── Join/
    │   ├── JoinSemanticAnalyzer.php
    │   ├── JoinSemanticInfo.php
    │   └── JoinEquivalenceAnalyzer.php
    │
    ├── Aggregate/
    │   ├── AggregateSemanticAnalyzer.php
    │   ├── GroupingSemanticInfo.php
    │   └── AggregateScope.php
    │
    ├── Correlation/
    │   ├── CorrelationAnalyzer.php
    │   ├── CorrelatedReference.php
    │   └── CorrelationInfo.php
    │
    ├── Constraint/
    │   ├── ConstraintId.php
    │   ├── SemanticConstraint.php
    │   ├── SemanticConstraintSet.php
    │   ├── ConstraintSetBuilder.php
    │   └── QueryConstraintAnalyzer.php
    │
    ├── Dependency/
    │   ├── QueryDependency.php
    │   ├── QueryDependencySet.php
    │   └── QueryDependencyAnalyzer.php
    │
    ├── Capability/
    │   ├── CapabilityRequirement.php
    │   ├── CapabilityRequirementSet.php
    │   └── CapabilityRequirementDeriver.php
    │
    ├── Graph/
    │   ├── SemanticQueryGraph.php
    │   ├── SemanticGraphNode.php
    │   ├── SemanticGraphEdge.php
    │   └── SemanticQueryGraphBuilder.php
    │
    ├── Lineage/
    │   ├── DataLineage.php
    │   └── LineageAnalyzer.php
    │
    ├── Budget/
    │   ├── SemanticAnalysisBudget.php
    │   └── SemanticAnalysisBudgetTracker.php
    │
    ├── Extension/
    │   ├── SemanticExtensionRegistry.php
    │   ├── SemanticExtensionDescriptor.php
    │   └── SemanticExtensionConformanceSuite.php
    │
    ├── Diagnostic/
    │   ├── SemanticDiagnostic.php
    │   ├── SemanticDiagnosticCode.php
    │   └── SemanticDiagnosticRenderer.php
    │
    └── Exception/
        ├── SemanticQueryException.php
        ├── UnknownRelationException.php
        ├── UnknownColumnException.php
        ├── AmbiguousColumnException.php
        ├── QueryTypeMismatchException.php
        ├── UnknownFunctionException.php
        ├── InvalidOperatorException.php
        ├── InvalidGroupingException.php
        ├── InvalidCorrelationException.php
        ├── SemanticAnalysisBudgetExceededException.php
        └── SemanticInvariantViolationException.php
```

---

# 257. Dependency direction

```text
Query AST
   │
   ▼
Validation
   │
   ▼
Semantic Engine
   │
   ├── Query Type System
   ├── SchemaView
   ├── Function Registry
   ├── Operator Registry
   └── Semantic Extensions
   │
   ▼
SemanticQueryArtifact
   │
   ▼
Optimizer
   │
   ▼
Planner
```

---

# 258. Forbidden dependency direction

Nunca:

```text
Semantic Engine
    │
    ├──► SQL Compiler
    ├──► Query Executor
    ├──► Connection Manager
    ├──► Connection Pool
    ├──► PDO
    ├──► EntityManager
    └──► UnitOfWork
```

---

# 259. SEM-001

El Semantic Query Engine recibirá únicamente queries estructuralmente válidas.

---

# 260. SEM-002

El Semantic Engine no modificará el Query AST original.

---

# 261. SEM-003

La información semántica residirá en artifacts/tables separados.

---

# 262. SEM-004

Cada symbol tendrá identidad semántica independiente de su nombre textual.

---

# 263. SEM-005

Cada relation visible tendrá identidad semántica propia.

---

# 264. SEM-006

La misma tabla física podrá originar múltiples relation symbols.

---

# 265. SEM-007

Una referencia deberá resolver de manera inequívoca.

---

# 266. SEM-008

La ambigüedad nunca se resolverá mediante heurísticas silenciosas.

---

# 267. SEM-009

Los scopes serán explícitos.

---

# 268. SEM-010

Las correlated references serán explícitas.

---

# 269. SEM-011

Semantic recursion no será confundida con AST cycles.

---

# 270. SEM-012

Query Type inference utilizará semantic types, no tipos PHP.

---

# 271. SEM-013

Runtime parameter values no serán necesarios para inferencia normal.

---

# 272. SEM-014

Type conflicts producirán errores explícitos.

---

# 273. SEM-015

Las coerciones serán modeladas explícitamente.

---

# 274. SEM-016

No existirán coerciones basadas accidentalmente en PHP.

---

# 275. SEM-017

Schema nullability y query nullability serán conceptos distintos.

---

# 276. SEM-018

Outer joins podrán modificar nullability semántica.

---

# 277. SEM-019

Functions se resolverán mediante semantic descriptors.

---

# 278. SEM-020

Operators se resolverán mediante semantic descriptors.

---

# 279. SEM-021

SQL syntax no será responsabilidad de function/operator semantic resolution.

---

# 280. SEM-022

SQL three-valued logic será preservada.

---

# 281. SEM-023

Volatility será parte de la información semántica.

---

# 282. SEM-024

Determinism podrá modelarse independientemente.

---

# 283. SEM-025

Aggregate scopes serán explícitos.

---

# 284. SEM-026

Grouping violations deberán detectarse antes de SQL execution.

---

# 285. SEM-027

Window semantics serán distintas de aggregate semantics.

---

# 286. SEM-028

Join semantics deberán preservar outer-join behavior.

---

# 287. SEM-029

Null-rejection properties podrán derivarse para optimización segura.

---

# 288. SEM-030

Wildcard expansion dependiente de schema ocurrirá en Semantic Analysis, no Normalization.

---

# 289. SEM-031

Derived relations expondrán output schema semántico.

---

# 290. SEM-032

CTEs serán relations semánticas.

---

# 291. SEM-033

Set operations deberán resolver output arity y types.

---

# 292. SEM-034

INSERT deberá validar compatibilidad target/source.

---

# 293. SEM-035

UPDATE deberá validar compatibilidad assignment/target.

---

# 294. SEM-036

Semantic constraints estarán separados de database schema constraints.

---

# 295. SEM-037

Schema constraints podrán aportar evidencia semántica.

---

# 296. SEM-038

Constraint inference será conservadora.

---

# 297. SEM-039

Semantic Query Graph no será un Execution Plan.

---

# 298. SEM-040

Semantic Query Graph no reemplazará al AST.

---

# 299. SEM-041

Semantic elements conservarán lineage hacia AST cuando corresponda.

---

# 300. SEM-042

Data lineage podrá representarse explícitamente.

---

# 301. SEM-043

Query dependencies serán explícitas.

---

# 302. SEM-044

Capability requirements serán derivadas, no resueltas mediante vendor checks.

---

# 303. SEM-045

Semantic Engine podrá operar sin Connection física.

---

# 304. SEM-046

SchemaView será una dependencia de lectura, no un Schema Manager mutable.

---

# 305. SEM-047

Semantic Engine no ejecutará introspection implícita.

---

# 306. SEM-048

Semantic Engine será compatible con análisis offline.

---

# 307. SEM-049

Semantic Engine no dependerá de ORM.

---

# 308. SEM-050

Semantic Engine no dependerá de Multitenancy.

---

# 309. SEM-051

Semantic Engine no dependerá de Authorization.

---

# 310. SEM-052

Semantic Engine no dependerá obligatoriamente de Cache.

---

# 311. SEM-053

Semantic Engine no dependerá obligatoriamente de Telemetry.

---

# 312. SEM-054

Shared semantic services deberán ser immutable/stateless.

---

# 313. SEM-055

Analysis state será operation-scoped.

---

# 314. SEM-056

No existirá global current scope.

---

# 315. SEM-057

No existirá global current schema.

---

# 316. SEM-058

No existirá global mutable symbol table.

---

# 317. SEM-059

El sistema será seguro para persistent workers.

---

# 318. SEM-060

El sistema será seguro para concurrent/coroutine execution.

---

# 319. SEM-061

Semantic analysis tendrá resource budgets.

---

# 320. SEM-062

Fixpoint algorithms estarán bounded.

---

# 321. SEM-063

Extension registries quedarán frozen antes del runtime.

---

# 322. SEM-064

Extension conflicts no usarán last-registered-wins.

---

# 323. SEM-065

Semantic extension resolution será determinista.

---

# 324. SEM-066

No habrá hidden I/O dentro de semantic extension handlers.

---

# 325. SEM-067

No habrá Service Locator dentro del Semantic Engine.

---

# 326. SEM-068

No habrá acceso directo a environment globals.

---

# 327. SEM-069

No habrá checks directos de nombre de database vendor en core semantic rules.

---

# 328. SEM-070

Diferencias de Platform se expresarán mediante semantic profiles/capabilities.

---

# 329. SEM-071

Portable Semantic Analysis deberá ser posible cuando no exista target definitivo.

---

# 330. SEM-072

Target Semantic Refinement deberá ser posible cuando la semántica dependa del Platform.

---

# 331. SEM-073

Semantic artifacts serán inmutables después de finalization.

---

# 332. SEM-074

No podrá finalizarse un artifact con unresolved required symbols.

---

# 333. SEM-075

No podrá finalizarse un artifact con fatal type conflicts.

---

# 334. SEM-076

Unknown y unresolved serán conceptos distintos.

---

# 335. SEM-077

Semantic fingerprints no incluirán parameter runtime values.

---

# 336. SEM-078

Semantic caches deberán considerar SchemaView version.

---

# 337. SEM-079

Semantic caches deberán considerar semantic policy.

---

# 338. SEM-080

Semantic caches deberán considerar extension set.

---

# 339. SEM-081

Tenant-specific artifacts no podrán reutilizarse entre tenants sin identidad semántica equivalente demostrable.

---

# 340. SEM-082

Diagnostics no deberán exponer parameter values sensibles.

---

# 341. SEM-083

Diagnostic suggestions nunca resolverán símbolos automáticamente.

---

# 342. SEM-084

Projection order será preservado.

---

# 343. SEM-085

Relation output order será determinista.

---

# 344. SEM-086

Set-operation output order será determinista.

---

# 345. SEM-087

Symbol IDs no dependerán de memory addresses.

---

# 346. SEM-088

Semantic results no dependerán del orden accidental de hash maps.

---

# 347. SEM-089

Semantic results no dependerán del orden accidental de extension discovery.

---

# 348. SEM-090

Semantic analysis no dependerá del tiempo actual salvo semantic function metadata explícita.

---

# 349. SEM-091

Semantic analysis no dependerá de random values.

---

# 350. SEM-092

Function volatility será visible para fases posteriores.

---

# 351. SEM-093

Predicate dependencies serán visibles para fases posteriores.

---

# 352. SEM-094

Relation dependencies serán visibles para fases posteriores.

---

# 353. SEM-095

Correlation information será visible para Planner/Optimizer.

---

# 354. SEM-096

Constraint information será visible para Optimizer.

---

# 355. SEM-097

Capability requirements serán visibles para Planner/Compiler.

---

# 356. SEM-098

Semantic Analysis deberá fallar antes del Compiler ante referencias inequívocamente inválidas.

---

# 357. SEM-099

El Compiler no deberá volver a realizar Symbol Resolution.

---

# 358. SEM-100

El Planner no deberá volver a realizar Schema Resolution básica.

---

# 359. SEM-101

El Optimizer consumirá semántica resuelta, no reinterpretará el AST desde cero.

---

# 360. SEM-102

El Executor no realizará Semantic Analysis.

---

# 361. Anti-pattern — AST mutation

Incorrecto:

```php
$node->resolvedColumn = $column;
$node->resolvedType = $type;
```

Correcto:

```text
NodeId
  │
  ▼
SemanticInfo
```

---

# 362. Anti-pattern — schema lookup from connection

Incorrecto:

```php
$pdo->query('DESCRIBE users');
```

desde `SemanticAnalyzer`.

Correcto:

```text
SemanticAnalyzer
     │
     ▼
SchemaView
```

---

# 363. Anti-pattern — vendor conditionals

Incorrecto:

```php
if ($driver === 'pgsql') {
    // semantic rule
}
```

Correcto:

```text
PlatformSemanticProfile
```

---

# 364. Anti-pattern — ORM dependency

Incorrecto:

```text
Semantic Analyzer
→ EntityManager
→ UnitOfWork
```

---

# 365. Anti-pattern — resolving by first match

Incorrecto:

```text
column id found in users and orders
→ choose users
```

Correcto:

```text
AmbiguousColumnReferenceException
```

---

# 366. Anti-pattern — compiler resolves columns

Incorrecto:

```text
SQL Compiler
→ discover what "u.email" means
```

El Compiler debe recibir semántica ya resuelta.

---

# 367. Anti-pattern — optimizer resolves types

Incorrecto:

```text
Optimizer
→ infer parameter type
```

El Optimizer deberá consumir tipos resueltos.

---

# 368. Anti-pattern — runtime semantic state singleton

Incorrecto:

```php
final class SemanticAnalyzer
{
    private array $symbols = [];
    private ?Scope $currentScope = null;
}
```

si la instancia es compartida.

---

# 369. Anti-pattern — semantic analysis as SQL parser

El Semantic Engine analiza:

```text
VoltStack Query AST
```

no SQL arbitrario.

---

# 370. Anti-pattern — hidden introspection

No:

```text
column not found in cache
→ silently connect to DB
→ introspect schema
```

La adquisición del `SchemaView` será responsabilidad externa.

---

# 371. Fórmula principal

```text
SemanticQueryArtifact
=
Analyze(
    ValidatedQueryArtifact,
    SchemaView,
    QueryTypeSystem,
    SemanticPolicy,
    SemanticExtensions
)
```

---

# 372. Semantic meaning formula

```text
Meaning(Query)
=
Scopes
+
Symbols
+
Relations
+
Types
+
Functions
+
Operators
+
Nullability
+
Correlations
+
Constraints
+
Dependencies
+
Requirements
```

---

# 373. Semantic graph formula

```text
SemanticQueryGraph
=
Resolved Semantic Nodes
+
Semantic Relationships
+
Dependency Edges
+
Constraint Edges
+
Lineage Edges
```

---

# 374. Query artifact evolution

```text
Query AST
    │
    ▼
Normalized Query AST
    │
    ▼
Validated Query Artifact
    │
    ▼
Semantic Query Artifact
    │
    ▼
Optimized Query Artifact
    │
    ▼
Logical Query Plan
    │
    ▼
Physical Query Plan
    │
    ▼
Compiled Query
```

---

# 375. Architectural boundary

```text
                 STRUCTURAL WORLD
                       │
       Query AST       │
           │           │
           ▼           │
      Normalization    │
           │           │
           ▼           │
       Validation      │
           │           │
═══════════╪═══════════╪══════════════════
           │           │
           ▼           │
     Semantic Engine   │
                       │
                 SEMANTIC WORLD
                       │
           ┌───────────┼───────────┐
           ▼           ▼           ▼
        Symbols      Types      Relations
           │           │           │
           └───────────┼───────────┘
                       ▼
                 Constraints
                       │
                       ▼
                Semantic Graph
                       │
═══════════════════════╪═══════════════════
                       ▼
                    Optimizer
                       │
                 TRANSFORMATION
                       │
                       ▼
                    Planner
                       │
                    PLANNING
```

---

# 376. Garantías del SemanticQueryArtifact

Una vez finalizado correctamente, las siguientes fases podrán asumir:

1. los scopes están definidos;
2. las references requeridas están resueltas;
3. los relation symbols son inequívocos;
4. los column symbols son inequívocos;
5. los aliases cumplen sus reglas de visibility;
6. los CTEs poseen resolución semántica válida;
7. las correlated references están identificadas;
8. las expressions poseen Query Types resueltos cuando sean requeridos;
9. los parameters poseen constraints de tipo coherentes;
10. las functions requeridas están resueltas;
11. los operators requeridos están resueltos;
12. aggregate scopes son coherentes;
13. grouping semantics son válidas;
14. joins poseen semántica conocida;
15. query nullability está derivada;
16. set operations son semánticamente compatibles;
17. DML assignments son compatibles;
18. output relation está definida;
19. constraints derivables están disponibles;
20. dependencies están disponibles;
21. capability requirements están declaradas;
22. lineage semántico puede consultarse;
23. no existen fatal semantic diagnostics.

---

# 377. Qué NO garantiza

`SemanticQueryArtifact` todavía no garantiza:

```text
que el target Platform soporte todos los requirements;
que exista una estrategia física óptima;
que una capability tenga que emularse;
que el query sea barato;
que exista un índice adecuado;
que el connection endpoint esté disponible;
que el servidor responda;
que la transacción pueda comenzar;
que el SQL ya haya sido generado;
que el query se ejecute correctamente.
```

---

# 378. Resultado arquitectónico

Con esta capa, VoltStack separará claramente:

```text
Query Structure
      │
      ▼
Query Meaning
      │
      ▼
Query Optimization
      │
      ▼
Query Planning
      │
      ▼
SQL Representation
      │
      ▼
Execution
```

---

# 379. Diferencia respecto a un Query Builder tradicional

Una arquitectura simple suele hacer:

```text
Builder
   │
   ▼
SQL String
```

VoltStack utilizará:

```text
Builder
   │
   ▼
Query Model
   │
   ▼
AST
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
Semantic Query Graph
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ▼
Compiler
   │
   ▼
SQL
```

---

# 380. Beneficio fundamental

Esto permite que las fases posteriores no tengan que preguntarse repetidamente:

```text
¿Qué significa users.id?
¿Qué tipo tiene :id?
¿A qué scope pertenece esta columna?
¿Este predicate depende de qué relation?
¿Este join puede producir NULL?
¿Esta función es volatile?
¿Esta subquery está correlacionada?
```

El Semantic Query Engine responderá esas preguntas una sola vez de forma estructurada.

---

# 381. Regla final de arquitectura

> Ninguna fase posterior deberá reinterpretar desde cero el significado de una Query AST que ya haya atravesado correctamente el Semantic Query Engine.

---

# 382. Cierre

`35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md` establece la frontera arquitectónica entre:

```text
Validated Structure
```

y:

```text
Resolved Meaning
```

El bloque completo será:

```text
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

Su transformación global será:

```text
ValidatedQueryArtifact
        │
        ▼
Semantic Analysis
        │
        ├── Scopes
        ├── Symbols
        ├── Schema
        ├── Relations
        ├── Types
        ├── Functions
        ├── Operators
        ├── Aggregates
        ├── Correlations
        ├── Constraints
        ├── Dependencies
        └── Capability Requirements
        │
        ▼
SemanticQueryArtifact
        │
        ▼
SemanticQueryGraph
```

---

# 383. Próximo documento

```text
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
```

profundizará en el motor que ejecutará estas fases, incluyendo:

```text
Semantic Analysis Coordinator
Analysis Passes
Pass Dependencies
Semantic State
Resolution Iterations
Fixpoint Processing
Error Recovery
Semantic Diagnostics
Analysis Budgets
Extension Passes
Artifact Finalization
```

La separación será:

```text
35 → arquitectura global del Semantic Query Engine

36 → funcionamiento interno del proceso de Semantic Analysis

37 → resolución de symbols

38 → resolución schema-aware

39 → inferencia de Query Types

40 → resolución de relations y joins

41 → análisis y propagación de constraints

42 → construcción final del Semantic Query Graph
```

Con ello, `35` funciona como el **contrato arquitectónico maestro** de todo el bloque semántico.