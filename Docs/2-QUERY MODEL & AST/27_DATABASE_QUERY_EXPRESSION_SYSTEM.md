# 27_DATABASE_QUERY_EXPRESSION_SYSTEM.md

# VoltStack Quantum Database
## Query Expression System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 27 — Query Expression System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / AST / Expression System  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el sistema oficial de expresiones de consulta de:

```text
VoltStack/Quantum/Database
```

El Expression System representa operaciones que producen valores dentro del Query AST.

Ejemplos:

```text
users.id
users.price * users.quantity
COUNT(*)
LOWER(users.email)
COALESCE(users.nickname, users.name)
CASE ... END
CAST(...)
CURRENT_TIMESTAMP
subquery
JSON extraction
window functions
```

La arquitectura general será:

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
Expression System
      │
      ▼
Semantic Analysis
      │
      ├── Symbol Resolution
      ├── Type Inference
      ├── Function Resolution
      └── Capability Requirements
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
Dialect
      │
      ▼
SQL
```

---

# 2. Objetivo

VoltStack deberá proporcionar un sistema de expresiones:

```text
Typed
+
Immutable
+
Composable
+
Semantic
+
Portable-first
+
Target-neutral
+
Extensible
+
Optimizable
```

El Expression System no será una colección de strings SQL.

---

# 3. Regla maestra

> Una expresión describe qué valor u operación desea representar la consulta, no cómo deberá escribirse dicha operación en SQL.

Por tanto:

```text
Expression
=
Semantic Intent
+
Operands
+
Structural Options
```

No:

```text
Expression
=
SQL Fragment
```

---

# 4. Expression ≠ SQL

Ejemplo incorrecto:

```php
new ExpressionNode('LOWER(users.email)');
```

Ejemplo conceptual correcto:

```text
FunctionCallNode
├── function: string.lower
└── arguments
    └── ColumnReferenceNode
        └── users.email
```

Posteriormente:

```text
Semantic Function
        │
        ▼
Planner
        │
        ▼
Dialect
```

decidirán la representación concreta.

---

# 5. Expression ≠ Predicate

V1 mantendrá la separación:

```text
ExpressionNode
≠
PredicateNode
```

aunque posteriormente el Type System pueda considerar determinados predicates como valores booleanos.

La separación mejora:

- validación;
- claridad;
- análisis;
- APIs;
- optimización;
- portabilidad.

---

# 6. Contrato base

Conceptualmente:

```php
interface ExpressionNode extends QueryAstNode
{
}
```

El contrato deberá permanecer deliberadamente pequeño.

No deberá contener:

```php
public function toSql(): string;
public function execute(): mixed;
public function inferType(): DatabaseType;
public function platform(): Platform;
```

---

# 7. Arquitectura

```text
ExpressionNode
│
├── Reference
├── Parameter
├── Literal
├── Function
├── Aggregate
├── Arithmetic
├── Unary
├── Conditional
├── Cast
├── Subquery
├── Tuple
├── String
├── Numeric
├── Date/Time
├── JSON
├── Window
├── Special Value
├── Raw
└── Extension
```

---

# 8. Taxonomía inicial

```text
ExpressionNode
│
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
├── StringExpressionNode
├── NumericExpressionNode
├── DateTimeExpressionNode
├── JsonExpressionNode
├── WindowFunctionNode
├── SpecialValueExpressionNode
├── RawExpressionNode
└── ExtensionExpressionNode
```

Estas son familias conceptuales.

No todas deberán transformarse obligatoriamente en interfaces PHP.

---

# 9. Principio de granularidad

VoltStack deberá evitar dos extremos:

```text
Expression("anything")
```

y:

```text
AddExpression
SubtractExpression
MultiplyExpression
DivideExpression
ModuloExpression
...
```

cuando una representación estructurada con operador sea suficiente.

---

# 10. Estrategia recomendada

Utilizar:

```text
Generic structural node
+
Typed semantic operator
```

cuando la forma estructural sea común.

Ejemplo:

```text
ArithmeticExpressionNode
├── ArithmeticOperator::ADD
├── left
└── right
```

---

# 11. Nodes especializados

Se utilizará un node especializado cuando exista una diferencia importante en:

- children;
- semántica;
- type inference;
- validation;
- capability requirements;
- optimizer behavior;
- compilation.

---

# 12. ColumnReferenceNode

Representa una referencia a una columna.

```text
ColumnReferenceNode
├── qualifier?
└── column
```

Ejemplos:

```text
email

users.email

u.email
```

---

# 13. Column reference unresolved

Inicialmente:

```text
ColumnReferenceNode("email")
```

puede estar unresolved.

El AST no necesita conocer todavía:

```text
users.email
VARCHAR
nullable
primary key
```

---

# 14. Symbol resolution

Será responsabilidad de:

```text
Semantic Query Engine
      │
      ▼
Symbol Resolution System
```

---

# 15. Qualified reference

Conceptualmente:

```php
final readonly class ColumnReferenceNode implements ExpressionNode
{
    public function __construct(
        public Identifier $column,
        public ?Identifier $qualifier = null,
    ) {}
}
```

---

# 16. Qualifier semantics

El qualifier podrá representar:

- table alias;
- source alias;
- CTE alias;
- derived source alias.

Su resolución exacta pertenece al Semantic Engine.

---

# 17. No table object

Incorrecto:

```php
new ColumnReferenceNode(
    column: 'email',
    table: $schemaTable,
);
```

El node no deberá acoplarse al Schema Model.

---

# 18. ParameterExpressionNode

Representa un parámetro bindable.

```text
ParameterExpressionNode
├── ParameterId
└── TypeHint?
```

---

# 19. Separación parameter/value

```text
AST
└── ParameterExpressionNode(p1)

BindingSet
└── p1 → runtime value
```

---

# 20. Parameter TypeHint

Podrá contener una intención de tipo:

```text
integer
string
uuid
datetime
decimal
json
```

sin almacenar detalles del driver.

---

# 21. TypeHint ≠ PDO type

No:

```text
PDO::PARAM_STR
```

dentro del AST.

---

# 22. Parameter placeholders

El node no conocerá:

```text
?
:p1
$1
```

Esto pertenece al Compiler/Dialect/Binding Profile.

---

# 23. LiteralExpressionNode

Representa valores literales estructuralmente seguros.

---

# 24. Literales core

Inicialmente:

```text
NULL
TRUE
FALSE
```

podrán tener representación especializada o un literal tipado.

---

# 25. Runtime values

Valores de aplicación deberán convertirse normalmente en parámetros.

Ejemplo:

```php
->where('email', $email)
```

produce:

```text
ParameterExpressionNode
```

no:

```text
LiteralExpressionNode($email)
```

---

# 26. Literal categories

Podrán existir:

```text
NullLiteral
BooleanLiteral
IntegerLiteral
DecimalLiteral
StringLiteral
BinaryLiteral
```

pero su uso deberá estar controlado.

---

# 27. Literal policy

Una política podrá decidir:

```text
STRUCTURAL_LITERAL
PARAMETERIZE
FORBIDDEN_LITERAL
```

---

# 28. Constant expressions

Algunas constantes pueden ser útiles para:

- optimizer;
- schema expressions;
- generated expressions;
- internal rewrites.

---

# 29. FunctionCallNode

Representa una llamada a función semántica.

```text
FunctionCallNode
├── FunctionId
├── arguments
└── options
```

---

# 30. FunctionId

No deberá ser necesariamente el nombre SQL.

Ejemplos:

```text
string.lower
string.upper
string.length

numeric.abs
numeric.round

datetime.current_timestamp
datetime.extract

aggregate.count
aggregate.sum

json.extract
json.contains
```

---

# 31. Semantic function names

Esto permite:

```text
string.length
```

compilarse según el target.

---

# 32. Function registry

El Semantic Engine podrá utilizar:

```text
FunctionRegistry
│
├── FunctionDescriptor
├── FunctionSignature
├── TypeRules
├── CapabilityRequirements
└── SemanticProperties
```

---

# 33. FunctionDescriptor

Conceptualmente:

```text
FunctionDescriptor
├── FunctionId
├── signatures
├── returnTypeRule
├── nullabilityRule
├── volatility
├── determinism
├── aggregate?
├── window?
├── capabilityRequirements
└── portability
```

---

# 34. Function resolution

```text
FunctionCallNode
      │
      ▼
FunctionResolver
      │
      ├── argument count
      ├── argument types
      ├── overloads
      └── context
      │
      ▼
ResolvedFunction
```

---

# 35. Function overloading

El mismo FunctionId podrá tener varias signatures.

Ejemplo conceptual:

```text
numeric.round(decimal)
numeric.round(decimal, integer)
numeric.round(float)
```

---

# 36. Variadic functions

Funciones como:

```text
COALESCE
```

podrán tener firmas variádicas.

---

# 37. Function syntax special cases

No todas las funciones tienen sintaxis:

```text
NAME(arg1, arg2)
```

Ejemplos:

```text
CURRENT_TIMESTAMP
EXTRACT(...)
CAST(...)
```

Por ello:

```text
semantic function
≠
literal SQL function syntax
```

---

# 38. Specialized nodes vs functions

Algunas construcciones merecen nodes especializados.

Por ejemplo:

```text
CastExpressionNode
CaseExpressionNode
WindowFunctionNode
```

aunque SQL las parezca funciones.

---

# 39. AggregateExpressionNode

Representa una agregación.

```text
AggregateExpressionNode
├── AggregateFunctionId
├── argument?
├── quantifier
├── filter?
└── order?
```

---

# 40. Aggregate functions

Inicialmente:

```text
aggregate.count
aggregate.sum
aggregate.avg
aggregate.min
aggregate.max
```

---

# 41. Aggregate argument

`COUNT(*)` deberá distinguirse de:

```text
COUNT(column)
```

---

# 42. Aggregate wildcard

Podrá utilizar:

```text
AggregateWildcard
```

como value object estructural.

No como `ColumnReference("*")`.

---

# 43. Aggregate quantifier

```text
DEFAULT
DISTINCT
ALL
```

---

# 44. Aggregate FILTER

Cuando exista semánticamente:

```text
AggregateExpressionNode
└── filter: PredicateNode?
```

sujeto a capabilities.

---

# 45. Ordered aggregate

Algunas agregaciones pueden aceptar ordering.

Deberá modelarse estructuralmente cuando sea soportado.

---

# 46. Aggregate semantic validation

Semantic Analysis verificará:

- tipos de argumentos;
- número de argumentos;
- contexto;
- nesting;
- capabilities.

---

# 47. Aggregate context

Una agregación podrá ser inválida en determinados contextos.

Ejemplo:

```text
WHERE
```

según el modelo SQL.

La validación pertenece al Semantic Query Engine.

---

# 48. ArithmeticExpressionNode

Representa operaciones aritméticas binarias.

```text
ArithmeticExpressionNode
├── operator
├── left
└── right
```

---

# 49. ArithmeticOperator

Inicialmente:

```text
ADD
SUBTRACT
MULTIPLY
DIVIDE
MODULO
POWER
```

donde `POWER` podrá ser capability/function-dependent.

---

# 50. Operator semantics

El enum representa intención.

No syntax.

---

# 51. Example

```text
price * quantity
```

AST:

```text
ArithmeticExpressionNode
├── MULTIPLY
├── ColumnReference(price)
└── ColumnReference(quantity)
```

---

# 52. Numeric type inference

Posteriormente:

```text
integer + integer
decimal + integer
float / integer
```

será analizado por:

```text
Query Type System
```

---

# 53. Arithmetic errors

Casos como:

```text
string + datetime
```

deberán detectarse semánticamente.

---

# 54. UnaryExpressionNode

```text
UnaryExpressionNode
├── UnaryOperator
└── operand
```

---

# 55. UnaryOperator

Inicialmente:

```text
PLUS
NEGATE
BITWISE_NOT
```

según capabilities.

---

# 56. Logical NOT

Pertenecerá al Predicate System:

```text
NotPredicateNode
```

no a `UnaryExpressionNode`.

---

# 57. String expressions

Las operaciones de strings deberán ser semánticas.

Ejemplos:

```text
string.concat
string.lower
string.upper
string.trim
string.substring
string.length
string.replace
```

---

# 58. Concatenation

No asumir:

```text
||
```

ni:

```text
CONCAT(...)
```

en el AST.

---

# 59. StringConcatExpressionNode

Podría representarse como:

```text
FunctionCallNode(string.concat)
```

o como node especializado si existen razones de optimización/semántica.

---

# 60. V1 recommendation

Usar:

```text
FunctionCallNode
```

para operaciones de string comunes salvo que necesiten estructura especial.

---

# 61. Numeric expressions

Funciones comunes:

```text
numeric.abs
numeric.ceil
numeric.floor
numeric.round
numeric.sqrt
numeric.power
numeric.random
```

---

# 62. Random semantics

`numeric.random` deberá tener metadata semántica:

```text
VOLATILE
NON_DETERMINISTIC
```

---

# 63. Determinism

Las expresiones podrán clasificarse semánticamente:

```text
DETERMINISTIC
STABLE
VOLATILE
UNKNOWN
```

---

# 64. Determinism location

No necesariamente será una propiedad mutable del node.

Podrá derivarse mediante:

```text
ExpressionSemanticProperties
```

---

# 65. Volatility

Modelo conceptual:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

si se requiere una clasificación más cercana a semántica de bases de datos.

---

# 66. Why volatility matters

Afecta:

- constant folding;
- expression deduplication;
- predicate reordering;
- caching;
- repeated evaluation;
- optimization safety.

---

# 67. Example

No deberá asumirse:

```text
random() = random()
```

aunque ambos nodes tengan la misma forma.

---

# 68. Structural equality vs evaluation equivalence

Dos expresiones estructuralmente iguales no necesariamente producen el mismo valor si son volátiles.

---

# 69. CaseExpressionNode

Representa `CASE`.

Dos formas:

```text
searched CASE
simple CASE
```

---

# 70. Searched CASE

```text
CaseExpressionNode
├── operand: null
├── branches
│   ├── Predicate → Expression
│   └── Predicate → Expression
└── elseExpression?
```

---

# 71. Simple CASE

```text
CaseExpressionNode
├── operand: Expression
├── branches
│   ├── Expression → Expression
│   └── Expression → Expression
└── elseExpression?
```

---

# 72. CaseBranch model

Podrán existir dos tipos:

```text
SearchedCaseBranchNode
SimpleCaseBranchNode
```

para evitar unions ambiguas.

---

# 73. Alternative

Un solo `CaseExpressionNode` con modo explícito:

```text
CaseMode::SEARCHED
CaseMode::SIMPLE
```

---

# 74. Recommendation

Preferir tipos estructuralmente claros antes que campos `mixed`.

---

# 75. CASE type inference

El resultado se inferirá a partir de:

```text
THEN expressions
+
ELSE expression
```

mediante el Query Type System.

---

# 76. CASE nullability

También será derivada semánticamente.

---

# 77. CastExpressionNode

```text
CastExpressionNode
├── expression
└── targetType
```

---

# 78. TargetType

Será un tipo semántico VoltStack.

Ejemplo:

```text
StringType
IntegerType
DecimalType
DateTimeType
JsonType
UuidType
```

---

# 79. No native type syntax

No almacenar:

```text
VARCHAR(255)
SIGNED
::jsonb
```

en el core portable.

---

# 80. Platform-specific casts

Podrán existir mediante:

```text
PlatformTypeReference
```

o extension node explícito.

---

# 81. Explicit vs implicit conversion

El sistema deberá distinguir:

```text
ExplicitCast
ImplicitCoercion
DriverConversion
HydrationConversion
```

---

# 82. Cast node

Sólo representa:

```text
explicit query-level conversion
```

---

# 83. Implicit coercion

Será metadata/decision del Semantic Type System.

---

# 84. SubqueryExpressionNode

Representa una consulta usada como expresión.

```text
SubqueryExpressionNode
└── QueryRootNode
```

---

# 85. Scalar subquery

Semantic Analysis deberá verificar cuando el contexto requiera:

```text
one column
```

y, donde sea posible, cardinality expectations.

---

# 86. Subquery categories

Podrán distinguirse semánticamente:

```text
SCALAR
ROW
TABLE
EXISTS_TARGET
IN_TARGET
```

---

# 87. EXISTS

La forma:

```text
EXISTS(subquery)
```

pertenece al Predicate System.

---

# 88. IN subquery

Igualmente será:

```text
InPredicateNode
```

con un query source.

---

# 89. TupleExpressionNode

```text
TupleExpressionNode
└── ExpressionNodeList
```

---

# 90. Tuple use cases

Ejemplos:

```text
(a, b)
VALUES
row comparisons
composite IN
multi-column assignment
```

dependiendo de capabilities.

---

# 91. Tuple ≠ array

No confundir:

```text
TupleExpression
```

con un tipo ARRAY de PostgreSQL.

---

# 92. Row expression

Podrá utilizarse el término:

```text
RowValueExpression
```

si ofrece mejor precisión semántica.

---

# 93. DateTime Expression System

Las operaciones de fecha/hora deberán permanecer semánticas.

---

# 94. Date/time functions

Ejemplos:

```text
datetime.current_date
datetime.current_time
datetime.current_timestamp
datetime.extract
datetime.add
datetime.subtract
datetime.diff
datetime.truncate
```

---

# 95. Current timestamp

Conceptualmente:

```text
CurrentTimestampExpressionNode
```

o:

```text
FunctionCallNode(datetime.current_timestamp)
```

---

# 96. Recommendation

Funciones especiales muy frecuentes podrán tener nodes especializados si mejoran:

- semantics;
- optimizer rules;
- portability;
- diagnostics.

---

# 97. Clock semantics

Deberá distinguirse cuando sea necesario entre:

```text
statement time
transaction time
server clock time
application clock time
```

---

# 98. Application time

Normalmente deberá resolverse fuera del AST:

```text
Application Clock
      │
      ▼
Parameter
```

---

# 99. Server time

Será una expresión semántica.

---

# 100. Date interval

Podrá existir:

```text
IntervalExpression
```

o un `IntervalValue` tipado.

No deberá reducirse prematuramente a syntax SQL.

---

# 101. Date arithmetic

Ejemplo semántico:

```text
DateAddExpression
├── datetime
├── amount
└── unit
```

puede ser preferible a:

```text
ArithmeticExpression(datetime + interval)
```

si mejora portabilidad.

---

# 102. DateTimeUnit

Ejemplos:

```text
SECOND
MINUTE
HOUR
DAY
WEEK
MONTH
QUARTER
YEAR
```

con capability validation.

---

# 103. JSON Expression System

JSON deberá ser una familia semántica importante.

---

# 104. JSON operations

Inicialmente:

```text
json.extract
json.extract_scalar
json.set
json.remove
json.merge
json.array_length
json.object_keys
json.type
```

y predicates:

```text
json.contains
json.has_key
json.path_exists
```

---

# 105. JSON path

No deberá ser siempre un raw string.

Podrá existir:

```text
JsonPath
├── Root
└── segments
```

---

# 106. JsonPathSegment

Ejemplos:

```text
PropertySegment
ArrayIndexSegment
WildcardSegment
RecursiveSegment
```

según capabilities.

---

# 107. Portable JSON subset

VoltStack deberá definir un subconjunto semántico portable.

---

# 108. Native JSON extensions

Operaciones avanzadas podrán estar namespaced:

```text
postgresql.jsonb.*
mysql.json.*
sqlite.json.*
```

sólo cuando no exista semántica portable equivalente.

---

# 109. JSON result type

Debe distinguirse:

```text
JSON value
```

de:

```text
scalar text value
```

---

# 110. Example

```text
JsonExtractExpression
```

puede devolver `JsonType`.

Mientras:

```text
JsonExtractScalarExpression
```

podrá devolver otro tipo inferido/string.

---

# 111. JSON operators

El AST no almacenará:

```text
->>
#>>
@>
JSON_EXTRACT
json_extract
```

como representación portable.

---

# 112. Compiler mapping

```text
JSON semantic expression
          │
          ▼
Planner
          │
          ▼
Capability Strategy
          │
          ▼
Target Compiler/Dialect
```

---

# 113. Conditional functions

Funciones como:

```text
COALESCE
NULLIF
```

deberán formar parte del modelo semántico.

---

# 114. Coalesce

Conceptualmente:

```text
CoalesceExpressionNode
└── ExpressionNodeList
```

o:

```text
FunctionCallNode(conditional.coalesce)
```

---

# 115. Recommendation

V1 puede usar funciones semánticas registradas.

Specialized nodes sólo cuando la estructura lo justifique.

---

# 116. Null semantics

El Expression System deberá respetar SQL three-valued logic.

---

# 117. NULL ≠ ordinary value

No deberán aplicarse automáticamente reglas PHP como:

```text
null == null
```

a expresiones SQL.

---

# 118. NULL comparison

Se modelará mediante Predicate System:

```text
NullPredicateNode
```

cuando corresponda.

---

# 119. Nullability analysis

Semantic Analysis podrá producir:

```text
NON_NULL
NULLABLE
UNKNOWN
```

---

# 120. Expression nullability

Será derivada de:

- operands;
- function semantics;
- schema metadata;
- joins;
- CASE;
- casts;
- platform semantics.

---

# 121. WindowFunctionNode

Representa una función evaluada sobre una ventana.

```text
WindowFunctionNode
├── function
└── windowSpecification
```

---

# 122. Window function source

La función podrá ser:

- aggregate;
- ranking;
- navigation;
- analytic.

---

# 123. Examples

```text
window.row_number
window.rank
window.dense_rank
window.lag
window.lead
window.first_value
window.last_value
window.nth_value
```

---

# 124. WindowSpecification

Definida estructuralmente por:

```text
PARTITION
ORDER
FRAME
```

sin syntax SQL.

---

# 125. Named windows

Podrá existir:

```text
NamedWindowReferenceNode
```

cuando se implemente soporte.

---

# 126. Aggregate + window

Una agregación utilizada con `OVER` deberá representarse sin duplicar semántica innecesariamente.

Posible:

```text
WindowFunctionNode
├── AggregateExpressionNode
└── WindowSpecification
```

---

# 127. Window capability validation

El Planner/Semantic Engine consultará:

```text
query.window.*
```

capabilities.

---

# 128. Ranking result types

Ejemplo:

```text
ROW_NUMBER
RANK
```

podrán inferirse como integer-like semantic types.

---

# 129. SpecialValueExpressionNode

Podrá representar valores especiales del servidor.

Ejemplos:

```text
CURRENT_DATE
CURRENT_TIME
CURRENT_TIMESTAMP
DEFAULT
```

cuando realmente sean expresiones válidas en el contexto.

---

# 130. DEFAULT context

`DEFAULT` no es universalmente una expresión válida.

Su uso deberá estar restringido por contexto.

---

# 131. Context-sensitive expression

Algunas expresiones sólo son válidas en:

```text
INSERT
UPDATE
SELECT
ORDER BY
GROUP BY
RETURNING
```

etc.

---

# 132. Expression context

Semantic validation recibirá:

```text
ExpressionContext
```

---

# 133. ExpressionContext examples

```text
SELECT_PROJECTION
WHERE_OPERAND
JOIN_CONDITION
GROUP_BY
HAVING
ORDER_BY
INSERT_VALUE
UPDATE_ASSIGNMENT
RETURNING
WINDOW_PARTITION
WINDOW_ORDER
```

---

# 134. Context is external

No almacenar:

```text
$this->context = WHERE;
```

dentro de cada expression node.

---

# 135. Expression semantic properties

El Semantic Engine podrá generar:

```text
ExpressionSemanticInfo
├── resolvedType
├── nullability
├── volatility
├── determinism
├── resolvedFunction
├── resolvedSymbols
├── capabilityRequirements
├── portability
├── constantness
└── semanticFlags
```

---

# 136. Side table

```text
AstOccurrenceId
      │
      ▼
ExpressionSemanticInfo
```

---

# 137. Constantness

Clasificación posible:

```text
CONSTANT
PARAMETER_DEPENDENT
ROW_DEPENDENT
QUERY_DEPENDENT
VOLATILE
UNKNOWN
```

---

# 138. Why constantness matters

Permite optimizaciones como:

```text
constant folding
predicate simplification
expression reuse
```

sin alterar semántica.

---

# 139. Expression dependencies

Podrá calcularse:

```text
ExpressionDependencySet
```

con:

- columns;
- parameters;
- functions;
- subqueries;
- capabilities.

---

# 140. Column dependency

Ejemplo:

```text
price * quantity
```

depende de:

```text
price
quantity
```

---

# 141. Parameter dependency

Ejemplo:

```text
price * :factor
```

depende también de `factor`.

---

# 142. Correlated subquery dependency

Un subquery podrá depender de símbolos externos.

Esto se resolverá durante Semantic Analysis.

---

# 143. Correlation

No deberá modelarse mediante referencia PHP al parent query.

---

# 144. Correct model

```text
ColumnReferenceNode
      │
      ▼
Semantic Resolver
      │
      ▼
Outer Scope Symbol
```

---

# 145. Expression scope

Semantic Analysis deberá mantener:

```text
SemanticScope
```

para resolver referencias.

---

# 146. Expression precedence

El AST elimina gran parte del problema de precedencia textual.

---

# 147. Example

```text
a + b * c
```

será:

```text
ADD
├── a
└── MULTIPLY
    ├── b
    └── c
```

---

# 148. Parentheses

No deberán almacenarse sólo por razones de syntax.

La estructura del árbol ya representa grouping.

---

# 149. Explicit grouping

Sólo se preservará metadata/grouping explícito si afecta:

- raw rendering;
- semantic interpretation;
- diagnostics;
- target syntax.

---

# 150. Associativity

El Optimizer podrá normalizar:

```text
(a + b) + c
```

si el tipo y semántica permiten asociatividad segura.

---

# 151. Numeric precision caution

No asumir que:

```text
(a + b) + c
=
a + (b + c)
```

para todos los tipos numéricos.

---

# 152. Optimizer safety

Las reglas deberán consultar:

```text
ResolvedType
SemanticProperties
Volatility
Platform Semantics
```

cuando sea necesario.

---

# 153. Expression normalization

Normalizaciones estructurales seguras pueden incluir:

- canonical node shapes;
- flattening seguro;
- normalized function IDs;
- normalized ParameterIds;
- canonical literal representation.

---

# 154. Normalization ≠ optimization

Normalización:

```text
equivalent representation
```

Optimización:

```text
alternative representation intended to improve execution
```

---

# 155. Expression folding

Constant folding pertenece al Optimizer, no al node constructor.

---

# 156. Example

```text
2 + 3
```

podría convertirse posteriormente en:

```text
5
```

si:

- type semantics known;
- overflow semantics known;
- operation deterministic;
- target semantics preserved.

---

# 157. Function volatility and folding

Nunca fold automáticamente:

```text
random()
current_timestamp
```

sin conocer sus semánticas.

---

# 158. Common subexpression elimination

Podrá existir en el futuro.

Pero:

```text
structural equality
```

no es suficiente.

También se necesita:

```text
determinism
+
scope
+
type
+
evaluation semantics
```

---

# 159. Expression rewrite rules

Ejemplos futuros:

```text
x + 0 → x
x * 1 → x
double cast removal
constant CASE simplification
```

sólo cuando sean semánticamente seguras.

---

# 160. No PHP semantic assumptions

No utilizar reglas PHP directamente sobre SQL expressions.

---

# 161. Type System integration

Cada expression deberá poder ser analizada por:

```text
Query Type System
```

---

# 162. ExpressionTypeResolver

Conceptualmente:

```text
ExpressionNode
+
Semantic Context
+
Resolved Children
      │
      ▼
ExpressionTypeResolver
      │
      ▼
ResolvedQueryType
```

---

# 163. Type resolution registry

Podrá utilizar:

```text
AstNodeKind
→ ExpressionTypeRule
```

---

# 164. Core type rules

Ejemplos:

```text
ColumnReference → column semantic type
Parameter → declared/inferred type
Literal → literal type
Arithmetic → operator promotion rule
Function → signature return rule
CASE → common result type
CAST → target type
Subquery → projected scalar type
```

---

# 165. Type inference direction

Podrá ser bidireccional.

Ejemplo:

```text
column(uuid) = parameter(p1)
```

puede inferir:

```text
p1 : uuid
```

---

# 166. Parameter inference

El Parameter Node permanece igual.

El resultado:

```text
p1 → UuidType
```

vive en semantic metadata/binding plan.

---

# 167. Function type inference

Ejemplo:

```text
COALESCE(integer, integer)
→ integer
```

---

# 168. CASE common type

```text
CASE
  WHEN ... THEN integer
  ELSE decimal
END
```

podrá resultar en un tipo común calculado.

---

# 169. Type incompatibility

Si no existe tipo común válido:

```text
QueryTypeMismatchException
```

antes de Compilation cuando sea posible.

---

# 170. Capability integration

Una expression podrá generar requirements.

Ejemplo:

```text
JsonExtractExpression
→ query.json.extract
```

---

# 171. CapabilityRequirementSet

Semantic Analysis podrá acumular:

```text
Expression Requirements
+
Predicate Requirements
+
Query Requirements
```

---

# 172. Capability checks

No deberán estar distribuidos como:

```php
if ($platform === 'postgresql') {
}
```

---

# 173. Correct

```text
Expression
      │
      ▼
Requirement Resolver
      │
      ▼
CapabilityRequirement
      │
      ▼
Planner
```

---

# 174. Planner integration

El Planner decidirá:

```text
Native
Emulated
Alternative Strategy
Unsupported
```

---

# 175. Example concatenation

Semantic expression:

```text
string.concat(a, b)
```

Target strategies:

```text
MySQL/MariaDB → function strategy
PostgreSQL    → operator/function strategy
SQLite        → operator strategy
```

El AST no cambia.

---

# 176. Expression portability

Clasificación:

```text
PORTABLE
PORTABLE_WITH_CAPABILITY
PORTABLE_WITH_EMULATION
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

---

# 177. Portability is contextual

La misma expression podrá ser portable para un target set y no para otro.

---

# 178. RawExpressionNode

Escape hatch explícito:

```text
RawExpressionNode
├── fragment
├── parameterReferences
├── portability
└── trust metadata
```

---

# 179. Raw expression rules

Debe:

- permanecer identificable;
- utilizar bindings;
- evitar interpolation;
- afectar fingerprint;
- limitar optimizer assumptions.

---

# 180. Raw SQL is opaque

Por defecto el Semantic Engine no deberá intentar interpretar arbitrary raw SQL.

---

# 181. Raw result type

Cuando sea necesario:

```text
RawExpressionNode
└── explicitTypeHint?
```

---

# 182. Unknown raw type

Si no existe hint:

```text
UnknownQueryType
```

hasta donde el pipeline pueda operar.

---

# 183. Trusted raw APIs

Podrá existir una API deliberadamente explícita:

```php
DB::raw(...)
```

pero deberá considerarse escape hatch.

---

# 184. Raw ≠ unsafe interpolation

La existencia de Raw no elimina binding seguro.

---

# 185. ExtensionExpressionNode

Permite nuevas expresiones mediante paquetes.

---

# 186. Extension requirements

Una nueva expression deberá registrar, según aplique:

```text
Node Descriptor
Semantic Handler
Type Rule
Capability Requirements
Optimizer Rules
Planner Strategy
Compiler Handler
Fingerprint Handler
Diagnostics
```

---

# 187. Example extension

```text
geo.distance
```

podría registrar:

```text
GeoDistanceExpressionNode
```

sin modificar el core.

---

# 188. Extension namespace

Ejemplo:

```text
package.geo.expression.distance
```

---

# 189. No runtime extension registration

Los handlers deberán registrarse durante bootstrap y congelarse.

---

# 190. Extension completeness

Una extensión no deberá declararse activa si faltan handlers obligatorios para el pipeline donde pretende operar.

---

# 191. Unsupported target

Si una expression extension no puede ejecutarse:

```text
CapabilityNotSupportedException
```

o una excepción semántica especializada.

---

# 192. Expression registry architecture

No deberá existir necesariamente un único:

```text
ExpressionRegistry
```

con todas las responsabilidades.

---

# 193. Specialized registries

Preferir:

```text
FunctionRegistry
ExpressionSemanticHandlerRegistry
ExpressionTypeRuleRegistry
ExpressionCompilerRegistry
ExpressionDiagnosticRegistry
```

cuando exista una frontera real.

---

# 194. Avoid universal registry

No:

```text
Registry
├── drivers
├── functions
├── AST nodes
├── types
├── compilers
└── ORM mappings
```

---

# 195. Query Builder API

La API pública deberá permanecer sencilla.

Ejemplo conceptual:

```php
DB::table('orders')
    ->select([
        'id',
        DB::expr()->multiply('price', 'quantity')->as('total'),
    ]);
```

---

# 196. Expression builder

Podrá existir:

```text
ExpressionBuilder
```

como API ergonómica.

---

# 197. ExpressionBuilder responsibility

Sólo:

```text
developer input
→ ExpressionNode
```

---

# 198. ExpressionBuilder must not

No deberá:

- resolve schema;
- infer final types;
- compile SQL;
- inspect connection;
- detect vendor;
- execute.

---

# 199. Column helper

Ejemplo:

```php
expr()->column('users.email');
```

produce:

```text
ColumnReferenceNode
```

---

# 200. Parameter helper

```php
expr()->parameter($value);
```

podrá coordinar:

```text
ParameterId
+
BindingSet
```

mediante el Query Builder construction context.

---

# 201. Value helper

```php
expr()->value($value);
```

por defecto deberá parameterizar.

---

# 202. Literal helper

Una API distinta deberá existir para literales deliberados:

```php
expr()->literal(...)
```

---

# 203. Function helper

```php
expr()->function('string.lower', ...);
```

---

# 204. Convenience functions

Podrán existir:

```php
expr()->lower(...)
expr()->count(...)
expr()->sum(...)
expr()->coalesce(...)
```

sin cambiar el AST resultante.

---

# 205. Arithmetic fluent API

Ejemplo:

```php
expr()
    ->column('price')
    ->multiply(expr()->column('quantity'));
```

si el diseño lo hace legible.

---

# 206. Alternative functional API

```php
expr()->multiply(
    expr()->column('price'),
    expr()->column('quantity'),
);
```

---

# 207. API vs AST

Varias APIs públicas pueden producir el mismo node.

---

# 208. ORM integration

El ORM utilizará el mismo Expression System.

No deberá crear un segundo árbol de expresiones.

---

# 209. Repository integration

Ejemplo:

```text
Repository Query API
        │
        ▼
Expression System
```

---

# 210. Persistence integration

Persistence Engine podrá generar expressions para:

- assignments;
- identifiers;
- generated values;
- version columns.

---

# 211. Schema expressions

El Query Expression System no deberá reutilizarse automáticamente como Schema Expression AST.

---

# 212. Why

Aunque exista solapamiento:

```text
query expression
```

y:

```text
schema default/generated expression
```

tienen contextos y restricciones diferentes.

---

# 213. Shared primitives

Podrán compartir value objects/semantic function IDs cuando sea apropiado.

No necesariamente el mismo root model.

---

# 214. Security

Valores de usuario deberán parameterizarse por defecto.

---

# 215. Identifier input

Los identifiers deberán validarse estructuralmente.

---

# 216. Dynamic identifiers

No podrán convertirse en parameters.

Deberán pasar por:

```text
Identifier Validation
→ Dialect Quoting
```

---

# 217. Raw security

Raw APIs deberán:

- ser explícitas;
- evitar interpolación;
- soportar bindings;
- redaction;
- diagnostics.

---

# 218. Function security

Un FunctionId dinámico proporcionado por usuario no deberá convertirse directamente en SQL function name.

---

# 219. Correct

```text
FunctionId
→ Registered Function Descriptor
→ Compiler Strategy
```

---

# 220. Unknown functions

La API avanzada podrá soportar funciones nativas desconocidas mediante un escape hatch explícito.

---

# 221. NativeFunctionExpression

Si se ofrece:

```text
NativeFunctionExpression
```

deberá ser:

```text
PLATFORM_SPECIFIC
```

y no fingir portabilidad.

---

# 222. Expression depth

Consultas maliciosas o generadas automáticamente podrían producir árboles extremadamente profundos.

---

# 223. Resource governance

VoltStack podrá imponer:

```text
maxExpressionDepth
maxExpressionNodes
maxFunctionArguments
maxTupleSize
maxCaseBranches
```

---

# 224. Limits

Deberán ser configurables mediante Resource Governance, no hardcoded arbitrariamente.

---

# 225. Traversal safety

El traverser deberá poder:

- detectar profundidad;
- controlar presupuesto;
- evitar stack overflow;
- cancelar procesamiento.

---

# 226. Expression complexity

Podrá calcularse:

```text
ExpressionComplexity
```

para diagnostics/resource governance.

---

# 227. Complexity ≠ database cost

No confundir:

```text
AST complexity
```

con:

```text
database execution cost
```

---

# 228. Fingerprinting

Una expression deberá contribuir deterministicamente al Query Fingerprint.

---

# 229. Example

```text
ADD(column(price), parameter(p1))
```

---

# 230. Shape fingerprint

Puede normalizar:

```text
parameter IDs
```

sin incluir runtime values.

---

# 231. Function fingerprint

Debe incluir:

```text
FunctionId
+
arguments
+
semantic options
```

---

# 232. Cast fingerprint

Debe incluir:

```text
target semantic type
```

---

# 233. CASE fingerprint

Debe preservar:

- branch order;
- conditions;
- result expressions;
- ELSE.

---

# 234. Volatile expressions

Pueden seguir teniendo structural fingerprint.

Pero su presencia podrá afectar:

```text
optimization
cacheability
```

---

# 235. Query result cache

La presencia de una expresión volátil puede volver una consulta:

```text
NOT_RESULT_CACHEABLE
```

dependiendo de política.

---

# 236. Compiled query cache

No necesariamente.

Una consulta con:

```text
CURRENT_TIMESTAMP
```

puede seguir teniendo SQL compilado cacheable.

---

# 237. Important distinction

```text
Compiled Query Cacheability
≠
Result Cacheability
```

---

# 238. Expression diagnostics

VoltStack deberá poder explicar:

```text
Expression
Resolved Type
Nullability
Volatility
Portability
Capabilities
Function Resolution
```

---

# 239. Example diagnostic

```text
Expression:
  Function: json.extract_scalar

Arguments:
  1. users.profile : JsonType
  2. $.name        : JsonPath

Resolved Type:
  StringType

Portability:
  PORTABLE_WITH_CAPABILITY

Required Capability:
  query.json.extract_scalar
```

---

# 240. Debug tree

```text
ArithmeticExpression[MULTIPLY]
├── ColumnReference[price]
│   └── type: Decimal(12,2)
└── ColumnReference[quantity]
    └── type: Integer

Resolved:
└── Decimal(...)
```

Las annotations provienen del Semantic Model, no del node.

---

# 241. Error diagnostics

Errores deberán mostrar:

- node kind;
- expression path;
- expected type;
- actual type;
- function/operator;
- capability;
- source location si está disponible.

---

# 242. No secret values

Diagnostics no deberán incluir runtime parameter values por defecto.

---

# 243. Expression exceptions

Jerarquía conceptual:

```text
QueryExpressionException
├── InvalidExpressionException
├── UnknownFunctionException
├── InvalidFunctionArgumentException
├── AmbiguousFunctionException
├── InvalidOperatorException
├── ExpressionTypeMismatchException
├── InvalidCastException
├── UnsupportedExpressionException
├── ExpressionDepthExceededException
└── RawExpressionException
```

---

# 244. Construction vs semantic errors

Distinguir:

```text
Malformed expression structure
```

de:

```text
semantically invalid expression
```

---

# 245. Testing

El sistema deberá tener:

```text
Unit Tests
Structural Tests
Semantic Tests
Type Inference Tests
Capability Tests
Optimizer Tests
Compiler Conformance Tests
Cross-platform Tests
Property Tests
Persistent Runtime Tests
```

---

# 246. Unit tests

Por node:

```text
construction
immutability
children
fingerprint
diagnostics
```

---

# 247. Function tests

Por función:

```text
signature
overload
return type
nullability
volatility
capabilities
target compilation
```

---

# 248. Cross-platform expression tests

Ejemplo:

```text
string.concat
```

deberá probarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 249. Semantic equivalence tests

No sólo comparar SQL generado.

También verificar resultados reales cuando sea posible.

---

# 250. Property testing

Especialmente útil para:

- expression tree transformations;
- fingerprint determinism;
- immutability;
- normalization idempotence.

---

# 251. Persistent runtime tests

Reutilizar Expression descriptors/registries entre miles de requests y comprobar ausencia de state leakage.

---

# 252. Concurrency tests

Especialmente para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

y cualquier runtime con operaciones concurrentes.

---

# 253. Application-scoped components

Podrán ser:

```text
FunctionRegistry
ExpressionSemanticHandlerRegistry
ExpressionTypeRuleRegistry
immutable descriptors
```

si están congelados.

---

# 254. Request/operation scoped

Deberán ser:

```text
BindingSet
SemanticScope
ExpressionSemanticInfo
QueryContext
CompilationContext
```

---

# 255. No current expression singleton

Nunca:

```php
ExpressionContext::current();
```

mediante estado global mutable.

---

# 256. Suggested namespaces

```text
VoltStack\Quantum\Database\Query\Expression\
```

---

# 257. Proposed structure

```text
Query/
├── Ast/
│   └── Node/
│       └── Expression/
│           ├── ColumnReferenceNode.php
│           ├── ParameterExpressionNode.php
│           ├── LiteralExpressionNode.php
│           ├── FunctionCallNode.php
│           ├── AggregateExpressionNode.php
│           ├── ArithmeticExpressionNode.php
│           ├── UnaryExpressionNode.php
│           ├── CaseExpressionNode.php
│           ├── CastExpressionNode.php
│           ├── SubqueryExpressionNode.php
│           ├── TupleExpressionNode.php
│           ├── WindowFunctionNode.php
│           ├── RawExpressionNode.php
│           └── ExtensionExpressionNode.php
│
└── Expression/
    ├── Contract/
    │   ├── ExpressionSemanticHandler.php
    │   ├── ExpressionTypeRule.php
    │   └── ExpressionExtension.php
    │
    ├── Function/
    │   ├── FunctionId.php
    │   ├── FunctionDescriptor.php
    │   ├── FunctionSignature.php
    │   ├── FunctionRegistry.php
    │   ├── FunctionResolver.php
    │   ├── FunctionVolatility.php
    │   └── Core/
    │
    ├── Operator/
    │   ├── ArithmeticOperator.php
    │   ├── UnaryOperator.php
    │   ├── OperatorDescriptor.php
    │   └── OperatorRegistry.php
    │
    ├── Conditional/
    │   ├── CaseMode.php
    │   └── CaseTypeResolver.php
    │
    ├── DateTime/
    │   ├── DateTimeFunction.php
    │   ├── DateTimeUnit.php
    │   └── IntervalValue.php
    │
    ├── Json/
    │   ├── JsonFunction.php
    │   ├── JsonPath.php
    │   └── JsonPathSegment.php
    │
    ├── Window/
    │   ├── WindowFunction.php
    │   ├── WindowSpecification.php
    │   └── WindowFrame.php
    │
    ├── Semantic/
    │   ├── ExpressionSemanticInfo.php
    │   ├── ExpressionSemanticAnalyzer.php
    │   ├── ExpressionDependencySet.php
    │   ├── ExpressionConstantness.php
    │   └── ExpressionPortability.php
    │
    ├── Type/
    │   ├── ExpressionTypeResolver.php
    │   ├── ExpressionTypeRuleRegistry.php
    │   └── TypePromotionRule.php
    │
    ├── Capability/
    │   └── ExpressionCapabilityResolver.php
    │
    ├── Builder/
    │   └── ExpressionBuilder.php
    │
    ├── Validation/
    │   └── ExpressionValidator.php
    │
    ├── Diagnostics/
    │   └── ExpressionDiagnosticRenderer.php
    │
    ├── Extension/
    │   └── ExpressionExtensionRegistry.php
    │
    └── Exception/
```

---

# 258. Dependency direction

```text
Expression AST
      │
      ▼
Expression Semantic Services
      │
      ▼
Query Semantic Engine
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
Expression AST
      │
      ▼
Compiler
      │
      ▼
mutate AST
```

---

# 259. Expression System dependencies

Puede depender de:

```text
Query AST contracts
Query Type abstractions
Identifiers
Parameters
immutable Support value objects
```

---

# 260. Forbidden dependencies

No deberá depender directamente de:

```text
PDO
Driver
Connection
ConnectionPool
EntityManager
UnitOfWork
HTTP Request
FrankenPHP API
RoadRunner API
OpenSwoole API
```

---

# 261. DB-EXPR-001

Toda expresión core será un `ExpressionNode`.

---

# 262. DB-EXPR-002

Toda expresión publicada será inmutable.

---

# 263. DB-EXPR-003

Una expresión nunca generará SQL.

---

# 264. DB-EXPR-004

Una expresión nunca ejecutará SQL.

---

# 265. DB-EXPR-005

Una expresión nunca accederá a Connection o Driver.

---

# 266. DB-EXPR-006

Los nombres de funciones core serán semánticos.

---

# 267. DB-EXPR-007

El FunctionId no deberá asumirse igual al nombre SQL.

---

# 268. DB-EXPR-008

Los operadores representarán semántica, no tokens SQL.

---

# 269. DB-EXPR-009

Los valores runtime se parameterizarán por defecto.

---

# 270. DB-EXPR-010

Los ParameterIds serán independientes de placeholders SQL.

---

# 271. DB-EXPR-011

Los identifiers permanecerán sin quoting de Dialect.

---

# 272. DB-EXPR-012

La resolución de columnas será responsabilidad del Semantic Engine.

---

# 273. DB-EXPR-013

Los nodes no contendrán Schema objects resueltos.

---

# 274. DB-EXPR-014

Los tipos resueltos se almacenarán fuera del AST.

---

# 275. DB-EXPR-015

La nullability resuelta se almacenará fuera del AST.

---

# 276. DB-EXPR-016

La volatility se derivará semánticamente.

---

# 277. DB-EXPR-017

La igualdad estructural no implicará igualdad de evaluación.

---

# 278. DB-EXPR-018

Las expresiones volátiles limitarán optimizaciones que requieran determinismo.

---

# 279. DB-EXPR-019

El Optimizer nunca asumirá que una función desconocida es determinista.

---

# 280. DB-EXPR-020

Una función desconocida tendrá semántica conservadora.

---

# 281. DB-EXPR-021

CASE preservará el orden de sus branches.

---

# 282. DB-EXPR-022

Los casts core utilizarán tipos semánticos.

---

# 283. DB-EXPR-023

Los casts nativos deberán declararse explícitamente como target-specific.

---

# 284. DB-EXPR-024

SubqueryExpressionNode contendrá un Query AST, no SQL.

---

# 285. DB-EXPR-025

Las correlated references se resolverán mediante Semantic Scope.

---

# 286. DB-EXPR-026

No existirán parent-query pointers mutables.

---

# 287. DB-EXPR-027

TupleExpression no será equivalente a un array type.

---

# 288. DB-EXPR-028

Las operaciones JSON core serán semánticas y target-neutral.

---

# 289. DB-EXPR-029

Los operadores JSON nativos no aparecerán en core AST.

---

# 290. DB-EXPR-030

Las operaciones date/time core serán semánticas.

---

# 291. DB-EXPR-031

Application clock values se parameterizarán normalmente.

---

# 292. DB-EXPR-032

Server clock expressions serán explícitas.

---

# 293. DB-EXPR-033

Window functions se validarán mediante capabilities.

---

# 294. DB-EXPR-034

RawExpression será siempre identificable como escape hatch.

---

# 295. DB-EXPR-035

RawExpression no habilitará interpolación insegura.

---

# 296. DB-EXPR-036

RawExpression deberá soportar bindings seguros cuando aplique.

---

# 297. DB-EXPR-037

Una raw expression podrá limitar Semantic Analysis y Optimizer.

---

# 298. DB-EXPR-038

ExtensionExpression deberá utilizar NodeTypeId namespaced.

---

# 299. DB-EXPR-039

Los extension handlers se registrarán antes del registry freeze.

---

# 300. DB-EXPR-040

No se permitirá silent handler replacement.

---

# 301. DB-EXPR-041

La ausencia de capability requerida deberá producir fallo explícito.

---

# 302. DB-EXPR-042

Las emulaciones sólo serán válidas cuando preserven semántica aceptable.

---

# 303. DB-EXPR-043

La expresión no decidirá por sí misma si utilizar estrategia nativa o emulada.

---

# 304. DB-EXPR-044

La selección de estrategia pertenece al Planner.

---

# 305. DB-EXPR-045

La sintaxis pertenece al Compiler/Dialect.

---

# 306. DB-EXPR-046

El Type System no utilizará nombres de vendor para resolver tipos portables.

---

# 307. DB-EXPR-047

Las reglas de type inference deberán ser deterministas para el mismo semantic context.

---

# 308. DB-EXPR-048

Los Parameter TypeHints no serán tipos de PDO.

---

# 309. DB-EXPR-049

Los fingerprints no incluirán runtime parameter values.

---

# 310. DB-EXPR-050

Los fingerprints sí incluirán estructura, FunctionIds, operadores y opciones semánticas.

---

# 311. DB-EXPR-051

Las expresiones inmutables podrán compartirse estructuralmente.

---

# 312. DB-EXPR-052

Los SemanticInfo no podrán almacenarse globalmente por object identity sin scope.

---

# 313. DB-EXPR-053

Las registries application-scoped deberán congelarse antes de ejecución normal.

---

# 314. DB-EXPR-054

Las registries application-scoped deberán ser concurrency-safe.

---

# 315. DB-EXPR-055

BindingSet será independiente del Expression AST.

---

# 316. DB-EXPR-056

ExpressionContext será externo al node.

---

# 317. DB-EXPR-057

Las restricciones contextuales serán validadas por Semantic Validation.

---

# 318. DB-EXPR-058

No se utilizarán closures runtime como expressions core.

---

# 319. DB-EXPR-059

No se almacenarán servicios mutables dentro de expressions.

---

# 320. DB-EXPR-060

El Expression System será independiente del ORM.

---

# 321. DB-EXPR-061

ORM y Query Builder utilizarán el mismo Expression AST.

---

# 322. DB-EXPR-062

Persistence Engine generará expressions, nunca SQL directo.

---

# 323. DB-EXPR-063

Schema expressions no se mezclarán automáticamente con Query expressions.

---

# 324. DB-EXPR-064

Toda transformación de expression deberá producir nodes inmutables.

---

# 325. DB-EXPR-065

Las transformaciones podrán reutilizar subtrees no modificados.

---

# 326. DB-EXPR-066

Constant folding pertenecerá al Optimizer.

---

# 327. DB-EXPR-067

Normalización y optimización permanecerán conceptualmente separadas.

---

# 328. DB-EXPR-068

Las reglas algebraicas deberán respetar precisión, overflow y target semantics.

---

# 329. DB-EXPR-069

No se aplicarán reglas de igualdad PHP directamente a SQL expressions.

---

# 330. DB-EXPR-070

La lógica NULL seguirá semántica de base de datos, no PHP.

---

# 331. DB-EXPR-071

Las expresiones podrán declarar o derivar Resource Complexity.

---

# 332. DB-EXPR-072

El traverser deberá protegerse contra árboles patológicamente profundos.

---

# 333. DB-EXPR-073

Los límites de complejidad serán gobernados por Resource Governance.

---

# 334. DB-EXPR-074

Compiled Query Cacheability y Result Cacheability serán conceptos independientes.

---

# 335. DB-EXPR-075

Una expresión volátil no implica automáticamente que el SQL compilado no pueda cachearse.

---

# 336. DB-EXPR-076

Una expresión volátil podrá impedir Result Cache según política.

---

# 337. DB-EXPR-077

Diagnostics no expondrán parameter values sensibles por defecto.

---

# 338. DB-EXPR-078

Las funciones nativas no registradas deberán utilizar una API explícita de escape hatch.

---

# 339. DB-EXPR-079

El core portable no dependerá de MySQL, MariaDB, PostgreSQL o SQLite classes.

---

# 340. DB-EXPR-080

El Expression System será consumible por cualquier target que implemente las capabilities y estrategias requeridas.

---

# 341. Anti-pattern — String Expression

Incorrecto:

```php
DB::expr('price * quantity');
```

como representación interna principal.

---

# 342. Correcto

```text
ArithmeticExpressionNode
├── MULTIPLY
├── ColumnReference(price)
└── ColumnReference(quantity)
```

---

# 343. Anti-pattern — SQL function names everywhere

Incorrecto:

```php
new FunctionCallNode('JSON_EXTRACT');
```

en el core portable.

---

# 344. Correcto

```text
FunctionCallNode
└── FunctionId(json.extract)
```

---

# 345. Anti-pattern — vendor checks

Incorrecto:

```php
if ($platform->name() === 'postgresql') {
    return '||';
}
```

dentro del Expression System.

---

# 346. Correcto

```text
string.concat
      │
      ▼
Capability/Planner
      │
      ▼
Target Compiler
```

---

# 347. Anti-pattern — resolved type mutation

Incorrecto:

```php
$expression->type = $resolvedType;
```

---

# 348. Correcto

```text
AstOccurrenceId
      │
      ▼
ExpressionSemanticInfo
      │
      └── ResolvedType
```

---

# 349. Anti-pattern — user value literal

Incorrecto:

```text
StringLiteral(userInput)
```

por defecto.

---

# 350. Correcto

```text
ParameterExpressionNode(p1)

BindingSet:
p1 → userInput
```

---

# 351. Anti-pattern — CURRENT_TIMESTAMP evaluated in PHP

Incorrecto cuando se desea tiempo del servidor:

```php
new ParameterExpressionNode(new DateTimeImmutable());
```

---

# 352. Correcto

```text
ServerCurrentTimestampExpression
```

---

# 353. Anti-pattern — universal function registry

Incorrecto:

```text
OneRegistry
├── AST
├── functions
├── compilers
├── drivers
├── ORM
└── types
```

---

# 354. Correcto

Registries especializados con fronteras claras.

---

# 355. Anti-pattern — PostgreSQL AST

Incorrecto:

```text
PostgresJsonbContainsNode
```

en el core cuando existe semántica portable.

---

# 356. Correcto

```text
JsonContains
+
Capability Requirement
+
PostgreSQL Strategy
```

---

# 357. Anti-pattern — fake portability

No convertir una operación nativa en portable simplemente cambiando su nombre.

---

# 358. Portability requirement

La semántica deberá ser suficientemente equivalente entre targets.

---

# 359. Expression pipeline completo

```text
Developer API
      │
      ▼
ExpressionBuilder
      │
      ▼
ExpressionNode
      │
      ▼
AST Structural Validation
      │
      ▼
Semantic Scope Resolution
      │
      ├── Columns
      ├── Parameters
      └── Functions
      │
      ▼
Type Inference
      │
      ▼
Semantic Properties
      │
      ├── Type
      ├── Nullability
      ├── Volatility
      ├── Constantness
      ├── Dependencies
      └── Capability Requirements
      │
      ▼
Optimizer
      │
      ▼
Planner
      │
      ├── Native
      ├── Emulated
      └── Unsupported
      │
      ▼
Compiler
      │
      ▼
Dialect
      │
      ▼
Compiled Query
```

---

# 360. Example completo

API:

```php
DB::table('orders')
    ->select([
        'id',
        DB::expr()
            ->multiply('price', 'quantity')
            ->as('total'),
    ])
    ->where('status', 'paid');
```

Query construction:

```text
Parameter:
p1 → "paid"
```

AST:

```text
SelectNode
│
├── Projection
│   ├── ColumnReference(id)
│   │
│   └── ExpressionProjection
│       ├── ArithmeticExpression[MULTIPLY]
│       │   ├── ColumnReference(price)
│       │   └── ColumnReference(quantity)
│       └── Alias(total)
│
├── Source
│   └── TableSource(orders)
│
└── Predicate
    └── ComparisonPredicate[EQUAL]
        ├── ColumnReference(status)
        └── ParameterExpression(p1)
```

Semantic result:

```text
price
→ Decimal(12,2)

quantity
→ Integer

price * quantity
→ Decimal(...)

status
→ String

p1
→ inferred String
```

La expresión permanece sin modificar.

---

# 361. Function model example

Consulta conceptual:

```text
LOWER(email)
```

AST:

```text
FunctionCallNode
├── FunctionId: string.lower
└── ColumnReference(email)
```

Semantic:

```text
Input:
StringType

Output:
StringType

Volatility:
IMMUTABLE

Nullability:
inherits argument

Capability:
query.function.string.lower
```

Compilation:

```text
Target Compiler
      │
      ▼
target syntax
```

---

# 362. JSON example

API conceptual:

```php
expr()->jsonExtractScalar(
    expr()->column('profile'),
    '$.name',
);
```

AST:

```text
JsonExtractScalarExpression
├── ColumnReference(profile)
└── JsonPath
    └── Property(name)
```

Semantic:

```text
profile → JsonType
result  → StringType
```

Requirements:

```text
query.json.extract_scalar
```

Planner:

```text
MySQL      → native strategy
MariaDB    → native strategy
PostgreSQL → native semantic strategy
SQLite     → native if JSON capability available
```

El node no contiene ninguna de esas decisiones.

---

# 363. CASE example

```text
CaseExpression
├── WHEN
│   ├── Predicate(status = p1)
│   └── Literal/Parameter(...)
├── WHEN
│   ├── Predicate(status = p2)
│   └── Literal/Parameter(...)
└── ELSE
    └── ...
```

El Type System determinará un tipo común para todos los resultados.

---

# 364. Window example

```text
WindowFunctionNode
├── Function: window.row_number
└── WindowSpecification
    ├── Partition
    │   └── customer_id
    ├── Order
    │   └── created_at DESC
    └── Frame
        └── ...
```

---

# 365. Modelo de seguridad

```text
Application Value
       │
       ▼
Parameter
       │
       ▼
Expression AST
       │
       ▼
Binding Plan
       │
       ▼
Driver Binding
```

Nunca:

```text
Application Value
       │
       ▼
String Concatenation
       │
       ▼
SQL
```

---

# 366. Modelo de portabilidad

```text
Semantic Expression
        │
        ▼
Capability Requirement
        │
        ▼
Strategy Resolver
        │
   ┌────┼─────┐
   ▼    ▼     ▼
Native Emulated Unsupported
   │    │
   └────┴──────► Planner
                    │
                    ▼
                 Compiler
```

---

# 367. Modelo de tipos

```text
Expression AST
      │
      ▼
Semantic Resolver
      │
      ▼
Child Types
      │
      ▼
Expression Type Rule
      │
      ▼
ResolvedQueryType
      │
      ├── nullability
      ├── conversion requirements
      └── binding implications
```

---

# 368. Modelo de optimización

```text
Expression
    │
    ▼
SemanticInfo
    │
    ├── type
    ├── volatility
    ├── nullability
    ├── constantness
    └── dependencies
    │
    ▼
Safe Rewrite Rules
    │
    ▼
New Immutable Expression
```

---

# 369. Principio central de extensibilidad

Una extensión no deberá pedir:

> “¿Cómo inserto este string SQL?”

Deberá declarar:

> “¿Qué nueva semántica de expresión estoy agregando y qué necesita cada fase para comprenderla?”

---

# 370. Definition of done para una expresión core

Una expresión core estará completa cuando tenga:

```text
AST representation
Structural validation
Semantic interpretation
Type rules
Nullability rules
Volatility/determinism rules
Capability requirements
Fingerprint behavior
Diagnostic representation
Planner strategy
Compiler support
Cross-platform tests
```

según corresponda.

---

# 371. Orden recomendado de implementación

## Fase 1 — Core expressions

```text
ColumnReference
Parameter
Literal
Arithmetic
Unary
FunctionCall
Aggregate
CASE
CAST
Subquery
Tuple
```

## Fase 2 — Semantic function library

```text
String
Numeric
Conditional
Date/Time
```

## Fase 3 — Advanced expressions

```text
JSON
Window
Advanced aggregates
```

## Fase 4 — Extensions

```text
Platform-specific
Package-defined
Advanced database features
```

---

# 372. Core portable expression set

V1 deberá priorizar un conjunto pequeño pero sólido:

```text
Column
Parameter
NULL
Boolean
Numeric
Arithmetic
String functions
Aggregate
CASE
COALESCE
CAST
Date/time basics
Subquery
Tuple
```

antes de intentar cubrir cada función existente de cada motor.

---

# 373. Feature richness strategy

VoltStack no deberá limitarse al mínimo común denominador.

La estrategia será:

```text
Portable Semantic Core
+
Capability-aware Advanced Features
+
Explicit Platform Extensions
+
Raw Escape Hatch
```

---

# 374. Relación con documentos anteriores

```text
24_DATABASE_QUERY_MODEL.md
        │
        ▼
25_DATABASE_QUERY_AST_SYSTEM.md
        │
        ▼
26_DATABASE_QUERY_AST_NODE_MODEL.md
        │
        ▼
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

Este documento especializa:

```text
ExpressionNode
```

dentro del AST.

---

# 375. Relación con el siguiente documento

El Expression System produce valores.

El Predicate System produce condiciones lógicas.

Por ello:

```text
Expression System
        │
        ├─────────────┐
        │             │
        ▼             ▼
     Values       Operands
                      │
                      ▼
              Predicate System
```

---

# 376. Próximo documento

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

deberá definir:

```text
Comparison
AND
OR
NOT
NULL predicates
BETWEEN
IN
EXISTS
LIKE
pattern matching
regex
JSON predicates
three-valued logic
predicate normalization
predicate simplification
predicate dependencies
predicate capability requirements
predicate extension model
```

---

# 377. Fórmula arquitectónica

```text
Expression
=
Immutable Semantic Value Computation
```

Mientras:

```text
Resolved Expression
=
Expression
+
Semantic Scope
+
Resolved Symbols
+
Resolved Types
+
Semantic Properties
+
Capability Requirements
```

Y:

```text
Compiled Expression
=
Resolved Expression
+
Plan Strategy
+
Target Dialect
+
Platform Capabilities
```

---

# 378. Regla maestra final

> **El Query Expression System de VoltStack representará operaciones de valor mediante un AST semántico, tipado e inmutable, completamente separado de la sintaxis SQL, del motor de base de datos y del runtime de ejecución.**

En resumen:

```text
Expression
    │
    ├── describes value intent
    ├── contains operands
    ├── remains immutable
    ├── remains target-neutral
    └── can be analyzed

Expression
    │
    ├── does NOT generate SQL
    ├── does NOT execute
    ├── does NOT know PDO
    ├── does NOT know Connection
    ├── does NOT know current Platform
    ├── does NOT own resolved types
    └── does NOT own runtime values
```

---

# 379. Decisión arquitectónica final

La implementación oficial deberá seguir:

```text
Semantic Expressions
        +
Immutable AST
        +
External Semantic Metadata
        +
Typed Function Registry
        +
Typed Operator Model
        +
Capability Requirements
        +
Planner-selected Strategies
        +
Dialect Compilation
```

De esta forma VoltStack podrá ofrecer una API sencilla al estilo Laravel mientras mantiene internamente una arquitectura preparada para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

y futuros motores sin contaminar el Query Builder o el AST con SQL específico del proveedor.

---

# 380. Resultado

Con este documento queda definida la frontera:

```text
Developer API
      │
      ▼
ExpressionBuilder
      │
      ▼
Expression AST
      │
      ▼
Semantic Expression Model
      │
      ▼
Type / Function / Capability Resolution
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

estableciendo la base necesaria para construir el siguiente subsistema:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

sin mezclar valores, condiciones, SQL, ejecución ni comportamiento específico del motor.