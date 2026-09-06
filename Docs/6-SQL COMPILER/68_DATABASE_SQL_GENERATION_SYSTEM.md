# 68_DATABASE_SQL_GENERATION_SYSTEM.md

# VoltStack Quantum Database
## SQL Generation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 68 — SQL Generation System  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`SQL Generation System` define la capa responsable de transformar una representación SQL estructurada, validada, adaptada al dialecto y preparada para emisión en SQL textual determinista.

Su responsabilidad central es:

```text
SqlEmissionTree
+
SqlAliasMap
+
SqlPlaceholderPlan
+
SqlRenderingContext
        │
        ▼
SQL Generation System
        │
        ▼
RenderedSqlArtifact
```

Este sistema constituye la fase de rendering del Compiler Pipeline definido en:

```text
67_DATABASE_SQL_COMPILER_PIPELINE.md
```

y se apoya en las fronteras arquitectónicas establecidas por:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
```

La regla principal será:

```text
SQL Generation
=
Representation Rendering
```

y nunca:

```text
SQL Generation
=
Semantic Interpretation
+
Query Optimization
+
Physical Planning
+
String Guessing
```

---

# 2. Objetivo arquitectónico

VoltStack deberá generar SQL que sea:

- semánticamente fiel a la representación recibida;
- válido para el dialecto seleccionado;
- correctamente escapado;
- correctamente parametrizado;
- determinista;
- portable dentro de las capacidades declaradas;
- seguro frente a inyección;
- reproducible entre procesos;
- trazable hacia sus nodos de origen;
- compatible con prepared statements;
- compatible con compiled query caching;
- eficiente en memoria;
- seguro en persistent workers;
- extensible sin convertir el renderer en un sistema de callbacks arbitrarios.

---

# 3. Frontera fundamental

El sistema recibe:

```text
DialectAdapted
+
RepresentationValidated
+
AliasPlanned
+
PlaceholderPlanned
SQL representation
```

y produce:

```text
Rendered SQL
```

Por tanto:

```text
SQL Generation
≠
SQL Compilation completa
```

SQL Generation es sólo una fase de SQL Compilation.

---

# 4. Posición en el pipeline

```text
CompilableDatabaseOperation
        │
        ▼
Structural SQL Lowering
        │
        ▼
SqlEmissionTree
        │
        ▼
Dialect Adaptation
        │
        ▼
Representation Validation
        │
        ▼
Alias Planning
        │
        ▼
Placeholder Planning
        │
        ▼
┌──────────────────────────────┐
│ SQL GENERATION SYSTEM        │
└──────────────┬───────────────┘
               │
               ▼
      RenderedSqlArtifact
               │
               ▼
Binding Layout Compilation
               │
               ▼
Result Contract Compilation
               │
               ▼
CompiledDatabaseCommand
```

---

# 5. Principio maestro

El renderer no deberá preguntarse:

> ¿Qué quiso decir el desarrollador?

Eso ya fue resuelto.

Tampoco:

> ¿Cuál es la mejor forma lógica de ejecutar esto?

Eso pertenece al Optimizer.

Ni:

> ¿Debo utilizar Hash Join o Nested Loop?

Eso pertenece al Planner.

El renderer responde únicamente:

> ¿Cómo se representa textualmente esta estructura SQL ya seleccionada para este dialecto?

---

# 6. Fórmula

```text
RenderedSqlArtifact
=
Render(
    SqlEmissionTree,
    SqlAliasMap,
    SqlPlaceholderPlan,
    SqlRenderingContext
)
```

con la condición:

```text
Meaning(RenderedSql)
=
Meaning(SqlEmissionTree)
```

dentro de las reglas del dialecto objetivo.

---

# 7. Entrada principal

La entrada conceptual será:

```php
final readonly class SqlGenerationInput
{
    public function __construct(
        public SqlEmissionTree $tree,
        public SqlAliasMap $aliases,
        public SqlPlaceholderPlan $placeholders,
    ) {}
}
```

---

# 8. Salida principal

```php
final readonly class RenderedSqlArtifact
{
    public function __construct(
        public RenderedSql $sql,
        public SqlSourceMap $sourceMap,
        public SqlRenderingMetadata $metadata,
    ) {}
}
```

---

# 9. RenderedSql

No deberá utilizarse un `string` desnudo en todas las capas.

Se recomienda:

```php
final readonly class RenderedSql
{
    public function __construct(
        public string $value,
    ) {}
}
```

Esto permite distinguir:

```text
Raw user string
≠
SQL fragment
≠
Rendered SQL
≠
Debug SQL
```

---

# 10. SQL Generation ≠ Raw SQL

`RenderedSql` significa:

> SQL producido por el Compiler.

No significa:

> SQL confiable suministrado externamente.

---

# 11. Arquitectura general

```text
                  SqlGenerationInput
                          │
                          ▼
                ┌────────────────────┐
                │ SqlRenderer        │
                └─────────┬──────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        Statement      Clause      Expression
        Renderer       Renderer     Renderer
              │           │           │
              └───────────┼───────────┘
                          ▼
                    SqlTextWriter
                          │
             ┌────────────┼─────────────┐
             ▼            ▼             ▼
        Identifier    Placeholder    Literal
         Renderer       Renderer      Renderer
             │            │             │
             └────────────┼─────────────┘
                          ▼
                    SqlSourceMap
                          │
                          ▼
                RenderedSqlArtifact
```

---

# 12. Componentes principales

El sistema se divide conceptualmente en:

```text
SqlRenderer
SqlRenderingContext
SqlTextWriter
SqlStatementRenderer
SqlClauseRenderer
SqlExpressionRenderer
SqlPredicateRenderer
SqlRelationRenderer
SqlIdentifierRenderer
SqlLiteralRenderer
SqlPlaceholderRenderer
SqlOperatorRenderer
SqlFunctionRenderer
SqlTypeRenderer
SqlSourceMapBuilder
SqlFormattingPolicy
SqlParenthesisPolicy
SqlPrecedenceModel
SqlRenderingRegistry
SqlRenderingBudget
```

---

# 13. SqlRenderer

Contrato principal:

```php
interface SqlRenderer
{
    public function render(
        SqlGenerationInput $input,
        SqlRenderingContext $context,
    ): RenderedSqlArtifact;
}
```

---

# 14. SqlRenderer responsibility

Coordina:

```text
root statement rendering
writer lifecycle
source map
budget
rendering registry
diagnostics
final output
```

No deberá contener un enorme:

```php
switch ($node::class) {
    // 300 cases
}
```

---

# 15. SqlRenderingContext

```php
final readonly class SqlRenderingContext
{
    public function __construct(
        public SqlDialect $dialect,
        public PlatformCapabilitySnapshot $capabilities,
        public SqlAliasMap $aliases,
        public SqlPlaceholderPlan $placeholders,
        public SqlFormattingPolicy $formatting,
        public SqlRenderingRegistry $registry,
        public SqlRenderingBudgetTracker $budget,
        public SqlRenderingProfile $profile,
    ) {}
}
```

---

# 16. Context inmutable

`SqlRenderingContext` será un snapshot.

No deberá contener:

```text
PDO
Connection
ResultSet
EntityManager
UnitOfWork
HTTP Request
mutable TenantContext global
current transaction
```

---

# 17. Operation-local state

Estado mutable temporal deberá vivir en:

```text
SqlRenderingSession
```

o componentes equivalentes operation-scoped.

---

# 18. SqlRenderingSession

```php
final class SqlRenderingSession
{
    public function __construct(
        public readonly SqlRenderingContext $context,
        public readonly SqlTextWriter $writer,
        public readonly SqlSourceMapBuilder $sourceMap,
        public readonly SqlRenderingDiagnosticCollector $diagnostics,
        public readonly SqlRenderingBudgetTracker $budget,
    ) {}
}
```

---

# 19. Lifecycle

```text
create session
     │
     ▼
render root
     │
     ▼
validate writer state
     │
     ▼
finalize SQL
     │
     ▼
finalize source map
     │
     ▼
destroy session
```

---

# 20. SqlEmissionTree

El sistema no renderizará directamente:

```text
QueryBuilder
SemanticQueryArtifact
LogicalQueryPlan
PhysicalQueryPlan
```

Renderizará:

```text
SqlEmissionTree
```

---

# 21. Razón

El `SqlEmissionTree` representa una frontera explícita:

```text
Database semantics
        │
        ▼
SQL representation semantics
        │
        ▼
text rendering
```

---

# 22. Modelo de nodos

Ejemplo conceptual:

```text
SqlStatementNode
├── SqlSelectStatement
├── SqlInsertStatement
├── SqlUpdateStatement
├── SqlDeleteStatement
└── ExtensionSqlStatement

SqlClauseNode
├── SqlProjectionClause
├── SqlFromClause
├── SqlWhereClause
├── SqlJoinClause
├── SqlGroupByClause
├── SqlHavingClause
├── SqlWindowClause
├── SqlOrderByClause
├── SqlLimitClause
├── SqlReturningClause
└── ...

SqlExpressionNode
├── SqlColumnReference
├── SqlParameterReference
├── SqlLiteralExpression
├── SqlFunctionCall
├── SqlBinaryExpression
├── SqlUnaryExpression
├── SqlCaseExpression
├── SqlCastExpression
├── SqlSubqueryExpression
└── ...

SqlPredicateNode
├── SqlComparisonPredicate
├── SqlLogicalPredicate
├── SqlNullPredicate
├── SqlBetweenPredicate
├── SqlInPredicate
├── SqlExistsPredicate
├── SqlLikePredicate
└── ...
```

---

# 23. SQL AST vs Query AST

Deben permanecer separados conceptualmente.

```text
Query AST
=
database-independent query meaning

SqlEmissionTree
=
SQL-target representation
```

---

# 24. No second semantic engine

El `SqlEmissionTree` no deberá convertirse en un nuevo sistema semántico paralelo.

Su propósito es:

```text
representation
```

no:

```text
reinterpretation
```

---

# 25. Statement rendering

Cada statement tendrá renderer especializado.

```php
interface SqlStatementRenderer
{
    public function supports(SqlStatementNode $statement): bool;

    public function render(
        SqlStatementNode $statement,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 26. Select renderer

Ejemplo:

```text
SqlSelectStatementRenderer
```

puede emitir:

```text
WITH
SELECT
FROM
JOIN
WHERE
GROUP BY
HAVING
WINDOW
ORDER BY
LIMIT/OFFSET
LOCKING
```

en el orden definido por el dialecto.

---

# 27. Clause order

El orden de cláusulas no deberá depender del orden de inserción de objetos.

---

# 28. Example

Aunque internamente el objeto tenga:

```text
where
projection
limit
from
order
```

el renderer deberá producir el orden sintáctico correcto.

---

# 29. Clause ordering authority

Será responsabilidad de:

```text
Statement Renderer
+
Dialect Rendering Contract
```

---

# 30. Select conceptual rendering

```php
final class SelectStatementRenderer
{
    public function render(
        SqlSelectStatement $statement,
        SqlRenderingSession $session,
    ): void {
        // conceptual:
        // render CTEs
        // render SELECT
        // render projection
        // render FROM
        // render JOINs
        // render WHERE
        // render GROUP BY
        // render HAVING
        // render WINDOW
        // render ORDER BY
        // render pagination
        // render locking
    }
}
```

---

# 31. INSERT rendering

Debe representar estructuralmente:

```text
target
columns
VALUES / SELECT / DEFAULT VALUES
conflict behavior
RETURNING
```

---

# 32. UPDATE rendering

```text
target
aliases
assignments
sources
joins if supported representation requires them
predicate
ordering where supported
limit where supported
returning
```

---

# 33. DELETE rendering

```text
target
using/source clauses
joins
predicate
ordering
limit
returning
```

según la representación adaptada al dialecto.

---

# 34. No dialect guessing in statement renderer

El statement renderer no hará:

```php
if ($platformName === 'postgres') {
   ...
}
```

por toda la implementación.

---

# 35. Dialect rendering contract

Preferir:

```php
interface SqlDialectRenderingContract
{
    public function identifierRules(): IdentifierRenderingRules;

    public function literalRules(): LiteralRenderingRules;

    public function operatorRules(): OperatorRenderingRules;

    public function statementRules(): StatementRenderingRules;
}
```

---

# 36. Clause renderer

Puede existir:

```php
interface SqlClauseRenderer
{
    public function render(
        SqlClauseNode $clause,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 37. Clause rendering ≠ clause optimization

`WHERE` renderer no simplifica predicados.

`JOIN` renderer no reordena joins.

`ORDER BY` renderer no elimina claves redundantes.

---

# 38. Expression rendering

El `SqlExpressionRenderer` transforma:

```text
SqlExpressionNode
```

en representación textual.

---

# 39. Expression examples

```text
ColumnReference
ParameterReference
Literal
FunctionCall
BinaryOperation
UnaryOperation
Cast
Case
Tuple
Subquery
Aggregate
WindowExpression
RawExpression
ExtensionExpression
```

---

# 40. Expression renderer contract

```php
interface SqlExpressionRenderer
{
    public function render(
        SqlExpressionNode $expression,
        SqlExpressionRenderingContext $context,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 41. Expression context

Necesita información como:

```text
parent operator
precedence requirement
associativity context
expression position
expected syntactic category
```

---

# 42. SQL precedence

Uno de los problemas centrales será preservar correctamente precedencia.

Ejemplo semántico:

```text
(a + b) * c
```

no puede renderizarse como:

```sql
a + b * c
```

---

# 43. Precedence model

```php
interface SqlPrecedenceModel
{
    public function precedence(
        SqlExpressionNode $expression,
    ): SqlPrecedence;

    public function requiresParentheses(
        SqlExpressionNode $child,
        SqlExpressionRenderingContext $parent,
    ): bool;
}
```

---

# 44. Parenthesis rule

La regla será:

```text
Emit parentheses
iff
required to preserve structure/semantics
or explicitly required by dialect/rendering policy.
```

---

# 45. Over-parenthesization

Puede ser semánticamente seguro:

```sql
((a + b))
```

pero canonical rendering deberá evitar paréntesis innecesarios cuando exista una regla determinista segura.

---

# 46. Under-parenthesization

Está prohibido.

---

# 47. Associativity

No basta con precedencia.

Ejemplo:

```text
a - (b - c)
```

no equivale a:

```text
a - b - c
```

---

# 48. Operator descriptor

Cada operador deberá conocer:

```text
precedence
associativity
arity
rendering token/form
capabilities
```

---

# 49. SqlOperatorDescriptor

```php
final readonly class SqlOperatorDescriptor
{
    public function __construct(
        public SqlOperatorId $id,
        public SqlOperatorArity $arity,
        public SqlPrecedence $precedence,
        public SqlAssociativity $associativity,
        public SqlOperatorRenderingStrategy $rendering,
    ) {}
}
```

---

# 50. Semantic operator ≠ SQL token

Ejemplo conceptual:

```text
Semantic operator:
JSON_CONTAINS

SQL representation:
target dependent
```

Dialect Adaptation deberá haber elegido una representación soportada.

---

# 51. Predicate rendering

Debe conservar SQL 3VL.

---

# 52. Logical predicate

Ejemplo:

```text
AND(P1, OR(P2, P3))
```

debe renderizarse preservando agrupación:

```sql
P1 AND (P2 OR P3)
```

---

# 53. Predicate renderer no simplifica

No:

```text
P AND TRUE
→ P
```

Eso pertenece al Optimizer/Rewrite system.

---

# 54. Comparison rendering

Ejemplos:

```text
=
<>
<
<=
>
>=
```

pero siempre desde descriptors estructurados.

---

# 55. NULL

Nunca renderizar:

```sql
column = NULL
```

cuando el modelo semántico representa:

```text
IS NULL
```

---

# 56. Null predicate

Será first-class:

```text
SqlIsNullPredicate
SqlIsNotNullPredicate
```

---

# 57. BETWEEN

Debe conservarse si el emission tree lo representa como:

```text
SqlBetweenPredicate
```

No es obligatorio descomponerlo a:

```text
x >= a AND x <= b
```

---

# 58. IN

Representación estructurada:

```text
SqlInPredicate
├── left
└── source
    ├── expression list
    ├── subquery
    └── platform-specific structured source
```

---

# 59. Empty IN

No deberá descubrirse durante rendering mediante:

```php
if (count($values) === 0)
```

sobre runtime bindings.

La estrategia ya deberá estar representada.

---

# 60. EXISTS

```text
SqlExistsPredicate
└── SqlSubquery
```

renderiza:

```sql
EXISTS (...)
```

sin convertirlo a `COUNT`.

---

# 61. LIKE

El renderer manejará:

```text
LIKE expression
escape clause
dialect representation
```

pero no inferirá collation ni case semantics.

---

# 62. Function rendering

Las funciones se renderizarán mediante:

```text
SqlFunctionRendererRegistry
```

---

# 63. Function semantic identity

No depender del nombre PHP de una clase.

Preferir:

```text
FunctionId
```

---

# 64. Function rendering descriptor

```php
interface SqlFunctionRenderer
{
    public function functionId(): SqlFunctionId;

    public function render(
        SqlFunctionCall $function,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 65. Built-in vs platform function

Ejemplo:

```text
CURRENT_TIMESTAMP
```

podría tener forma especial y no ser:

```sql
CURRENT_TIMESTAMP()
```

en todos los targets.

---

# 66. Function renderer no hace capability fallback

Si la función no tiene representación válida:

```text
Compilation error
```

---

# 67. Aggregate rendering

Debe soportar estructuralmente:

```text
function
DISTINCT
arguments
ordering if supported
FILTER
```

---

# 68. Example

```text
AggregateExpression
├── COUNT
├── DISTINCT
└── user_id
```

→

```sql
COUNT(DISTINCT "user_id")
```

según quoting/context.

---

# 69. Ordered aggregate

Si la representación objetivo lo soporta:

```sql
STRING_AGG(name, ',' ORDER BY created_at)
```

será generado desde estructura, no desde raw string.

---

# 70. Window rendering

Ejemplo:

```text
WindowExpression
├── ROW_NUMBER
└── WindowSpecification
    ├── partition
    ├── ordering
    └── frame
```

---

# 71. Window SQL

Conceptualmente:

```sql
ROW_NUMBER() OVER (
    PARTITION BY ...
    ORDER BY ...
    ROWS BETWEEN ...
)
```

---

# 72. Window renderer

No decide effective frame.

Ese dato ya debe haber sido resuelto antes.

---

# 73. CASE rendering

Estructura:

```text
CASE
├── operand?
├── WHEN
│   ├── condition
│   └── result
├── WHEN
│   ├── condition
│   └── result
└── ELSE
```

---

# 74. CASE formatting

Canonical:

```sql
CASE WHEN ... THEN ... ELSE ... END
```

Pretty:

```sql
CASE
    WHEN ... THEN ...
    ELSE ...
END
```

---

# 75. CAST rendering

No concatenar:

```php
$expr . '::' . $type
```

como lógica universal.

---

# 76. Type renderer

```php
interface SqlTypeRenderer
{
    public function render(
        SqlPlatformType $type,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 77. Identifier rendering

Es una frontera crítica de seguridad.

---

# 78. Identifier ≠ string

Preferir:

```text
Identifier
QualifiedIdentifier
SchemaIdentifier
TableIdentifier
ColumnIdentifier
AliasIdentifier
```

---

# 79. Identifier renderer

```php
interface SqlIdentifierRenderer
{
    public function render(
        SqlIdentifier $identifier,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 80. Quoting

La estrategia dependerá del dialecto.

Ejemplos conceptuales:

```text
"identifier"
`identifier`
```

---

# 81. No manual quoting

Está prohibido:

```php
'"' . $name . '"'
```

fuera del componente responsable.

---

# 82. Quoting escaping

Si un identifier contiene el carácter de quote correspondiente, deberá escaparse según reglas del target.

---

# 83. Qualified identifiers

No renderizar:

```text
"users.id"
```

cuando la estructura representa:

```text
users.id
```

Debe producir conceptualmente:

```text
"users"."id"
```

---

# 84. Identifier components

```text
QualifiedIdentifier
├── namespace?
├── relation?
└── localName
```

cada componente se renderiza por separado.

---

# 85. Wildcard

`*` no es un identifier ordinario.

---

# 86. Structured wildcard

Preferir:

```text
SqlWildcard
SqlQualifiedWildcard
```

---

# 87. Example

```text
SqlQualifiedWildcard(users)
```

→

```sql
"users".*
```

---

# 88. Reserved words

El sistema podrá utilizar quoting determinista para evitar depender de listas parciales de palabras reservadas.

---

# 89. Quoting policy

Opciones posibles:

```text
ALWAYS
WHEN_REQUIRED
PRESERVE
```

pero canonical mode deberá utilizar una política estable.

---

# 90. Case folding

No deberá confiarse accidentalmente en case folding del motor.

La semántica del identifier debe haber sido resuelta antes.

---

# 91. Alias rendering

Alias provienen exclusivamente de:

```text
SqlAliasMap
```

---

# 92. No alias generation during rendering

Incorrecto:

```php
$alias = 't' . (++$this->counter);
```

dentro del renderer.

---

# 93. Alias reference

```text
SemanticRelationId
      │
      ▼
SqlAliasMap
      │
      ▼
SqlAlias
      │
      ▼
IdentifierRenderer
```

---

# 94. Placeholder rendering

Los placeholders provienen de:

```text
SqlPlaceholderPlan
```

---

# 95. Placeholder renderer

```php
interface SqlPlaceholderRenderer
{
    public function render(
        SqlPlaceholderId $placeholder,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 96. Placeholder forms

Ejemplos:

```text
?
$1
$2
:p1
```

---

# 97. No values

Nunca:

```sql
WHERE email = 'john@example.com'
```

a partir de un runtime parameter.

Debe ser:

```sql
WHERE email = ?
```

o equivalente.

---

# 98. Placeholder plan authority

El renderer no renumerará placeholders por su cuenta.

---

# 99. Repeated parameter

Si:

```text
P1 occurs twice
```

la forma concreta depende del plan.

---

# 100. Positional example

```sql
a = ? OR b = ?
```

Binding layout:

```text
slot 1 → P1
slot 2 → P1
```

---

# 101. Numbered example

```sql
a = $1 OR b = $1
```

si el target/driver permite reuse y el placeholder plan lo decidió.

---

# 102. Literal rendering

No todo valor SQL debe convertirse en parameter.

Existen literals estructurales.

---

# 103. Literal categories

```text
NULL
TRUE
FALSE
numeric compile-time literal
string compile-time literal where explicitly allowed
interval representation
keyword literal
special SQL constant
```

---

# 104. Runtime value ≠ compile-time literal

La mayoría de valores provenientes de aplicación deberán ser parameters.

---

# 105. Boolean literals

La representación depende del dialecto/capabilities.

Puede ser:

```sql
TRUE
FALSE
```

o representación equivalente.

---

# 106. Numeric literal

Debe utilizar formato:

```text
locale-independent
```

Nunca:

```text
1,25
```

por locale.

---

# 107. Floating point

Debe evitarse representación que:

```text
changes value
uses locale
produces NaN/Infinity unsupported by target
```

---

# 108. String literals

Deberán utilizarse únicamente para valores estructuralmente compile-time y mediante `SqlLiteralRenderer`.

---

# 109. No ad hoc escaping

Nunca:

```php
str_replace("'", "''", $value)
```

disperso por renderers.

---

# 110. Raw SQL rendering

`RawSqlExpression` constituye una excepción controlada.

---

# 111. Raw policy

Antes de llegar al renderer debe haber pasado:

```text
raw validation
trust validation
parameter validation
capability validation
security validation
```

---

# 112. Raw renderer

No deberá intentar interpretar su contenido.

---

# 113. Raw parameters

Deben seguir utilizando placeholders estructurados cuando el raw contract los declare.

---

# 114. No interpolation

Incorrecto:

```text
Raw("age > $age")
```

Correcto conceptualmente:

```text
RawTemplate
├── text segment
├── ParameterReference(P1)
└── text segment
```

---

# 115. Raw template model

Para APIs seguras puede utilizarse:

```text
RawSqlTemplate
├── StaticSqlSegment
├── SqlParameterSlot
├── StaticSqlSegment
└── SqlIdentifierSlot
```

con restricciones distintas por slot.

---

# 116. Identifier slot ≠ parameter slot

Los identifiers no se bindearán como valores.

---

# 117. Static raw segment

Debe ser tratado como código SQL, no como runtime data.

---

# 118. SqlTextWriter

Componente central de emisión.

---

# 119. Contract

```php
interface SqlTextWriter
{
    public function keyword(string $keyword): void;

    public function identifier(RenderedIdentifier $identifier): void;

    public function placeholder(RenderedPlaceholder $placeholder): void;

    public function literal(RenderedLiteral $literal): void;

    public function punctuation(string $token): void;

    public function operator(string $operator): void;

    public function whitespace(): void;

    public function appendTrustedSql(string $sql): void;

    public function length(): int;

    public function finalize(): RenderedSql;
}
```

---

# 120. Typed writer operations

El objetivo es evitar que cualquier componente pueda hacer:

```php
$writer->append($untrustedValue);
```

---

# 121. Generic append

Si existe, deberá ser internal/private o exigir un tipo que certifique contenido compilado.

---

# 122. Trusted SQL

`appendTrustedSql()` sólo podrá recibir contenido proveniente de:

```text
compiler-owned static syntax
validated raw SQL nodes
registered dialect renderer
```

---

# 123. Writer responsibility

Gestionará:

```text
buffer
length
budget
optional whitespace normalization
source offsets
```

---

# 124. Writer does not understand semantics

No sabe qué significa:

```text
JOIN
WHERE
RETURNING
```

Sólo emite tokens autorizados.

---

# 125. Token-oriented generation

Internamente puede modelarse como:

```text
SqlTokenStream
```

antes de materializar el string.

---

# 126. Token model

Ejemplo:

```text
Keyword(SELECT)
Whitespace
Identifier(users)
Punctuation(.)
Identifier(id)
Whitespace
Keyword(FROM)
Whitespace
Identifier(users)
```

---

# 127. Token stream optionality

V1 puede escribir directamente a buffer siempre que conserve:

```text
typed emission
determinism
source mapping
budget accounting
```

---

# 128. Future token stream

Un `SqlTokenStream` puede facilitar:

```text
formatters
debugging
syntax highlighting
source maps
canonicalization
```

sin cambiar contratos externos.

---

# 129. Formatting policy

El contenido SQL y su presentación deberán separarse.

---

# 130. SqlFormattingPolicy

```php
interface SqlFormattingPolicy
{
    public function whitespaceBetween(
        SqlTokenCategory $left,
        SqlTokenCategory $right,
    ): SqlWhitespaceDecision;

    public function newlineBefore(
        SqlClauseKind $clause,
    ): bool;

    public function indentation(): SqlIndentationPolicy;
}
```

---

# 131. Canonical rendering

Canonical mode deberá buscar:

```text
stable
compact
deterministic
cache-friendly
test-friendly
```

---

# 132. Canonical example

```sql
SELECT "u"."id","u"."email" FROM "users" AS "u" WHERE "u"."active"=$1 ORDER BY "u"."id" ASC LIMIT 20
```

La forma exacta se fijará mediante convención del Compiler.

---

# 133. Pretty rendering

```sql
SELECT
    "u"."id",
    "u"."email"
FROM
    "users" AS "u"
WHERE
    "u"."active" = $1
ORDER BY
    "u"."id" ASC
LIMIT 20
```

---

# 134. Canonical vs pretty semantics

Debe cumplirse:

```text
Meaning(CanonicalSql)
=
Meaning(PrettySql)
```

---

# 135. Canonical output version

La política tendrá versión:

```text
SqlCanonicalRenderingVersion
```

---

# 136. Why version it

Cambiar:

```text
spaces
parentheses
alias policy
keyword case
```

puede afectar:

```text
golden tests
compiled caches
SQL fingerprints
telemetry grouping
```

---

# 137. Keyword case

Canonical policy podría establecer:

```text
UPPERCASE SQL KEYWORDS
```

sin depender del input.

---

# 138. Identifier case

No deberá alterarse arbitrariamente durante formatting.

---

# 139. Whitespace safety

Debe impedir concatenaciones como:

```text
SELECTFROM
```

pero evitar whitespace accidental no determinista.

---

# 140. Comments

Los comentarios no formarán parte del SQL canónico por defecto.

---

# 141. Debug comments

Podrán incluir:

```text
query ID
compiler phase marker
source node marker
```

si:

```text
debug mode enabled
```

y nunca contienen información sensible.

---

# 142. Optimizer hints

Comentarios especiales de vendor no se introducirán arbitrariamente aquí.

Si existe soporte futuro:

```text
StructuredPlatformHint
→ Dialect Adaptation
→ StructuredSqlHintNode
→ Renderer
```

---

# 143. No raw optimizer hint strings

No:

```php
$query->hint('/*+ WHATEVER */');
```

como API core universal.

---

# 144. Parenthesis subsystem

Debe ser un componente formal.

---

# 145. Parenthesis context

```php
final readonly class SqlExpressionRenderingContext
{
    public function __construct(
        public ?SqlOperatorId $parentOperator,
        public SqlChildPosition $position,
        public SqlPrecedence $requiredPrecedence,
    ) {}
}
```

---

# 146. Child positions

```text
LEFT
RIGHT
OPERAND
ARGUMENT
PREDICATE
ORDER_EXPRESSION
FRAME_BOUND
```

---

# 147. Binary operator example

AST:

```text
Multiply
├── Add(a,b)
└── c
```

render:

```sql
(a + b) * c
```

---

# 148. Right associativity example

AST:

```text
Subtract
├── a
└── Subtract(b,c)
```

render:

```sql
a - (b - c)
```

---

# 149. Logical example

```text
AND
├── P1
└── OR
    ├── P2
    └── P3
```

render:

```sql
P1 AND (P2 OR P3)
```

---

# 150. Unary operators

Ejemplo:

```text
NOT
```

debe considerar precedencia.

---

# 151. NOT example

```text
NOT(OR(P1,P2))
```

→

```sql
NOT (P1 OR P2)
```

---

# 152. Function argument grouping

No deberán agregarse paréntesis que cambien syntactic category.

---

# 153. Subquery parentheses

Un subquery en expresión requiere normalmente:

```sql
(...)
```

---

# 154. Derived table

Igualmente:

```sql
FROM (...) AS alias
```

según target.

---

# 155. CTE rendering

El renderer recibe:

```text
SqlCteDefinition
```

ya ordenado/dependency-validado.

---

# 156. No dependency resolution

No ordenará CTEs según referencias por su cuenta.

---

# 157. Recursive CTE

Si el dialecto requiere:

```sql
WITH RECURSIVE
```

la adaptación debe proporcionar la representación correspondiente.

---

# 158. Set operation rendering

Debe preservar árbol y grouping.

Ejemplo:

```text
Union(
    A,
    Intersect(B,C)
)
```

puede requerir:

```sql
A UNION (B INTERSECT C)
```

dependiendo de precedencia SQL del dialecto.

---

# 159. Set operation precedence

Tendrá su propio:

```text
SqlSetOperationPrecedenceModel
```

si es necesario.

---

# 160. UNION ALL

Debe distinguirse de:

```text
UNION DISTINCT
```

aunque el dialecto permita omitir `DISTINCT`.

---

# 161. Explicit semantic representation

Canonical renderer podrá utilizar la forma definida por dialect policy.

---

# 162. ORDER BY

Cada item contiene:

```text
expression
direction
null ordering
collation if represented
```

---

# 163. NULLS FIRST/LAST

Sólo se renderizará si:

```text
emission representation
+
dialect
```

lo permiten.

No se inventará emulación en rendering.

---

# 164. Pagination

El renderer no decide entre:

```text
LIMIT/OFFSET
FETCH FIRST
TOP
ROW_NUMBER emulation
```

Esa decisión debe haber ocurrido antes.

---

# 165. Locking

Igualmente:

```text
FOR UPDATE
FOR SHARE
NOWAIT
SKIP LOCKED
```

se renderizan desde estructura adaptada.

---

# 166. RETURNING

Se renderiza cuando el target representation lo contiene.

---

# 167. Upsert

No habrá un renderer universal basado en concatenar:

```text
ON CONFLICT
ON DUPLICATE KEY
```

La adaptación específica habrá creado nodos apropiados.

---

# 168. JSON expressions

La representación será target-aware antes del rendering.

---

# 169. Example

Semantic:

```text
JsonExtract(document, path)
```

podría haber sido adaptado a:

```text
PostgresJsonOperatorExpression
```

o:

```text
MysqlJsonExtractFunctionExpression
```

El renderer sólo emite la representación seleccionada.

---

# 170. Extension rendering

Las extensiones utilizarán registries typed.

---

# 171. SqlRenderingRegistry

```php
interface SqlRenderingRegistry
{
    public function statementRenderer(
        SqlStatementNode $node,
    ): SqlStatementRenderer;

    public function expressionRenderer(
        SqlExpressionNode $node,
    ): SqlExpressionRenderer;

    public function predicateRenderer(
        SqlPredicateNode $node,
    ): SqlPredicateRenderer;
}
```

---

# 172. Registry lifecycle

```text
discover
→ validate
→ resolve conflicts
→ freeze
```

---

# 173. Unknown node

Resultado:

```text
UnsupportedSqlEmissionNodeException
```

---

# 174. No raw fallback

Nunca:

```text
unknown node
→ cast to string
```

---

# 175. Renderer identity

Cada renderer deberá tener:

```text
RendererId
RendererVersion
SupportedNodeKinds
```

---

# 176. Conflict detection

Dos renderers reclamando el mismo node kind sin resolución explícita:

```text
AmbiguousSqlRendererException
```

---

# 177. No last-write-wins

Registro:

```text
RendererA
RendererB
```

no significa que B sustituya silenciosamente a A.

---

# 178. Source mapping

`SqlSourceMap` conecta posiciones de SQL con estructuras de compilación.

---

# 179. Example

SQL:

```sql
SELECT "u"."id" FROM "users" AS "u" WHERE "u"."active"=$1
```

Map:

```text
0..20   → SelectStatement S1
7..15   → ColumnExpression E4
21..40  → Relation R2
41..63  → Predicate P7
60..62  → ParameterOccurrence PO1
```

---

# 180. SourceSpan

```php
final readonly class SqlSourceSpan
{
    public function __construct(
        public int $startOffset,
        public int $endOffset,
    ) {}
}
```

---

# 181. SourceMapEntry

```php
final readonly class SqlSourceMapEntry
{
    public function __construct(
        public SqlSourceSpan $sqlSpan,
        public CompilationSourceReference $source,
    ) {}
}
```

---

# 182. Source references

Podrán apuntar a:

```text
QueryNodeId
SemanticNodeId
PlanNodeId
ExecutionNodeId
SqlEmissionNodeId
ParameterOccurrenceId
SecurityPolicyId
ExtensionNodeId
```

según provenance disponible.

---

# 183. Source map hierarchy

Un mismo span puede tener múltiples niveles:

```text
SQL span
├── emission node
├── execution-plan node
├── semantic expression
└── original query node
```

---

# 184. Source map ≠ SQL parser

No reconstruye estructura desde texto.

Se genera durante emisión.

---

# 185. Uses

Servirá para:

```text
database error mapping
debug toolbar
query explain
developer diagnostics
test failures
telemetry
security audit
```

---

# 186. Error mapping example

DB devuelve:

```text
syntax error near character 58
```

VoltStack puede buscar:

```text
offset 58
→ Predicate P7
→ original query expression
```

---

# 187. Driver position differences

Algunos motores reportan:

```text
byte offset
character offset
line/column
token
```

La integración deberá normalizar cuando sea posible.

---

# 188. UTF-8 offsets

Debe definirse explícitamente si los spans usan:

```text
bytes
Unicode code points
```

---

# 189. Recommended V1

Utilizar:

```text
UTF-8 byte offsets
```

porque corresponden naturalmente al buffer generado.

---

# 190. Line map

Puede derivarse opcionalmente para Pretty SQL.

---

# 191. Source map levels

```php
enum SqlSourceMapLevel
{
    case NONE;
    case STATEMENT;
    case CLAUSE;
    case NODE;
    case DETAILED;
}
```

---

# 192. Production

Default posible:

```text
CLAUSE
```

o `NODE` si benchmarks muestran overhead aceptable.

---

# 193. Debug

```text
DETAILED
```

---

# 194. Budget

Source map estará sujeto a:

```text
maxSourceMapEntries
```

---

# 195. Budget degradation

Si source-map detail es opcional, puede degradarse:

```text
DETAILED
→ NODE
→ CLAUSE
```

sin alterar SQL.

---

# 196. Important distinction

Esto es diferente de truncar SQL.

---

# 197. Correctness budget vs diagnostic budget

Separar:

```text
CompilationCorrectnessBudget
DiagnosticEnrichmentBudget
```

---

# 198. Correctness exhaustion

```text
fail
```

---

# 199. Diagnostic exhaustion

Puede reducir metadata opcional.

---

# 200. Rendering budget

```php
final readonly class SqlRenderingBudget
{
    public function __construct(
        public int $maxVisitedNodes,
        public int $maxNestingDepth,
        public int $maxRenderedBytes,
        public int $maxTokens,
        public int $maxSourceMapEntries,
        public int $maxExtensionInvocations,
        public int $maxRawSqlBytes,
    ) {}
}
```

---

# 201. Complexity protection

Previene estructuras maliciosas como:

```text
deeply nested AND
deep CASE
huge VALUES list
massive generated projection
recursive extension expansion
```

---

# 202. Depth

Traversal no deberá depender ilimitadamente de PHP recursion.

---

# 203. Iterative traversal

Para árboles profundos, puede utilizarse traversal iterativo.

---

# 204. Stack protection

Se deberá considerar explícitamente para:

```text
10,000 nested expressions
```

aunque normalmente query validation ya impondrá límites.

---

# 205. Multiple defense layers

```text
Builder budget
Semantic budget
Optimizer budget
Planner budget
Compiler budget
Rendering budget
```

no son redundantes: protegen distintas fronteras.

---

# 206. Determinism

Mismo:

```text
SqlEmissionTree
SqlAliasMap
SqlPlaceholderPlan
Dialect
RenderingPolicy
RendererVersions
```

deberá producir:

```text
same canonical SQL
```

---

# 207. Formula

```text
Render(X,C) = Render(X,C)
```

para context snapshots equivalentes.

---

# 208. No wall clock

Rendering no dependerá de:

```text
current time
```

---

# 209. No random IDs

No utilizar:

```text
uniqid()
random_bytes()
spl_object_id()
```

para aliases/placeholders/output.

---

# 210. No locale dependence

Rendering no dependerá de:

```text
setlocale()
```

---

# 211. No timezone dependence

Excepto si un literal compile-time explícito incluye representación timezone ya definida.

---

# 212. No process identity

No depender de:

```text
PID
worker ID
memory address
```

---

# 213. Canonical traversal

Maps/sets sin orden semántico deberán ordenarse mediante criterios estables cuando su rendering necesite orden.

---

# 214. But do not sort semantic sequences

No ordenar:

```text
SELECT columns
ORDER BY items
UNION operands
function arguments
VALUES columns
```

cuando el orden sea observable.

---

# 215. Deterministic ≠ sorted everything

Regla:

```text
Preserve semantic order
+
canonicalize only unordered representation
```

---

# 216. Security model

El SQL Generation System es una frontera de seguridad crítica.

---

# 217. Injection prevention

La separación será:

```text
SQL structure
→ compiler

runtime data
→ placeholders

identifiers
→ typed identifier renderer
```

---

# 218. Unsafe concatenation

Queda prohibida en renderers core.

---

# 219. Security rule

```text
Untrusted runtime string
must never reach SqlTextWriter
as executable SQL syntax.
```

---

# 220. Identifier injection

Identifiers dinámicos deberán pasar por:

```text
structured identifier validation
+
identifier renderer
```

---

# 221. Parameter injection

Runtime data deberá pasar por:

```text
ParameterId
→ Placeholder
→ BindingLayout
```

---

# 222. Raw escape hatch

Será explícito, auditable y restringido.

---

# 223. Security provenance

Si un predicate proviene de:

```text
TenantPolicy
AuthorizationPolicy
SecurityPolicy
```

el source map podrá conservar dicha provenance.

---

# 224. SQL comments and secrets

Nunca incluir runtime values sensibles en comentarios de debug.

---

# 225. Logging

El renderer produce SQL parametrizado.

Por tanto logs deberían mostrar:

```sql
WHERE email = $1
```

y no necesariamente el valor.

---

# 226. Binding telemetry

Valores se manejarán por política separada de redaction.

---

# 227. Raw logging

Raw SQL puede contener información sensible por diseño del usuario.

Deberá marcarse:

```text
RawSqlSensitivity
```

cuando corresponda.

---

# 228. SQL Generation and capabilities

Rendering sólo puede emitir features declaradas disponibles.

---

# 229. Capability example

```text
supportsReturning()
supportsWindowFunctions()
supportsFilterClause()
supportsNullOrderingSyntax()
supportsRecursiveCte()
```

---

# 230. Capability check timing

Idealmente:

```text
validation/adaptation
```

antes del renderer.

El renderer puede mantener assertions defensivas.

---

# 231. Defensive assertion

Si llega:

```text
SqlReturningClause
```

pero target contract dice:

```text
supportsReturning = false
```

deberá fallar.

---

# 232. No silent omission

Nunca:

```text
RETURNING unsupported
→ just don't render it
```

---

# 233. Platform compiler separation

El core generation system será común.

Los documentos siguientes definirán:

```text
69 MySQL
70 MariaDB
71 PostgreSQL
72 SQLite
```

---

# 234. Common vs platform-specific

```text
Common SQL Generation
├── traversal
├── writer
├── source maps
├── formatting
├── identifiers abstraction
├── placeholders abstraction
├── precedence
├── diagnostics
└── budget

Platform Compiler
├── concrete syntax
├── quoting
├── function forms
├── operators
├── pagination syntax
├── locking syntax
├── upsert syntax
└── RETURNING behavior
```

---

# 235. Platform specialization

No duplicar todo el renderer por plataforma.

---

# 236. Incorrect architecture

```text
MysqlSelectRenderer
PostgresSelectRenderer
SqliteSelectRenderer
MariaDbSelectRenderer
```

cada uno con 2,000 líneas casi iguales.

---

# 237. Preferred architecture

```text
Common Select Renderer
        │
        ▼
Dialect Rendering Contract
        │
 ┌──────┼────────┬────────┐
 ▼      ▼        ▼        ▼
MySQL MariaDB PostgreSQL SQLite
```

con specialized renderers sólo donde realmente diverja la estructura.

---

# 238. Dialect strategy granularity

Puede existir:

```text
PaginationRenderer
UpsertRenderer
ReturningRenderer
LockingRenderer
JsonRenderer
```

por target.

---

# 239. Avoid dialect God Object

No convertir:

```text
PostgresDialect
```

en una clase con cientos de `if`.

---

# 240. Rendering capability providers

Preferir pequeños contracts especializados.

---

# 241. Example

```php
interface PaginationSyntaxRenderer
{
    public function render(
        SqlPaginationClause $pagination,
        SqlRenderingSession $session,
    ): void;
}
```

---

# 242. Extension SQL nodes

Una extensión podrá introducir:

```text
ExtensionSqlExpression
ExtensionSqlPredicate
ExtensionSqlClause
```

si está registrada.

---

# 243. Extension contract

Debe declarar:

```text
ExtensionId
Version
SupportedNodeKinds
SupportedDialects/Capabilities
Renderer
Fingerprint contribution
Security classification
```

---

# 244. Extension renderer restrictions

No podrá:

```text
open connection
inspect database
access EntityManager
access current Request
mutate registry
read runtime bindings
```

---

# 245. Extension output

Deberá pasar por:

```text
SqlTextWriter
```

y sus budgets.

---

# 246. Extension raw output

Si necesita syntax arbitraria deberá usar un API explícito de trusted compiler syntax.

---

# 247. Extension conflicts

Serán detectados durante bootstrap.

---

# 248. Persistent runtime safety

Shared:

```text
SqlRenderingRegistry
SqlDialectRenderingContract
Renderer descriptors
Formatting policies
Operator descriptors
Function descriptors
```

deberán ser:

```text
immutable/frozen
```

---

# 249. Per-operation

```text
SqlRenderingSession
SqlTextWriter
SqlSourceMapBuilder
DiagnosticCollector
BudgetTracker
```

serán operation-scoped.

---

# 250. FrankenPHP model

```text
Worker
├── Frozen Compiler Registry
├── Frozen Renderer Registry
├── Frozen Dialect Contracts
│
├── Request A
│   └── RenderingSession A
│
├── Request B
│   └── RenderingSession B
│
└── Job C
    └── RenderingSession C
```

---

# 251. State reset

No deberá requerirse limpiar cientos de propiedades de renderers compartidos porque éstos no deberían contener estado mutable por query.

---

# 252. Concurrency

Dos compilaciones podrán renderizar simultáneamente si sus sessions son independientes.

---

# 253. Error model

Root:

```text
SqlGenerationException
```

---

# 254. Specific exceptions

```text
UnsupportedSqlEmissionNodeException
AmbiguousSqlRendererException
SqlIdentifierRenderingException
SqlPlaceholderRenderingException
SqlLiteralRenderingException
SqlOperatorRenderingException
SqlFunctionRenderingException
SqlPrecedenceException
SqlSourceMapException
SqlRenderingBudgetExceededException
SqlRenderingInvariantException
```

---

# 255. Diagnostics

Cada error deberá intentar incluir:

```text
CompilationId
EmissionNodeId
NodeKind
DialectId
RendererId
SourceReference
Capability requirement
```

sin exponer secretos.

---

# 256. Example diagnostic

```text
SQL generation failed.

Dialect:
PostgreSQL

Node:
E42

Kind:
ExtensionExpression<vector_distance>

Renderer:
not registered

Source:
QueryExpression Q17

Required extension:
voltstack/vector

Compilation phase:
SQL_GENERATION
```

---

# 257. No SQL string-only diagnostics

Evitar:

```text
syntax generation failed near blah
```

sin referencia estructural.

---

# 258. Invariant validation before render

El renderer podrá asumir que:

```text
all required aliases exist
all parameter occurrences have placeholder assignments
all emission node kinds are supported
```

pero deberá mantener assertions defensivas.

---

# 259. Invariant validation after render

Verificar:

```text
writer balanced
source spans closed
SQL not empty
budget respected
all expected placeholders emitted
all required nodes rendered
```

---

# 260. Balanced structures

Si el writer usa scopes:

```text
openParenthesis()
closeParenthesis()
```

deberá validar balance.

---

# 261. Better approach

Sin embargo, paréntesis deberían surgir de rendering estructurado, no de un stack textual usado para arreglar errores.

---

# 262. Placeholder emission ledger

Puede existir:

```text
PlaceholderEmissionLedger
```

---

# 263. Purpose

Registrar:

```text
planned placeholder occurrences
vs
rendered placeholder occurrences
```

---

# 264. Invariant

```text
PlannedOccurrences
=
RenderedOccurrences
```

salvo placeholders explícitamente no-runtime según modelo.

---

# 265. Alias emission ledger

Opcionalmente:

```text
AliasEmissionLedger
```

para validar references.

---

# 266. Source map ledger

Puede comprobar que:

```text
required source-mapped security nodes
```

fueron renderizados.

---

# 267. Testing strategy

El sistema requiere testing intensivo.

---

# 268. Unit tests

Para:

```text
identifier quoting
literal rendering
placeholder rendering
operator precedence
parentheses
function rendering
formatting
source maps
budget
```

---

# 269. Golden SQL tests

Entrada estructurada fija:

```text
SqlEmissionTree
```

deberá producir SQL exacto.

---

# 270. Example

```text
Input:
Multiply(Add(a,b),c)

Expected:
(a + b) * c
```

---

# 271. Negative golden tests

```text
a + b * c
```

deberá rechazarse como expected output para esa estructura.

---

# 272. Cross-platform golden tests

La misma operación puede generar:

```text
MySQL expected SQL
MariaDB expected SQL
PostgreSQL expected SQL
SQLite expected SQL
```

---

# 273. Semantic-equivalence integration tests

Cuando sea posible:

```text
same semantic query
→ compile on real target
→ execute fixture
→ compare expected result
```

---

# 274. Injection tests

Inputs maliciosos:

```text
identifier names
raw fragments
runtime values
string literals
```

deberán comprobar que no escapan sus fronteras.

---

# 275. Property tests

Ejemplos:

```text
rendering deterministic
parenthesis preserves AST structure
identifier quote/unquote roundtrip where defined
all planned placeholders emitted
```

---

# 276. Fuzzing

Especialmente útil para:

```text
nested expressions
operator combinations
CASE
set operations
subqueries
identifiers
raw templates
```

---

# 277. Differential tests

Puede compararse contra parsers/motores SQL en test environment.

---

# 278. Parser roundtrip tests

Para subset soportado:

```text
EmissionTree
→ Rendered SQL
→ external/test parser
→ syntax valid
```

No implica que parser externo sea runtime dependency.

---

# 279. Source map tests

Dado SQL:

```text
offset X
```

deberá resolverse al node correcto.

---

# 280. UTF-8 tests

Identifiers como:

```text
"café"
"名"
"über"
```

deberán validar offsets/quoting según capacidades.

---

# 281. Persistent runtime tests

Ejecutar:

```text
Query A
Query B
Query A
Query C
```

en el mismo worker y verificar:

```text
Query A canonical SQL identical
no aliases leaked
no placeholder counters leaked
no source-map entries leaked
```

---

# 282. Concurrency tests

Múltiples sessions sobre los mismos registries frozen.

---

# 283. Budget tests

Generar árboles deliberadamente grandes y comprobar fallo controlado.

---

# 284. Extension tests

Cada renderer extension deberá tener:

```text
registration test
conflict test
capability test
budget test
determinism test
security test
cross-worker isolation test
```

---

# 285. Performance model

Rendering deberá aproximarse a:

```text
O(number of emitted nodes + output size)
```

para el camino normal.

---

# 286. Avoid repeated concatenation

No construir grandes strings mediante:

```php
$sql .= ...
```

si provoca comportamiento cuadrático bajo ciertas condiciones.

---

# 287. Buffer strategy

Puede utilizar:

```text
array of chunks
→ implode
```

o un writer optimizado.

---

# 288. Benchmark required

La implementación concreta deberá decidirse mediante benchmarks PHP reales.

---

# 289. Avoid premature micro-optimization

La arquitectura no deberá comprometer claridad por asumir que:

```text
array chunks
```

siempre es superior.

---

# 290. Rendering cache

No se introducirá un cache interno oculto por node.

---

# 291. Why

Puede crear:

```text
cross-query state
memory retention
incorrect context reuse
```

---

# 292. Shared immutable fragment cache

Si en futuro se justifica, deberá ser:

```text
explicit
fingerprinted
bounded
context-aware
```

---

# 293. Compiled query cache

La reutilización principal corresponde al documento:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 294. SQL source fingerprint

Opcionalmente:

```text
RenderedSqlFingerprint
```

puede calcularse para:

```text
telemetry grouping
debugging
driver prepared-statement caches
```

---

# 295. RenderedSqlFingerprint ≠ CompiledQueryFingerprint

Porque el segundo incluye más contexto.

---

# 296. Example

```text
same SQL text
+
different BindingLayout
```

puede requerir distintos compiled artifacts.

---

# 297. Result alias mapping

Cuando el Compiler introduce aliases internos:

```sql
SELECT
    "users"."id" AS "__vs_c1",
    "orders"."id" AS "__vs_c2"
```

el result contract conservará:

```text
__vs_c1 → semantic users.id output
__vs_c2 → semantic orders.id output
```

---

# 298. Internal alias namespace

VoltStack deberá reservar un namespace/pattern de aliases internos que minimice colisiones.

---

# 299. Example

```text
__vs_r1
__vs_c1
__vs_e1
```

---

# 300. Reserved alias collision

Si el usuario utiliza el mismo nombre:

```text
alias planner
```

deberá resolverlo antes del renderer.

---

# 301. Renderer must not rename

El renderer sólo utiliza el alias final.

---

# 302. SQL Generation and semantic provenance

Aunque el renderer no entienda toda la semántica, puede transportar provenance.

---

# 303. Example

```text
SqlPredicateNode P12
origin:
SecurityPolicy SP4
```

Al renderizar:

```sql
"tenant_id" = $3
```

source map conserva:

```text
SQL span
→ P12
→ SP4
```

---

# 304. Explainability

Esto permite que herramientas futuras digan:

```text
Predicate:
tenant_id = ?

Origin:
Multitenancy policy

Introduced by:
Tenant query policy

Compiled as:
"u"."tenant_id" = $3
```

---

# 305. Query debugging

La toolbar podrá mostrar:

```text
Semantic Query
Logical Plan
Physical Plan
Execution Plan
Generated SQL
Bindings metadata
Source mapping
```

sin mezclar responsabilidades.

---

# 306. Directory architecture

```text
VoltStack/Quantum/Database/Query/Compiler/Generation
├── Contract
│   ├── SqlRenderer.php
│   ├── SqlStatementRenderer.php
│   ├── SqlClauseRenderer.php
│   ├── SqlExpressionRenderer.php
│   ├── SqlPredicateRenderer.php
│   ├── SqlIdentifierRenderer.php
│   ├── SqlLiteralRenderer.php
│   ├── SqlPlaceholderRenderer.php
│   ├── SqlFunctionRenderer.php
│   ├── SqlOperatorRenderer.php
│   └── SqlTypeRenderer.php
│
├── Context
│   ├── SqlRenderingContext.php
│   ├── SqlRenderingSession.php
│   ├── SqlExpressionRenderingContext.php
│   └── SqlRenderingProfile.php
│
├── Model
│   ├── RenderedSql.php
│   ├── RenderedIdentifier.php
│   ├── RenderedPlaceholder.php
│   ├── RenderedLiteral.php
│   └── RenderedSqlArtifact.php
│
├── Writer
│   ├── SqlTextWriter.php
│   ├── BufferedSqlTextWriter.php
│   ├── SqlToken.php
│   ├── SqlTokenCategory.php
│   └── SqlTokenStream.php
│
├── Statement
│   ├── SelectStatementRenderer.php
│   ├── InsertStatementRenderer.php
│   ├── UpdateStatementRenderer.php
│   └── DeleteStatementRenderer.php
│
├── Clause
│   ├── ProjectionClauseRenderer.php
│   ├── FromClauseRenderer.php
│   ├── JoinClauseRenderer.php
│   ├── WhereClauseRenderer.php
│   ├── GroupByClauseRenderer.php
│   ├── HavingClauseRenderer.php
│   ├── WindowClauseRenderer.php
│   ├── OrderByClauseRenderer.php
│   ├── PaginationClauseRenderer.php
│   ├── LockingClauseRenderer.php
│   └── ReturningClauseRenderer.php
│
├── Expression
│   ├── ColumnReferenceRenderer.php
│   ├── ParameterReferenceRenderer.php
│   ├── LiteralExpressionRenderer.php
│   ├── FunctionCallRenderer.php
│   ├── BinaryExpressionRenderer.php
│   ├── UnaryExpressionRenderer.php
│   ├── CaseExpressionRenderer.php
│   ├── CastExpressionRenderer.php
│   ├── SubqueryExpressionRenderer.php
│   ├── AggregateExpressionRenderer.php
│   └── WindowExpressionRenderer.php
│
├── Predicate
│   ├── ComparisonPredicateRenderer.php
│   ├── LogicalPredicateRenderer.php
│   ├── NullPredicateRenderer.php
│   ├── BetweenPredicateRenderer.php
│   ├── InPredicateRenderer.php
│   ├── ExistsPredicateRenderer.php
│   └── LikePredicateRenderer.php
│
├── Identifier
│   ├── IdentifierRenderingRules.php
│   ├── QualifiedIdentifierRenderer.php
│   └── IdentifierQuoteEscaper.php
│
├── Literal
│   ├── SqlLiteralRenderingRules.php
│   ├── BooleanLiteralRenderer.php
│   ├── NumericLiteralRenderer.php
│   ├── StringLiteralRenderer.php
│   └── NullLiteralRenderer.php
│
├── Placeholder
│   └── PlannedPlaceholderRenderer.php
│
├── Operator
│   ├── SqlOperatorDescriptor.php
│   ├── SqlOperatorRegistry.php
│   ├── SqlPrecedence.php
│   ├── SqlAssociativity.php
│   └── SqlPrecedenceModel.php
│
├── Function
│   ├── SqlFunctionRendererRegistry.php
│   └── BuiltIn
│
├── Formatting
│   ├── SqlFormattingPolicy.php
│   ├── CanonicalSqlFormattingPolicy.php
│   ├── PrettySqlFormattingPolicy.php
│   ├── SqlIndentationPolicy.php
│   └── SqlCanonicalRenderingVersion.php
│
├── SourceMap
│   ├── SqlSourceMap.php
│   ├── SqlSourceMapBuilder.php
│   ├── SqlSourceMapEntry.php
│   ├── SqlSourceSpan.php
│   ├── SqlSourceMapLevel.php
│   └── CompilationSourceReference.php
│
├── Registry
│   ├── SqlRenderingRegistry.php
│   ├── FrozenSqlRenderingRegistry.php
│   └── SqlRendererDescriptor.php
│
├── Budget
│   ├── SqlRenderingBudget.php
│   └── SqlRenderingBudgetTracker.php
│
├── Diagnostic
│   ├── SqlRenderingDiagnostic.php
│   └── SqlRenderingDiagnosticCollector.php
│
├── Raw
│   ├── RawSqlRenderer.php
│   ├── RawSqlTemplateRenderer.php
│   └── RawSqlRenderingPolicy.php
│
└── Exception
    ├── SqlGenerationException.php
    ├── UnsupportedSqlEmissionNodeException.php
    ├── AmbiguousSqlRendererException.php
    ├── SqlIdentifierRenderingException.php
    ├── SqlPlaceholderRenderingException.php
    ├── SqlLiteralRenderingException.php
    ├── SqlOperatorRenderingException.php
    ├── SqlFunctionRenderingException.php
    ├── SqlPrecedenceException.php
    ├── SqlSourceMapException.php
    ├── SqlRenderingBudgetExceededException.php
    └── SqlRenderingInvariantException.php
```

---

# 307. Invariantes arquitectónicos

## DB-SQLGEN-001

SQL Generation recibirá una representación SQL estructurada.

## DB-SQLGEN-002

SQL Generation no recibirá directamente Query Builder.

## DB-SQLGEN-003

SQL Generation no recibirá directamente ORM entities.

## DB-SQLGEN-004

SQL Generation no realizará semantic analysis.

## DB-SQLGEN-005

SQL Generation no realizará query optimization.

## DB-SQLGEN-006

SQL Generation no realizará logical planning.

## DB-SQLGEN-007

SQL Generation no realizará physical planning.

## DB-SQLGEN-008

SQL Generation no ejecutará queries.

## DB-SQLGEN-009

SQL Generation no abrirá conexiones.

## DB-SQLGEN-010

SQL Generation no inspeccionará schema mediante I/O.

## DB-SQLGEN-011

`SqlEmissionTree` será distinto de Query AST.

## DB-SQLGEN-012

`SqlEmissionTree` no será un segundo semantic engine.

## DB-SQLGEN-013

Rendering preservará significado.

## DB-SQLGEN-014

Rendering será target-specific.

## DB-SQLGEN-015

Rendering será determinista.

## DB-SQLGEN-016

Statement rendering será estructurado.

## DB-SQLGEN-017

Clause ordering será explícito.

## DB-SQLGEN-018

Clause ordering no dependerá de insertion order accidental.

## DB-SQLGEN-019

Expression rendering respetará precedencia.

## DB-SQLGEN-020

Expression rendering respetará asociatividad.

## DB-SQLGEN-021

Under-parenthesization estará prohibido.

## DB-SQLGEN-022

Canonical rendering evitará paréntesis innecesarios cuando sea seguro.

## DB-SQLGEN-023

Predicate rendering preservará SQL 3VL.

## DB-SQLGEN-024

Predicate rendering no simplificará predicates.

## DB-SQLGEN-025

NULL predicates serán estructurados.

## DB-SQLGEN-026

`IS NULL` no será inferido desde `= NULL`.

## DB-SQLGEN-027

BETWEEN podrá conservarse first-class.

## DB-SQLGEN-028

IN source será estructurado.

## DB-SQLGEN-029

Empty IN no se resolverá inspeccionando runtime values.

## DB-SQLGEN-030

EXISTS no se reescribirá como COUNT durante rendering.

## DB-SQLGEN-031

Function rendering utilizará semantic/rendering IDs.

## DB-SQLGEN-032

Function rendering no dependerá de FQCN accidental.

## DB-SQLGEN-033

Unsupported function representation producirá error.

## DB-SQLGEN-034

Aggregate DISTINCT será preservado.

## DB-SQLGEN-035

Aggregate FILTER será preservado cuando esté representado.

## DB-SQLGEN-036

Window effective frame ya estará resuelto antes del renderer.

## DB-SQLGEN-037

Window rendering no reinterpretará window semantics.

## DB-SQLGEN-038

CASE structure será preservada.

## DB-SQLGEN-039

CAST será target-aware.

## DB-SQLGEN-040

Identifiers serán typed.

## DB-SQLGEN-041

Identifiers no serán strings arbitrarios.

## DB-SQLGEN-042

Identifier quoting estará centralizado.

## DB-SQLGEN-043

Qualified identifiers renderizarán componentes individualmente.

## DB-SQLGEN-044

Wildcard será distinto de ordinary identifier.

## DB-SQLGEN-045

Alias generation ocurrirá antes del rendering.

## DB-SQLGEN-046

Renderer no inventará aliases.

## DB-SQLGEN-047

Aliases serán obtenidos desde `SqlAliasMap`.

## DB-SQLGEN-048

Placeholder generation ocurrirá antes del rendering.

## DB-SQLGEN-049

Renderer no renumerará placeholders arbitrariamente.

## DB-SQLGEN-050

Placeholders provendrán de `SqlPlaceholderPlan`.

## DB-SQLGEN-051

Runtime values no serán interpolados.

## DB-SQLGEN-052

Repeated parameters respetarán placeholder plan.

## DB-SQLGEN-053

Compile-time literals serán distintos de runtime parameters.

## DB-SQLGEN-054

Numeric literal rendering será locale-independent.

## DB-SQLGEN-055

String escaping estará centralizado.

## DB-SQLGEN-056

Raw SQL será explícito.

## DB-SQLGEN-057

Raw SQL no bypassará security validation.

## DB-SQLGEN-058

Raw SQL no bypassará parameterization rules.

## DB-SQLGEN-059

Raw SQL no será interpretado por el renderer.

## DB-SQLGEN-060

Raw identifiers serán distintos de raw value slots.

## DB-SQLGEN-061

`SqlTextWriter` controlará output.

## DB-SQLGEN-062

Untrusted values no llegarán al writer como syntax.

## DB-SQLGEN-063

Writer accounting incluirá output size.

## DB-SQLGEN-064

Writer no interpretará query semantics.

## DB-SQLGEN-065

Token-oriented emission será compatible con la arquitectura.

## DB-SQLGEN-066

Token stream no será obligatorio para V1.

## DB-SQLGEN-067

Formatting estará separado de semantics.

## DB-SQLGEN-068

Canonical formatting será estable.

## DB-SQLGEN-069

Pretty formatting no cambiará semantics.

## DB-SQLGEN-070

Canonical rendering tendrá versión.

## DB-SQLGEN-071

Keyword casing será determinista.

## DB-SQLGEN-072

Comments no formarán parte del canonical SQL por defecto.

## DB-SQLGEN-073

Debug comments no contendrán secretos.

## DB-SQLGEN-074

Vendor hints serán estructurados si se soportan.

## DB-SQLGEN-075

No habrá raw vendor hint strings como core abstraction.

## DB-SQLGEN-076

Parenthesis policy será formal.

## DB-SQLGEN-077

Parenthesis decisions utilizarán precedence/associativity.

## DB-SQLGEN-078

Subqueries serán parenthesized estructuralmente.

## DB-SQLGEN-079

Set operation grouping será preservado.

## DB-SQLGEN-080

Set operation multiplicity semantics será preservada.

## DB-SQLGEN-081

ORDER BY item order será preservado.

## DB-SQLGEN-082

NULL ordering no será inventado por renderer.

## DB-SQLGEN-083

Pagination strategy no será elegida por renderer.

## DB-SQLGEN-084

Locking strategy no será elegida por renderer.

## DB-SQLGEN-085

RETURNING unsupported no será omitido silenciosamente.

## DB-SQLGEN-086

Upsert strategy no será elegida por renderer.

## DB-SQLGEN-087

JSON representation deberá haber sido adaptada previamente.

## DB-SQLGEN-088

Unknown emission nodes producirán error.

## DB-SQLGEN-089

Unknown emission nodes no caerán a `__toString()`.

## DB-SQLGEN-090

Rendering registry será typed.

## DB-SQLGEN-091

Rendering registry será frozen después de bootstrap.

## DB-SQLGEN-092

Renderer conflicts serán detectados.

## DB-SQLGEN-093

No habrá last-write-wins silencioso.

## DB-SQLGEN-094

Renderer descriptors serán versionables.

## DB-SQLGEN-095

Source map se generará durante rendering.

## DB-SQLGEN-096

Source map no dependerá de reparsing SQL.

## DB-SQLGEN-097

Source spans utilizarán convención de offsets explícita.

## DB-SQLGEN-098

V1 utilizará UTF-8 byte offsets salvo decisión posterior documentada.

## DB-SQLGEN-099

Source-map granularity será configurable.

## DB-SQLGEN-100

Source-map diagnostics podrán degradarse sin cambiar SQL.

## DB-SQLGEN-101

Correctness budget será distinto de diagnostic budget.

## DB-SQLGEN-102

Correctness budget exhaustion fallará rendering.

## DB-SQLGEN-103

Diagnostic budget exhaustion podrá reducir metadata opcional.

## DB-SQLGEN-104

Rendering tendrá límites de nodos.

## DB-SQLGEN-105

Rendering tendrá límite de profundidad.

## DB-SQLGEN-106

Rendering tendrá límite de bytes.

## DB-SQLGEN-107

Rendering tendrá límite de extension invocations.

## DB-SQLGEN-108

Raw SQL contará contra budgets.

## DB-SQLGEN-109

Rendering no dependerá del wall clock.

## DB-SQLGEN-110

Rendering no dependerá de random IDs.

## DB-SQLGEN-111

Rendering no dependerá del locale.

## DB-SQLGEN-112

Rendering no dependerá del PID.

## DB-SQLGEN-113

Rendering no dependerá de memory addresses.

## DB-SQLGEN-114

Semantic sequences no serán ordenadas arbitrariamente.

## DB-SQLGEN-115

Unordered internal collections se canonicalizarán cuando corresponda.

## DB-SQLGEN-116

SQL structure estará separada de runtime data.

## DB-SQLGEN-117

Dynamic identifiers pasarán por IdentifierRenderer.

## DB-SQLGEN-118

Runtime data pasará por placeholders.

## DB-SQLGEN-119

Raw escape hatch será auditable.

## DB-SQLGEN-120

Security provenance podrá mantenerse hasta source map.

## DB-SQLGEN-121

Renderer no logueará bindings sensibles.

## DB-SQLGEN-122

Capabilities serán explícitas.

## DB-SQLGEN-123

Unsupported capability no causará silent omission.

## DB-SQLGEN-124

Platform-specific rendering reutilizará infraestructura común.

## DB-SQLGEN-125

No se duplicará innecesariamente todo el renderer por plataforma.

## DB-SQLGEN-126

Dialect-specific syntax tendrá componentes especializados.

## DB-SQLGEN-127

Dialect object no deberá convertirse en God Object.

## DB-SQLGEN-128

Extensions estarán sujetas al mismo writer.

## DB-SQLGEN-129

Extensions estarán sujetas a budgets.

## DB-SQLGEN-130

Extensions no abrirán conexiones.

## DB-SQLGEN-131

Extensions no inspeccionarán runtime bindings.

## DB-SQLGEN-132

Extensions no mutarán registry durante rendering.

## DB-SQLGEN-133

Shared renderer infrastructure será immutable.

## DB-SQLGEN-134

Rendering session será operation-scoped.

## DB-SQLGEN-135

No habrá state leakage entre queries.

## DB-SQLGEN-136

Renderer será compatible con FrankenPHP.

## DB-SQLGEN-137

Renderer será compatible con RoadRunner.

## DB-SQLGEN-138

Renderer será compatible con OpenSwoole.

## DB-SQLGEN-139

Concurrent rendering sobre frozen registries será seguro.

## DB-SQLGEN-140

Errors incluirán node/source context cuando sea posible.

## DB-SQLGEN-141

Errors no expondrán secretos.

## DB-SQLGEN-142

Planned placeholders y emitted placeholders deberán coincidir.

## DB-SQLGEN-143

Required security nodes deberán poder verificarse.

## DB-SQLGEN-144

Testing incluirá golden SQL.

## DB-SQLGEN-145

Testing incluirá cross-platform cases.

## DB-SQLGEN-146

Testing incluirá injection attempts.

## DB-SQLGEN-147

Testing incluirá precedence/parenthesis properties.

## DB-SQLGEN-148

Testing incluirá persistent-worker isolation.

## DB-SQLGEN-149

Testing incluirá budgets.

## DB-SQLGEN-150

Rendering complexity normal tenderá a O(nodes + output size).

## DB-SQLGEN-151

Compiled query cache será distinto de rendering cache.

## DB-SQLGEN-152

RenderedSqlFingerprint será distinto de CompiledQueryFingerprint.

## DB-SQLGEN-153

Internal result aliases serán trazables al output semántico.

## DB-SQLGEN-154

Renderer no cambiará aliases después de Alias Planning.

## DB-SQLGEN-155

Semantic provenance podrá atravesar rendering sin que renderer reinterprete semantics.

## DB-SQLGEN-156

Toda syntax ejecutable deberá provenir de una fuente estructural autorizada.

## DB-SQLGEN-157

Todo runtime data deberá permanecer fuera del SQL textual salvo excepción compile-time explícita.

## DB-SQLGEN-158

Toda decisión dialectal deberá ser explícita o provenir del dialect contract.

## DB-SQLGEN-159

Toda decisión de rendering deberá ser reproducible.

## DB-SQLGEN-160

SQL Generation jamás deberá alterar el significado observable del `SqlEmissionTree`.

---

# 308. Anti-patrones

## 308.1 Query Builder generando SQL

```php
$query->toSql();
```

no deberá significar que Query Builder contiene el renderer.

Una API de conveniencia podrá delegar:

```text
Builder
→ Query Engine
→ Compiler
→ Renderer
```

---

## 308.2 SQL mediante concatenación de valores

Incorrecto:

```php
$sql = "SELECT * FROM users WHERE email = '$email'";
```

---

## 308.3 Identifier interpolation

Incorrecto:

```php
$sql = "ORDER BY {$column}";
```

---

## 308.4 Placeholder discovery por regex

Incorrecto:

```php
preg_match_all('/\?/', $sql);
```

---

## 308.5 Reparar precedencia después

Incorrecto:

```text
render expression
→ discover wrong precedence
→ regex parentheses
```

---

## 308.6 Platform conditionals dispersos

Incorrecto:

```php
if ($platform === 'mysql') { ... }
if ($platform === 'pgsql') { ... }
```

en cada renderer.

---

## 308.7 Unknown node fallback

Incorrecto:

```php
$writer->append((string) $node);
```

---

## 308.8 Global alias counter

Incorrecto:

```php
static $alias = 0;
```

---

## 308.9 Global placeholder counter

Incorrecto:

```php
static $parameter = 0;
```

---

## 308.10 Runtime database introspection

Incorrecto:

```text
renderer
→ inspect database
→ discover index
→ change SQL
```

---

## 308.11 SQL parsing as validation architecture

Incorrecto:

```text
generate
→ parse generated SQL
→ reconstruct meaning
```

---

## 308.12 Raw SQL as universal escape

Incorrecto:

```text
unsupported semantic feature
→ emit raw SQL automatically
```

---

# 309. Ejemplo completo

Entrada:

```text
SqlSelectStatement S1
├── Projection
│   ├── Column(R1,id)
│   └── Column(R1,email)
├── From
│   └── Relation(R1, users)
├── Where
│   └── And
│       ├── Eq(Column(R1,active), ParameterOccurrence PO1)
│       └── Gte(Column(R1,created_at), ParameterOccurrence PO2)
├── OrderBy
│   └── Column(R1,created_at) DESC
└── Limit
    └── Literal(20)
```

Alias map:

```text
R1 → u
```

Placeholder plan:

```text
PO1 → $1
PO2 → $2
```

Dialect:

```text
PostgreSQL
```

---

# 310. Rendering traversal

```text
S1
 │
 ├─ SELECT
 │   ├─ Column(R1,id)
 │   └─ Column(R1,email)
 │
 ├─ FROM
 │   └─ Relation(R1)
 │
 ├─ WHERE
 │   └─ AND
 │       ├─ Eq(...)
 │       └─ Gte(...)
 │
 ├─ ORDER BY
 │
 └─ LIMIT
```

---

# 311. Identifier resolution

```text
R1
→ alias u
```

Column:

```text
Column(R1,id)
→ "u"."id"
```

---

# 312. Placeholder resolution

```text
PO1 → $1
PO2 → $2
```

---

# 313. Canonical output

```sql
SELECT "u"."id","u"."email" FROM "users" AS "u" WHERE "u"."active"=$1 AND "u"."created_at">=$2 ORDER BY "u"."created_at" DESC LIMIT 20
```

---

# 314. Pretty output

```sql
SELECT
    "u"."id",
    "u"."email"
FROM
    "users" AS "u"
WHERE
    "u"."active" = $1
    AND "u"."created_at" >= $2
ORDER BY
    "u"."created_at" DESC
LIMIT 20
```

---

# 315. Source map

```text
0..143
→ Statement S1

7..15
→ Column id

16..27
→ Column email

34..48
→ Relation R1

49..105
→ Where predicate

62..64
→ ParameterOccurrence PO1

92..94
→ ParameterOccurrence PO2

106..135
→ OrderBy

136..143
→ Limit
```

Offsets ilustrativos.

---

# 316. Binding relation

El SQL Generation System no crea runtime values.

Produce:

```text
$1
$2
```

Después:

```text
Binding Layout Compiler

$1 → Parameter P1 → BOOLEAN
$2 → Parameter P2 → DATETIME
```

Y sólo en ejecución:

```text
BindingSet

P1 → true
P2 → 2026-09-05T00:00:00
```

---

# 317. Separation achieved

```text
SQL text
≠
Binding values
```

---

# 318. Arquitectura final

```text
                 SqlEmissionTree
                        │
                        ▼
             ┌─────────────────────┐
             │ SQL Generation      │
             │ System              │
             └──────────┬──────────┘
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
   Statements       Expressions      Predicates
        │               │                │
        └───────────────┼────────────────┘
                        ▼
              Precedence Model
                        │
                        ▼
                 Dialect Rules
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
      Identifiers   Placeholders   Literals
           │            │            │
           └────────────┼────────────┘
                        ▼
                  SqlTextWriter
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        Rendered SQL          SqlSourceMap
             │                     │
             └──────────┬──────────┘
                        ▼
              RenderedSqlArtifact
```

---

# 319. Relación con los documentos siguientes

Este documento define el motor común de generación.

Los siguientes especializarán el comportamiento por plataforma:

```text
69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
```

Después:

```text
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
```

formalizará las extensiones.

Luego:

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

profundizará en:

```text
parameters
placeholders
binding layouts
prepared-statement contracts
specialization
```

Finalmente:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

definirá la reutilización de artifacts compilados.

---

# 320. Block 6 status

```text
Block 6 — SQL Compiler

66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
   ✓

67_DATABASE_SQL_COMPILER_PIPELINE.md
   ✓

68_DATABASE_SQL_GENERATION_SYSTEM.md
   ✓

69_DATABASE_MYSQL_SQL_COMPILER.md
   next

70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 321. Principio final

La regla fundamental será:

```text
The SQL renderer renders decisions.
It does not make query decisions.
```

Por tanto:

```text
Semantic Engine
determines meaning

Optimizer
determines equivalent preferable logical form

Planner
determines execution strategy

Execution Plan
determines executable operation structure

SQL Compiler
determines SQL representation

SQL Generation System
renders that representation
```

---

# 322. Fórmula final

```text
SafeSqlGeneration
=
StructuredEmission
+
TypedIdentifiers
+
PlannedPlaceholders
+
ExplicitDialectRules
+
CorrectPrecedence
+
DeterministicFormatting
+
SourceMapping
+
InjectionBoundaries
+
FrozenExtensions
+
BoundedRendering
+
PersistentRuntimeIsolation
```

Y la transformación final:

```text
SqlEmissionTree
        │
        │ no semantic guessing
        │ no optimization
        │ no planning
        │ no database I/O
        │ no runtime interpolation
        ▼
SQL Generation System
        │
        ├── Statement Rendering
        ├── Clause Rendering
        ├── Expression Rendering
        ├── Predicate Rendering
        ├── Identifier Rendering
        ├── Placeholder Rendering
        ├── Literal Rendering
        ├── Precedence / Parentheses
        ├── Formatting
        ├── Source Mapping
        └── Budget Enforcement
        │
        ▼
RenderedSqlArtifact
```

> En VoltStack, generar SQL no significará construir strings a partir de una consulta. Significará renderizar de forma determinista, segura y trazable una representación SQL estructurada cuya semántica, estrategia y capacidades ya fueron resueltas por las capas anteriores.