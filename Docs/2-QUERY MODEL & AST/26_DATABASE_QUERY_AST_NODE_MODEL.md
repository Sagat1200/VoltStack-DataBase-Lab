# 26_DATABASE_QUERY_AST_NODE_MODEL.md

# VoltStack Quantum Database
## Query AST Node Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 26 — Query AST Node Model  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / AST Node Model  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el modelo formal de nodos del **Query Abstract Syntax Tree** de `VoltStack/Quantum/Database`.

El documento anterior:

```text
25_DATABASE_QUERY_AST_SYSTEM.md
```

estableció el AST como representación estructural canónica de una consulta.

Este documento define ahora:

- qué es exactamente un AST node;
- qué familias de nodes existen;
- cómo se identifican;
- cómo se componen;
- cómo se recorren;
- cómo se comparan;
- cómo se transforman;
- qué metadata pueden contener;
- qué información queda prohibida;
- cómo se extiende el modelo;
- qué invariantes deberá respetar cada node.

La arquitectura general continúa siendo:

```text
Query Builder
      │
      ▼
Query Model
      │
      ▼
Query AST
      │
      ▼
AST NODE MODEL
      │
      ▼
Semantic Analysis
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

# 2. Objetivo central

VoltStack deberá disponer de un modelo de nodos:

```text
Typed
+
Immutable
+
Composable
+
Deterministic
+
Traversable
+
Transformable
+
Portable-first
+
Extensible
```

sin convertir el AST en una jerarquía de objetos excesivamente acoplada.

---

# 3. Regla maestra del Node Model

> Un AST node representa una unidad estructural de una consulta; no representa una conexión, una ejecución, una entidad ORM ni un fragmento SQL compilado.

Formalmente:

```text
AST Node
=
Node Kind
+
Semantic Fields
+
Ordered Children
+
Safe Structural Metadata
```

y nunca:

```text
AST Node
=
SQL
+
PDO
+
Connection
+
Runtime State
```

---

# 4. Node ≠ objeto arbitrario

No todo objeto utilizado durante una consulta será un AST node.

Por ejemplo:

```text
QueryAstNode           → sí
ExpressionNode         → sí
PredicateNode          → sí
TableSourceNode        → sí

QueryContext           → no
SemanticSymbol         → no
Connection             → no
EntityManager          → no
CompiledQuery          → no
ExecutionPlan          → no
Platform               → no
CapabilitySnapshot     → no
```

---

# 5. Modelo conceptual mínimo

Conceptualmente:

```php
interface QueryAstNode
{
    public function kind(): AstNodeKind;
}
```

Esta interfaz deberá permanecer pequeña.

No deberá convertirse en:

```php
interface QueryAstNode
{
    public function toSql(): string;
    public function execute(): mixed;
    public function optimize(): self;
    public function resolvePlatform(): Platform;
    public function connection(): Connection;
}
```

---

# 6. Contrato base mínimo

La recomendación arquitectónica será:

```php
interface QueryAstNode
{
    public function kind(): AstNodeKind;
}
```

y mover operaciones como:

- traversal;
- equality;
- hashing;
- validation;
- transformation;
- diagnostics;

a servicios externos.

---

# 7. Por qué mantener pequeño QueryAstNode

Esto evita que cada node tenga que conocer:

```text
Visitor
Transformer
Compiler
Platform
Dialect
Semantic Analyzer
Optimizer
Diagnostics
Fingerprinting
```

---

# 8. Arquitectura orientada a datos estructurados

Los nodes deberán comportarse principalmente como:

```text
immutable structured data
```

y no como objetos activos que controlan todo el pipeline.

---

# 9. Node behavior

Se permitirán métodos relacionados directamente con su estructura.

Ejemplos aceptables:

```php
$node->kind();
$node->expression();
$node->alias();
$node->items();
```

No:

```php
$node->compile();
$node->execute();
$node->connect();
```

---

# 10. Taxonomía principal

El modelo conceptual será:

```text
QueryAstNode
│
├── QueryRootNode
│
├── SourceNode
│
├── ProjectionNode
│
├── ExpressionNode
│
├── PredicateNode
│
├── IdentifierNode
│
├── ParameterNode
│
├── JoinConditionNode
│
├── InsertSourceNode
│
├── ConflictTargetNode
│
├── ConflictActionNode
│
├── SetOperationNode
│
├── ClauseNode
│
└── ExtensionAstNode
```

No todas estas categorías deberán convertirse obligatoriamente en interfaces PHP.

---

# 11. Categoría conceptual ≠ interfaz obligatoria

Una categoría se convierte en interfaz cuando exista una necesidad real de:

- polymorphism;
- type restriction;
- extension contract;
- visitor dispatch;
- validation boundary.

---

# 12. Evitar interface explosion

Incorrecto:

```text
Node
StatementNode
QueryNode
ExecutableNode
RootNode
SelectableNode
ReadableNode
SqlNode
DatabaseNode
```

sin una diferencia contractual real.

---

# 13. Familias oficiales

El AST deberá organizar inicialmente sus nodes en las siguientes familias:

```text
01 Root / Statement
02 Source
03 Projection
04 Identifier
05 Parameter
06 Expression
07 Predicate
08 Join
09 CTE
10 Insert
11 Mutation
12 Conflict
13 Returning
14 Grouping
15 Ordering
16 Pagination
17 Locking
18 Set Operation
19 Window
20 Raw / Escape Hatch
21 Metadata
22 Extension
```

---

# 14. Root Nodes

Los root nodes representan consultas completas.

```text
QueryRootNode
├── SelectNode
├── InsertNode
├── UpdateNode
└── DeleteNode
```

---

# 15. QueryRootNode

Contrato conceptual:

```php
interface QueryRootNode extends QueryAstNode
{
    public function queryType(): QueryType;
}
```

---

# 16. QueryType

Inicialmente:

```php
enum QueryType
{
    case SELECT;
    case INSERT;
    case UPDATE;
    case DELETE;
}
```

Podrá evolucionar sin convertir el enum en sustituto de polymorphism.

---

# 17. SelectNode

Estructura conceptual:

```text
SelectNode
│
├── WithNode
├── DistinctSpecification
├── ProjectionNodeList
├── SourceNodeList
├── JoinNodeList
├── WherePredicate?
├── GroupByNode
├── HavingPredicate?
├── SetOperationNodeList
├── OrderByNode
├── PaginationNode?
├── LockNode?
└── AstMetadata
```

---

# 18. InsertNode

```text
InsertNode
│
├── WithNode
├── MutationTargetNode
├── InsertColumnList
├── InsertSourceNode
├── ConflictNode?
├── ReturningNode?
└── AstMetadata
```

---

# 19. UpdateNode

```text
UpdateNode
│
├── WithNode
├── MutationTargetNode
├── AssignmentNodeList
├── SourceNodeList
├── JoinNodeList
├── WherePredicate?
├── OrderByNode
├── PaginationNode?
├── ReturningNode?
└── AstMetadata
```

---

# 20. DeleteNode

```text
DeleteNode
│
├── WithNode
├── MutationTargetNode
├── SourceNodeList
├── JoinNodeList
├── WherePredicate?
├── OrderByNode
├── PaginationNode?
├── ReturningNode?
└── AstMetadata
```

---

# 21. Root node completeness

Un root node deberá representar la consulta completa desde el punto de vista estructural.

No deberá depender del Query Builder para reconstruir partes faltantes.

---

# 22. Root node immutability

Una vez construido:

```text
SelectNode
```

no deberá modificarse.

Una nueva cláusula produce otro `SelectNode`.

---

# 23. Source Nodes

Los source nodes representan fuentes de datos.

```text
SourceNode
├── TableSourceNode
├── SubquerySourceNode
├── CteReferenceNode
├── DerivedTableNode
├── ValuesSourceNode
├── FunctionSourceNode
└── ExtensionSourceNode
```

---

# 24. SourceNode contract

Conceptualmente:

```php
interface SourceNode extends QueryAstNode
{
}
```

No necesita más métodos si no existe comportamiento universal.

---

# 25. TableSourceNode

```text
TableSourceNode
├── QualifiedIdentifierNode
└── AliasNode?
```

Ejemplo lógico:

```text
users AS u
```

se representa como:

```text
TableSourceNode
├── Identifier
│   └── users
└── Alias
    └── u
```

---

# 26. SubquerySourceNode

```text
SubquerySourceNode
├── QueryRootNode
└── AliasNode
```

---

# 27. Alias requirement

Un derived source podrá requerir alias aunque una tabla normal no.

La validación estructural podrá imponer esta regla.

---

# 28. CteReferenceNode

```text
CteReferenceNode
├── CteIdentifier
└── Alias?
```

No contendrá necesariamente el CTE completo.

---

# 29. CTE reference ≠ object pointer

Preferir una referencia estructural:

```text
CteReference
→ Identifier
```

en lugar de:

```text
CteReference
→ direct mutable pointer to CteNode
```

---

# 30. DerivedTableNode

Podrá utilizarse cuando sea necesario distinguir semánticamente:

```text
derived query
```

de un subquery expression.

---

# 31. FunctionSourceNode

Permitirá modelar fuentes tabulares basadas en funciones cuando la plataforma/capability lo permita.

Ejemplo conceptual:

```text
FunctionSourceNode
├── FunctionExpression
└── Alias
```

---

# 32. Projection Nodes

Representan elementos de SELECT/RETURNING.

```text
ProjectionNode
├── ExpressionProjectionNode
├── WildcardProjectionNode
└── QualifiedWildcardProjectionNode
```

---

# 33. ExpressionProjectionNode

```text
ExpressionProjectionNode
├── ExpressionNode
└── AliasNode?
```

---

# 34. WildcardProjectionNode

Representa:

```text
*
```

semánticamente.

No deberá representarse como un identifier ordinario.

---

# 35. QualifiedWildcardProjectionNode

Representa:

```text
source.*
```

sin almacenar la syntax SQL.

---

# 36. Projection alias

El alias deberá utilizar un identifier estructurado.

No:

```text
"total AS amount"
```

---

# 37. Identifier Nodes

Los identifiers representan nombres estructurales.

Conceptualmente:

```text
IdentifierNode
├── SimpleIdentifierNode
├── QualifiedIdentifierNode
├── AliasNode
├── SchemaIdentifierNode
├── TableIdentifierNode
├── ColumnIdentifierNode
├── CteIdentifierNode
└── ConstraintIdentifierNode
```

---

# 38. ¿Node o Value Object?

No todos los identifiers necesitan ser nodes completos.

Puede resultar más apropiado:

```text
Identifier
QualifiedIdentifier
Alias
```

como value objects inmutables usados por nodes.

---

# 39. Decisión recomendada

Utilizar:

```text
AST Node
→ para elementos estructurales del árbol

Value Object
→ para valores estructurales atómicos
```

---

# 40. Ejemplo

```php
final readonly class TableSourceNode implements SourceNode
{
    public function __construct(
        public QualifiedIdentifier $table,
        public ?Identifier $alias,
    ) {}
}
```

puede ser preferible a convertir cada segmento en node.

---

# 41. Node granularity

La granularidad deberá equilibrar:

- claridad;
- type safety;
- traversal;
- allocations;
- extensibilidad;
- diagnostics.

---

# 42. No AST atomization

Evitar:

```text
IdentifierNode
└── IdentifierStringNode
    └── CharacterNode[]
```

---

# 43. Identifier invariant

Los identifiers deberán estar:

```text
unquoted
```

en el AST.

---

# 44. QualifiedIdentifier

Conceptualmente:

```text
QualifiedIdentifier
├── namespace/schema?
├── source/table?
└── name
```

pero deberá soportar los modelos de namespaces de cada plataforma sin asumir PostgreSQL.

---

# 45. Parameter Nodes

Los parámetros representan valores runtime bindables.

```text
ParameterExpressionNode
├── ParameterId
├── TypeHint?
└── Metadata
```

---

# 46. ParameterId

Deberá ser estable dentro de una consulta.

Ejemplo:

```text
p1
p2
p3
```

La representación externa podrá diferir.

---

# 47. ParameterId ≠ placeholder

`ParameterId("p1")` no significa:

```text
:p1
```

ni:

```text
?
```

---

# 48. Parameter value

Preferencia:

```text
AST
└── ParameterId

BindingSet
└── ParameterId → value
```

---

# 49. Parameter node equality

La igualdad estructural deberá considerar la identidad semántica del parámetro según el modelo elegido.

---

# 50. Parameter canonicalization

Para Query Shape fingerprints podrá existir una representación donde IDs secuenciales se normalicen.

Ejemplo:

```text
p17
p82
```

podrían canonicalizarse como:

```text
p1
p2
```

si la estructura relativa es equivalente.

---

# 51. Expression Nodes

Se detallarán en:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

Taxonomía inicial:

```text
ExpressionNode
├── ColumnReferenceNode
├── ParameterExpressionNode
├── LiteralExpressionNode
├── FunctionCallNode
├── AggregateExpressionNode
├── ArithmeticExpressionNode
├── UnaryExpressionNode
├── CaseExpressionNode
├── CastExpressionNode
├── SubqueryExpressionNode
├── TupleExpressionNode
├── JsonExpressionNode
├── DateTimeExpressionNode
├── WindowFunctionNode
├── RawExpressionNode
└── ExtensionExpressionNode
```

---

# 52. ExpressionNode contract

```php
interface ExpressionNode extends QueryAstNode
{
}
```

---

# 53. Expressions no compilan

No:

```php
$expression->toSql();
```

---

# 54. ColumnReferenceNode

Conceptualmente:

```text
ColumnReferenceNode
├── Qualifier?
└── ColumnIdentifier
```

Ejemplo:

```text
u.email
```

---

# 55. Unresolved reference

El node puede permanecer unresolved:

```text
ColumnReferenceNode("email")
```

hasta Semantic Analysis.

---

# 56. Resolved symbol not embedded

No convertirlo en:

```text
ColumnReferenceNode
├── name = email
├── tableObject = users
├── databaseType = varchar
└── platformColumn = ...
```

---

# 57. LiteralExpressionNode

Representará constantes estructurales seguras.

Ejemplos:

```text
NULL
TRUE
FALSE
```

y otros valores cuando el Type System lo permita.

---

# 58. Literal ≠ arbitrary runtime value

Datos de aplicación deberán ser parameters por defecto.

---

# 59. FunctionCallNode

```text
FunctionCallNode
├── FunctionId
└── ExpressionNodeList
```

---

# 60. FunctionId

Deberá ser semántico.

Ejemplo:

```text
string.lower
aggregate.count
datetime.current_timestamp
```

cuando sea portable.

---

# 61. ArithmeticExpressionNode

```text
ArithmeticExpressionNode
├── Operator
├── Left
└── Right
```

---

# 62. UnaryExpressionNode

```text
UnaryExpressionNode
├── Operator
└── Operand
```

---

# 63. CaseExpressionNode

```text
CaseExpressionNode
├── Operand?
├── CaseBranchNodeList
└── ElseExpression?
```

---

# 64. CaseBranchNode

```text
CaseBranchNode
├── Condition
└── ResultExpression
```

---

# 65. CastExpressionNode

```text
CastExpressionNode
├── Expression
└── TargetType
```

TargetType será semántico.

---

# 66. SubqueryExpressionNode

```text
SubqueryExpressionNode
└── QueryRootNode
```

---

# 67. TupleExpressionNode

```text
TupleExpressionNode
└── ExpressionNodeList
```

---

# 68. Predicate Nodes

Se desarrollarán en:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Taxonomía inicial:

```text
PredicateNode
├── TruePredicateNode
├── FalsePredicateNode
├── ComparisonPredicateNode
├── AndPredicateNode
├── OrPredicateNode
├── NotPredicateNode
├── NullPredicateNode
├── BetweenPredicateNode
├── InPredicateNode
├── ExistsPredicateNode
├── LikePredicateNode
├── RegexPredicateNode
├── JsonPredicateNode
├── RawPredicateNode
└── ExtensionPredicateNode
```

---

# 69. PredicateNode

```php
interface PredicateNode extends QueryAstNode
{
}
```

---

# 70. Predicate vs Expression

V1 mantendrá categorías separadas.

```text
ExpressionNode
≠
PredicateNode
```

aunque el Semantic Type System pueda considerar predicates como expresiones booleanas.

---

# 71. ComparisonPredicateNode

```text
ComparisonPredicateNode
├── ComparisonOperator
├── LeftExpression
└── RightExpression
```

---

# 72. AndPredicateNode

```text
AndPredicateNode
└── PredicateNodeList
```

---

# 73. N-ary AND

Preferir:

```text
AND
├── A
├── B
└── C
```

sobre:

```text
AND
├── A
└── AND
    ├── B
    └── C
```

cuando la canonicalization pueda hacerlo sin alterar semántica.

---

# 74. N-ary OR

Igualmente:

```text
OR
├── A
├── B
└── C
```

---

# 75. Grouping preservation

Cuando grouping explícito sea semánticamente relevante, deberá preservarse.

---

# 76. NotPredicateNode

```text
NotPredicateNode
└── PredicateNode
```

---

# 77. NullPredicateNode

```text
NullPredicateNode
├── Expression
└── Negated
```

Representa:

```text
IS NULL
IS NOT NULL
```

semánticamente.

---

# 78. BetweenPredicateNode

```text
BetweenPredicateNode
├── Expression
├── LowerBound
├── UpperBound
└── Negated
```

---

# 79. InPredicateNode

```text
InPredicateNode
├── Expression
├── ValueSource
└── Negated
```

---

# 80. InValueSource

```text
InValueSource
├── ExpressionListSource
└── SubquerySource
```

---

# 81. ExistsPredicateNode

```text
ExistsPredicateNode
├── QueryRootNode
└── Negated
```

---

# 82. LikePredicateNode

```text
LikePredicateNode
├── Expression
├── Pattern
├── Escape?
├── CaseSensitivitySemantics
└── Negated
```

La sintaxis concreta será Dialect concern.

---

# 83. Join Nodes

```text
JoinNode
├── JoinType
├── SourceNode
└── JoinConditionNode
```

---

# 84. JoinType

Semánticamente:

```text
INNER
LEFT
RIGHT
FULL
CROSS
```

y extensiones cuando existan capabilities.

---

# 85. JoinConditionNode

```text
JoinConditionNode
├── PredicateJoinConditionNode
├── UsingJoinConditionNode
├── NaturalJoinConditionNode
└── NoJoinConditionNode
```

---

# 86. PredicateJoinConditionNode

```text
PredicateJoinConditionNode
└── PredicateNode
```

---

# 87. UsingJoinConditionNode

```text
UsingJoinConditionNode
└── IdentifierList
```

---

# 88. NaturalJoinConditionNode

Será un node explícito cuando la API lo permita.

---

# 89. Cross join

Un CROSS JOIN normalmente utilizará:

```text
NoJoinConditionNode
```

---

# 90. CTE Nodes

```text
WithNode
└── CommonTableExpressionNodeList
```

---

# 91. CommonTableExpressionNode

```text
CommonTableExpressionNode
├── CteIdentifier
├── ColumnIdentifierList
├── QueryRootNode
└── CteOptions
```

---

# 92. Recursive flag

La recursividad podrá vivir en:

```text
WithNode
```

o ser derivada de los CTEs.

La decisión deberá permanecer consistente.

---

# 93. CTE materialization

Si se soporta:

```text
CteMaterializationPreference
├── DEFAULT
├── MATERIALIZED
└── NOT_MATERIALIZED
```

será una intención/capability-dependent feature.

---

# 94. Insert Nodes

```text
InsertSourceNode
├── ValuesInsertSourceNode
├── QueryInsertSourceNode
└── DefaultValuesInsertSourceNode
```

---

# 95. ValuesInsertSourceNode

```text
ValuesInsertSourceNode
└── InsertRowNodeList
```

---

# 96. InsertRowNode

```text
InsertRowNode
└── ExpressionNodeList
```

---

# 97. QueryInsertSourceNode

```text
QueryInsertSourceNode
└── QueryRootNode
```

Normalmente el root deberá ser SELECT-compatible.

---

# 98. DefaultValuesInsertSourceNode

Representa la intención:

```text
insert using defaults
```

sin almacenar la syntax final.

---

# 99. Mutation Nodes

Mutation comprende elementos compartidos por INSERT/UPDATE/DELETE.

---

# 100. MutationTargetNode

```text
MutationTargetNode
├── QualifiedIdentifier
└── Alias?
```

---

# 101. AssignmentNode

```text
AssignmentNode
├── AssignmentTarget
└── ExpressionNode
```

---

# 102. AssignmentTarget

Normalmente será una columna target.

Podrá convertirse en una abstracción propia para soportar features futuras.

---

# 103. Assignment list

```text
AssignmentNodeList
```

deberá:

- ser inmutable;
- preservar orden;
- impedir tipos inválidos.

---

# 104. Duplicate assignments

La validación deberá detectar:

```text
SET name = ...
SET name = ...
```

cuando la semántica de VoltStack lo considere inválido.

---

# 105. Conflict Nodes

```text
ConflictNode
├── ConflictTargetNode?
└── ConflictActionNode
```

---

# 106. ConflictTargetNode

Taxonomía posible:

```text
ConflictTargetNode
├── ConflictColumnTargetNode
├── ConflictConstraintTargetNode
├── ConflictInferenceTargetNode
└── ExtensionConflictTargetNode
```

---

# 107. ConflictActionNode

```text
ConflictActionNode
├── IgnoreConflictActionNode
├── UpdateConflictActionNode
└── ExtensionConflictActionNode
```

---

# 108. UpdateConflictActionNode

```text
UpdateConflictActionNode
├── AssignmentNodeList
└── PredicateNode?
```

---

# 109. Portable semantics

El node no se llamará:

```text
OnDuplicateKeyUpdateNode
```

en el core portable.

---

# 110. Returning Nodes

```text
ReturningNode
└── ProjectionNodeList
```

---

# 111. Returning reuse

La misma infraestructura de projection podrá utilizarse para:

```text
SELECT projection
RETURNING projection
```

si la semántica común es suficiente.

---

# 112. Context remains external

El ProjectionNode no deberá necesitar saber:

```text
I am SELECT
```

o:

```text
I am RETURNING
```

salvo que una diferencia semántica lo requiera.

---

# 113. Grouping Nodes

```text
GroupByNode
└── ExpressionNodeList
```

---

# 114. Grouping extensions

Futuro:

```text
GroupingSetNode
RollupNode
CubeNode
```

mediante capabilities/extensibility.

---

# 115. Having

No necesita obligatoriamente:

```text
HavingNode
```

si:

```text
?PredicateNode $having
```

preserva suficientemente la posición estructural.

---

# 116. Clause nodes only when useful

No crear clases sólo para imitar keywords SQL.

---

# 117. Ordering Nodes

```text
OrderByNode
└── OrderItemNodeList
```

---

# 118. OrderItemNode

```text
OrderItemNode
├── ExpressionNode
├── SortDirection
└── NullOrdering
```

---

# 119. SortDirection

```text
ASC
DESC
```

---

# 120. NullOrdering

```text
DEFAULT
FIRST
LAST
```

---

# 121. DEFAULT semantics

`DEFAULT` significa:

```text
use target/platform default semantics
```

no asumir una posición universal.

---

# 122. Pagination Nodes

```text
PaginationNode
├── LimitExpression?
└── OffsetExpression?
```

---

# 123. Pagination values

Podrán representarse mediante parámetros/valores estructurados según el Query Model.

---

# 124. No SQL syntax

No almacenar:

```text
LIMIT 10 OFFSET 20
```

---

# 125. Pagination invariant

Valores negativos deberán rechazarse antes de Compilation cuando sean conocidos.

---

# 126. Locking Nodes

```text
LockNode
├── LockMode
├── LockWaitPolicy
└── LockTargetList
```

---

# 127. LockMode

Modelo semántico portable inicial:

```text
UPDATE
SHARE
```

y modos avanzados mediante capabilities:

```text
NO_KEY_UPDATE
KEY_SHARE
```

---

# 128. LockWaitPolicy

```text
WAIT
NOWAIT
SKIP_LOCKED
```

---

# 129. Lock targets

Podrán ser referencias a fuentes/aliases.

---

# 130. Lock node ≠ transaction

El node expresa intención de locking.

No inicia una transacción.

---

# 131. Set Operation Nodes

```text
SetOperationNode
├── UnionNode
├── IntersectNode
└── ExceptNode
```

o:

```text
SetOperationNode
├── operation
├── quantifier
└── query
```

---

# 132. Recommended representation

Preferir una estructura composable:

```text
SetOperationNode
├── SetOperationType
├── SetQuantifier
└── QueryRootNode
```

dentro de una secuencia asociada al query principal.

---

# 133. SetOperationType

```text
UNION
INTERSECT
EXCEPT
```

---

# 134. SetQuantifier

```text
DISTINCT
ALL
```

---

# 135. Grouping

La estructura deberá preservar:

```text
(A UNION B) INTERSECT C
```

frente a:

```text
A UNION (B INTERSECT C)
```

cuando sea necesario.

---

# 136. Window Nodes

```text
WindowFunctionNode
├── FunctionExpression
└── WindowSpecificationNode
```

---

# 137. WindowSpecificationNode

```text
WindowSpecificationNode
├── PartitionExpressionList
├── OrderByNode
└── WindowFrameNode?
```

---

# 138. WindowFrameNode

```text
WindowFrameNode
├── FrameUnit
├── StartBoundary
└── EndBoundary?
```

---

# 139. FrameUnit

```text
ROWS
RANGE
GROUPS
```

sujeto a capabilities.

---

# 140. Frame boundary

Conceptualmente:

```text
UNBOUNDED_PRECEDING
PRECEDING
CURRENT_ROW
FOLLOWING
UNBOUNDED_FOLLOWING
```

con expresión cuando corresponda.

---

# 141. Raw Nodes

El escape hatch deberá ser explícito.

```text
RawAstNode
├── RawExpressionNode
├── RawPredicateNode
└── PlatformSpecificAstNode
```

---

# 142. TrustedRawExpressionNode

Conceptualmente:

```text
TrustedRawExpressionNode
├── RawFragment
├── ParameterReferences
├── PortabilityLevel
└── TrustDescriptor
```

---

# 143. RawFragment

Será un value object explícito.

No un string mezclado accidentalmente con expressions normales.

---

# 144. Raw parameter binding

Incluso Raw deberá permitir:

```text
fragment + bindings
```

en lugar de interpolación.

---

# 145. Raw limitations

Un raw node podrá limitar:

- semantic analysis;
- optimizer rewrites;
- portability;
- capability inference;
- fingerprint reuse.

---

# 146. PlatformSpecificAstNode

Cuando una feature sea genuinamente específica:

```text
PlatformSpecificAstNode
├── PlatformRequirement
└── StructuredPayload
```

preferiblemente mediante extension contracts.

---

# 147. Extension Nodes

```text
ExtensionAstNode
```

será la frontera para extensiones estructurales.

---

# 148. Extension node identity

Cada extension node deberá declarar:

```text
ExtensionId
NodeTypeId
NodeSchemaVersion
```

---

# 149. NodeTypeId

Ejemplo conceptual:

```text
voltstack.query.expression.json_extract
vendor.postgresql.expression.array_overlap
package.foo.query.geo_distance
```

---

# 150. Stable node type IDs

No deberán depender del nombre PHP de la clase para serialización/fingerprinting público.

---

# 151. Extension node requirements

Una extensión deberá definir, cuando aplique:

- structural schema;
- children;
- validation;
- traversal;
- fingerprinting;
- semantic handler;
- capability requirements;
- optimizer behavior;
- planner behavior;
- compiler behavior;
- diagnostics.

---

# 152. Unknown extension

Si una etapa requerida no puede procesarlo:

```text
UnsupportedAstNodeException
```

---

# 153. Node Kind

Todo node deberá poseer un tipo estructural identificable.

---

# 154. AstNodeKind

Podrá implementarse mediante:

```php
final readonly class AstNodeKind
{
    public function __construct(
        public string $value,
    ) {}
}
```

en lugar de un enum cerrado si se necesita extensibilidad.

---

# 155. Core node kinds

Ejemplos:

```text
query.select
query.insert
query.update
query.delete

source.table
source.subquery

projection.expression
projection.wildcard

expression.column
expression.parameter
expression.literal
expression.function

predicate.comparison
predicate.and
predicate.or

join.standard

cte.definition

mutation.assignment

ordering.item
```

---

# 156. Namespaced kinds

Los IDs deberán estar namespaced para evitar colisiones.

---

# 157. Node Kind ≠ PHP class

Una clase puede cambiar internamente sin cambiar necesariamente el kind estable.

---

# 158. Node descriptor

Podrá existir:

```text
AstNodeDescriptor
├── kind
├── category
├── schemaVersion
├── childSchema
├── stability
└── extensionOwner
```

---

# 159. AstNodeDescriptor use

Útil para:

- tooling;
- extensions;
- diagnostics;
- serialization;
- validation.

---

# 160. No runtime reflection dependency

El hot path no deberá depender obligatoriamente de reflection para descubrir children.

---

# 161. Child Model

Cada node deberá poseer children conocidos estructuralmente.

---

# 162. Child schema

Ejemplo:

```text
ComparisonPredicateNode
├── left: ExpressionNode
└── right: ExpressionNode
```

---

# 163. Required child

Un child obligatorio nunca deberá ser `null`.

---

# 164. Optional child

Ejemplo:

```text
ExpressionProjectionNode
└── alias: Identifier?
```

---

# 165. Repeated children

Ejemplo:

```text
AndPredicateNode
└── predicates: PredicateNodeList
```

---

# 166. Child order

Será determinista.

---

# 167. Structural ordering

En:

```text
Comparison(left, right)
```

left y right no son intercambiables estructuralmente.

---

# 168. Ordered collections

Por defecto las colecciones preservarán orden.

---

# 169. Semantic unordered sets

Si una estructura es realmente un set semántico, deberá modelarse explícitamente.

No asumir que todas las listas pueden ordenarse para mejorar fingerprints.

---

# 170. Child ownership

Los nodes inmutables podrán compartir children.

No existe ownership mutable exclusivo.

---

# 171. Parent references

Los children no deberán mantener referencias mutables hacia su parent.

---

# 172. No bidirectional AST graph

Evitar:

```text
Parent → Child
Child → Parent
```

porque:

- crea ciclos;
- complica GC;
- complica serialization;
- dificulta structural sharing.

---

# 173. Parent lookup

Cuando sea necesario:

```text
AstTraversalContext
```

podrá mantener el path actual externamente.

---

# 174. Node Path

Conceptualmente:

```text
AstNodePath
├── root
├── segments
└── current
```

---

# 175. Path segment

Ejemplo:

```text
where
and[1]
comparison.right
```

---

# 176. Node Identity

Se distinguirán cuatro conceptos:

```text
Runtime Object Identity
Occurrence Identity
Structural Identity
Semantic Identity
```

---

# 177. Runtime Object Identity

Es simplemente:

```text
PHP object identity
```

No tendrá significado arquitectónico persistente.

---

# 178. Occurrence Identity

Identifica una aparición concreta dentro de un AST.

Podrá utilizar:

```text
AstNodeId
```

---

# 179. Structural Identity

Dos nodes son estructuralmente iguales cuando poseen:

```text
same kind
+
same semantic fields
+
structurally equal children
```

---

# 180. Semantic Identity

Sólo puede determinarse después del Semantic Engine en muchos casos.

Ejemplo:

```text
id
```

y:

```text
users.id
```

pueden resolver al mismo símbolo.

---

# 181. Structural equality ≠ semantic equality

Regla obligatoria.

---

# 182. AstNodeId

Si se utiliza:

```php
final readonly class AstNodeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 183. AstNodeId generation

Podrá ser:

- traversal-derived;
- factory-generated;
- deterministic;
- ephemeral.

La estrategia deberá depender del uso.

---

# 184. Recommendation

Para Semantic side tables, se recomienda una identidad de ocurrencia estable dentro del snapshot.

---

# 185. Node IDs and structural sharing

Si un mismo immutable child se comparte en dos posiciones, deberá decidirse si:

```text
object identity
```

representa una o dos ocurrencias.

---

# 186. Recommended occurrence model

La posición en el árbol será la identidad de ocurrencia conceptual.

El mismo object compartido puede aparecer en múltiples paths.

---

# 187. Consequence

Semantic annotations no deberán depender únicamente de:

```text
spl_object_id()
```

---

# 188. Semantic annotation key

Podrá utilizar:

```text
AstOccurrenceId
```

derivado del snapshot/path.

---

# 189. Equality service

La igualdad estructural deberá residir preferentemente en:

```text
AstStructuralComparator
```

---

# 190. Why external comparator

Permite:

- diferentes equality modes;
- excluir metadata;
- evitar métodos complejos en nodes;
- versionar reglas.

---

# 191. Equality modes

Podrán existir:

```text
STRICT_STRUCTURAL
QUERY_SHAPE
SEMANTIC_CANDIDATE
```

---

# 192. Strict structural

Considera todos los campos estructurales relevantes.

---

# 193. Query shape

Puede ignorar:

- runtime values;
- source locations;
- diagnostics;
- normalized parameter IDs.

---

# 194. Semantic candidate

Sólo puede afirmar equivalencia potencial.

La equivalencia semántica real pertenece al Semantic Engine.

---

# 195. Node Hashing

Deberá ser consistente con el modo de igualdad utilizado.

---

# 196. Structural hash

Conceptualmente:

```text
hash(
    kind,
    structural fields,
    ordered child hashes
)
```

---

# 197. Hash ≠ equality

Colisiones deberán resolverse mediante comparación cuando sea necesario.

---

# 198. Metadata model

Los nodes podrán asociarse a metadata segura.

---

# 199. Metadata categories

```text
AstMetadata
├── StructuralMetadata
├── DiagnosticMetadata
├── ProvenanceMetadata
└── ExtensionMetadata
```

---

# 200. StructuralMetadata

Sólo cuando afecte significado estructural.

---

# 201. DiagnosticMetadata

Ejemplos:

```text
source file
line
builder operation
debug label
```

No afecta igualdad semántica.

---

# 202. ProvenanceMetadata

Ejemplo:

```text
APPLICATION
ORM
REPOSITORY
PERSISTENCE
FRAMEWORK
EXTENSION
```

---

# 203. ExtensionMetadata

Deberá utilizar claves namespaced y tipos definidos.

---

# 204. No arbitrary mixed bag

Evitar:

```php
array<string, mixed> $metadata;
```

como contrato sin control.

---

# 205. MetadataBag

Si se utiliza un bag:

```text
TypedMetadataBag
```

deberá exigir descriptors/keys tipadas.

---

# 206. Metadata immutability

Toda metadata incluida en un node inmutable deberá ser también inmutable.

---

# 207. Metadata lifecycle

No incluir:

- Request;
- User;
- Tenant object;
- Connection;
- EntityManager;
- mutable service;
- logger;
- callback scoped.

---

# 208. SourceLocation

Value object conceptual:

```text
SourceLocation
├── origin
├── file?
├── line?
├── column?
└── operation?
```

---

# 209. Source origin

Podrá ser:

```text
BUILDER
ORM
PERSISTENCE
FRAMEWORK
RAW
EXTENSION
```

---

# 210. Privacy

Paths de archivos y datos de aplicación podrán ocultarse en producción.

---

# 211. Node Collections

Las listas de nodes serán value collections inmutables.

---

# 212. Example

```php
final readonly class ExpressionNodeList
{
    /** @var list<ExpressionNode> */
    private array $items;
}
```

---

# 213. Collection API

Conceptualmente:

```text
count()
isEmpty()
first()
at()
all()
map()
```

siempre sin mutación.

---

# 214. withAdded()

Podrá existir:

```php
$list->withAdded($node);
```

retornando otra colección.

---

# 215. No mutable append

No:

```php
$list[] = $node;
```

después de construcción.

---

# 216. Collection validation

La colección deberá validar elementos una sola vez cuando sea posible.

---

# 217. Non-empty collections

Podrán existir:

```text
NonEmptyExpressionNodeList
NonEmptyPredicateNodeList
```

si reducen estados inválidos.

---

# 218. Use with moderation

No crear docenas de tipos si una validación de constructor es suficiente.

---

# 219. AST factories

Construcción recomendada:

```text
QueryAstFactory
├── StatementFactory
├── ExpressionFactory
├── PredicateFactory
└── ExtensionFactory
```

sólo si la complejidad lo justifica.

---

# 220. Constructors remain valid

No se deberá prohibir construcción directa de value objects/nodes internos si es segura.

---

# 221. Factory purpose

Factories pueden:

- canonicalizar;
- validar;
- reutilizar constants;
- generar ParameterId;
- aplicar defaults estructurales.

---

# 222. Factory must not perform semantics

No deberá consultar Schema Metadata.

---

# 223. Canonical singleton nodes

Nodes sin estado podrán reutilizarse.

Ejemplo:

```text
TruePredicateNode
FalsePredicateNode
WildcardProjectionNode
```

si son completamente inmutables.

---

# 224. No global mutable singleton

El singleton reutilizable deberá ser puro.

---

# 225. Null Objects

Para estructuras vacías podrán utilizarse:

```text
EmptyWithNode
EmptyOrderByNode
```

sólo cuando simplifiquen el modelo.

---

# 226. Avoid fake nodes

No crear:

```text
NoWhereNode
NoHavingNode
NoLockNode
```

si `null` comunica mejor ausencia.

---

# 227. Required vs optional

Regla:

```text
empty collection
```

para cláusulas presentes conceptualmente pero sin elementos;

```text
null
```

para componentes realmente ausentes.

---

# 228. AST Visitor Dispatch

Existen varias estrategias posibles.

---

# 229. Strategy A — visitor methods per node

```php
$visitor->visitSelect($node);
$visitor->visitComparison($node);
```

Ventaja:

- type safety.

Desventaja:

- extension friction.

---

# 230. Strategy B — kind-based dispatch

```php
$visitor->visit($node);
```

y registry por `AstNodeKind`.

Ventaja:

- extensibilidad.

---

# 231. Recommendation

Modelo híbrido:

```text
Core typed handlers
+
Extension handler registry
```

---

# 232. Core visitor

Los nodes oficiales podrán tener dispatch optimizado.

---

# 233. Extension visitor

Los extension nodes se resolverán por:

```text
NodeTypeId
→ Handler
```

---

# 234. Traversal metadata

El traverser deberá conocer el child schema sin reflection obligatoria.

---

# 235. Child accessor descriptor

Podrá existir metadata compilada:

```text
AstChildDescriptor
├── name
├── cardinality
├── expectedType
└── ordering
```

---

# 236. Cardinality

```text
ONE
OPTIONAL
MANY
```

---

# 237. AstNodeSchema

Conceptualmente:

```text
AstNodeSchema
├── NodeTypeId
├── Fields
├── Children
├── MetadataRules
└── Version
```

---

# 238. Core schemas

Podrán estar definidos estáticamente/compilados.

---

# 239. Extension schemas

Se registran durante bootstrap y se congelan.

---

# 240. Schema use

Permite:

- generic traversal;
- validation;
- serialization;
- debug rendering;
- extension tooling.

---

# 241. Schema ≠ database schema

Importante:

```text
AstNodeSchema
```

describe estructura del node.

No tablas/columnas del database.

---

# 242. AST Transformation

Un transformer deberá reconstruir sólo la ruta modificada.

---

# 243. Example

Original:

```text
Select
├── Projection
├── Source
└── Where
    └── A
```

Transformado:

```text
Select'
├── Projection ───────── reused
├── Source ───────────── reused
└── Where'
    └── B
```

---

# 244. Node reconstruction

Cada node podrá tener métodos internos tipo:

```php
withPredicate(...)
withChildren(...)
```

si conservan inmutabilidad.

---

# 245. Alternative

Un `AstNodeRebuilder` externo puede reconstruir nodes.

---

# 246. Recommendation

Para core nodes pequeños, métodos `with*()` selectivos son aceptables.

Para transformación genérica, utilizar `AstNodeRebuilder`.

---

# 247. No dozens of withers

No convertir cada node en una API enorme de mutación simulada.

---

# 248. Transformation context

Puede contener:

```text
AstNodePath
TransformationPhase
RuleId
Budget
```

No:

```text
PDO
Request
Container
```

---

# 249. Node validation

Cada node deberá tener invariantes locales.

---

# 250. Local invariant

Ejemplo:

```text
ComparisonPredicateNode
```

debe tener:

```text
left expression
operator
right expression
```

---

# 251. Cross-node invariant

Ejemplo:

```text
Insert column count
=
row expression count
```

requiere validación de una estructura mayor.

---

# 252. Semantic invariant

Ejemplo:

```text
column exists
```

no pertenece al Node Model.

---

# 253. Validation levels

```text
L1 Node Construction
L2 AST Structural Validation
L3 Semantic Validation
L4 Capability Validation
L5 Plan Validation
L6 Compilation Validation
```

---

# 254. Invalid states

Siempre que sea razonable:

> Make invalid states unrepresentable.

---

# 255. Example

En lugar de:

```php
new ComparisonPredicateNode(
    left: null,
    operator: null,
    right: null,
);
```

el constructor deberá exigir valores válidos.

---

# 256. But avoid overvalidation

No intentar validar en constructor información que aún no existe.

---

# 257. Node immutability enforcement

Preferencias:

```text
final
readonly
private validated collections
immutable value objects
```

---

# 258. Inheritance caution

`readonly` + inheritance compleja puede dificultar evolución.

Preferir composición.

---

# 259. Base abstract node

No es obligatorio tener:

```php
abstract class AbstractAstNode
```

---

# 260. If used

Sólo deberá contener comportamiento realmente común y estable.

No:

```text
metadata
children
compile
visitor
context
platform
```

todo mezclado.

---

# 261. Recommended core style

Ejemplo:

```php
final readonly class ComparisonPredicateNode implements PredicateNode
{
    public function __construct(
        public ExpressionNode $left,
        public ComparisonOperator $operator,
        public ExpressionNode $right,
    ) {}

    public function kind(): AstNodeKind
    {
        return CoreAstNodeKind::comparisonPredicate();
    }
}
```

---

# 262. External traversal

```text
AstTraverser
→ knows child schema
→ visits left
→ visits right
```

---

# 263. Node does not own traverser

No:

```php
$node->walk($visitor);
```

como requisito obligatorio.

---

# 264. Node Serialization

Si se habilita, deberá utilizar:

```text
NodeTypeId
SchemaVersion
Fields
Children
Safe Metadata
```

---

# 265. No PHP serialize contract

No usar `serialize()` como formato arquitectónico oficial.

---

# 266. Why

Porque acopla:

- nombres de clases;
- implementación;
- propiedades;
- seguridad;
- versiones PHP.

---

# 267. Deserialization validation

Todo AST deserializado deberá volver a validarse.

---

# 268. Extension deserialization

Si falta una extensión requerida:

```text
UnknownAstExtensionException
```

---

# 269. Fingerprinting nodes

Cada node contribuye mediante:

```text
NodeTypeId
+
structural fields
+
children
```

---

# 270. Metadata fingerprint policy

Cada metadata descriptor deberá indicar:

```text
STRUCTURAL
DIAGNOSTIC
IGNORED_FOR_SHAPE
```

o equivalente.

---

# 271. Parameter values

No deberán entrar en Query Shape fingerprint.

---

# 272. Raw fragments

Sí deberán afectar el fingerprint estructural porque alteran la consulta.

---

# 273. Fingerprint and extensions

Cada extension node deberá proporcionar una representación determinista.

---

# 274. Non-deterministic extension

Si una extensión no puede fingerprintarse de forma estable:

```text
cacheability = false
```

---

# 275. Node Cacheability

Podrá existir análisis:

```text
AstCacheability
├── CACHEABLE
├── CONDITIONALLY_CACHEABLE
└── NOT_CACHEABLE
```

---

# 276. Cacheability is derived

No deberá ser un mutable flag arbitrario dentro de cada node.

---

# 277. Node purity

Un node core deberá ser puro:

```text
same fields
=
same structural meaning
```

---

# 278. No callbacks

Evitar callbacks/closures dentro de AST nodes.

---

# 279. Why

Closures dificultan:

- fingerprinting;
- serialization;
- caching;
- concurrency;
- diagnostics.

---

# 280. Dynamic behavior

Debe resolverse antes de construir AST o mediante extension/service externo.

---

# 281. Example dynamic value

Incorrecto:

```text
ParameterNode
└── closure returning current user
```

---

# 282. Correct

```text
Application
→ resolve current user ID
→ BindingSet
→ ParameterId
```

---

# 283. Current timestamp semantics

Si se desea tiempo del servidor:

```text
CurrentTimestampExpressionNode
```

no una closure PHP.

---

# 284. Node portability

Cada core node deberá clasificarse conceptualmente:

```text
PORTABLE
CAPABILITY_DEPENDENT
```

---

# 285. Extension nodes

Podrán ser:

```text
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
```

---

# 286. Raw nodes

Serán:

```text
RAW
```

---

# 287. Portability classification use

Útil para:

- diagnostics;
- query linting;
- multi-platform tests;
- migration tooling.

---

# 288. Node capability requirements

No necesariamente se almacenarán directamente en el node.

Podrán derivarse mediante:

```text
AstNodeKind
+
Semantic Context
→ CapabilityRequirementSet
```

---

# 289. Why derived

Una misma estructura puede tener requirements diferentes según tipos/contexto.

---

# 290. Example

```text
JsonContainsExpression
```

puede requerir capabilities distintas según target/type.

---

# 291. AST and semantic annotations

El Semantic Engine producirá side tables.

Ejemplo:

```text
AstOccurrenceId
    │
    ├── ResolvedType
    ├── ResolvedSymbol
    ├── Nullability
    ├── Volatility
    ├── CapabilityRequirements
    └── SemanticProperties
```

---

# 292. Node remains unchanged

```text
Canonical AST
```

se mantiene target-neutral.

---

# 293. Node and optimization annotations

Optimizer tampoco deberá llenar nodes con:

```text
estimatedCost
selectedIndex
joinAlgorithm
```

---

# 294. Those belong to

```text
Logical Plan
Physical Plan
Optimization Metadata
```

---

# 295. Node and compilation annotations

No almacenar:

```text
compiledSql
placeholderPosition
quotedIdentifier
```

dentro del AST.

---

# 296. Those belong to Compiler

```text
CompilationContext
CompiledQuery
```

---

# 297. Node and execution state

Nunca almacenar:

```text
rowsAffected
cursor
statement
executionTime
retryCount
```

---

# 298. Node and telemetry

Telemetry puede referenciar:

```text
AstFingerprint
QueryShape
NodeCount
```

pero no mutar el AST.

---

# 299. Node lifecycle

Normal:

```text
Construct
   │
   ▼
Normalize
   │
   ▼
Validate
   │
   ▼
Publish immutable AST
   │
   ▼
Analyze
   │
   ▼
Transform / Optimize
   │
   ▼
Eventually GC/cache
```

---

# 300. Publish point

Una vez publicado como Canonical AST:

```text
no mutation
```

---

# 301. Construction phase mutability

Internamente una factory podrá usar builders mutables temporales por performance.

---

# 302. But

Esos builders:

```text
are not AST nodes
```

y no deberán escapar.

---

# 303. Persistent runtime safety

Un node podrá sobrevivir entre requests únicamente si:

- es immutable;
- no contiene request state;
- no contiene tenant objects;
- no contiene mutable services;
- no contiene connections;
- no contiene runtime callbacks.

---

# 304. FrankenPHP example

Seguro:

```text
Cached AST Shape
├── Table(users)
└── Predicate
    ├── Column(id)
    └── Parameter(p1)
```

No seguro:

```text
Cached AST
└── Parameter
    └── value = current request user
```

---

# 305. OpenSwoole concurrency

Dos coroutines podrán leer el mismo immutable AST.

No podrán compartir:

```text
mutable BindingSet
mutable SemanticContext
mutable ExecutionContext
```

---

# 306. AST vs Bindings

Separación:

```text
AST
+
BindingSet
```

---

# 307. BindingSet lifecycle

Normalmente:

```text
operation/request scope
```

---

# 308. AST lifecycle

Puede ser:

```text
operation
```

o cache-safe application/worker scope si cumple requisitos.

---

# 309. Testing model

Cada node core deberá tener unit tests para:

- construction;
- invariants;
- kind;
- children;
- equality;
- fingerprint;
- traversal;
- transformation;
- diagnostics.

---

# 310. Golden AST tests

Ejemplo:

```text
Builder Input
→ Expected AST Tree
```

---

# 311. Round-trip structural tests

Si existe serialization:

```text
AST
→ serialize
→ deserialize
→ structurally equal AST
```

---

# 312. Property tests

Útiles para:

- immutable transformations;
- equality symmetry;
- equality transitivity;
- hash consistency;
- normalization idempotence.

---

# 313. Architecture tests

Deberán detectar dependencias prohibidas desde:

```text
Query\Ast
```

hacia:

```text
Driver
Connection
ORM
Runtime
HTTP
PDO
```

---

# 314. Extension conformance suite

Una extensión AST deberá pasar pruebas para:

- stable kind;
- schema;
- traversal;
- validation;
- fingerprint;
- immutability;
- handler completeness;
- capability declaration.

---

# 315. Node diagnostic renderer

Podrá existir:

```text
AstNodeRenderer
```

para representación humana.

---

# 316. Example

```text
ComparisonPredicateNode
├── operator: EQUAL
├── left
│   └── ColumnReference(email)
└── right
    └── Parameter(p1)
```

---

# 317. No value leakage

En producción:

```text
Parameter(p1)
```

no:

```text
Parameter(p1 = "secret@example.com")
```

salvo política explícita de debugging seguro.

---

# 318. Suggested namespaces

```text
VoltStack\Quantum\Database\Query\Ast\
```

---

# 319. Proposed structure

```text
Query/
└── Ast/
    ├── Contract/
    │   ├── QueryAstNode.php
    │   ├── QueryRootNode.php
    │   ├── SourceNode.php
    │   ├── ProjectionNode.php
    │   ├── ExpressionNode.php
    │   ├── PredicateNode.php
    │   └── ExtensionAstNode.php
    │
    ├── Node/
    │   ├── Statement/
    │   ├── Source/
    │   ├── Projection/
    │   ├── Expression/
    │   ├── Predicate/
    │   ├── Join/
    │   ├── Cte/
    │   ├── Insert/
    │   ├── Mutation/
    │   ├── Conflict/
    │   ├── Returning/
    │   ├── Grouping/
    │   ├── Ordering/
    │   ├── Pagination/
    │   ├── Locking/
    │   ├── SetOperation/
    │   ├── Window/
    │   ├── Raw/
    │   └── Extension/
    │
    ├── Value/
    │   ├── AstNodeKind.php
    │   ├── AstNodeId.php
    │   ├── AstOccurrenceId.php
    │   ├── AstNodePath.php
    │   ├── Identifier.php
    │   └── ParameterId.php
    │
    ├── Collection/
    │   ├── ExpressionNodeList.php
    │   ├── PredicateNodeList.php
    │   ├── ProjectionNodeList.php
    │   ├── SourceNodeList.php
    │   ├── JoinNodeList.php
    │   └── AssignmentNodeList.php
    │
    ├── Schema/
    │   ├── AstNodeSchema.php
    │   ├── AstNodeDescriptor.php
    │   ├── AstChildDescriptor.php
    │   └── AstNodeSchemaRegistry.php
    │
    ├── Metadata/
    │   ├── AstMetadata.php
    │   ├── SourceLocation.php
    │   ├── ProvenanceMetadata.php
    │   └── MetadataKey.php
    │
    ├── Factory/
    │   └── AstNodeFactory.php
    │
    ├── Traversal/
    │   ├── AstTraverser.php
    │   └── AstTraversalContext.php
    │
    ├── Comparison/
    │   └── AstStructuralComparator.php
    │
    ├── Transformation/
    │   ├── AstTransformer.php
    │   └── AstNodeRebuilder.php
    │
    ├── Validation/
    │   └── AstNodeValidator.php
    │
    ├── Fingerprint/
    │   └── AstNodeFingerprint.php
    │
    ├── Diagnostics/
    │   ├── AstNodeRenderer.php
    │   └── AstTreeRenderer.php
    │
    ├── Extension/
    │   ├── AstExtensionRegistry.php
    │   └── ExtensionNodeDescriptor.php
    │
    └── Exception/
```

---

# 320. Namespace boundaries

`Query\Ast\Node` no deberá importar:

```text
ORM
Connection
Driver
Execution
Runtime
Telemetry implementation
```

---

# 321. Node category matrix

| Familia | Ejemplo | Children | Target-aware |
|---|---|---:|---|
| Root | SelectNode | sí | no |
| Source | TableSourceNode | sí | no |
| Projection | ExpressionProjectionNode | sí | no |
| Identifier | Identifier | no | no |
| Parameter | ParameterExpressionNode | no/metadata | no |
| Expression | FunctionCallNode | sí | no |
| Predicate | ComparisonPredicateNode | sí | no |
| Join | JoinNode | sí | no |
| CTE | CommonTableExpressionNode | sí | no |
| Insert | ValuesInsertSourceNode | sí | no |
| Mutation | AssignmentNode | sí | no |
| Conflict | ConflictNode | sí | no |
| Returning | ReturningNode | sí | no |
| Grouping | GroupByNode | sí | no |
| Ordering | OrderItemNode | sí | no |
| Pagination | PaginationNode | sí | no |
| Locking | LockNode | sí | no |
| Set Operation | SetOperationNode | sí | no |
| Window | WindowSpecificationNode | sí | no |
| Raw | RawExpressionNode | opcional | explícitamente no portable |
| Extension | ExtensionAstNode | variable | por descriptor |

---

# 322. Node composition example

Consulta conceptual:

```php
DB::table('users')
    ->select('id', 'name')
    ->where('active', true)
    ->orderBy('name')
    ->limit(20);
```

AST:

```text
SelectNode
│
├── ProjectionNodeList
│   ├── ExpressionProjectionNode
│   │   └── ColumnReferenceNode(id)
│   └── ExpressionProjectionNode
│       └── ColumnReferenceNode(name)
│
├── SourceNodeList
│   └── TableSourceNode
│       └── Identifier(users)
│
├── Where
│   └── ComparisonPredicateNode
│       ├── EQUAL
│       ├── ColumnReferenceNode(active)
│       └── ParameterExpressionNode(p1)
│
├── OrderByNode
│   └── OrderItemNode
│       ├── ColumnReferenceNode(name)
│       ├── ASC
│       └── DEFAULT
│
└── PaginationNode
    └── limit = 20
```

---

# 323. What this tree does not contain

No contiene:

```text
SELECT
FROM
WHERE
ORDER BY
LIMIT

?
:p1

PDO
MySQL
PostgreSQL
SQLite

Entity
Connection
Driver
```

como dependencias de compilación/ejecución.

---

# 324. DB-AST-NODE-001

Todo core AST node deberá poseer un `AstNodeKind` estable.

---

# 325. DB-AST-NODE-002

`AstNodeKind` no deberá depender obligatoriamente del nombre PHP de la clase.

---

# 326. DB-AST-NODE-003

Todo core node será inmutable una vez publicado.

---

# 327. DB-AST-NODE-004

Todo child de un node inmutable deberá ser también inmutable o tratado como immutable value.

---

# 328. DB-AST-NODE-005

Un node no ejecutará SQL.

---

# 329. DB-AST-NODE-006

Un node no generará SQL.

---

# 330. DB-AST-NODE-007

Un node no abrirá conexiones.

---

# 331. DB-AST-NODE-008

Un node no contendrá PDO/native connections.

---

# 332. DB-AST-NODE-009

Un node no contendrá EntityManager.

---

# 333. DB-AST-NODE-010

Un node no contendrá UnitOfWork.

---

# 334. DB-AST-NODE-011

Un node no contendrá ExecutionContext.

---

# 335. DB-AST-NODE-012

Un node no contendrá request-scoped mutable state.

---

# 336. DB-AST-NODE-013

Los identifiers permanecerán sin quoting SQL.

---

# 337. DB-AST-NODE-014

Los ParameterIds serán independientes de placeholders SQL.

---

# 338. DB-AST-NODE-015

Runtime values permanecerán separados del AST cuando sea posible.

---

# 339. DB-AST-NODE-016

Structural equality no utilizará object identity.

---

# 340. DB-AST-NODE-017

Structural equality no implicará semantic equality.

---

# 341. DB-AST-NODE-018

Semantic equality no será responsabilidad del Node Model.

---

# 342. DB-AST-NODE-019

Diagnostic metadata no modificará structural meaning.

---

# 343. DB-AST-NODE-020

SourceLocation no formará parte de Query Shape.

---

# 344. DB-AST-NODE-021

Los children tendrán orden determinista.

---

# 345. DB-AST-NODE-022

Las colecciones de children serán inmutables.

---

# 346. DB-AST-NODE-023

No existirán parent pointers mutables.

---

# 347. DB-AST-NODE-024

Recursive queries no utilizarán object cycles.

---

# 348. DB-AST-NODE-025

Structural sharing estará permitido.

---

# 349. DB-AST-NODE-026

Semantic annotations residirán fuera del node.

---

# 350. DB-AST-NODE-027

Optimization metadata residirá fuera del node.

---

# 351. DB-AST-NODE-028

Compilation metadata residirá fuera del node.

---

# 352. DB-AST-NODE-029

Execution metadata residirá fuera del node.

---

# 353. DB-AST-NODE-030

Platform y CapabilitySnapshot no serán children del AST.

---

# 354. DB-AST-NODE-031

Vendor-specific behavior no contaminará core nodes.

---

# 355. DB-AST-NODE-032

Raw SQL deberá usar nodes/escape hatches explícitos.

---

# 356. DB-AST-NODE-033

Raw nodes deberán permanecer distinguibles durante todo el pipeline.

---

# 357. DB-AST-NODE-034

Extension nodes deberán tener IDs namespaced.

---

# 358. DB-AST-NODE-035

Extension nodes deberán declarar schema version cuando sean serializables/cacheables.

---

# 359. DB-AST-NODE-036

Unknown required extension nodes nunca serán ignorados.

---

# 360. DB-AST-NODE-037

Todo extension node deberá ser recorrible de forma segura.

---

# 361. DB-AST-NODE-038

Un extension node cacheable deberá ser determinista e inmutable.

---

# 362. DB-AST-NODE-039

Los node schemas estarán congelados durante runtime normal.

---

# 363. DB-AST-NODE-040

No existirá un universal mutable node registry.

---

# 364. DB-AST-NODE-041

No se utilizará reflection obligatoria en el hot path si puede evitarse.

---

# 365. DB-AST-NODE-042

Los constructors deberán impedir estados localmente inválidos cuando exista suficiente información.

---

# 366. DB-AST-NODE-043

Los constructors no realizarán semantic database validation.

---

# 367. DB-AST-NODE-044

Los nodes no consultarán Schema Metadata directamente.

---

# 368. DB-AST-NODE-045

Los nodes no consultarán Service Container.

---

# 369. DB-AST-NODE-046

Los nodes no accederán a configuración global.

---

# 370. DB-AST-NODE-047

Los nodes no accederán a variables de entorno.

---

# 371. DB-AST-NODE-048

Los nodes no contendrán closures runtime salvo un mecanismo de extensión explícitamente no cacheable, y preferiblemente se prohibirán.

---

# 372. DB-AST-NODE-049

Node fingerprints serán consistentes con structural equality.

---

# 373. DB-AST-NODE-050

Fingerprints no incluirán secretos/runtime parameter values por defecto.

---

# 374. DB-AST-NODE-051

Los node kinds formarán parte del fingerprint.

---

# 375. DB-AST-NODE-052

El orden de children formará parte del fingerprint cuando sea semánticamente significativo.

---

# 376. DB-AST-NODE-053

Las transformaciones no mutarán nodes existentes.

---

# 377. DB-AST-NODE-054

Un transformer podrá reutilizar nodes sin cambios.

---

# 378. DB-AST-NODE-055

El mismo immutable node podrá ser compartido por múltiples AST snapshots cuando sea seguro.

---

# 379. DB-AST-NODE-056

Semantic side tables no utilizarán exclusivamente `spl_object_id()` como identidad.

---

# 380. DB-AST-NODE-057

Las ocurrencias estructurales deberán poder distinguirse aun con structural sharing.

---

# 381. DB-AST-NODE-058

Los root nodes deberán representar consultas estructuralmente completas.

---

# 382. DB-AST-NODE-059

Query AST y Schema AST mantendrán modelos de nodes separados.

---

# 383. DB-AST-NODE-060

El Query AST Node Model será la única taxonomía estructural canónica consumida por el Semantic Query Engine.

---

# 384. Anti-pattern — God Node

Incorrecto:

```php
final class QueryNode
{
    public string $sql;

    public array $bindings;

    public ?PDO $pdo;

    public ?EntityManager $entityManager;

    public ?Platform $platform;

    public array $children;

    public array $metadata;

    public function execute(): mixed {}
}
```

---

# 385. Anti-pattern — Generic Array AST

Incorrecto:

```php
[
    'type' => 'select',
    'data' => [
        'whatever' => ...
    ],
]
```

como representación interna principal.

---

# 386. Why

Esto pierde:

- type safety;
- IDE support;
- invariants;
- discoverability;
- extension contracts;
- static analysis.

---

# 387. Anti-pattern — keyword classes

No crear automáticamente:

```text
SelectKeywordNode
FromKeywordNode
WhereKeywordNode
AsKeywordNode
OrderKeywordNode
```

El AST modela estructura, no tokens SQL.

---

# 388. Anti-pattern — every value is a node

No convertir:

```text
ASC
DESC
TRUE
FALSE
```

en jerarquías innecesarias si enums/value objects son suficientes.

---

# 389. Anti-pattern — every clause is a node

No crear un `WhereNode` sólo porque SQL tiene la palabra `WHERE`.

Si:

```text
?PredicateNode $where
```

expresa mejor la estructura, utilizarlo.

---

# 390. Anti-pattern — direct platform subclassing

No:

```text
PostgreSqlSelectNode
MySqlSelectNode
SQLiteSelectNode
```

para consultas normales.

---

# 391. Correct

```text
SelectNode
+
Semantic Requirements
+
Capabilities
+
Planner
+
Target Compiler
```

---

# 392. Anti-pattern — mutable annotations

No:

```php
$node->resolvedType = ...;
$node->resolvedColumn = ...;
$node->chosenIndex = ...;
$node->sql = ...;
```

---

# 393. Correct

```text
Canonical AST
     │
     ├── Semantic Annotation Map
     ├── Optimization Metadata
     ├── Plan
     └── Compilation Context
```

---

# 394. Anti-pattern — current tenant node

No:

```text
CurrentTenantNode
└── TenantObject
```

si depende de mutable request state.

---

# 395. Correct tenant filtering

```text
Tenant Context
      │
      ▼
Query Policy
      │
      ▼
Predicate Injection
      │
      ▼
Canonical AST
      │
      └── Parameter(tenantId)
```

con el valor en BindingSet.

---

# 396. Anti-pattern — value interpolation

No:

```text
RawExpression("email = '$email'")
```

---

# 397. Correct

```text
ComparisonPredicate
├── Column(email)
└── Parameter(p1)

BindingSet
└── p1 → email value
```

---

# 398. Anti-pattern — object identity fingerprint

No:

```text
hash(spl_object_id($node))
```

---

# 399. Correct

```text
hash(
    NodeKind
    + StructuralFields
    + ChildFingerprints
)
```

---

# 400. Anti-pattern — shared mutable collection

No:

```text
SelectNode
└── ArrayObject<ExpressionNode>
```

modificable después de publicación.

---

# 401. Correct

```text
SelectNode
└── Immutable ProjectionNodeList
```

---

# 402. Anti-pattern — AST inheritance tree demasiado profundo

Evitar:

```text
Node
└── AbstractNode
    └── StatementNode
        └── QueryStatementNode
            └── ReadStatementNode
                └── SelectStatementNode
```

sin beneficios reales.

---

# 403. Preferred

```text
small interfaces
+
final immutable classes
+
composition
```

---

# 404. Node Model final

La arquitectura oficial queda:

```text
                    QueryAstNode
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
 QueryRootNode       SourceNode       ExpressionNode
       │                                   │
       │                                   ├── Column
       │                                   ├── Parameter
       │                                   ├── Literal
       │                                   ├── Function
       │                                   ├── Aggregate
       │                                   ├── Arithmetic
       │                                   ├── Case
       │                                   ├── Cast
       │                                   └── Subquery
       │
       ├── SelectNode
       ├── InsertNode
       ├── UpdateNode
       └── DeleteNode

                    PredicateNode
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Comparison           Logic          Specialized
                        │                │
                    AND/OR/NOT     NULL/IN/EXISTS/
                                   BETWEEN/LIKE

Other Structural Nodes
│
├── Projection
├── Join
├── CTE
├── Assignment
├── Conflict
├── Returning
├── Grouping
├── Ordering
├── Pagination
├── Locking
├── Set Operation
├── Window
├── Raw
└── Extension
```

---

# 405. Node processing model

```text
                  ┌──────────────────┐
                  │ Immutable Node   │
                  └────────┬─────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      Traverser        Validator       Comparator
          │                │                │
          ├────────────────┼────────────────┤
          │                │                │
          ▼                ▼                ▼
       Visitor        Transformer      Fingerprinter
                           │
                           ▼
                    New Immutable AST
```

---

# 406. Architectural formula

```text
AST Node
=
Immutable Structural Value
```

Mientras:

```text
Semantic Meaning
=
AST Node
+
Semantic Context
+
Symbol Resolution
+
Type Resolution
```

Y:

```text
Compiled SQL
=
Semantic Query
+
Plan
+
Dialect
+
Platform
+
Capabilities
+
Compiler
```

---

# 407. Separación crítica

```text
Node
≠
Semantic Symbol

Node
≠
Logical Plan Node

Node
≠
Physical Plan Node

Node
≠
SQL Fragment

Node
≠
Execution Operation
```

VoltStack deberá mantener estas representaciones separadas.

---

# 408. Regla maestra final

> **Los AST nodes de VoltStack serán valores estructurales inmutables y tipados que describen la forma de una consulta sin conocer cómo será resuelta, optimizada, compilada o ejecutada.**

En forma resumida:

```text
AST Node
     │
     ├── knows structure
     ├── knows children
     ├── knows semantic intent local
     └── knows its node kind

AST Node
     │
     ├── does NOT know SQL
     ├── does NOT know Driver
     ├── does NOT know Connection
     ├── does NOT know Platform instance
     ├── does NOT know ORM
     ├── does NOT know runtime
     └── does NOT know execution
```

---

# 409. Resultado arquitectónico

Con `25_DATABASE_QUERY_AST_SYSTEM.md` y este documento, VoltStack obtiene dos niveles claramente separados:

```text
25 Query AST System
│
│  Define:
│  ├── arquitectura AST
│  ├── lifecycle
│  ├── normalization
│  ├── traversal
│  ├── transformation
│  └── relación con Semantic/Optimizer/Planner/Compiler
│
└── 26 Query AST Node Model
    │
    ├── node taxonomy
    ├── node kinds
    ├── children
    ├── collections
    ├── identity
    ├── structural equality
    ├── metadata
    ├── extension nodes
    └── node invariants
```

Esto evita que las siguientes capas tengan que interpretar estructuras ambiguas.

---

# 410. Próximo nivel

El siguiente documento deberá especializar una de las familias más importantes:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

La secuencia queda:

```text
24 Query Model
       │
       ▼
25 Query AST System
       │
       ▼
26 Query AST Node Model
       │
       ├─────────────────────┐
       ▼                     ▼
27 Expression System    28 Predicate System
       │                     │
       └──────────┬──────────┘
                  ▼
29 Parameter & Binding
                  │
                  ▼
30 Query Type System
                  │
                  ▼
31 Query Metadata
                  │
                  ▼
32 Query Context
                  │
                  ▼
33 Query Normalization
                  │
                  ▼
34 Query Validation
                  │
                  ▼
35 Semantic Query Architecture
```

---

# 411. Decisión final

La implementación de V1 deberá favorecer:

```text
Small Interfaces
+
Final Immutable Nodes
+
Typed Value Objects
+
Typed Immutable Collections
+
External Traversal
+
External Transformation
+
External Semantic Annotations
+
Composition over Inheritance
```

Esta combinación proporciona una base suficientemente rigurosa para construir posteriormente:

```text
Expression Engine
Predicate Engine
Semantic Analyzer
Symbol Resolver
Type Inference
Query Optimizer
Logical Planner
Physical Planner
SQL Compiler
```

sin introducir dependencias circulares ni convertir el AST en un objeto monolítico.