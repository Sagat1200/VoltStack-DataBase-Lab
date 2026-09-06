# 25_DATABASE_QUERY_AST_SYSTEM.md

# VoltStack Quantum Database
## Query AST System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 25 — Query AST System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Canonical Intermediate Representation  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el **Query Abstract Syntax Tree System — Query AST System** de `VoltStack/Quantum/Database`.

El Query AST será la representación interna canónica utilizada por VoltStack para analizar, transformar, validar, optimizar, planificar y finalmente compilar consultas.

Su posición arquitectónica será:

```text
Query Builder
      │
      ▼
Query Model
      │
      ▼
QUERY AST
      │
      ▼
Semantic Analysis
      │
      ▼
Semantic Query
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
CompiledQuery
      │
      ▼
Executor
```

El AST deberá representar **estructura e intención**, no SQL específico de un motor.

---

# 2. Decisión arquitectónica principal

VoltStack utilizará un AST propio.

No utilizará como representación interna primaria:

- strings SQL;
- arrays arbitrarios;
- objetos del Query Builder;
- objetos PDO;
- estructuras internas de Doctrine;
- estructuras internas de Laravel;
- sintaxis específica de MySQL;
- sintaxis específica de PostgreSQL;
- sintaxis específica de MariaDB;
- sintaxis específica de SQLite.

El AST será parte de la arquitectura propia de VoltStack.

---

# 3. Regla fundamental

```text
Query Model
    │
    ▼
Canonical AST
    │
    ▼
Semantic Engine
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

Nunca:

```text
Query Model
    │
    ▼
SQL String
```

---

# 4. AST ≠ SQL

Un AST:

```text
SelectNode
├── Projection
├── Source
├── Predicate
└── Ordering
```

no es equivalente a:

```sql
SELECT id, name
FROM users
WHERE active = ?
ORDER BY name;
```

El primero representa estructura.

El segundo representa una compilación concreta.

---

# 5. AST ≠ Query Model

Aunque ambos representan una consulta, tienen responsabilidades diferentes.

| Query Model | Query AST |
|---|---|
| cercano al Builder | cercano al Engine |
| orientado a construcción | orientado a procesamiento |
| conserva conceptos de API | representación canónica |
| puede contener sugar semántico | reduce sugar |
| fácil de construir | fácil de recorrer |
| puede tener estructuras convenience | estructura normalizada |
| frontera de DX | IR interna |

---

# 6. AST como Intermediate Representation

El Query AST será una **IR — Intermediate Representation** del Query Engine.

Conceptualmente:

```text
Developer Query
      │
      ▼
High-Level Representation
      │
      ▼
Canonical AST
      │
      ▼
Semantic Representation
      │
      ▼
Logical Plan
      │
      ▼
Physical Plan
      │
      ▼
SQL
```

---

# 7. Objetivos

El AST deberá ser:

- estructurado;
- canónico;
- inmutable;
- portable;
- tipado;
- determinista;
- recorrible;
- transformable;
- extensible de forma controlada;
- independiente del Driver;
- independiente del Connection;
- independiente del ORM;
- independiente del runtime;
- independiente del SQL final.

---

# 8. No objetivos

El AST no deberá:

- ejecutar consultas;
- abrir conexiones;
- generar SQL directamente;
- contener PDO;
- contener EntityManager;
- contener UnitOfWork;
- contener entidades;
- conocer el pool;
- seleccionar replicas;
- manejar transacciones;
- resolver credenciales;
- consultar el Service Container;
- almacenar estado global mutable.

---

# 9. Arquitectura general

```text
Query Model
    │
    ▼
QueryAstFactory
    │
    ▼
AstRootNode
    │
    ├── Statement Nodes
    ├── Source Nodes
    ├── Expression Nodes
    ├── Predicate Nodes
    ├── Clause Nodes
    ├── Identifier Nodes
    ├── Parameter Nodes
    └── Extension Nodes
            │
            ▼
      AST Infrastructure
            │
      ┌─────┼──────────────┐
      ▼     ▼              ▼
 Traversal Visitor    Transformer
      │     │              │
      └─────┼──────────────┘
            ▼
       Validation
            │
            ▼
      Fingerprinting
            │
            ▼
     Semantic Engine
```

---

# 10. AST root

Toda consulta estructurada deberá poseer un root node.

Conceptualmente:

```php
interface QueryAstNode
{
}
```

y:

```php
interface QueryRootNode extends QueryAstNode
{
    public function queryType(): QueryType;
}
```

---

# 11. Root node taxonomy

Inicialmente:

```text
QueryRootNode
├── SelectNode
├── InsertNode
├── UpdateNode
└── DeleteNode
```

Podrán agregarse nuevos root nodes mediante mecanismos de extensión oficiales.

---

# 12. No God AST Node

No utilizar:

```php
final class QueryNode
{
    public ?array $select;
    public ?array $insert;
    public ?array $update;
    public ?array $delete;

    public mixed $whatever;
}
```

Preferir tipos especializados.

---

# 13. Node taxonomy

La taxonomía conceptual será:

```text
QueryAstNode
│
├── QueryRootNode
│
├── SourceNode
│
├── ExpressionNode
│
├── PredicateNode
│
├── ClauseNode
│
├── IdentifierNode
│
├── ParameterNode
│
└── ExtensionNode
```

No necesariamente todos deberán compartir interfaces artificiales si no existe comportamiento común real.

---

# 14. Composition over inheritance

El AST utilizará principalmente composición.

```text
SelectNode
├── WithNode
├── ProjectionNodeList
├── SourceNodeList
├── JoinNodeList
├── PredicateNode
├── GroupByNode
├── HavingNode
├── OrderByNode
├── PaginationNode
├── LockNode
└── Metadata
```

---

# 15. AST inmutable

Los AST nodes deberán ser inmutables.

Preferencia PHP:

```php
final readonly class SelectNode implements QueryRootNode
{
    // ...
}
```

cuando resulte apropiado.

---

# 16. Razón de inmutabilidad

La inmutabilidad facilita:

- concurrencia;
- persistent workers;
- structural sharing;
- caching;
- fingerprinting;
- transformations;
- debugging;
- reproducibilidad;
- reasoning;
- tests.

---

# 17. No mutation in place

Incorrecto:

```php
$node->where = $newPredicate;
```

Preferir:

```text
AST A
 │
 ▼
Transformer
 │
 ▼
AST B
```

---

# 18. Structural sharing

Una transformación podrá reutilizar nodos inmutables que no hayan cambiado.

Ejemplo:

```text
AST A
├── Projection ─────────────┐
├── Source ─────────────────┤ reused
└── Predicate A             │
                            │
AST B                       │
├── Projection ◄────────────┤
├── Source ◄────────────────┘
└── Predicate B
```

---

# 19. Node identity

El AST deberá distinguir entre:

```text
Structural Identity
```

y:

```text
Runtime Object Identity
```

Dos objetos PHP diferentes podrán representar exactamente el mismo nodo estructural.

---

# 20. Structural equality

Conceptualmente:

```text
Column(users.id)
==
Column(users.id)
```

aunque sean instancias diferentes.

---

# 21. Node IDs

No todos los nodes necesitan un ID persistente.

Podrán existir IDs cuando sean necesarios para:

- diagnostics;
- semantic references;
- rewrite tracking;
- provenance;
- mapping entre etapas.

---

# 22. Node ID ≠ semantic identity

Un `AstNodeId` identifica una ocurrencia del AST.

No necesariamente identifica:

- una columna real;
- una tabla real;
- una entidad;
- un símbolo semántico.

---

# 23. Node IDs deterministas

Si se utilizan IDs en fingerprints o caches, deberán ser deterministas o excluirse del fingerprint.

---

# 24. Source location

Los nodes podrán contener opcionalmente:

```text
SourceLocation
├── origin
├── file
├── line
├── column
└── builder operation
```

para diagnostics.

---

# 25. Source location no semántica

La ubicación del código no deberá cambiar el significado de la consulta.

Por tanto:

```text
SourceLocation
```

normalmente deberá excluirse de structural equality y semantic fingerprints.

---

# 26. Node metadata

Podrá existir metadata asociada a nodes.

Deberá distinguirse entre:

```text
SemanticMetadata
DiagnosticMetadata
ExtensionMetadata
```

---

# 27. No arbitrary metadata

Evitar un:

```php
array<string, mixed> $metadata;
```

sin reglas.

Las extensiones deberán utilizar namespaces y contratos tipados.

---

# 28. SelectNode

Representará un SELECT canónico.

Conceptualmente:

```php
final readonly class SelectNode implements QueryRootNode
{
    public function __construct(
        public WithNode $with,
        public DistinctNode $distinct,
        public ProjectionNodeList $projection,
        public SourceNodeList $sources,
        public JoinNodeList $joins,
        public ?PredicateNode $where,
        public GroupByNode $groupBy,
        public ?PredicateNode $having,
        public SetOperationNodeList $setOperations,
        public OrderByNode $orderBy,
        public ?PaginationNode $pagination,
        public ?LockNode $locking,
        public AstMetadata $metadata,
    ) {}
}
```

La implementación final podrá ajustar la forma exacta.

---

# 29. InsertNode

Conceptualmente:

```text
InsertNode
├── With
├── Target
├── Columns
├── InsertSource
├── Conflict
├── Returning
└── Metadata
```

---

# 30. UpdateNode

```text
UpdateNode
├── With
├── Target
├── Assignments
├── Auxiliary Sources
├── Joins
├── Predicate
├── Ordering
├── Pagination
├── Returning
└── Metadata
```

---

# 31. DeleteNode

```text
DeleteNode
├── With
├── Target
├── Auxiliary Sources
├── Predicate
├── Ordering
├── Pagination
├── Returning
└── Metadata
```

---

# 32. AST canonicalization

El AST deberá utilizar una representación suficientemente canónica.

Ejemplo de Builder:

```php
$query
    ->where('active', true)
    ->where('verified', true);
```

podrá convertirse a:

```text
AndPredicateNode
├── EqualityPredicateNode
│   ├── ColumnReferenceNode(active)
│   └── ParameterNode(p1)
│
└── EqualityPredicateNode
    ├── ColumnReferenceNode(verified)
    └── ParameterNode(p2)
```

---

# 33. Canonical form

Una representación canónica facilita:

- semantic analysis;
- optimizer rules;
- fingerprinting;
- cache hits;
- compiler simplicity;
- testing.

---

# 34. Canonicalization ≠ optimization

Esta distinción será obligatoria.

```text
Canonicalization
=
convert equivalent construction forms
into common structural representation
```

Mientras:

```text
Optimization
=
transform query while preserving semantics
to improve execution characteristics
```

---

# 35. Ejemplo de canonicalization

Estas APIs:

```php
->where('id', '=', 10)
```

y:

```php
->where('id', 10)
```

podrán producir el mismo:

```text
EqualityPredicateNode
```

---

# 36. Ejemplo de optimization

```text
NOT (a != b)
```

podría eventualmente simplificarse según las reglas semánticas aplicables.

Eso no corresponde necesariamente a AST construction.

---

# 37. AST normalization stages

Se recomienda distinguir:

```text
Query Model
    │
    ▼
Structural Normalization
    │
    ▼
Canonical AST
    │
    ▼
Semantic Normalization
    │
    ▼
Semantic Query Representation
    │
    ▼
Optimization
```

---

# 38. Structural normalization

Podrá incluir:

- eliminar Builder sugar;
- normalizar operadores equivalentes;
- convertir listas a colecciones canónicas;
- agrupar predicates;
- normalizar aliases;
- asignar parameter identities;
- validar estructura básica.

---

# 39. Structural normalization no resolverá

No deberá resolver:

- tipos reales de columnas;
- existencia de tablas;
- aliases contra schema;
- capabilities de plataforma;
- estrategia SQL;
- índices;
- join costs.

---

# 40. Unresolved AST

El AST inicial podrá contener referencias no resueltas.

Ejemplo:

```text
UnresolvedColumnReferenceNode("email")
```

---

# 41. Semantic resolution

Posteriormente:

```text
UnresolvedColumnReferenceNode("email")
            │
            ▼
Semantic Analyzer
            │
            ▼
ResolvedColumnSymbol
```

---

# 42. AST preservation

Se recomienda no mutar el AST original para convertir referencias unresolved en resolved.

El Semantic Engine podrá construir una representación semántica separada.

---

# 43. AST vs Semantic Graph

```text
AST
=
syntactic/structural relationships

Semantic Graph
=
resolved semantic relationships
```

---

# 44. Ejemplo

AST:

```text
ColumnReferenceNode("id")
```

Semantic Graph:

```text
ColumnReference
    │
    ▼
Symbol#42
    │
    ▼
users.id : bigint
```

---

# 45. AST no necesita Schema

La construcción básica del AST deberá ser posible sin conectarse a la base de datos.

---

# 46. Offline construction

Debe ser posible:

```text
Application
→ Builder
→ Query Model
→ AST
```

sin servidor disponible.

---

# 47. AST Source Nodes

Taxonomía conceptual:

```text
SourceNode
├── TableSourceNode
├── SubquerySourceNode
├── CteReferenceNode
├── DerivedTableNode
├── FunctionSourceNode
└── ExtensionSourceNode
```

---

# 48. TableSourceNode

Conceptualmente:

```php
final readonly class TableSourceNode implements SourceNode
{
    public function __construct(
        public QualifiedIdentifierNode $identifier,
        public ?AliasNode $alias,
    ) {}
}
```

---

# 49. No quoted table names

Incorrecto:

```text
TableSourceNode("`users`")
```

Correcto:

```text
TableSourceNode
└── IdentifierNode(users)
```

---

# 50. SubquerySourceNode

```text
SubquerySourceNode
├── SelectNode
└── AliasNode
```

---

# 51. CTE AST

Conceptualmente:

```text
WithNode
├── recursive: semantic flag
└── CommonTableExpressionNodeList
    ├── CommonTableExpressionNode
    └── CommonTableExpressionNode
```

---

# 52. CTE node

```text
CommonTableExpressionNode
├── Name
├── Optional Column Names
├── Query Root
└── Options
```

---

# 53. Recursive CTE

El AST representará la intención de recursividad.

No almacenará:

```text
WITH RECURSIVE
```

como string.

---

# 54. Join AST

```text
JoinNode
├── JoinType
├── SourceNode
├── JoinCondition
└── Metadata
```

---

# 55. Join condition taxonomy

Conceptualmente:

```text
JoinCondition
├── PredicateJoinCondition
├── UsingJoinCondition
├── NaturalJoinCondition
└── NoJoinCondition
```

si las variantes son necesarias.

---

# 56. Join semantic validity

El AST podrá representar una feature antes de saber si el target la soporta.

La validación de capabilities ocurrirá posteriormente.

---

# 57. Expression AST

Se desarrollará formalmente en:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

pero conceptualmente:

```text
ExpressionNode
├── ColumnReferenceNode
├── ParameterExpressionNode
├── LiteralNode
├── FunctionCallNode
├── AggregateNode
├── ArithmeticNode
├── CaseNode
├── CastNode
├── SubqueryExpressionNode
├── TupleNode
└── ExtensionExpressionNode
```

---

# 58. Predicate AST

Se desarrollará en:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Conceptualmente:

```text
PredicateNode
├── ComparisonPredicateNode
├── AndPredicateNode
├── OrPredicateNode
├── NotPredicateNode
├── NullPredicateNode
├── BetweenPredicateNode
├── InPredicateNode
├── ExistsPredicateNode
├── LikePredicateNode
└── ExtensionPredicateNode
```

---

# 59. Predicate is semantic boolean expression

VoltStack deberá decidir explícitamente si:

```text
PredicateNode
```

es subtipo de:

```text
ExpressionNode
```

o un dominio separado.

---

# 60. Recomendación

Mantener inicialmente:

```text
ExpressionNode
```

y:

```text
PredicateNode
```

como categorías distintas con interoperabilidad explícita.

Esto ayuda a evitar aceptar expresiones no booleanas accidentalmente donde se requiere una condición.

---

# 61. Future unification

Si el Type System demuestra que una jerarquía unificada es más correcta, podrá evolucionarse internamente.

No deberá decidirse únicamente por imitar SQL.

---

# 62. Identifier AST

Taxonomía conceptual:

```text
IdentifierNode
├── SimpleIdentifierNode
├── QualifiedIdentifierNode
├── AliasIdentifierNode
├── SchemaIdentifierNode
├── TableIdentifierNode
└── ColumnIdentifierNode
```

No es obligatorio crear clases diferentes cuando un value object tipado sea suficiente.

---

# 63. Identifier value

Los identifiers almacenarán el nombre lógico sin quoting SQL.

---

# 64. Identifier quoting

Ocurre:

```text
Identifier AST
     │
     ▼
Compiler
     │
     ▼
Dialect Identifier Strategy
     │
     ▼
SQL identifier
```

---

# 65. Identifier validation

Se distinguen:

```text
structural identifier validation
```

de:

```text
semantic identifier resolution
```

---

# 66. Structural validation

Ejemplos:

- identifier vacío;
- segmentos inválidos;
- alias estructuralmente inválido.

---

# 67. Semantic validation

Ejemplos:

- tabla inexistente;
- columna inexistente;
- alias ambiguo;
- columna ambigua.

---

# 68. Parameter AST

Los parámetros deberán poseer identidad semántica independiente del placeholder final.

```text
ParameterNode
├── ParameterId
├── Optional Type Hint
└── Metadata
```

---

# 69. Parameter value separation

Cuando sea posible:

```text
AST Shape
     │
     └── ParameterId

Bindings
     │
     └── ParameterId → Runtime Value
```

---

# 70. Benefit

Permite que:

```text
WHERE email = "a@example.com"
```

y:

```text
WHERE email = "b@example.com"
```

compartan la misma estructura AST.

---

# 71. Parameter ordering

El AST no deberá depender innecesariamente del placeholder ordering de un Driver.

---

# 72. Placeholder allocation

Será responsabilidad posterior de:

```text
Compiler
+
SqlBindingProfile
+
ParameterAllocator
```

---

# 73. Literal nodes

No todo valor deberá convertirse automáticamente en literal SQL.

Valores dinámicos deberán ser parámetros.

---

# 74. LiteralNode

Se reservará para valores estructurales seguros cuando sea semánticamente apropiado.

Ejemplos:

```text
NULL
TRUE
FALSE
```

o constantes internas según Type System y compiler.

---

# 75. No user interpolation

Incorrecto:

```text
LiteralNode($_GET['email'])
```

como forma de saltarse bindings.

---

# 76. Projection AST

```text
ProjectionNodeList
├── ExpressionProjectionNode
├── ExpressionProjectionNode
└── WildcardProjectionNode
```

---

# 77. Projection alias

```text
ExpressionProjectionNode
├── Expression
└── Optional Alias
```

---

# 78. Wildcard

El wildcard será un node explícito.

No:

```text
ColumnIdentifier("*")
```

si eso pierde semántica.

---

# 79. Qualified wildcard

```text
QualifiedWildcardNode
└── SourceReference
```

---

# 80. Assignment AST

UPDATE:

```text
AssignmentNode
├── Target
└── Expression
```

---

# 81. Insert source AST

```text
InsertSourceNode
├── ValuesInsertSourceNode
├── QueryInsertSourceNode
└── DefaultValuesInsertSourceNode
```

---

# 82. Values source

```text
ValuesInsertSourceNode
└── InsertRowNodeList
    ├── InsertRowNode
    └── InsertRowNode
```

---

# 83. Row structural invariant

Cada row deberá mantener cardinalidad compatible con la target column list.

La validación estructural podrá detectarlo.

---

# 84. Conflict AST

Upsert deberá representarse como:

```text
ConflictNode
├── Target
└── Action
```

---

# 85. Conflict target

```text
ConflictTargetNode
├── ColumnSet
├── ConstraintReference
├── Inference
└── Optional Predicate
```

dependiendo del modelo semántico requerido.

---

# 86. Conflict action

```text
ConflictActionNode
├── IgnoreConflictNode
└── UpdateConflictNode
```

---

# 87. No vendor upsert AST

No crear:

```text
OnDuplicateKeyNode
```

como representación portable principal.

Tampoco:

```text
PostgresOnConflictSqlNode
```

en el core portable.

---

# 88. Vendor-specific extension

Si existe una feature genuinamente vendor-specific, deberá entrar por:

```text
ExtensionNode
+
Capability Requirement
+
Platform-specific Compiler Extension
```

---

# 89. Returning AST

```text
ReturningNode
└── ProjectionNodeList
```

---

# 90. Returning is semantic

El AST puede representarlo aunque todavía no se conozca la plataforma.

---

# 91. Capability resolution later

```text
ReturningNode
      │
      ▼
Semantic Requirements
      │
      ▼
Capability Resolver
      │
      ▼
Planner Strategy
```

---

# 92. Grouping AST

```text
GroupByNode
└── ExpressionNodeList
```

---

# 93. Having AST

```text
Having
└── PredicateNode
```

El contexto HAVING se conserva por su posición estructural.

---

# 94. Ordering AST

```text
OrderByNode
└── OrderItemNodeList
```

---

# 95. Order item

```text
OrderItemNode
├── Expression
├── Direction
└── NullOrdering
```

---

# 96. Pagination AST

```text
PaginationNode
├── Limit
└── Offset
```

---

# 97. Pagination semantic representation

No deberá asumir:

```text
LIMIT x OFFSET y
```

como syntax final.

---

# 98. Lock AST

```text
LockNode
├── Mode
├── WaitPolicy
└── TargetList
```

---

# 99. Lock requirements

Semantic Analysis podrá derivar:

```text
row locking capability
primary connection
transaction semantics
```

---

# 100. Set operations AST

```text
SetOperationNode
├── Operation
├── Left
└── Right
```

o una representación equivalente.

---

# 101. Set operation tree

Ejemplo:

```text
UnionNode
├── SelectNode A
└── SelectNode B
```

---

# 102. Associativity

La estructura deberá preservar agrupación cuando pueda afectar semántica.

---

# 103. Parentheses

No deberán modelarse como strings.

La estructura del árbol representa grouping.

---

# 104. AST traversal

El sistema deberá ofrecer traversal estándar.

Conceptualmente:

```php
interface AstTraverserInterface
{
    public function traverse(QueryAstNode $node, AstVisitorInterface $visitor): void;
}
```

---

# 105. Traversal order

Deberán existir reglas deterministas.

Por ejemplo:

```text
PRE_ORDER
POST_ORDER
```

cuando sea necesario.

---

# 106. Default traversal

Se recomienda una estrategia canónica documentada para cada node.

---

# 107. Child ordering

El orden de children deberá ser estable.

Esto afecta:

- fingerprints;
- visitors;
- diagnostics;
- tests;
- transformations.

---

# 108. Visitor

Visitor será útil para operaciones read-only.

Ejemplos:

- diagnostics;
- parameter collection;
- dependency collection;
- source inspection;
- fingerprinting;
- metrics.

---

# 109. Visitor contract

Conceptualmente:

```php
interface AstVisitorInterface
{
    public function enter(QueryAstNode $node): void;

    public function leave(QueryAstNode $node): void;
}
```

La API real podrá ser más tipada.

---

# 110. Typed visitors

Podrán existir visitors especializados.

```text
ParameterCollector
SourceCollector
AstDebugPrinter
AstFingerprintVisitor
```

---

# 111. No giant visitor

Evitar:

```text
OneVisitorWithEveryPossibleBehavior
```

---

# 112. Transformer

Las transformaciones deberán utilizar una abstracción separada del Visitor read-only.

Conceptualmente:

```php
interface AstTransformerInterface
{
    public function transform(QueryRootNode $query): QueryRootNode;
}
```

---

# 113. Transformer behavior

```text
AST A
  │
  ▼
Transformer
  │
  ▼
AST B
```

---

# 114. No in-place rewriting

Una rewrite rule nunca deberá modificar un node compartido.

---

# 115. NodeTransformer

Podrá existir:

```text
NodeTransformer
```

para transformaciones locales.

---

# 116. TreeTransformer

Y:

```text
TreeTransformer
```

para recorridos completos.

---

# 117. Transformation result

Para optimizar structural sharing:

```text
unchanged node
→ same immutable instance may be reused

changed node
→ new instance
```

---

# 118. Rewrite tracking

En development podrá registrarse:

```text
RewriteTrace
├── Rule
├── BeforeFingerprint
├── AfterFingerprint
└── Reason
```

---

# 119. Rewrite trace no producción por defecto

No deberá introducir overhead elevado en producción.

---

# 120. AST validation

Deberá existir un:

```text
AstValidator
```

para invariantes estructurales.

---

# 121. Structural validation examples

Detectará:

- SELECT sin estructura válida;
- INSERT sin target;
- UPDATE sin assignments;
- malformed CTE;
- malformed join;
- invalid set operation shape;
- invalid pagination;
- malformed parameter references;
- duplicate structural IDs cuando no sean válidos.

---

# 122. AST validator no Schema

No validará necesariamente:

```text
users.email exists
```

Eso corresponde al Semantic Engine.

---

# 123. Validation stages

```text
AST Construction
      │
      ▼
Structural Validation
      │
      ▼
Semantic Validation
      │
      ▼
Capability Validation
      │
      ▼
Plan Validation
      │
      ▼
Compiler Validation
```

---

# 124. Fail early

Los errores deberán detectarse en la etapa más temprana que posea información suficiente.

---

# 125. AST fingerprinting

VoltStack deberá soportar fingerprints deterministas.

Conceptualmente:

```text
AstFingerprint
```

---

# 126. Fingerprint purpose

Podrá utilizarse para:

- caching;
- deduplication;
- diagnostics;
- rewrite detection;
- query shape analysis;
- testing.

---

# 127. Fingerprint exclusions

Normalmente excluir:

- runtime parameter values;
- source code locations;
- object IDs;
- timestamps;
- request IDs;
- trace IDs.

---

# 128. Fingerprint inclusions

Normalmente incluir:

- node type;
- child structure;
- identifiers;
- operators;
- clauses;
- parameter type shape;
- semantic flags;
- extension node identity.

---

# 129. AST fingerprint ≠ compiled query fingerprint

```text
AstFingerprint
```

no es suficiente para identificar SQL compilado.

---

# 130. Compiled cache key

Conceptualmente:

```text
CompiledQueryCacheKey
=
Ast/Semantic Fingerprint
+
Dialect Fingerprint
+
Platform Fingerprint
+
Capability Fingerprint
+
Binding Profile
+
Compiler Version
```

---

# 131. Fingerprint algorithm

El algoritmo deberá ser:

- determinista;
- versionado;
- independiente de object identity;
- suficientemente rápido.

---

# 132. Fingerprint version

Ejemplo conceptual:

```text
AstFingerprintVersion(1)
```

Cambios estructurales incompatibles podrán incrementar versión.

---

# 133. Structural hash

Cada node podrá contribuir:

```text
NodeKind
+
Semantic Fields
+
Ordered Child Hashes
```

---

# 134. Commutativity caution

No reordenar children únicamente porque un operador parezca conmutativo.

SQL posee:

- NULL semantics;
- volatile functions;
- side effects en algunas extensiones;
- evaluation concerns.

Las rewrites deberán basarse en reglas semánticas demostrables.

---

# 135. AST serialization

La serialización del AST no será requisito obligatorio de V1.

---

# 136. Si se implementa

Deberá ser:

- explícita;
- versionada;
- segura;
- independiente de `serialize()`;
- capaz de rechazar extension nodes desconocidos.

---

# 137. AST cache serialization

Un futuro cache de AST compilado podrá utilizar un formato interno.

No deberá considerarse automáticamente API pública.

---

# 138. AST debug printer

Deberá existir una representación humana.

Ejemplo:

```text
SelectNode
├── Projection
│   ├── Column(users.id)
│   └── Column(users.name)
├── Source
│   └── Table(users)
└── Where
    └── Equal
        ├── Column(users.active)
        └── Parameter<bool>(p1)
```

---

# 139. Debug AST ≠ executable SQL

La representación de debug nunca deberá enviarse al Driver.

---

# 140. AST visualization

En development podrá ofrecerse:

```text
ASCII Tree
JSON-like Debug Tree
Graph Representation
```

---

# 141. Debug toolbar integration

`Quantum/Telemetry` podrá mostrar:

```text
Builder
Query Model
AST
Semantic Plan
Execution Plan
Compiled SQL
```

según configuración.

---

# 142. Production safety

No deberán exponerse automáticamente:

- parameter values sensibles;
- secrets;
- credentials;
- tenant secrets;
- raw personal data.

---

# 143. AST provenance

Opcionalmente podrá conocerse el origen:

```text
Application
ORM
Repository
Migration Internal Query
Framework
Extension
```

---

# 144. Provenance purpose

Útil para:

- diagnostics;
- telemetry;
- policies;
- debugging.

No deberá crear dependencias inversas.

---

# 145. AST extension model

VoltStack deberá permitir extensión controlada del AST.

---

# 146. Extension node contract

Conceptualmente:

```php
interface ExtensionAstNode extends QueryAstNode
{
    public function extensionId(): ExtensionId;

    public function nodeType(): ExtensionNodeType;
}
```

---

# 147. Extension requirements

Una extensión AST deberá declarar:

- node identity;
- semantic handler;
- traversal behavior;
- validation behavior;
- fingerprint behavior;
- planning support;
- compiler support;
- capability requirements.

según corresponda.

---

# 148. No opaque extension nodes

No permitir:

```text
ExtensionNode
└── mixed $data
```

sin schema ni handlers.

---

# 149. Extension node schema

Cada extension node deberá poseer estructura definida y versionada.

---

# 150. Unknown extension node

Si una etapa no sabe procesar un node requerido:

```text
UnsupportedAstNodeException
```

en lugar de ignorarlo.

---

# 151. No silent node dropping

Regla crítica:

> Ninguna etapa podrá descartar silenciosamente un AST node que pueda afectar la semántica.

---

# 152. Extension registry

Podrá existir:

```text
AstExtensionRegistry
```

especializado.

No será un universal registry.

---

# 153. Registry freeze

Registro:

```text
Bootstrap
→ Mutable registration
→ Validation
→ Freeze
→ Runtime immutable
```

---

# 154. Extension ordering

Si múltiples extensiones transforman AST:

```text
dependency graph
+
explicit ordering
```

deberá determinar el orden.

---

# 155. No last-wins

Conflictos de node handlers deberán producir error explícito salvo decoration/replacement declarado.

---

# 156. Vendor-specific AST

El core AST deberá favorecer semántica portable.

---

# 157. Ejemplo correcto

```text
JsonExtractExpressionNode
```

si existe una semántica framework definida.

---

# 158. Ejemplo incorrecto

```text
PostgreSqlArrowArrowNode
```

como primera opción del core.

---

# 159. Escape hatch

Cuando una feature sea genuinamente específica:

```text
PlatformSpecificExpressionNode
```

o extension node explícito podrá utilizarse.

---

# 160. Portability marker

El node podrá declarar:

```text
PORTABLE
CAPABILITY_DEPENDENT
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

cuando resulte útil.

---

# 161. AST and Capability System

El AST no resolverá capabilities por sí mismo.

---

# 162. Correct flow

```text
AST
 │
 ▼
Semantic Analysis
 │
 ▼
Capability Requirements
 │
 ▼
EffectiveCapabilitySet
 │
 ▼
Planner
```

---

# 163. No vendor checks in AST

Prohibido:

```php
if ($database === 'mysql') {
    ...
}
```

dentro de AST nodes.

---

# 164. No Platform objects in nodes

No:

```text
SelectNode
└── PostgreSqlPlatform
```

---

# 165. Platform-specific requirement

Puede representarse mediante metadata/requirement cuando una extensión lo necesite.

---

# 166. AST and Type System

El AST podrá contener:

```text
DeclaredType
TypeHint
UnknownType
```

pero la inferencia completa ocurrirá después.

---

# 167. Type states

Conceptualmente:

```text
UNKNOWN
DECLARED
INFERRED
RESOLVED
```

La representación exacta deberá evitar mutar nodes durante análisis.

---

# 168. Type annotations

Se recomienda que los resultados de inferencia vivan en:

```text
SemanticModel
```

en lugar de modificar AST nodes.

---

# 169. AST and symbols

AST references no serán necesariamente symbols resueltos.

---

# 170. Symbol resolution

```text
AST Identifier
      │
      ▼
Symbol Resolver
      │
      ▼
Semantic Symbol
```

---

# 171. Symbol table

No deberá almacenarse globalmente dentro del AST.

---

# 172. AST and Query Context

Query Context no deberá convertirse en un child del AST.

---

# 173. Context separation

```text
AST
=
query structure

QueryContext
=
external processing context
```

---

# 174. QueryContext examples

Podrá contener:

- target connection;
- platform snapshot;
- schema metadata snapshot;
- execution scope;
- extension context;
- policies.

---

# 175. No contextual contamination

El mismo AST podrá analizarse contra diferentes targets.

---

# 176. Ejemplo

```text
AST X
├── analyze against PostgreSQL
├── analyze against MySQL
└── analyze against SQLite
```

si su semántica es portable.

---

# 177. Benefit

Esto permite:

- portability analysis;
- migration tools;
- query linting;
- offline compilation;
- multi-platform testing.

---

# 178. AST and ORM

ORM podrá producir Query Model/AST mediante la misma infraestructura.

---

# 179. No ORM nodes in core AST

Evitar:

```text
EntityNode(User)
```

como elemento del AST SQL/query core.

---

# 180. Correct flow

```text
ORM Query
   │
   ▼
ORM Translator
   │
   ▼
Query Model
   │
   ▼
AST
```

---

# 181. Persistence flow

```text
UnitOfWork
   │
   ▼
Persistence Planner
   │
   ▼
Insert/Update/Delete Query Model
   │
   ▼
AST
```

---

# 182. AST remains ORM-independent

El Query Engine deberá funcionar completamente sin ORM.

---

# 183. AST and Schema

DDL utilizará un:

```text
Schema AST
```

separado.

---

# 184. Query AST ≠ Schema AST

```text
Query AST
→ DML / query operations

Schema AST
→ DDL / schema operations
```

---

# 185. Shared primitives

Podrán compartir:

- identifiers;
- type descriptors;
- expression primitives;
- metadata infrastructure;

si las dependencias permanecen limpias.

---

# 186. No universal AST

No crear necesariamente:

```text
OneAstToRuleThemAll
```

para Query + Schema + Migration + ORM.

---

# 187. Migration

Migration Planner podrá producir Schema AST/operations.

No deberá deformar Query AST para representar DDL.

---

# 188. AST and transactions

No habrá:

```text
BeginTransactionNode
CommitNode
RollbackNode
```

dentro del Query AST normal.

Transaction Manager es otra capa.

---

# 189. Transaction requirements

Una query podrá generar requisitos semánticos.

Ejemplo:

```text
LockNode
→ transaction-related requirements
```

---

# 190. AST and Connection

AST no contiene:

- Connection;
- ConnectionLease;
- NativeConnection;
- Pool;
- Driver.

---

# 191. AST and runtime

AST no deberá conocer:

- FrankenPHP;
- RoadRunner;
- OpenSwoole;
- HTTP;
- Queue;
- Scheduler.

---

# 192. Persistent runtime safety

Los AST nodes inmutables podrán sobrevivir técnicamente más tiempo que un request siempre que no contengan state scoped.

---

# 193. No request references

No almacenar:

```text
Request
CurrentUser
Tenant object
EntityManager
DatabaseContext
Transaction
Connection
```

en nodes cacheables.

---

# 194. Tenant semantics

Si una query requiere tenant filtering:

```text
Tenant Policy
    │
    ▼
Query Transformation
    │
    ▼
AST with explicit predicate
```

o una capa semántica equivalente.

No mediante mutable global tenant state dentro del AST.

---

# 195. Security policy transformations

Ejemplo:

```text
Original AST
    │
    ▼
Security Query Policy
    │
    ▼
Restricted AST
```

---

# 196. Policy transformation visibility

En development deberá poder diagnosticarse:

```text
original
+
policy transformations
+
effective AST
```

cuando sea seguro.

---

# 197. AST rewrite architecture

Las transformaciones deberán organizarse en pipelines explícitos.

```text
AST
 │
 ▼
Normalization Pipeline
 │
 ▼
Policy Pipeline
 │
 ▼
Semantic Analysis
 │
 ▼
Optimizer Rewrite Pipeline
```

---

# 198. Pipelines must not blur stages

Una normalization rule no deberá empezar a seleccionar índices.

Una security policy no deberá compilar SQL.

Un optimizer rule no deberá abrir conexiones.

---

# 199. AST normalization rules

Ejemplos posibles:

```text
NormalizeComparisonOperator
NormalizePredicateGrouping
NormalizeProjectionShape
NormalizeInsertSource
NormalizeAliases
NormalizeSetOperationStructure
```

---

# 200. Rule contract

Conceptualmente:

```php
interface AstNormalizationRule
{
    public function supports(QueryAstNode $node): bool;

    public function normalize(
        QueryAstNode $node,
        AstNormalizationContext $context,
    ): QueryAstNode;
}
```

---

# 201. Deterministic rules

Misma entrada + mismo contexto:

```text
same output
```

---

# 202. Idempotence

Cuando sea posible:

```text
normalize(normalize(AST))
=
normalize(AST)
```

---

# 203. Fixed-point processing

Si algunas reglas requieren iteración, el pipeline deberá:

- tener límites;
- detectar ciclos;
- ser determinista;
- reportar non-convergence.

---

# 204. No infinite rewrites

El sistema deberá detectar:

```text
Rule A: X → Y
Rule B: Y → X
```

---

# 205. Rewrite budget

Podrá existir:

```text
AstRewriteBudget
```

para evitar loops patológicos.

---

# 206. Normalization context

Debe ser mínimo.

No incluir arbitrariamente:

```text
Container
Request
PDO
```

---

# 207. AST versioning

El formato conceptual del AST tendrá una versión interna.

---

# 208. AstVersion

Conceptualmente:

```text
AstVersion
├── major
└── minor
```

si la serialización/cache lo necesita.

---

# 209. Versioning purpose

Importante para:

- persisted caches;
- extension compatibility;
- compiled artifacts;
- tooling.

---

# 210. Internal API evolution

El AST podrá evolucionar más rápido que la API pública del Query Builder.

---

# 211. Extension stability

Los contratos marcados como `Extension` requerirán mayor estabilidad.

---

# 212. AST public exposure

El AST no deberá convertirse accidentalmente en API pública sólo porque pueda inspeccionarse.

---

# 213. API classification

```text
Public
Extension
Internal
Implementation
```

---

# 214. Public

Podrá incluir:

- interfaces de tooling cuidadosamente seleccionadas;
- diagnostic views;
- stable extension contracts.

---

# 215. Internal

Podrá incluir:

- node implementations;
- normalization internals;
- traversal optimizations;
- fingerprint encoding.

---

# 216. AST builder

Podrá existir un:

```text
AstFactory
```

o:

```text
AstNodeFactory
```

para construcción controlada.

---

# 217. AstNodeFactory

Puede centralizar:

- canonical empty nodes;
- validated collections;
- interned immutable values;
- node construction rules.

---

# 218. Avoid God Factory

No deberá concentrar:

- semantic analysis;
- optimization;
- planning;
- compilation.

---

# 219. Empty nodes

Para colecciones puede preferirse:

```text
ProjectionNodeList::empty()
JoinNodeList::empty()
```

en lugar de `null`.

---

# 220. Optional clauses

Para una cláusula realmente ausente:

```text
?LockNode
```

puede ser correcto.

---

# 221. Null vs empty

La diferencia deberá ser semánticamente intencional.

---

# 222. Node collections

Colecciones especializadas:

```text
ProjectionNodeList
SourceNodeList
JoinNodeList
ExpressionNodeList
PredicateNodeList
OrderItemNodeList
AssignmentNodeList
CteNodeList
SetOperationNodeList
```

---

# 223. Collection immutability

También deberán ser inmutables.

---

# 224. Collection invariants

Ejemplo:

```text
ProjectionNodeList
```

sólo acepta projection nodes.

---

# 225. Generic AstNodeList

Puede existir internamente como base utilitaria.

No deberá destruir type safety.

---

# 226. Performance considerations

Un AST completo puede contener muchos objetos.

VoltStack deberá medir:

- allocations;
- traversal cost;
- transformation cost;
- memory;
- fingerprint cost.

---

# 227. No premature scalarization

No reemplazar todo por arrays y integers antes de medir.

La claridad arquitectónica tiene prioridad en V1.

---

# 228. Future compact AST

Si profiling demuestra necesidad, podrá existir una representación compacta interna.

---

# 229. Compact representation invariant

No deberá cambiar la semántica ni romper contratos de capas superiores.

---

# 230. AST interning

Value objects frecuentes podrán internarse si:

- son inmutables;
- el beneficio está demostrado;
- no generan memory leaks en persistent workers.

---

# 231. Persistent worker caution

Un global intern pool ilimitado sería peligroso.

---

# 232. Bounded caches

Cualquier cache AST global deberá ser:

- bounded;
- observable;
- resettable/versioned;
- libre de request references.

---

# 233. AST memory ownership

Normalmente:

```text
Query operation
owns
AST
```

---

# 234. Cache ownership

Si se cachea:

```text
Compiled Query Cache
```

deberá almacenar sólo artefactos seguros para application/worker scope.

---

# 235. AST error hierarchy

Conceptualmente:

```text
QueryAstException
├── InvalidAstException
├── InvalidAstNodeException
├── InvalidAstStructureException
├── UnsupportedAstNodeException
├── AstNormalizationException
├── AstTransformationException
├── AstTraversalException
├── AstFingerprintException
├── AstExtensionException
└── AstVersionException
```

---

# 236. Error context

Los errores podrán incluir:

- node kind;
- node path;
- safe source location;
- extension ID;
- normalization rule;
- fingerprint.

---

# 237. No sensitive values

Los parameter values deberán redacted por defecto.

---

# 238. Node path

Una utilidad conceptual:

```text
AstNodePath
```

podrá representar:

```text
Select
→ Where
→ And[1]
→ Comparison
→ RightOperand
```

---

# 239. Node path use

Útil para:

- diagnostics;
- validation;
- extension errors;
- tooling.

---

# 240. AST inspector

Podrá existir:

```text
AstInspector
```

read-only.

---

# 241. Inspector queries

Ejemplos:

```text
containsSubquery()
containsLocking()
containsMutation()
containsRawExpression()
collectParameters()
collectSources()
```

---

# 242. Inspector no optimizer

No deberá modificar el AST.

---

# 243. Query complexity metrics

El AST permitirá medir:

- node count;
- nesting depth;
- join count;
- subquery count;
- predicate complexity;
- parameter count.

---

# 244. Security limits

Podrán establecerse límites contra queries generadas patológicamente.

Ejemplos:

```text
max AST depth
max parameter count
max predicate count
max CTE count
max union branches
```

---

# 245. Limits are policy/config

No deberán hardcodearse arbitrariamente dentro de nodes.

---

# 246. Query complexity protection

Útil para:

- dynamic query APIs;
- GraphQL-like integrations;
- report builders;
- user-generated filters.

---

# 247. AST validation budget

Traversal/validation podrá tener budgets para prevenir estructuras abusivas.

---

# 248. Cycle safety

El AST conceptual debe ser un árbol o DAG controlado.

---

# 249. Cyclic references

No deberán permitirse referencias PHP cíclicas arbitrarias entre nodes.

---

# 250. Recursive query ≠ cyclic object graph

Un recursive CTE deberá representarse mediante referencias semánticas estructuradas.

No mediante:

```text
CTENode
└── points directly to itself
```

como ciclo de objetos.

---

# 251. DAG possibility

Structural sharing puede convertir la representación física en DAG.

Semánticamente seguirá comportándose como árbol estructurado.

---

# 252. Traversal cycle protection

La infraestructura deberá ser segura ante extension nodes defectuosos.

---

# 253. AST and volatile expressions

La futura semántica deberá poder distinguir:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

cuando una expresión lo requiera.

---

# 254. Why

Esto afecta qué optimizaciones son seguras.

---

# 255. AST does not guess volatility

Será proporcionada por:

- function metadata;
- platform semantics;
- extension metadata;
- semantic analysis.

---

# 256. AST and determinism

Deterministic AST construction no significa que la query sea determinista.

Ejemplo:

```text
RandomFunctionNode
```

puede formar un AST determinista que representa una operación runtime no determinista.

---

# 257. AST and NULL semantics

El AST deberá conservar suficiente estructura para que Semantic Engine respete SQL three-valued logic.

---

# 258. No unsafe boolean simplification

Ejemplo:

```text
NOT (x = NULL)
```

no deberá simplificarse usando lógica booleana convencional.

---

# 259. Semantic optimizer requirement

Las rewrites posteriores deberán considerar:

- NULL;
- types;
- collations;
- volatility;
- platform semantics.

---

# 260. AST and collation

Una expresión podrá representar collation semántica si la API la expone.

La syntax final será Dialect concern.

---

# 261. AST and casts

```text
CastExpressionNode
├── Expression
└── TargetType
```

---

# 262. Cast target

Será un type descriptor semántico, no:

```text
"VARCHAR(255)"
```

como SQL fragment.

---

# 263. AST and functions

```text
FunctionCallNode
├── FunctionIdentifier
└── Arguments
```

---

# 264. Semantic function ID

Preferir:

```text
FunctionId("string.lower")
```

o una abstracción equivalente cuando sea portable.

---

# 265. Native function

Funciones específicas podrán usar extension namespace.

---

# 266. Function syntax

Dialect/Compiler decidirán si una función se expresa como:

- normal call;
- keyword;
- operator;
- special syntax.

---

# 267. AST and operators

Operadores deberán utilizar IDs/enums semánticos.

---

# 268. No SQL operator fragments

Evitar:

```text
Operator("ILIKE")
```

como core portable si la intención puede representarse semánticamente.

---

# 269. Operator extension

Un operador nativo puede registrarse explícitamente como extensión.

---

# 270. AST and JSON

Representar operaciones semánticas:

```text
JsonExtract
JsonContains
JsonPath
JsonArrayLength
```

cuando formen parte de la API.

No almacenar syntax de `->`, `->>`, etc. en core.

---

# 271. AST and date/time

Igualmente:

```text
DateAdd
DateDiff
ExtractDatePart
```

deberán representar intención.

---

# 272. AST and window functions

Conceptualmente:

```text
WindowFunctionNode
├── Function
└── WindowSpecificationNode
```

---

# 273. Window specification

```text
WindowSpecificationNode
├── PartitionBy
├── OrderBy
└── Frame
```

---

# 274. Window frame

Deberá representarse estructuralmente.

No como:

```text
"ROWS BETWEEN ..."
```

---

# 275. AST and aggregate filters

Si se soporta:

```text
AggregateNode
├── Function
├── Arguments
├── Distinct
└── FilterPredicate
```

sujeto a capabilities/planning.

---

# 276. AST and CASE

```text
CaseExpressionNode
├── Operand?
├── WhenThenList
└── Else?
```

---

# 277. AST and tuples

```text
TupleExpressionNode
└── ExpressionList
```

útil para:

- row comparisons;
- composite IN;
- multi-column operations.

---

# 278. AST and EXISTS

```text
ExistsPredicateNode
└── SelectNode
```

---

# 279. AST and IN

```text
InPredicateNode
├── LeftExpression
└── ValueSource
    ├── ExpressionList
    └── Subquery
```

---

# 280. Empty IN

La semántica de:

```php
whereIn('id', [])
```

deberá normalizarse explícitamente.

---

# 281. No invalid SQL generation

El Builder/Query Model/AST normalization podrá convertirlo a una constante semántica:

```text
FalsePredicateNode
```

cuando corresponda.

---

# 282. NOT IN empty

Podrá convertirse a:

```text
TruePredicateNode
```

si las reglas semánticas de la API así lo definen.

---

# 283. Constant predicate nodes

Podrán existir:

```text
TruePredicateNode
FalsePredicateNode
```

---

# 284. Benefits

Facilitan:

- empty collections;
- policy composition;
- optimizer;
- deterministic compilation.

---

# 285. AST and raw expressions

Deberá existir un escape hatch explícito.

---

# 286. TrustedRawExpressionNode

Conceptualmente:

```text
TrustedRawExpressionNode
├── SQL fragment
├── Parameters
├── Portability metadata
└── Trust metadata
```

---

# 287. Raw node isolation

El uso de Raw:

- reduce optimización;
- reduce portability;
- reduce semantic analysis;
- puede afectar caching;
- requiere tratamiento de seguridad especial.

---

# 288. Raw AST is not ordinary AST

Las etapas deberán saber explícitamente que existe una zona opaca.

---

# 289. Raw SQL values

Aun dentro de Raw, valores dinámicos deberán utilizar bindings.

---

# 290. Raw node fingerprint

Deberá incluir el fragmento estructural y shape de parámetros, pero nunca secretos innecesarios.

---

# 291. Raw query root

Una consulta SQL completamente raw podrá utilizar:

```text
RawSqlQuery
```

fuera del AST estructurado normal.

---

# 292. Do not parse implicitly

VoltStack no deberá intentar parsear automáticamente cualquier SQL raw para convertirlo en AST salvo que en el futuro exista un parser SQL explícito.

---

# 293. AST creation pipeline

Propuesta:

```text
Query Model
    │
    ▼
QueryModelValidator
    │
    ▼
QueryAstFactory
    │
    ▼
Initial AST
    │
    ▼
AstStructuralNormalizer
    │
    ▼
Canonical AST
    │
    ▼
AstValidator
```

---

# 294. QueryAstFactory responsibility

Sólo:

```text
Query Model
→ AST
```

---

# 295. QueryAstFactory must not

No deberá:

- resolve schema;
- connect DB;
- optimize;
- choose platform strategy;
- compile SQL.

---

# 296. AstStructuralNormalizer responsibility

Convertir variantes estructurales equivalentes a representación canónica.

---

# 297. AstValidator responsibility

Garantizar invariantes del AST canónico.

---

# 298. Output contract

La salida podrá representarse conceptualmente como:

```text
CanonicalQueryAst
```

---

# 299. CanonicalQueryAst

Podría contener:

```text
root
astVersion
fingerprint?
diagnostic metadata?
```

sin convertirlo en God object.

---

# 300. AST snapshot

Para pipelines puede utilizarse:

```text
QueryAstSnapshot
```

que represente una versión inmutable del árbol.

---

# 301. Snapshot use

Útil para:

- rewrite tracing;
- diagnostics;
- testing;
- stage boundaries.

---

# 302. Avoid unnecessary snapshots

No duplicar árboles completos en producción si structural sharing es suficiente.

---

# 303. Semantic handoff

La frontera siguiente será:

```text
Canonical AST
     │
     ▼
SemanticAnalyzer
```

---

# 304. Semantic analyzer output

No deberá ser simplemente el mismo AST con propiedades mutadas.

Preferencia:

```text
SemanticQuery
├── AST reference
├── Symbol Graph
├── Type Map
├── Requirement Set
├── Semantic Annotations
└── Diagnostics
```

---

# 305. Semantic side tables

Información derivada podrá mantenerse en side tables inmutables:

```text
NodeId → ResolvedSymbol
NodeId → ResolvedType
NodeId → Nullability
NodeId → Volatility
NodeId → CapabilityRequirement
```

---

# 306. Benefit

Mantiene el AST:

- reusable;
- immutable;
- target-neutral.

---

# 307. Multiple semantic analyses

Un AST podrá analizarse contra:

```text
PostgreSQL Target
MySQL Target
SQLite Target
```

produciendo diferentes semantic snapshots.

---

# 308. AST target neutrality

Esto será una propiedad central de VoltStack.

---

# 309. AST and Optimizer

Optimizer no deberá trabajar directamente sobre Builder objects.

---

# 310. Correct

```text
AST / Semantic Query
      │
      ▼
Optimizer
```

---

# 311. Query rewrite vs plan optimization

También deberán separarse.

```text
AST Rewrite
→ semantic structure transformations

Plan Optimization
→ execution strategy transformations
```

---

# 312. Example AST rewrite

```text
AND(TRUE, predicate)
→ predicate
```

si es semánticamente seguro.

---

# 313. Example plan optimization

```text
Join order A-B-C
→ A-C-B
```

cuando el framework posea suficiente información y la optimización sea apropiada.

---

# 314. AST and Planner

Planner consumirá representación semántica y requirements.

No deberá reconstruir intención desde SQL.

---

# 315. AST and Compiler

Compiler recibirá un plan o representación compilable.

---

# 316. Compiler must not mutate AST

El mismo AST podrá ser compilado para múltiples targets.

---

# 317. Example

```text
AST
├── compile → MySQL
├── compile → PostgreSQL
└── compile → SQLite
```

si sus capabilities lo permiten.

---

# 318. Compiler output

```text
CompiledQuery
├── sql
├── bindings
├── parameterTypes
└── executionMetadata
```

---

# 319. SQL exists only later

SQL no deberá aparecer antes de Compilation salvo:

- Raw SQL escape hatch;
- diagnostic examples;
- explicit native extension fragments.

---

# 320. AST concurrency

AST inmutable podrá compartirse de forma segura entre fibers/coroutines si todos sus children también son inmutables.

---

# 321. Extension immutability requirement

Un extension node declarado shareable deberá ser inmutable.

---

# 322. Unsafe extension node

Si contiene mutable scoped state:

```text
must not be cacheable/shareable
```

y preferiblemente deberá rechazarse por arquitectura.

---

# 323. AST lifecycle

Normal:

```text
Operation
├── Query Model
├── AST
├── Semantic Query
├── Plan
└── Compiled Query
```

---

# 324. Cache lifecycle

Sólo artefactos comprobados como scope-safe podrán ascender a:

```text
worker/application cache
```

---

# 325. FrankenPHP

No se permitirá:

```text
Request A AST
└── mutable tenant/request state
       │
       ▼
Request B
```

---

# 326. RoadRunner/OpenSwoole

La misma regla aplica.

El AST no dependerá del runtime.

---

# 327. AST security

El AST contribuirá a seguridad evitando que datos dinámicos sean tratados como SQL.

---

# 328. Structural distinction

```text
Identifier
≠
Parameter
≠
Literal
≠
Raw SQL
```

será explícita.

---

# 329. Security benefit

Esto reduce:

- accidental interpolation;
- identifier/value confusion;
- raw SQL proliferation;
- SQL injection paths.

---

# 330. AST is not a security boundary alone

La seguridad completa requiere:

- safe Builder APIs;
- parameter binding;
- identifier validation;
- Compiler;
- Driver;
- policies;
- raw escape hatch restrictions.

---

# 331. Tainted input

En el futuro podrá existir metadata de trust para identifiers/raw fragments.

No deberá reemplazar validación real.

---

# 332. AST complexity limits

Configuración conceptual:

```text
query.ast.max_depth
query.ast.max_nodes
query.ast.max_parameters
query.ast.max_subqueries
query.ast.max_ctes
query.ast.max_set_operations
```

---

# 333. Defaults

Deberán ser suficientemente amplios para aplicaciones normales y configurables.

---

# 334. Internal framework queries

Podrán utilizar perfiles diferentes si existe una necesidad demostrada.

---

# 335. AST telemetry

Métricas posibles:

```text
database.query.ast.nodes
database.query.ast.depth
database.query.ast.parameters
database.query.ast.normalization.duration
database.query.ast.rewrites
```

---

# 336. Telemetry optional

El AST no dependerá directamente de `Quantum/Telemetry`.

Usará ports/events/hooks apropiados.

---

# 337. Debugging stages

VoltStack podrá mostrar:

```text
Query Builder
     │
     ▼
Query Model
     │
     ▼
Initial AST
     │
     ▼
Canonical AST
     │
     ▼
Semantic Query
     │
     ▼
Optimized Query
     │
     ▼
Plan
     │
     ▼
Compiled SQL
```

---

# 338. Explain architecture

Esto permitirá eventualmente:

```text
DB::explain(...)
```

mostrar no sólo el plan del servidor, sino también el pipeline interno de VoltStack.

---

# 339. Framework explain

Podrá distinguir:

```text
VoltStack Query Explain
```

de:

```text
Database Server EXPLAIN
```

---

# 340. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Ast\
```

---

# 341. Proposed directory structure

```text
Query/
└── Ast/
    ├── Contract/
    │   ├── QueryAstNode.php
    │   ├── QueryRootNode.php
    │   ├── SourceNode.php
    │   ├── ExpressionNode.php
    │   ├── PredicateNode.php
    │   └── ExtensionAstNode.php
    │
    ├── Statement/
    │   ├── SelectNode.php
    │   ├── InsertNode.php
    │   ├── UpdateNode.php
    │   └── DeleteNode.php
    │
    ├── Source/
    │   ├── TableSourceNode.php
    │   ├── SubquerySourceNode.php
    │   ├── CteReferenceNode.php
    │   └── DerivedTableNode.php
    │
    ├── Projection/
    │   ├── ProjectionNode.php
    │   ├── ExpressionProjectionNode.php
    │   ├── WildcardProjectionNode.php
    │   └── QualifiedWildcardNode.php
    │
    ├── Join/
    │   ├── JoinNode.php
    │   ├── JoinType.php
    │   └── JoinCondition.php
    │
    ├── Cte/
    │   ├── WithNode.php
    │   └── CommonTableExpressionNode.php
    │
    ├── Insert/
    │   ├── InsertSourceNode.php
    │   ├── ValuesInsertSourceNode.php
    │   ├── QueryInsertSourceNode.php
    │   ├── DefaultValuesInsertSourceNode.php
    │   └── InsertRowNode.php
    │
    ├── Mutation/
    │   ├── MutationTargetNode.php
    │   ├── AssignmentNode.php
    │   └── ReturningNode.php
    │
    ├── Conflict/
    │   ├── ConflictNode.php
    │   ├── ConflictTargetNode.php
    │   └── ConflictActionNode.php
    │
    ├── Grouping/
    ├── Ordering/
    ├── Pagination/
    ├── Locking/
    ├── SetOperation/
    ├── Identifier/
    ├── Parameter/
    ├── Expression/
    ├── Predicate/
    ├── Metadata/
    ├── Collection/
    ├── Factory/
    ├── Normalization/
    ├── Validation/
    ├── Traversal/
    ├── Visitor/
    ├── Transformation/
    ├── Fingerprint/
    ├── Diagnostics/
    ├── Extension/
    └── Exception/
```

---

# 342. Dependency direction

```text
Query Model
     │
     ▼
AST
     │
     ▼
Semantic
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

Nunca al revés.

---

# 343. Forbidden AST dependencies

Architecture tests deberán impedir dependencias hacia:

```text
PDO
Driver implementations
Connection Pool
EntityManager
UnitOfWork
IdentityMap
HTTP Request
FrankenPHP
RoadRunner
OpenSwoole
Service Container
Application Auth State
```

---

# 344. DB-AST-001

Todo query estructurado deberá tener un root node explícito.

---

# 345. DB-AST-002

El AST nunca generará SQL directamente.

---

# 346. DB-AST-003

El AST nunca ejecutará consultas.

---

# 347. DB-AST-004

El AST será independiente del Driver.

---

# 348. DB-AST-005

El AST será independiente de PDO.

---

# 349. DB-AST-006

El AST será independiente de Connection.

---

# 350. DB-AST-007

El AST será independiente del ORM.

---

# 351. DB-AST-008

El AST será independiente del runtime.

---

# 352. DB-AST-009

Los AST nodes serán inmutables por defecto.

---

# 353. DB-AST-010

Las transformaciones producirán nuevos trees/nodes en lugar de mutar los existentes.

---

# 354. DB-AST-011

Structural sharing estará permitido únicamente con objetos inmutables.

---

# 355. DB-AST-012

El AST representará intención y estructura, no syntax vendor-specific.

---

# 356. DB-AST-013

Identifiers no estarán pre-quoted.

---

# 357. DB-AST-014

Parameters no contendrán placeholder syntax nativa.

---

# 358. DB-AST-015

Valores dinámicos deberán representarse como parameters.

---

# 359. DB-AST-016

Raw SQL será explícito y distinguible.

---

# 360. DB-AST-017

No se descartarán extension nodes desconocidos silenciosamente.

---

# 361. DB-AST-018

El AST tendrá traversal determinista.

---

# 362. DB-AST-019

Child ordering será estable.

---

# 363. DB-AST-020

AST fingerprinting será determinista y versionado.

---

# 364. DB-AST-021

Runtime parameter values no formarán normalmente parte del structural fingerprint.

---

# 365. DB-AST-022

Diagnostic metadata no deberá alterar semantic equality.

---

# 366. DB-AST-023

Source locations no deberán alterar la semántica.

---

# 367. DB-AST-024

Canonicalization y optimization serán etapas diferentes.

---

# 368. DB-AST-025

Structural validation y semantic validation serán diferentes.

---

# 369. DB-AST-026

Capability validation no se realizará mediante vendor checks en nodes.

---

# 370. DB-AST-027

Un AST podrá construirse offline.

---

# 371. DB-AST-028

Un AST portable podrá analizarse contra múltiples targets.

---

# 372. DB-AST-029

Semantic analysis no mutará el AST original.

---

# 373. DB-AST-030

Información semántica derivada deberá mantenerse en estructuras separadas o annotations inmutables.

---

# 374. DB-AST-031

No habrá mutable global current AST.

---

# 375. DB-AST-032

No habrá mutable global current platform dentro del AST.

---

# 376. DB-AST-033

No habrá mutable global tenant state dentro del AST.

---

# 377. DB-AST-034

AST nodes cacheables no contendrán request-scoped references.

---

# 378. DB-AST-035

Recursive queries no crearán object cycles arbitrarios.

---

# 379. DB-AST-036

Collections de nodes serán inmutables y tipadas.

---

# 380. DB-AST-037

No existirá un God AST Node.

---

# 381. DB-AST-038

No existirá un universal AST registry.

---

# 382. DB-AST-039

Extension registries se congelarán antes del runtime normal.

---

# 383. DB-AST-040

Extension handlers deberán declarar compatibilidad.

---

# 384. DB-AST-041

AST transformations deberán ser deterministas para la misma entrada/contexto.

---

# 385. DB-AST-042

Normalization deberá ser idempotente cuando sea razonablemente posible.

---

# 386. DB-AST-043

Rewrite loops deberán detectarse.

---

# 387. DB-AST-044

AST traversal deberá protegerse contra estructuras inválidas de extensions.

---

# 388. DB-AST-045

Query AST y Schema AST serán dominios separados.

---

# 389. DB-AST-046

Transactions no se modelarán como Query AST statements normales.

---

# 390. DB-AST-047

ORM entities no serán nodes del Query AST core.

---

# 391. DB-AST-048

Platform objects no serán children del AST.

---

# 392. DB-AST-049

Compiler objects no serán children del AST.

---

# 393. DB-AST-050

Connection objects no serán children del AST.

---

# 394. DB-AST-051

AST debug output nunca será ejecutable por definición.

---

# 395. DB-AST-052

Sensitive parameter values estarán redacted en diagnostics.

---

# 396. DB-AST-053

Raw nodes deberán indicar explícitamente su pérdida de portability.

---

# 397. DB-AST-054

Vendor-specific behavior deberá entrar mediante extension/capability mechanisms.

---

# 398. DB-AST-055

El AST deberá preservar grouping semánticamente significativo.

---

# 399. DB-AST-056

No se aplicarán simplificaciones booleanas inseguras ignorando NULL semantics.

---

# 400. DB-AST-057

No se reordenarán expresiones sin conocer volatility y semantic safety.

---

# 401. DB-AST-058

El AST deberá soportar complexity inspection.

---

# 402. DB-AST-059

Los límites de complejidad serán policies configurables.

---

# 403. DB-AST-060

El AST será la representación estructural canónica de entrada al Semantic Query Engine.

---

# 404. Anti-pattern — SQL AST

Incorrecto:

```text
SelectNode
├── sql = "SELECT..."
└── bindings = [...]
```

Eso no es el AST de VoltStack.

---

# 405. Anti-pattern — mutable nodes

Incorrecto:

```php
$select->joins[] = $join;
```

después de publicar el AST.

---

# 406. Anti-pattern — driver-aware AST

Incorrecto:

```php
if ($driver === 'pdo_pgsql') {
    return new ReturningNode(...);
}
```

---

# 407. Anti-pattern — platform-aware node behavior

Incorrecto:

```php
$node->toSql($platform);
```

---

# 408. Correct model

```text
Node
   │
   ▼
Compiler
   │
   ├── Dialect
   ├── Platform
   └── Capabilities
```

---

# 409. Anti-pattern — ORM AST

Incorrecto:

```text
SelectNode
└── UserEntity::class
```

---

# 410. Anti-pattern — Request state

Incorrecto:

```text
SelectNode
└── Request
    └── currentTenant
```

---

# 411. Anti-pattern — raw fragments everywhere

Incorrecto:

```text
WhereNode("age > 18")
OrderNode("name DESC")
JoinNode("LEFT JOIN ...")
```

---

# 412. Correct structured form

```text
Where
└── GreaterThan
    ├── Column(age)
    └── Parameter(18)

Order
└── OrderItem
    ├── Column(name)
    └── DESC
```

---

# 413. Anti-pattern — AST as execution plan

Incorrecto:

```text
SelectNode
├── chosenReplica
├── chosenIndex
├── PDOStatement
└── retryCount
```

---

# 414. Correct separation

```text
AST
 │
 ▼
Semantic Query
 │
 ▼
Logical Plan
 │
 ▼
Physical Plan
 │
 ▼
Execution Plan
```

---

# 415. Anti-pattern — AST as Semantic Graph

No llenar el AST progresivamente con:

```text
resolvedTable
resolvedColumn
resolvedType
selectedPlatform
selectedCapability
selectedCompiler
selectedConnection
```

hasta convertirlo en un mutable state bag.

---

# 416. Correct semantic side model

```text
Canonical AST
       │
       ▼
Semantic Analysis
       │
       ├── SymbolTable
       ├── TypeMap
       ├── RequirementSet
       ├── SemanticGraph
       └── Diagnostics
```

---

# 417. Anti-pattern — platform-compatible means portable

Que MySQL y PostgreSQL puedan expresar una operación no significa que sus semánticas sean idénticas.

El AST expresa la intención.

El Capability/Semantic/Planner pipeline determina si puede preservarse.

---

# 418. Anti-pattern — parser assumptions

El Query AST de VoltStack no será necesariamente idéntico al AST que produciría un parser SQL.

---

# 419. Why

VoltStack comienza desde una API semántica, no desde texto SQL.

Por ello puede conservar información más rica que el SQL final.

---

# 420. Query AST vs SQL Parser AST

```text
VoltStack Query AST
=
framework semantic structural IR

SQL Parser AST
=
parsed representation of SQL grammar
```

Son problemas relacionados pero diferentes.

---

# 421. Future SQL parser

Si VoltStack incorpora uno:

```text
Raw SQL
   │
   ▼
SQL Parser AST
   │
   ▼
Normalization/Translation
   │
   ▼
VoltStack Query AST?
```

sólo cuando la conversión sea semánticamente segura.

---

# 422. AST invariants summary

El AST deberá cumplir:

```text
Immutable
Portable-first
Structured
Typed
Canonical
Deterministic
Traversable
Transformable
Fingerprintable
Extensible
Runtime-neutral
Driver-neutral
ORM-neutral
Connection-neutral
SQL-free
```

excepto escape hatches explícitos.

---

# 423. Core pipeline

```text
┌─────────────────────────────┐
│        Query Builder        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│         Query Model         │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Query AST Factory     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│         Initial AST         │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ Structural Normalization    │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        Canonical AST        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Semantic Analysis       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Semantic Query         │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│         Optimizer           │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│          Planner            │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│         Compiler            │
└─────────────────────────────┘
```

---

# 424. Separación final de responsabilidades

| Componente | Pregunta |
|---|---|
| Query Builder | ¿Cómo construye cómodamente la consulta el desarrollador? |
| Query Model | ¿Qué operación quiere representar? |
| Query AST | ¿Cuál es su estructura canónica? |
| Semantic Engine | ¿Qué significa realmente esa estructura? |
| Capability System | ¿Qué puede hacer el target? |
| Optimizer | ¿Qué transformaciones semánticamente seguras convienen? |
| Planner | ¿Qué estrategia debe utilizarse? |
| Compiler | ¿Cómo se expresa esa estrategia en SQL? |
| Executor | ¿Cómo se ejecuta? |
| Driver | ¿Cómo se comunica VoltStack con el cliente nativo? |

---

# 425. Regla maestra

> **El Query AST de VoltStack será una representación canónica, inmutable, tipada y portable de la estructura de una consulta. Nunca será SQL disfrazado de objetos, nunca ejecutará consultas y nunca acumulará estado de las capas posteriores.**

La regla puede resumirse como:

```text
Intent
  │
  ▼
Query Model
  │
  ▼
Structure
  │
  ▼
AST
  │
  ▼
Meaning
  │
  ▼
Semantic Engine
  │
  ▼
Strategy
  │
  ▼
Planner
  │
  ▼
Syntax
  │
  ▼
Compiler
  │
  ▼
Execution
```

---

# 426. Decisión arquitectónica final

La arquitectura oficial será:

```text
Query Builder
     │
     ▼
Query Model
     │
     ▼
Query AST
     │
     ├── immutable
     ├── canonical
     ├── structured
     ├── typed
     ├── portable-first
     ├── traversable
     ├── transformable
     ├── fingerprintable
     └── extensible
     │
     ▼
Semantic Query Engine
```

Y queda expresamente prohibida la arquitectura:

```text
Builder
  │
  ▼
SQL fragments
  │
  ▼
PDO
```

como arquitectura interna principal de VoltStack.

---

# 427. Relación con documentos siguientes

```text
24_DATABASE_QUERY_MODEL.md
        │
        ▼
25_DATABASE_QUERY_AST_SYSTEM.md
        │
        ▼
26_DATABASE_QUERY_AST_NODE_MODEL.md
        │
        ├───────────────┐
        ▼               ▼
27 Expression       28 Predicate
        │               │
        └───────┬───────┘
                ▼
29 Parameters
                │
                ▼
30 Query Types
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

# 428. Próximo documento

El siguiente documento será:

```text
26_DATABASE_QUERY_AST_NODE_MODEL.md
```

Ese documento deberá bajar un nivel adicional y definir formalmente la **taxonomía de nodes**:

```text
QueryAstNode
├── Root Nodes
├── Statement Nodes
├── Source Nodes
├── Projection Nodes
├── Expression Nodes
├── Predicate Nodes
├── Identifier Nodes
├── Parameter Nodes
├── Join Nodes
├── CTE Nodes
├── Mutation Nodes
├── Conflict Nodes
├── Returning Nodes
├── Grouping Nodes
├── Ordering Nodes
├── Pagination Nodes
├── Locking Nodes
├── Set Operation Nodes
└── Extension Nodes
```

incluyendo sus contratos, children, invariantes, ownership, igualdad estructural, metadata, node kinds y reglas exactas de composición.