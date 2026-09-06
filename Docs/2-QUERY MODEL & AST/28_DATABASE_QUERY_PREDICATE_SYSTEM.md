# 28_DATABASE_QUERY_PREDICATE_SYSTEM.md

# VoltStack Quantum Database
## Query Predicate System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 28 — Query Predicate System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / AST / Predicate System  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el sistema oficial de predicados de consulta de:

```text
VoltStack/Quantum/Database
```

El Predicate System representa condiciones lógicas dentro del Query AST.

Ejemplos conceptuales:

```text
users.active = true

users.age >= 18

users.deleted_at IS NULL

users.id IN (...)

users.created_at BETWEEN start AND end

EXISTS (subquery)

name LIKE pattern

A AND B

A OR B

NOT A
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
      ├── Expression System
      │
      └── Predicate System
                │
                ▼
        Semantic Analysis
                │
                ├── Symbol Resolution
                ├── Type Analysis
                ├── Predicate Validation
                ├── Nullability Analysis
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

El Predicate System deberá ser:

```text
Semantic
+
Immutable
+
Composable
+
Typed
+
Target-neutral
+
Three-valued-logic aware
+
Optimizable
+
Capability-aware
+
Extensible
```

No será un sistema basado principalmente en fragmentos SQL.

---

# 3. Regla maestra

> Un predicado representa una condición lógica de consulta; nunca representa directamente la sintaxis SQL utilizada para expresar esa condición.

Por tanto:

```text
Predicate
=
Logical Intent
+
Operands
+
Predicate Structure
```

No:

```text
Predicate
=
SQL WHERE Fragment
```

---

# 4. Expression ≠ Predicate

VoltStack mantendrá como frontera arquitectónica:

```text
ExpressionNode
≠
PredicateNode
```

Una expresión representa:

```text
value computation
```

Un predicado representa:

```text
logical condition
```

Ejemplo:

```text
price * quantity
```

es una expresión.

Mientras:

```text
price * quantity > 100
```

es un predicado cuyos operands incluyen expresiones.

---

# 5. Relación fundamental

```text
ExpressionNode
      │
      ▼
    value
      │
      ▼
PredicateNode
      │
      ▼
 logical condition
```

---

# 6. Razón de la separación

La separación permite:

- validación contextual;
- type checking;
- three-valued logic;
- simplificación lógica;
- predicate pushdown;
- join analysis;
- capability detection;
- seguridad;
- diagnostics;
- optimización.

---

# 7. Contrato base

Conceptualmente:

```php
interface PredicateNode extends QueryAstNode
{
}
```

El contrato deberá permanecer mínimo.

Nunca:

```php
interface PredicateNode
{
    public function toSql(): string;

    public function evaluate(array $row): bool;

    public function platform(): Platform;
}
```

---

# 8. Arquitectura conceptual

```text
PredicateNode
│
├── ComparisonPredicate
├── LogicalPredicate
├── NotPredicate
├── NullPredicate
├── BetweenPredicate
├── InPredicate
├── ExistsPredicate
├── PatternPredicate
├── RegexPredicate
├── DistinctnessPredicate
├── BooleanTestPredicate
├── JsonPredicate
├── TuplePredicate
├── RawPredicate
└── ExtensionPredicate
```

---

# 9. Taxonomía inicial

El core podrá incluir conceptualmente:

```text
ComparisonPredicateNode
AndPredicateNode
OrPredicateNode
NotPredicateNode
NullPredicateNode
BetweenPredicateNode
InPredicateNode
ExistsPredicateNode
LikePredicateNode
RegexPredicateNode
DistinctnessPredicateNode
BooleanTestPredicateNode
JsonPredicateNode
RawPredicateNode
ExtensionPredicateNode
```

No todas las categorías necesitan interfaces PHP adicionales.

---

# 10. Predicate tree

Ejemplo:

```text
active = true
AND
(
    age >= 18
    OR
    verified_at IS NOT NULL
)
```

AST:

```text
AndPredicate
├── ComparisonPredicate
│   ├── Column(active)
│   ├── EQUAL
│   └── Parameter(true)
│
└── OrPredicate
    ├── ComparisonPredicate
    │   ├── Column(age)
    │   ├── GREATER_THAN_OR_EQUAL
    │   └── Parameter(18)
    │
    └── NullPredicate
        ├── Column(verified_at)
        └── IS_NOT_NULL
```

---

# 11. No parentheses nodes by default

La estructura del árbol representa precedencia.

No será necesario almacenar:

```text
(
)
```

como nodes sintácticos ordinarios.

---

# 12. ComparisonPredicateNode

Representa una comparación binaria.

```text
ComparisonPredicateNode
├── left: ExpressionNode
├── operator: ComparisonOperator
└── right: ExpressionNode
```

---

# 13. ComparisonOperator

Inicialmente:

```text
EQUAL
NOT_EQUAL
LESS_THAN
LESS_THAN_OR_EQUAL
GREATER_THAN
GREATER_THAN_OR_EQUAL
```

---

# 14. Operator semantic intent

El operador representa:

```text
semantic comparison
```

No representa directamente:

```text
=
<>
!=
<
<=
>
>=
```

---

# 15. NOT_EQUAL

El Dialect podrá elegir la sintaxis apropiada.

El AST no deberá decidir entre:

```text
<>
```

y:

```text
!=
```

---

# 16. Operand types

Semantic Analysis deberá validar la compatibilidad entre:

```text
left type
right type
operator
```

---

# 17. Example

```text
created_at > :date
```

podrá producir:

```text
ComparisonPredicate
├── ColumnReference(created_at)
├── GREATER_THAN
└── ParameterExpression(p1)
```

Posteriormente:

```text
p1
→ DateTimeType
```

puede inferirse.

---

# 18. Parameter inference

Un predicado:

```text
users.id = :p1
```

podrá inferir:

```text
p1 → UserId-compatible type
```

según metadata disponible.

---

# 19. Comparison type compatibility

No deberá asumirse que cualquier par de expressions es comparable.

Ejemplos potencialmente inválidos:

```text
json > datetime

blob < integer

tuple = scalar
```

salvo que exista semántica específica.

---

# 20. Comparison semantic rules

Podrá existir:

```text
ComparisonSemanticResolver
```

responsable de determinar:

- comparability;
- coercion;
- resulting truth domain;
- capabilities;
- portability.

---

# 21. NULL comparison

Una comparación ordinaria con `NULL` no deberá tratarse como igualdad PHP.

Incorrecto:

```text
column = NULL
```

cuando la intención es:

```text
column IS NULL
```

---

# 22. Query Builder normalization

Una API ergonómica podrá transformar:

```php
->where('deleted_at', null)
```

en:

```text
NullPredicateNode
```

en lugar de `ComparisonPredicateNode`.

---

# 23. Explicit SQL semantics

Si una API avanzada necesita representar literalmente una comparación SQL con NULL, deberá hacerlo de forma deliberada y semánticamente explícita.

---

# 24. NullPredicateNode

Representa:

```text
IS NULL
IS NOT NULL
```

---

# 25. Structure

```text
NullPredicateNode
├── expression
└── mode
```

---

# 26. NullPredicateMode

```text
IS_NULL
IS_NOT_NULL
```

---

# 27. Example

```text
NullPredicateNode
├── ColumnReference(deleted_at)
└── IS_NULL
```

---

# 28. Three-valued logic

SQL no utiliza únicamente:

```text
TRUE
FALSE
```

También existe:

```text
UNKNOWN
```

Por tanto:

```text
SQL Truth Domain
=
TRUE
FALSE
UNKNOWN
```

---

# 29. TruthValue

VoltStack deberá modelar conceptualmente:

```text
TruthValue
├── TRUE
├── FALSE
└── UNKNOWN
```

para análisis semántico y optimización.

---

# 30. TruthValue ≠ PHP bool

Nunca deberá suponerse:

```text
SQL predicate result
=
PHP bool
```

durante optimización.

---

# 31. NULL propagation

Ejemplo:

```text
5 = NULL
```

produce semánticamente:

```text
UNKNOWN
```

no:

```text
FALSE
```

---

# 32. WHERE semantics

Aunque un `WHERE` retiene normalmente filas donde el resultado es `TRUE`, el Predicate System no deberá colapsar prematuramente `UNKNOWN` a `FALSE`.

---

# 33. Context matters

La diferencia puede importar en:

- `NOT`;
- `CHECK`;
- joins;
- CASE;
- expression contexts;
- future schema predicates;
- optimizer transformations.

---

# 34. Three-valued AND

Tabla conceptual:

| A | B | A AND B |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| TRUE | UNKNOWN | UNKNOWN |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | FALSE |
| UNKNOWN | TRUE | UNKNOWN |
| UNKNOWN | FALSE | FALSE |
| UNKNOWN | UNKNOWN | UNKNOWN |

---

# 35. Three-valued OR

| A | B | A OR B |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| TRUE | UNKNOWN | TRUE |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | UNKNOWN |
| UNKNOWN | TRUE | TRUE |
| UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN |

---

# 36. Three-valued NOT

| A | NOT A |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

---

# 37. Optimizer consequence

Una regla aparentemente obvia como:

```text
NOT (x = NULL)
```

no puede tratarse utilizando lógica booleana PHP convencional.

---

# 38. AndPredicateNode

Representa una conjunción lógica.

Recomendación:

```text
AndPredicateNode
└── predicates: NonEmptyPredicateList
```

en lugar de limitarse a dos children.

---

# 39. N-ary logical nodes

VoltStack podrá representar:

```text
A AND B AND C AND D
```

como:

```text
AndPredicate
├── A
├── B
├── C
└── D
```

---

# 40. Benefit

Esto facilita:

- normalization;
- simplification;
- pushdown;
- fingerprinting;
- traversal;
- optimizer rules.

---

# 41. OrPredicateNode

De manera equivalente:

```text
OrPredicate
├── A
├── B
└── C
```

---

# 42. Non-empty invariant

Un `AndPredicateNode` u `OrPredicateNode` deberá contener al menos un child.

---

# 43. Empty logical conditions

La API constructora decidirá explícitamente cómo tratar listas vacías.

No crear automáticamente un node ambiguo.

---

# 44. Constant predicates

Podrá existir:

```text
ConstantPredicateNode
├── TRUE
├── FALSE
└── UNKNOWN
```

para normalización/optimización interna.

---

# 45. Public construction

No necesariamente deberá exponerse ampliamente al usuario.

Puede ser principalmente un primitive interno del Query AST.

---

# 46. NotPredicateNode

```text
NotPredicateNode
└── predicate
```

---

# 47. NOT operand

Su child será:

```text
PredicateNode
```

no una expresión arbitraria.

---

# 48. Boolean expression interoperability

Si una plataforma permite expresiones booleanas como predicates, la conversión deberá ser explícita mediante semantic adaptation.

---

# 49. Predicate negation

El Optimizer podrá aplicar reglas como:

```text
NOT(NOT A)
→ A
```

cuando sean seguras.

---

# 50. De Morgan

Potencialmente:

```text
NOT(A AND B)
→ NOT A OR NOT B
```

y:

```text
NOT(A OR B)
→ NOT A AND NOT B
```

pero las transformaciones deberán respetar la semántica ternaria.

---

# 51. BetweenPredicateNode

Representa una condición de rango.

```text
BetweenPredicateNode
├── value
├── lower
├── upper
└── negated
```

---

# 52. Example

```text
age BETWEEN 18 AND 65
```

se representa como:

```text
BetweenPredicate
├── Column(age)
├── Parameter(18)
├── Parameter(65)
└── negated: false
```

---

# 53. NOT BETWEEN

No requiere necesariamente:

```text
NotPredicate(BetweenPredicate)
```

Puede existir:

```text
negated: true
```

si preserva mejor la semántica y compilación.

---

# 54. Structural vs logical negation

La arquitectura deberá decidir consistentemente cuándo una construcción tiene negación intrínseca:

```text
NOT BETWEEN
NOT IN
NOT LIKE
IS NOT NULL
```

y cuándo utiliza `NotPredicateNode`.

---

# 55. Recommendation

Mantener flags/modes específicos cuando SQL y semántica los reconocen como una misma familia estructural.

---

# 56. Between type analysis

Los tres operands deberán ser comparables bajo reglas compatibles.

---

# 57. Symmetric BETWEEN

Algunos motores soportan variantes adicionales.

Podrán modelarse como:

```text
BetweenMode
├── ASYMMETRIC
└── SYMMETRIC
```

si se incorpora la feature.

---

# 58. Capability requirement

`SYMMETRIC` deberá ser capability-aware.

---

# 59. InPredicateNode

Representa membership.

```text
InPredicateNode
├── value
├── source
└── negated
```

---

# 60. IN sources

El source podrá ser:

```text
ExpressionList
Subquery
TupleList
ExtensionSource
```

---

# 61. Expression list

Ejemplo:

```text
id IN (:p1, :p2, :p3)
```

---

# 62. Subquery source

Ejemplo:

```text
user_id IN (
    SELECT ...
)
```

---

# 63. Tuple IN

Ejemplo conceptual:

```text
(a, b) IN ((1, 2), (3, 4))
```

sujeto a capabilities.

---

# 64. IN list values

Los valores runtime deberán parameterizarse normalmente.

---

# 65. Empty IN list

Una lista vacía requiere política explícita.

SQL:

```text
IN ()
```

no es portable ni universalmente válido.

---

# 66. Empty IN normalization

La construcción:

```text
x IN []
```

podrá normalizarse semánticamente a:

```text
FALSE
```

si la API define esa semántica.

Mientras:

```text
x NOT IN []
```

podrá normalizarse a:

```text
TRUE
```

---

# 67. Important caveat

Esta normalización pertenece a Query construction/normalization, no al Dialect.

---

# 68. NULL inside IN

La presencia de NULL en listas `IN` afecta three-valued logic.

Ejemplo:

```text
x IN (1, NULL)
```

no puede simplificarse ingenuamente.

---

# 69. NOT IN and NULL

Particularmente:

```text
x NOT IN (... NULL ...)
```

requiere gran cuidado semántico.

---

# 70. Optimizer rule

VoltStack nunca deberá transformar `NOT IN` ignorando posibles NULLs.

---

# 71. Large IN lists

El Planner podrá transformar listas grandes mediante estrategias como:

```text
chunking
temporary relation
array parameter
derived values
platform-specific strategy
```

si capabilities lo permiten.

---

# 72. Predicate AST remains semantic

El AST seguirá representando:

```text
membership
```

aunque el Planner elija otra estrategia física.

---

# 73. ExistsPredicateNode

Representa:

```text
EXISTS(query)
NOT EXISTS(query)
```

---

# 74. Structure

```text
ExistsPredicateNode
├── subquery
└── negated
```

---

# 75. EXISTS ignores projected value semantics

El valor exacto proyectado por el subquery no deberá interpretarse como resultado escalar ordinario.

---

# 76. Semantic validation

El subquery deberá ser válido como existence query.

---

# 77. Correlation

EXISTS es un caso importante de correlated subqueries.

Ejemplo:

```text
EXISTS
└── Subquery
    └── orders.user_id = users.id
```

---

# 78. Correlated symbol resolution

La referencia externa:

```text
users.id
```

será resuelta por Semantic Scope.

---

# 79. No parent pointers

El subquery no deberá almacenar un pointer mutable a su query padre.

---

# 80. LikePredicateNode

Representa pattern matching semántico.

```text
LikePredicateNode
├── value
├── pattern
├── escape?
├── caseSensitivity
└── negated
```

---

# 81. Pattern values

El pattern deberá parameterizarse cuando provenga de runtime.

---

# 82. Example

```php
->whereLike('name', '%john%')
```

podrá producir:

```text
LikePredicate
├── Column(name)
├── Parameter(p1)
├── caseSensitivity: DEFAULT
└── negated: false
```

---

# 83. LIKE wildcard escaping

Debe distinguirse entre:

```text
SQL pattern syntax
```

y:

```text
literal user search text
```

---

# 84. Search convenience API

Una API como:

```php
->whereContains('name', $text)
```

deberá escapar correctamente:

```text
%
_
escape character
```

antes de construir el pattern binding.

---

# 85. whereLike vs whereContains

No deberán ser exactamente la misma intención.

```text
whereLike()
```

puede aceptar pattern.

```text
whereContains()
```

acepta texto literal y construye un pattern seguro.

---

# 86. Pattern mode

Podrá existir una abstracción:

```text
PatternMode
├── RAW_PATTERN
├── CONTAINS
├── STARTS_WITH
└── ENDS_WITH
```

---

# 87. Case sensitivity

No asumir que todos los motores manejan `LIKE` igual.

Modelo:

```text
CaseSensitivity
├── DEFAULT
├── SENSITIVE
└── INSENSITIVE
```

---

# 88. Planner responsibility

El Planner determinará si:

```text
INSENSITIVE
```

es:

```text
native
collation-based
function-based
emulated
unsupported
```

---

# 89. ILIKE

El AST portable no deberá contener:

```text
ILIKE
```

como operador core.

---

# 90. PostgreSQL strategy

Puede compilar una intención:

```text
case-insensitive pattern match
```

mediante `ILIKE` si es apropiado.

---

# 91. Other targets

Pueden utilizar:

- collation;
- normalized operands;
- native semantics;
- alternative strategy.

---

# 92. Escape character

Deberá modelarse estructuralmente.

No concatenarse arbitrariamente en SQL.

---

# 93. RegexPredicateNode

Representa matching mediante expresión regular.

```text
RegexPredicateNode
├── value
├── pattern
├── options
└── negated
```

---

# 94. Regex portability

Regex deberá considerarse:

```text
PORTABLE_WITH_CAPABILITY
```

o incluso target-specific según las garantías semánticas ofrecidas.

---

# 95. Regex semantics differ

Los motores pueden diferir en:

- regex engine;
- flags;
- case sensitivity;
- Unicode;
- multiline;
- escaping;
- operator syntax.

---

# 96. No fake regex portability

VoltStack no deberá declarar dos implementaciones equivalentes si sus semánticas no lo son suficientemente.

---

# 97. Regex options

Podrá existir:

```text
RegexOptions
├── caseSensitivity
├── multiline
├── dotAll
└── unicode
```

según el portable subset definido.

---

# 98. DistinctnessPredicateNode

SQL ofrece semánticas útiles distintas de `=`.

Conceptualmente:

```text
IS DISTINCT FROM
IS NOT DISTINCT FROM
```

---

# 99. Why important

A diferencia de `=`:

```text
NULL IS NOT DISTINCT FROM NULL
```

puede evaluarse como verdadero.

---

# 100. Semantic operator

VoltStack podrá modelar:

```text
DistinctnessMode
├── DISTINCT
└── NOT_DISTINCT
```

---

# 101. Emulation

Si el target no tiene syntax nativa, el Planner podrá considerar emulación sólo si preserva semántica.

---

# 102. Null-safe equality

No deberá representarse simplemente como:

```text
EQUAL
```

porque la semántica es diferente.

---

# 103. BooleanTestPredicateNode

Puede representar operaciones como:

```text
IS TRUE
IS FALSE
IS UNKNOWN
IS NOT TRUE
IS NOT FALSE
IS NOT UNKNOWN
```

cuando formen parte del modelo portable/capability-aware.

---

# 104. Why separate

Estas operaciones interactúan explícitamente con three-valued logic.

---

# 105. BooleanTestMode

Conceptualmente:

```text
IS_TRUE
IS_FALSE
IS_UNKNOWN
IS_NOT_TRUE
IS_NOT_FALSE
IS_NOT_UNKNOWN
```

---

# 106. Boolean value compatibility

El operand podrá ser:

```text
PredicateNode
```

o una expresión boolean-like mediante un wrapper semántico.

La decisión deberá mantenerse tipada.

---

# 107. Predicate-as-expression boundary

SQL permite usar condiciones como valores en algunos motores/contextos.

VoltStack no deberá borrar por ello la frontera:

```text
Predicate
≠
Expression
```

---

# 108. Explicit bridge

Si se necesita, utilizar:

```text
PredicateValueExpression
```

o una conversión semántica equivalente.

---

# 109. Expression-to-predicate bridge

Igualmente podría existir:

```text
BooleanExpressionPredicate
```

para plataformas/contextos donde una expresión booleana puede actuar como condición.

---

# 110. Bridges are explicit

Nunca convertir implícitamente cualquier ExpressionNode en PredicateNode.

---

# 111. JsonPredicate family

Las condiciones JSON deberán representarse semánticamente.

Ejemplos:

```text
json.contains
json.has_key
json.path_exists
json.overlaps
```

---

# 112. JsonContainsPredicate

Conceptualmente:

```text
JsonContainsPredicate
├── document
├── candidate
└── path?
```

---

# 113. JsonHasKeyPredicate

```text
JsonHasKeyPredicate
├── document
└── key
```

---

# 114. JsonPathExistsPredicate

```text
JsonPathExistsPredicate
├── document
└── JsonPath
```

---

# 115. JSON syntax isolation

El AST no deberá contener:

```text
@>
?
JSON_CONTAINS
JSON_EXTRACT(...)
json_extract(...)
```

como representación portable.

---

# 116. JSON capability requirements

Ejemplos conceptuales:

```text
query.json.contains
query.json.key_exists
query.json.path_exists
```

---

# 117. JSON semantics

El sistema deberá considerar diferencias entre:

- JSON textual;
- JSON binary/native;
- scalar extraction;
- containment;
- path semantics;
- array semantics;
- numeric comparison;
- NULL vs JSON null.

---

# 118. SQL NULL ≠ JSON null

Esta distinción deberá preservarse.

---

# 119. JsonNull semantics

Podrá ser necesario distinguir:

```text
SQL NULL
JSON null
missing path
```

---

# 120. JSON predicate diagnostics

El Semantic Engine deberá poder explicar estas diferencias cuando afecten la consulta.

---

# 121. Tuple comparisons

VoltStack podrá soportar:

```text
(a, b) = (c, d)
```

mediante operands `TupleExpressionNode`.

---

# 122. Tuple comparability

Debe validarse:

```text
arity
+
corresponding type compatibility
+
platform capability
```

---

# 123. Tuple ordering

Comparaciones como:

```text
(a, b) < (c, d)
```

requieren semántica específica y capability validation.

---

# 124. Predicate composition API

El Query Builder podrá ofrecer:

```php
->where(...)
->orWhere(...)
->whereNull(...)
->whereNotNull(...)
->whereBetween(...)
->whereIn(...)
->whereExists(...)
```

sin exponer la complejidad interna.

---

# 125. Fluent ergonomics

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->where(function ($query) {
        $query
            ->where('age', '>=', 18)
            ->orWhereNotNull('verified_at');
    });
```

podrá construir:

```text
AndPredicate
├── active = p1
└── OrPredicate
    ├── age >= p2
    └── verified_at IS NOT NULL
```

---

# 126. Closures

Las closures son una API de construcción.

No deberán almacenarse en el AST.

---

# 127. Builder callback lifecycle

```text
Closure
   │
   ▼
PredicateBuilder
   │
   ▼
PredicateNode
```

La closure desaparece después de la construcción.

---

# 128. PredicateBuilder

Podrá existir:

```text
PredicateBuilder
```

como API ergonómica.

---

# 129. PredicateBuilder responsibility

Sólo:

```text
developer input
→ Predicate AST
```

---

# 130. PredicateBuilder must not

No deberá:

- generar SQL;
- consultar PDO;
- resolver Platform;
- ejecutar;
- hidratar;
- consultar EntityManager;
- mantener estado global.

---

# 131. Operator input normalization

Una API Laravel-like podrá aceptar:

```php
->where('age', '>=', 18)
```

y normalizar:

```text
'>='
→ ComparisonOperator::GREATER_THAN_OR_EQUAL
```

en el boundary del Builder.

---

# 132. Internal AST

Internamente no deberá conservar el string:

```text
'>='
```

como semántica principal.

---

# 133. Unknown operator

Un operador desconocido deberá producir:

```text
UnknownComparisonOperatorException
```

en vez de convertirse directamente en SQL.

---

# 134. Custom operators

Los operadores adicionales deberán pasar por el Extension System.

---

# 135. Native operator escape hatch

Puede existir una API explícita para operadores nativos.

Deberá marcar:

```text
PLATFORM_SPECIFIC
```

o:

```text
DIALECT_SPECIFIC
```

---

# 136. Logical normalization

El sistema podrá normalizar:

```text
And(
    A,
    And(B, C)
)
```

a:

```text
And(A, B, C)
```

---

# 137. OR normalization

Igualmente:

```text
Or(
    A,
    Or(B, C)
)
```

a:

```text
Or(A, B, C)
```

---

# 138. Normalization invariant

La normalización no deberá cambiar la semántica.

---

# 139. Canonical logical shape

Una forma canónica facilita:

- fingerprints;
- optimizer;
- testing;
- diagnostics;
- deduplication.

---

# 140. Predicate ordering

VoltStack no deberá reordenar arbitrariamente predicates durante construction.

---

# 141. Why

Aunque muchos predicates parezcan conmutativos, pueden existir:

- volatile functions;
- expensive expressions;
- platform behavior;
- errors;
- extension semantics.

---

# 142. Optimizer reordering

Sólo el Optimizer podrá reordenar cuando exista prueba suficiente de seguridad.

---

# 143. Predicate volatility

La volatility se deriva de sus operands.

Ejemplo:

```text
random() > 0.5
```

es un predicado volátil.

---

# 144. PredicateSemanticInfo

Podrá contener:

```text
PredicateSemanticInfo
├── truthDomain
├── nullability/unknownability
├── volatility
├── determinism
├── dependencies
├── capabilityRequirements
├── portability
├── selectivityHint?
└── semanticFlags
```

---

# 145. Truth domain analysis

Podrá clasificarse:

```text
BOOLEAN_TWO_VALUED
BOOLEAN_THREE_VALUED
ALWAYS_TRUE
ALWAYS_FALSE
ALWAYS_UNKNOWN
UNKNOWN
```

según contexto.

---

# 146. Unknownability

Un predicado puede ser:

```text
NEVER_UNKNOWN
MAY_BE_UNKNOWN
ALWAYS_UNKNOWN
```

---

# 147. Example

```text
column IS NULL
```

normalmente produce sólo:

```text
TRUE/FALSE
```

mientras:

```text
column = parameter
```

puede producir `UNKNOWN`.

---

# 148. Predicate dependencies

Podrá calcularse:

```text
PredicateDependencySet
```

incluyendo:

- columns;
- parameters;
- functions;
- subqueries;
- outer symbols;
- capabilities.

---

# 149. Dependency example

```text
price > :minimum
AND
status = :status
```

depende de:

```text
columns:
- price
- status

parameters:
- minimum
- status
```

---

# 150. Join analysis

Predicate dependencies serán esenciales para determinar:

```text
which sources a predicate references
```

---

# 151. Predicate source set

Ejemplo:

```text
orders.user_id = users.id
```

depende de:

```text
orders
users
```

---

# 152. Join predicate classification

Semantic Analysis podrá clasificar:

```text
LOCAL_FILTER
JOIN_CONDITION
CORRELATED_FILTER
CONSTANT_FILTER
UNKNOWN
```

---

# 153. Classification ≠ AST mutation

La clasificación vive en Semantic Metadata.

---

# 154. Predicate pushdown

El Optimizer podrá mover predicates hacia fuentes inferiores cuando sea seguro.

---

# 155. Example

```text
SELECT ...
FROM users
JOIN orders ...
WHERE users.active = true
```

podría permitir pushdown del filtro de `users`.

---

# 156. Pushdown safety

Debe considerar:

- outer joins;
- NULL semantics;
- volatility;
- correlation;
- aggregation;
- windowing;
- subqueries;
- platform semantics.

---

# 157. Outer join caution

Mover un predicate a través de:

```text
LEFT JOIN
```

puede cambiar resultados.

---

# 158. Predicate simplification

Posibles reglas:

```text
A AND TRUE → A

A OR FALSE → A

NOT NOT A → A
```

si se modela correctamente el truth domain.

---

# 159. Constant simplification

Ejemplos:

```text
TRUE AND FALSE
→ FALSE
```

---

# 160. UNKNOWN simplification

Debe utilizar las tablas de lógica ternaria.

Ejemplo:

```text
FALSE AND UNKNOWN
→ FALSE
```

pero:

```text
TRUE AND UNKNOWN
→ UNKNOWN
```

---

# 161. Predicate deduplication

Podría transformar:

```text
A AND A
→ A
```

sólo si repetir `A` no tiene semántica observable relevante.

---

# 162. Volatile predicate deduplication

No deberá deduplicarse:

```text
random() > 0.5
AND
random() > 0.5
```

basándose únicamente en igualdad estructural.

---

# 163. Contradiction detection

El Optimizer podrá detectar casos simples:

```text
x = 1
AND
x = 2
```

cuando type/domain semantics permitan demostrar contradicción.

---

# 164. No unsafe theorem prover

V1 no deberá intentar convertirse en un theorem prover general.

---

# 165. Range simplification

Casos futuros:

```text
x > 10
AND
x > 20
→
x > 20
```

si se preservan NULL/type semantics.

---

# 166. IN normalization

Casos como:

```text
x = 1 OR x = 2 OR x = 3
```

podrían convertirse en:

```text
x IN (1,2,3)
```

si la estrategia es beneficiosa y semánticamente equivalente.

---

# 167. Reverse transformation

Igualmente un `IN` podría transformarse en otra estrategia física.

Esto pertenece al Optimizer/Planner.

---

# 168. Predicate normalization ≠ SQL rewrite

Las transformaciones siguen operando sobre AST/semantic model.

---

# 169. Predicate fingerprint

Todo predicate deberá contribuir deterministicamente al Query Fingerprint.

---

# 170. Comparison fingerprint

Ejemplo:

```text
CMP[
  GT,
  COLUMN(age),
  PARAMETER(p1)
]
```

---

# 171. Logical fingerprint

Ejemplo:

```text
AND[
  CMP(...),
  NULL(...)
]
```

---

# 172. Parameter values

Los runtime values no forman parte del structural fingerprint.

---

# 173. Parameter shape

Sí podrán formar parte:

- ParameterId normalizado;
- inferred semantic type;
- expansion mode;
- structural role.

---

# 174. IN fingerprint

No deberá incluir valores concretos.

Pero la cantidad de elementos puede afectar query shape dependiendo de la estrategia de binding.

---

# 175. Variable-length IN

VoltStack deberá distinguir:

```text
semantic fingerprint
```

de:

```text
compiled query shape fingerprint
```

---

# 176. Example

Semánticamente:

```text
id IN collection
```

puede ser una misma forma.

Pero compilado como:

```text
IN (?, ?, ?)
```

vs:

```text
IN (?, ?, ?, ?, ?)
```

puede requerir cache keys diferentes.

---

# 177. Collection parameter

El futuro Parameter System podrá representar:

```text
CollectionParameter
```

para desacoplar membership semantics del número de placeholders.

---

# 178. Planner strategies for collections

Ejemplos:

```text
EXPANDED_PARAMETERS
ARRAY_PARAMETER
VALUES_RELATION
TEMPORARY_RELATION
CHUNKED_DISJUNCTION
```

según target/capabilities.

---

# 179. Predicate capability requirements

Ejemplos:

```text
query.predicate.regex
query.predicate.distinctness
query.predicate.tuple_comparison
query.predicate.boolean_test
query.json.contains
query.pattern.case_insensitive
```

---

# 180. Requirement collection

```text
Predicate AST
      │
      ▼
Semantic Analysis
      │
      ▼
CapabilityRequirementSet
```

---

# 181. Planner resolution

```text
Requirement
      │
      ▼
CapabilitySnapshot
      │
      ▼
Strategy Resolver
      │
      ├── Native
      ├── Emulated
      ├── Partial
      └── Unsupported
```

---

# 182. No vendor checks

Nunca:

```php
if ($platform->name() === 'postgresql') {
    // ILIKE
}
```

dentro del Predicate Builder.

---

# 183. Correct flow

```text
CaseInsensitiveLikePredicate
        │
        ▼
Capability Requirement
        │
        ▼
Planner
        │
        ▼
PostgreSQL Strategy
        │
        ▼
Dialect
```

---

# 184. Predicate portability

Podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_CAPABILITY
PORTABLE_WITH_EMULATION
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

---

# 185. Portable core

V1 deberá priorizar:

```text
comparison
AND
OR
NOT
NULL
BETWEEN
IN
EXISTS
LIKE
```

---

# 186. Advanced portable/capability-aware

Posteriormente:

```text
distinctness
boolean tests
regex
tuple comparisons
JSON predicates
```

---

# 187. RawPredicateNode

Escape hatch para condiciones SQL no modeladas.

```text
RawPredicateNode
├── fragment
├── bindings
├── portability
└── trust metadata
```

---

# 188. Raw predicate restrictions

Debe:

- ser explícito;
- soportar safe bindings;
- evitar interpolación;
- ser visible a diagnostics;
- limitar optimizer assumptions.

---

# 189. Raw predicate truth semantics

Por defecto:

```text
truth domain: UNKNOWN
volatility: UNKNOWN
dependencies: UNKNOWN/PARTIAL
```

salvo metadata explícita confiable.

---

# 190. Optimizer treatment

Un RawPredicate deberá actuar como una barrera conservadora para transformaciones que requieran comprender su semántica.

---

# 191. Raw predicate dependencies

Una API avanzada podría permitir declarar:

```text
referenced sources
referenced columns
volatility
```

pero VoltStack no deberá confiar ciegamente en metadata insegura para transformaciones críticas.

---

# 192. ExtensionPredicateNode

Los paquetes podrán agregar nuevas familias de predicates.

---

# 193. Extension requirements

Una extensión podrá necesitar registrar:

```text
Node Descriptor
Structural Validator
Semantic Handler
Type/Operand Rules
Truth Rules
Capability Requirements
Optimizer Rules
Planner Strategy
Compiler Handler
Fingerprint Handler
Diagnostic Renderer
```

---

# 194. Example

Un paquete geoespacial podría introducir:

```text
geo.intersects
geo.contains
geo.within_distance
```

---

# 195. Semantic extension

Ejemplo:

```text
GeoWithinDistancePredicate
├── geometryA
├── geometryB
└── distance
```

---

# 196. No SQL in extension AST

No:

```text
ST_DWithin(a, b, d)
```

como string interno principal.

---

# 197. Extension lifecycle

```text
Discover
→ Register
→ Validate
→ Resolve Dependencies
→ Register Handlers
→ Freeze
→ Runtime
```

---

# 198. Registry freeze

No deberá permitirse registrar predicate handlers arbitrariamente durante requests normales.

---

# 199. Specialized registries

Preferir:

```text
PredicateSemanticHandlerRegistry
PredicateOptimizerRuleRegistry
PredicateCompilerRegistry
PredicateDiagnosticRegistry
```

sobre un universal registry.

---

# 200. Predicate validation

Existirán al menos dos niveles:

```text
Structural Validation
Semantic Validation
```

---

# 201. Structural validation

Ejemplos:

- AND no vacío;
- BETWEEN tiene tres operands;
- EXISTS tiene query;
- IN tiene source válido;
- LIKE tiene pattern;
- extension node tiene NodeTypeId válido.

---

# 202. Semantic validation

Ejemplos:

- operands comparables;
- column resolvable;
- subquery valid;
- tuple arity compatible;
- regex capability;
- JSON type compatible;
- predicate allowed in current context.

---

# 203. PredicateContext

Podrá existir:

```text
PredicateContext
```

---

# 204. Contexts

Ejemplos:

```text
WHERE
JOIN_ON
HAVING
FILTER
CASE_WHEN
QUALIFY
POLICY_FILTER
ORM_SCOPE
```

donde las capabilities y el modelo lo permitan.

---

# 205. Context external

El PredicateNode no deberá almacenar:

```text
$this->context = WHERE
```

como mutable runtime state.

---

# 206. Contextual validation

Un predicate válido en:

```text
WHERE
```

puede no ser válido en otro contexto.

---

# 207. Aggregate interaction

Por ejemplo:

```text
COUNT(*) > 5
```

es estructuralmente un ComparisonPredicate.

Pero su contexto normal será:

```text
HAVING
```

no un `WHERE` pre-aggregation.

---

# 208. Semantic phase

La validación de aggregate scope deberá ocurrir en Semantic Analysis.

---

# 209. Window function interaction

Predicates que dependen de window functions requieren reglas contextuales y posiblemente query rewrites.

---

# 210. QUALIFY

Si algún target soporta `QUALIFY`, deberá ser capability-driven.

El core Query Model puede expresar la intención sin depender de la keyword.

---

# 211. Predicate aliases

No deberán existir aliases SQL como operands lógicos sin resolución semántica apropiada.

---

# 212. Correlated predicates

Podrán referenciar:

```text
local symbols
+
outer scope symbols
```

---

# 213. Correlation metadata

Semantic Analysis producirá:

```text
CorrelationSet
```

---

# 214. Why correlation matters

Afecta:

- decorrelation;
- subquery planning;
- caching;
- join conversion;
- predicate movement.

---

# 215. Decorrelation

El Optimizer podrá convertir ciertos:

```text
EXISTS correlated subquery
```

en estrategias equivalentes tipo semi-join.

---

# 216. Anti-join

`NOT EXISTS` podrá planearse conceptualmente como anti-join cuando sea seguro.

---

# 217. IN subquery planning

También podrá convertirse en:

```text
semi-join
```

dependiendo de NULL semantics y target strategy.

---

# 218. NOT IN caution

`NOT IN` no deberá convertirse ingenuamente en anti-join debido a NULL semantics.

---

# 219. SemiJoin logical plan

Estas transformaciones pertenecen al Logical Query Planner, no al Predicate AST.

---

# 220. Selectivity

El Predicate System podrá exponer información útil para estimar selectivity.

---

# 221. SelectivityHint

Conceptualmente:

```text
UNKNOWN
HIGH
MEDIUM
LOW
```

o una estimación cuantitativa futura.

---

# 222. No fake statistics

VoltStack no deberá inventar estadísticas de cardinalidad.

---

# 223. User hints

En el futuro podrían existir hints explícitos.

Deberán permanecer separados de la semántica lógica del predicate.

---

# 224. Query optimizer scope

VoltStack podrá optimizar:

- AST shape;
- redundant predicates;
- pushdown;
- subquery strategy;
- IN strategy;
- semantic rewrites.

No intentará sustituir el optimizer interno de MySQL/PostgreSQL/MariaDB/SQLite.

---

# 225. Constraint inference

Schema metadata podrá ayudar a simplificar predicates.

Ejemplo:

Si:

```text
users.id
```

es `NOT NULL`, entonces:

```text
users.id IS NULL
```

podría reconocerse como imposible en determinados contexts.

---

# 226. Caution

Outer joins pueden volver nullable una columna originalmente `NOT NULL`.

---

# 227. Effective nullability

El Semantic Engine deberá usar:

```text
effective query nullability
```

no sólo schema nullability.

---

# 228. Join-induced nullability

Ejemplo:

```text
users
LEFT JOIN orders
```

las columnas de `orders` pueden ser nullable en el result scope aunque el schema diga `NOT NULL`.

---

# 229. Predicate reasoning must use semantic scope

No consultar directamente Schema Metadata ignorando Query Semantics.

---

# 230. Parameter binding

Ejemplo:

```php
->where('email', $email)
```

produce:

```text
ComparisonPredicate
├── Column(email)
├── EQUAL
└── Parameter(p1)

BindingSet
└── p1 → $email
```

---

# 231. Binding remains external

El runtime value no vive en el Predicate AST.

---

# 232. Collection binding

```php
->whereIn('id', $ids)
```

podrá producir:

```text
InPredicate
├── Column(id)
└── CollectionParameter(p1)
```

o una lista de ParameterExpressions según la fase/modelo elegido.

---

# 233. Preferred abstraction

Para desacoplar AST del número de elementos runtime, el sistema deberá considerar:

```text
CollectionParameter
```

como primera clase.

---

# 234. Collection expansion

La expansión pertenece a:

```text
Planning
or
Compilation/Binding Planning
```

según la estrategia final.

---

# 235. Security

El Predicate System deberá asumir:

```text
user values → parameters
```

por defecto.

---

# 236. Dynamic column security

Columnas dinámicas deberán pasar por:

```text
Identifier
→ Validation
→ Semantic Resolution
→ Dialect Quoting
```

---

# 237. Dynamic operator security

Un operador recibido del usuario no deberá pasar directamente a SQL.

---

# 238. Dynamic sort/predicate fields

La aplicación deberá usar allowlists o metadata segura para exponer campos dinámicos.

---

# 239. Query filters from HTTP

Nunca:

```text
HTTP operator string
→ raw SQL
```

---

# 240. Correct

```text
HTTP filter
→ Application validation
→ Semantic operator
→ Predicate Builder
→ Predicate AST
```

---

# 241. Policy predicates

Authorization o Multitenancy podrían proporcionar predicates.

Ejemplo:

```text
tenant_id = current tenant
```

---

# 242. Boundary rule

Database core no deberá depender de Authorization o Multitenancy.

---

# 243. Integration model

```text
Optional Integration
      │
      ▼
Predicate Provider
      │
      ▼
Predicate AST
```

---

# 244. Tenant value

El tenant ID deberá convertirse normalmente en:

```text
ParameterExpression
```

no en global state dentro del predicate.

---

# 245. Global scopes

ORM podrá aplicar scopes mediante Predicate AST.

---

# 246. Soft deletes

Ejemplo:

```text
deleted_at IS NULL
```

podrá inyectarse como Predicate AST.

---

# 247. Scope transparency

Los predicates añadidos automáticamente deberán poder verse en diagnostics.

---

# 248. No invisible security predicates

Un filtro de seguridad/tenant deberá quedar registrado en Query Metadata/Diagnostics.

---

# 249. Predicate provenance

Podrá existir metadata:

```text
PredicateOrigin
├── USER
├── ORM_SCOPE
├── TENANT
├── AUTHORIZATION
├── SOFT_DELETE
├── OPTIMIZER
├── SYSTEM
└── EXTENSION
```

---

# 250. Origin outside immutable semantic core

La provenance podrá vivir en:

```text
QueryMetadata
```

o side metadata.

No necesariamente dentro del node.

---

# 251. Source location

Para diagnostics, puede registrarse:

```text
file
line
builder operation
extension
```

cuando esté disponible.

---

# 252. Predicate diagnostics

Ejemplo:

```text
Predicate:
  Comparison: GREATER_THAN_OR_EQUAL

Left:
  users.age
  Type: Integer
  Nullable: false

Right:
  Parameter p2
  Inferred Type: Integer

Truth Domain:
  TRUE | FALSE

Portability:
  PORTABLE
```

---

# 253. NULL-aware diagnostic

Ejemplo:

```text
Predicate:
  users.deleted_at = :p1

Warning:
  Parameter may be NULL.

Potential truth result:
  UNKNOWN

Suggestion:
  Use an explicit null predicate if NULL is intended semantically.
```

---

# 254. IN diagnostic

```text
Predicate:
  id IN collection(p1)

Collection size:
  runtime-dependent

Available strategies:
  expanded parameters
  target-specific collection binding

Selected strategy:
  determined during planning
```

---

# 255. Explain predicate

CLI/debug tooling podrá mostrar:

```text
Original Predicate
Normalized Predicate
Semantic Predicate
Optimized Predicate
Logical Plan Placement
Capability Requirements
```

---

# 256. No sensitive values

No mostrar valores de parameters por defecto.

---

# 257. Predicate exceptions

Jerarquía conceptual:

```text
QueryPredicateException
├── InvalidPredicateException
├── InvalidPredicateOperandException
├── UnknownComparisonOperatorException
├── IncompatibleComparisonException
├── InvalidBetweenPredicateException
├── InvalidInPredicateException
├── InvalidExistsPredicateException
├── InvalidPatternPredicateException
├── InvalidRegexPredicateException
├── InvalidTuplePredicateException
├── UnsupportedPredicateException
├── PredicateComplexityException
└── RawPredicateException
```

---

# 258. Capability exception

Cuando la estructura es semánticamente válida pero el target no puede soportarla:

```text
CapabilityNotSupportedException
```

será preferible a clasificarla como malformed predicate.

---

# 259. Predicate complexity governance

El sistema podrá controlar:

```text
maxPredicateDepth
maxPredicateNodes
maxLogicalChildren
maxInElements
maxSubqueryDepth
maxPatternLength
```

según Resource Governance.

---

# 260. Query generated attacks

Esto ayuda contra consultas generadas que intenten producir:

- miles de ORs;
- IN gigantes;
- nesting extremo;
- regex costosos;
- subqueries recursivos.

---

# 261. Complexity ≠ DB execution cost

Debe distinguirse:

```text
AST complexity
```

de:

```text
database cost
```

---

# 262. Regex risk

Regex puede requerir políticas adicionales por riesgo de ejecución costosa según motor.

---

# 263. Pattern escaping limits

Pattern length y wildcard policy podrán formar parte de capas superiores de aplicación.

Database core proporciona primitives seguros, no políticas de negocio.

---

# 264. Immutability

Todos los PredicateNodes publicados serán inmutables.

---

# 265. Transformations

Una transformación produce:

```text
old predicate
→ new predicate
```

Nunca:

```php
$predicate->children[] = $newChild;
```

---

# 266. Structural sharing

Subtrees no modificados podrán reutilizarse.

---

# 267. Persistent runtime safety

Predicate AST immutable podrá sobrevivir más tiempo si no contiene state scoped.

---

# 268. Scoped metadata

Nunca deberán sobrevivir accidentalmente entre requests:

```text
resolved tenant
parameter values
semantic scope
connection
transaction
runtime capability snapshot
```

---

# 269. Application-scoped safe components

Podrán sobrevivir:

```text
Predicate descriptors
immutable operator descriptors
frozen registries
optimizer rule definitions
compiler handler definitions
```

---

# 270. Concurrency

Los servicios application-scoped deberán ser:

```text
stateless
or
immutable
or
properly concurrency-safe
```

---

# 271. FrankenPHP

El mismo Predicate System deberá funcionar en workers persistentes sin almacenar request state en singletons.

---

# 272. RoadRunner

Misma regla.

---

# 273. OpenSwoole

Además deberá evitar:

```text
static current predicate context
```

por riesgo de concurrencia entre coroutines.

---

# 274. Testing architecture

El Predicate System deberá probarse mediante:

```text
Structural Tests
Semantic Tests
Truth Logic Tests
Type Compatibility Tests
Normalization Tests
Optimizer Tests
Capability Tests
Compiler Tests
Cross-platform Integration Tests
Property Tests
Security Tests
Persistent Runtime Tests
```

---

# 275. Truth logic tests

Las tablas de:

```text
AND
OR
NOT
```

deberán tener tests exhaustivos con:

```text
TRUE
FALSE
UNKNOWN
```

---

# 276. NULL tests

Casos esenciales:

```text
NULL comparisons
IS NULL
IS NOT NULL
IN with NULL
NOT IN with NULL
outer join nullability
distinctness
```

---

# 277. Comparison tests

Cubrir:

```text
numeric
string
date/time
uuid
enum
JSON
tuple
incompatible types
```

---

# 278. Pattern tests

Cubrir:

```text
LIKE
NOT LIKE
contains
starts with
ends with
escaping %
escaping _
custom escape character
case sensitivity
```

---

# 279. Cross-platform tests

Cada semantic predicate deberá probarse donde aplique contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 280. Semantic result tests

No sólo comparar SQL.

Cuando sea posible:

```text
same semantic query
→ equivalent result set
```

en todos los targets soportados.

---

# 281. Golden SQL tests

También serán útiles para verificar compilación determinista por Dialect.

---

# 282. Property tests

Especialmente:

```text
normalization idempotence
fingerprint determinism
immutable transformations
three-valued logic identities
```

---

# 283. Optimizer equivalence tests

Cada rewrite rule deberá demostrar equivalencia para:

```text
TRUE
FALSE
UNKNOWN
NULL operands
volatile operands
```

según corresponda.

---

# 284. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Predicate\
```

---

# 285. Proposed structure

```text
Query/
├── Ast/
│   └── Node/
│       └── Predicate/
│           ├── ComparisonPredicateNode.php
│           ├── AndPredicateNode.php
│           ├── OrPredicateNode.php
│           ├── NotPredicateNode.php
│           ├── ConstantPredicateNode.php
│           ├── NullPredicateNode.php
│           ├── BetweenPredicateNode.php
│           ├── InPredicateNode.php
│           ├── ExistsPredicateNode.php
│           ├── LikePredicateNode.php
│           ├── RegexPredicateNode.php
│           ├── DistinctnessPredicateNode.php
│           ├── BooleanTestPredicateNode.php
│           ├── JsonPredicateNode.php
│           ├── RawPredicateNode.php
│           └── ExtensionPredicateNode.php
│
└── Predicate/
    ├── Contract/
    │   ├── PredicateSemanticHandler.php
    │   ├── PredicateNormalizer.php
    │   ├── PredicateExtension.php
    │   └── PredicateDiagnosticRenderer.php
    │
    ├── Operator/
    │   ├── ComparisonOperator.php
    │   ├── DistinctnessMode.php
    │   └── BooleanTestMode.php
    │
    ├── Logical/
    │   ├── TruthValue.php
    │   ├── TruthDomain.php
    │   └── ThreeValuedLogic.php
    │
    ├── Membership/
    │   ├── InSource.php
    │   ├── ExpressionListInSource.php
    │   ├── SubqueryInSource.php
    │   └── CollectionParameterInSource.php
    │
    ├── Pattern/
    │   ├── PatternMode.php
    │   ├── CaseSensitivity.php
    │   ├── PatternEscaper.php
    │   └── RegexOptions.php
    │
    ├── Json/
    │   ├── JsonPredicateKind.php
    │   └── JsonPredicateSemantics.php
    │
    ├── Semantic/
    │   ├── PredicateSemanticAnalyzer.php
    │   ├── PredicateSemanticInfo.php
    │   ├── PredicateDependencySet.php
    │   ├── PredicateOrigin.php
    │   └── CorrelationSet.php
    │
    ├── Validation/
    │   ├── PredicateStructuralValidator.php
    │   └── PredicateSemanticValidator.php
    │
    ├── Normalization/
    │   ├── PredicateNormalizer.php
    │   └── Core/
    │
    ├── Capability/
    │   └── PredicateCapabilityResolver.php
    │
    ├── Builder/
    │   └── PredicateBuilder.php
    │
    ├── Diagnostics/
    │   └── PredicateDiagnosticRenderer.php
    │
    ├── Extension/
    │   └── PredicateExtensionRegistry.php
    │
    └── Exception/
```

---

# 286. Dependency direction

```text
Expression AST
      │
      ▼
Predicate AST
      │
      ▼
Predicate Semantic Analysis
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

---

# 287. Forbidden reverse dependencies

Nunca:

```text
Predicate AST
→ Connection

Predicate AST
→ Driver

Predicate AST
→ PDO

Predicate AST
→ EntityManager

Predicate AST
→ Runtime Server
```

---

# 288. DB-PRED-001

Todo predicate core será un `PredicateNode`.

---

# 289. DB-PRED-002

Todo PredicateNode publicado será inmutable.

---

# 290. DB-PRED-003

PredicateNode y ExpressionNode permanecerán conceptualmente separados.

---

# 291. DB-PRED-004

Un predicate nunca generará SQL.

---

# 292. DB-PRED-005

Un predicate nunca ejecutará consultas.

---

# 293. DB-PRED-006

Un predicate nunca accederá a Connection o Driver.

---

# 294. DB-PRED-007

Los operadores core serán semánticos, no strings SQL.

---

# 295. DB-PRED-008

Los runtime values se parameterizarán por defecto.

---

# 296. DB-PRED-009

Los Parameter values vivirán fuera del AST.

---

# 297. DB-PRED-010

Los identifiers permanecerán sin quoting específico de Dialect.

---

# 298. DB-PRED-011

La comparación con NULL no se interpretará mediante semántica PHP.

---

# 299. DB-PRED-012

`IS NULL` y `IS NOT NULL` tendrán representación semántica explícita.

---

# 300. DB-PRED-013

El Predicate System reconocerá SQL three-valued logic.

---

# 301. DB-PRED-014

`UNKNOWN` no se reducirá globalmente a `FALSE`.

---

# 302. DB-PRED-015

Las reglas de AND/OR/NOT respetarán three-valued logic.

---

# 303. DB-PRED-016

Las optimizaciones lógicas deberán demostrar equivalencia bajo three-valued logic.

---

# 304. DB-PRED-017

AndPredicate y OrPredicate podrán utilizar representación N-ary.

---

# 305. DB-PRED-018

Logical nodes vacíos no serán creados implícitamente.

---

# 306. DB-PRED-019

El orden original de predicates se preservará hasta que una fase autorizada decida transformarlo.

---

# 307. DB-PRED-020

Sólo el Optimizer podrá reordenar predicates por razones de optimización.

---

# 308. DB-PRED-021

La volatility deberá considerarse antes de reordenar o deduplicar.

---

# 309. DB-PRED-022

Dos predicates estructuralmente iguales no serán necesariamente evaluation-equivalent.

---

# 310. DB-PRED-023

NOT IN con NULL deberá conservar semántica SQL correcta.

---

# 311. DB-PRED-024

Una lista IN vacía tendrá una política semántica explícita.

---

# 312. DB-PRED-025

El Dialect no será responsable de decidir la semántica de IN vacío.

---

# 313. DB-PRED-026

IN podrá usar collection parameters sin acoplar el AST al número final de placeholders.

---

# 314. DB-PRED-027

La expansión de collections pertenecerá al Planning/Compilation Binding pipeline.

---

# 315. DB-PRED-028

EXISTS contendrá Query AST, no SQL.

---

# 316. DB-PRED-029

Las correlated references se resolverán mediante Semantic Scope.

---

# 317. DB-PRED-030

No existirán mutable parent-query pointers.

---

# 318. DB-PRED-031

Pattern matching distinguirá patterns de texto literal de búsqueda.

---

# 319. DB-PRED-032

Las APIs contains/startsWith/endsWith escaparán wildcards correctamente.

---

# 320. DB-PRED-033

Case-insensitive matching será semántico y capability-aware.

---

# 321. DB-PRED-034

`ILIKE` no aparecerá como operador del AST portable.

---

# 322. DB-PRED-035

Regex será capability-aware.

---

# 323. DB-PRED-036

VoltStack no declarará portabilidad regex cuando las semánticas sean incompatibles.

---

# 324. DB-PRED-037

Null-safe distinctness tendrá semántica distinta de equality ordinaria.

---

# 325. DB-PRED-038

Boolean tests preservarán UNKNOWN.

---

# 326. DB-PRED-039

Los bridges Predicate/Expression serán explícitos.

---

# 327. DB-PRED-040

JSON null, SQL NULL y missing JSON path no deberán colapsarse automáticamente.

---

# 328. DB-PRED-041

Los predicates JSON core no almacenarán operadores específicos del vendor.

---

# 329. DB-PRED-042

Tuple predicates validarán arity y tipos.

---

# 330. DB-PRED-043

Tuple comparisons serán capability-aware.

---

# 331. DB-PRED-044

PredicateBuilder sólo construirá AST.

---

# 332. DB-PRED-045

Closures de Builder nunca se almacenarán dentro del AST.

---

# 333. DB-PRED-046

Los strings de operadores públicos se normalizarán a semantic operators en el boundary.

---

# 334. DB-PRED-047

Operadores desconocidos no se enviarán directamente al Compiler.

---

# 335. DB-PRED-048

RawPredicate será un escape hatch explícito.

---

# 336. DB-PRED-049

RawPredicate utilizará bindings seguros cuando aplique.

---

# 337. DB-PRED-050

RawPredicate será tratado conservadoramente por Semantic Analysis y Optimizer.

---

# 338. DB-PRED-051

Los extension predicates utilizarán NodeTypeId namespaced.

---

# 339. DB-PRED-052

Los extension handlers se registrarán antes del registry freeze.

---

# 340. DB-PRED-053

No habrá silent extension replacement.

---

# 341. DB-PRED-054

Capability requirements se resolverán fuera del PredicateNode.

---

# 342. DB-PRED-055

El PredicateNode no seleccionará estrategia nativa/emulada.

---

# 343. DB-PRED-056

La selección de estrategia pertenece al Planner.

---

# 344. DB-PRED-057

La representación SQL pertenece al Compiler/Dialect.

---

# 345. DB-PRED-058

Predicate normalization y Predicate optimization permanecerán conceptualmente separadas.

---

# 346. DB-PRED-059

Normalization será determinista.

---

# 347. DB-PRED-060

Normalization será idempotente.

---

# 348. DB-PRED-061

PredicateSemanticInfo vivirá fuera del AST.

---

# 349. DB-PRED-062

Resolved types no se escribirán dentro del PredicateNode.

---

# 350. DB-PRED-063

Effective nullability vendrá del Query Semantic Model, no sólo del Schema.

---

# 351. DB-PRED-064

Outer join nullability deberá considerarse en reasoning.

---

# 352. DB-PRED-065

Predicate dependency analysis será source-aware.

---

# 353. DB-PRED-066

Predicate pushdown sólo ocurrirá cuando preserve semántica.

---

# 354. DB-PRED-067

Outer joins serán una frontera especial para pushdown.

---

# 355. DB-PRED-068

Subquery decorrelation pertenecerá al Optimizer/Planner.

---

# 356. DB-PRED-069

NOT EXISTS podrá transformarse a anti-join sólo cuando sea semánticamente seguro.

---

# 357. DB-PRED-070

NOT IN nunca se convertirá ingenuamente a anti-join ignorando NULL.

---

# 358. DB-PRED-071

Los fingerprints no contendrán runtime parameter values.

---

# 359. DB-PRED-072

Semantic fingerprint y compiled-shape fingerprint podrán ser diferentes.

---

# 360. DB-PRED-073

Predicate diagnostics no expondrán datos sensibles por defecto.

---

# 361. DB-PRED-074

Los predicates automáticos de ORM/tenant/security deberán ser diagnosticables.

---

# 362. DB-PRED-075

Database core no dependerá de Multitenancy o Authorization.

---

# 363. DB-PRED-076

Las integraciones externas producirán Predicate AST mediante contracts.

---

# 364. DB-PRED-077

Los límites de complejidad se gobernarán mediante Resource Governance.

---

# 365. DB-PRED-078

Los trees extremadamente profundos deberán detectarse antes de causar stack exhaustion.

---

# 366. DB-PRED-079

Las registries application-scoped deberán ser frozen y concurrency-safe.

---

# 367. DB-PRED-080

No existirá global mutable current PredicateContext.

---

# 368. DB-PRED-081

El sistema será seguro para FrankenPHP persistent workers.

---

# 369. DB-PRED-082

El sistema será adaptable a RoadRunner.

---

# 370. DB-PRED-083

El sistema será coroutine-safe para OpenSwoole mediante scopes apropiados.

---

# 371. DB-PRED-084

El Predicate System permanecerá independiente del ORM.

---

# 372. DB-PRED-085

ORM, Query Builder y Repository API reutilizarán el mismo Predicate AST.

---

# 373. DB-PRED-086

Persistence Engine podrá generar predicates para updates/deletes/version checks sin generar SQL.

---

# 374. DB-PRED-087

Optimizations no utilizarán PHP boolean semantics como sustituto de SQL semantics.

---

# 375. DB-PRED-088

Las capability decisions se basarán en EffectiveCapabilitySet.

---

# 376. DB-PRED-089

No existirán vendor conditionals en Query Builder o Predicate core.

---

# 377. DB-PRED-090

Toda operación avanzada deberá ser portable, capability-aware, explícitamente target-specific o raw; nunca ambiguamente portable.

---

# 378. Anti-pattern — WHERE string

Incorrecto:

```php
->whereRaw("age >= 18 AND active = 1")
```

como mecanismo interno principal.

Correcto:

```text
AndPredicate
├── Comparison(age >= p1)
└── Comparison(active = p2)
```

`whereRaw()` seguirá existiendo únicamente como escape hatch.

---

# 379. Anti-pattern — SQL operator AST

Incorrecto:

```php
new ComparisonPredicate(
    operator: '>='
);
```

como representación interna final.

Correcto:

```text
ComparisonOperator::GREATER_THAN_OR_EQUAL
```

---

# 380. Anti-pattern — boolean simplification PHP

Incorrecto:

```php
(bool) $predicate;
```

o asumir:

```text
UNKNOWN = FALSE
```

Correcto:

```text
ThreeValuedLogic
```

---

# 381. Anti-pattern — NOT IN rewrite

Incorrecto:

```text
x NOT IN (subquery)
→
NOT EXISTS(subquery)
```

sin analizar NULL semantics.

---

# 382. Anti-pattern — vendor-specific LIKE

Incorrecto:

```php
if ($postgres) {
    return 'ILIKE';
}
```

en Predicate Builder.

Correcto:

```text
PatternPredicate
├── caseSensitivity: INSENSITIVE
└── Planner Strategy
```

---

# 383. Anti-pattern — predicate mutation

Incorrecto:

```php
$predicate->negated = true;
```

Correcto:

```text
old predicate
→ new immutable predicate
```

---

# 384. Anti-pattern — schema nullability only

Incorrecto:

```text
schema says NOT NULL
→ query expression can never be NULL
```

porque outer joins pueden modificar effective nullability.

---

# 385. Anti-pattern — invisible tenant filter

Incorrecto:

```text
tenant filtering happens somewhere in SQL compiler
```

Correcto:

```text
Tenant Integration
      │
      ▼
Predicate AST
      │
      ▼
Query Metadata
      │
      ▼
normal query pipeline
```

---

# 386. Anti-pattern — custom SQL operator injection

Incorrecto:

```php
->where($field, $_GET['operator'], $value);
```

sin validación.

Correcto:

```text
external input
→ allowlist
→ semantic operator
→ Predicate AST
```

---

# 387. Query example

API:

```php
DB::table('users')
    ->where('active', true)
    ->where(function ($query) {
        $query
            ->where('age', '>=', 18)
            ->orWhereNotNull('verified_at');
    })
    ->whereIn('country', ['MX', 'US']);
```

Predicate AST:

```text
AndPredicate
├── ComparisonPredicate
│   ├── Column(active)
│   ├── EQUAL
│   └── Parameter(p1)
│
├── OrPredicate
│   ├── ComparisonPredicate
│   │   ├── Column(age)
│   │   ├── GREATER_THAN_OR_EQUAL
│   │   └── Parameter(p2)
│   │
│   └── NullPredicate
│       ├── Column(verified_at)
│       └── IS_NOT_NULL
│
└── InPredicate
    ├── Column(country)
    └── CollectionParameter(p3)
```

Bindings:

```text
p1 → true
p2 → 18
p3 → ["MX", "US"]
```

---

# 388. Semantic result example

```text
Predicate 1
active = p1

active:
BooleanType
Non-null

p1:
BooleanType inferred

Truth:
TRUE | FALSE
```

```text
Predicate 2
age >= p2

age:
IntegerType
Nullable

p2:
IntegerType inferred

Truth:
TRUE | FALSE | UNKNOWN
```

---

# 389. Pattern example

API:

```php
$query->whereContains('name', $search);
```

Semantic model:

```text
LikePredicate
├── Column(name)
├── PatternParameter(p1)
├── mode: CONTAINS
├── caseSensitivity: DEFAULT
└── negated: false
```

The Builder does not manually generate:

```text
LIKE '%value%'
```

as SQL.

---

# 390. Case-insensitive example

```php
$query->whereContains(
    'email',
    $search,
    caseSensitive: false,
);
```

AST:

```text
PatternPredicate
├── value: Column(email)
├── pattern: Parameter(p1)
├── mode: CONTAINS
└── caseSensitivity: INSENSITIVE
```

Planner:

```text
Target Capability Snapshot
            │
            ▼
Pattern Strategy Resolver
            │
     ┌──────┼─────────┐
     ▼      ▼         ▼
 native  emulated  unsupported
```

---

# 391. EXISTS example

```text
ExistsPredicate
└── SelectNode
    ├── Source: orders
    └── Predicate
        └── Comparison
            ├── orders.user_id
            └── users.id
```

Semantic analysis:

```text
orders.user_id
→ local scope

users.id
→ outer scope

CorrelationSet
→ users
```

---

# 392. JSON example

Semantic predicate:

```text
JsonContainsPredicate
├── Column(metadata)
└── Parameter(p1)
```

Requirements:

```text
query.json.contains
```

Planner:

```text
MySQL
→ native JSON containment strategy

MariaDB
→ resolved MariaDB strategy

PostgreSQL
→ JSON/JSONB-specific strategy

SQLite
→ capability-dependent JSON strategy
```

El AST permanece sin vendor syntax.

---

# 393. Predicate optimizer pipeline

```text
Predicate AST
      │
      ▼
Structural Normalization
      │
      ▼
Semantic Analysis
      │
      ├── Types
      ├── Nullability
      ├── Truth Domain
      ├── Dependencies
      ├── Volatility
      └── Capabilities
      │
      ▼
Predicate Optimizer
      │
      ├── Simplification
      ├── Deduplication
      ├── Range Analysis
      ├── Pushdown
      ├── Correlation Analysis
      └── Subquery Rewrites
      │
      ▼
Logical Query Plan
```

---

# 394. Three-valued optimization rule

Toda regla lógica deberá poder responder:

```text
Does this transformation preserve:

TRUE?
FALSE?
UNKNOWN?
NULL behavior?
Volatility?
Evaluation count?
```

Si no puede demostrarlo:

```text
do not rewrite
```

---

# 395. Security predicate pipeline

```text
Authorization / Tenant / ORM Scope
              │
              ▼
       Predicate Provider
              │
              ▼
        Predicate AST
              │
              ▼
         Query Metadata
              │
              ▼
       Semantic Analysis
              │
              ▼
           Planner
```

No:

```text
security module
→ raw SQL injection into compiler
```

---

# 396. Expression + Predicate architecture

```text
                Query AST
                   │
          ┌────────┴────────┐
          ▼                 ▼
   Expression System   Predicate System
          │                 │
          │                 ├── Comparison
          │                 ├── Logical
          │                 ├── Null
          │                 ├── Between
          │                 ├── IN
          │                 ├── EXISTS
          │                 ├── Pattern
          │                 └── JSON
          │                 │
          └────────┬────────┘
                   ▼
             Semantic Engine
```

---

# 397. Core relationship

```text
Expression
=
produces a value

Predicate
=
produces a logical condition

Predicate Operand
=
usually Expression

Logical Predicate Child
=
Predicate
```

---

# 398. V1 implementation priority

## Phase 1

```text
ComparisonPredicate
AndPredicate
OrPredicate
NotPredicate
NullPredicate
ConstantPredicate
```

## Phase 2

```text
BetweenPredicate
InPredicate
ExistsPredicate
LikePredicate
```

## Phase 3

```text
DistinctnessPredicate
BooleanTestPredicate
RegexPredicate
TuplePredicate
```

## Phase 4

```text
JSON predicates
advanced pattern semantics
extension predicates
advanced optimizer rules
```

---

# 399. Initial optimization priority

V1 deberá concentrarse en reglas seguras:

```text
logical flattening
constant predicate simplification
double negation
empty IN normalization
basic dependency analysis
safe duplicate elimination
```

No comenzar con rewrites agresivos.

---

# 400. Later optimization

Posteriormente:

```text
predicate pushdown
range merging
contradiction detection
IN conversion
subquery decorrelation
semi-join planning
anti-join planning
join predicate extraction
```

---

# 401. Relation with Query Model

```text
Query Model
      │
      ▼
Query AST
      │
      ├── Expression AST
      └── Predicate AST
```

El Query Model determina dónde aparece una condición.

El Predicate AST representa qué significa esa condición.

---

# 402. Relation with Semantic Engine

El Predicate System no conoce el schema resuelto.

```text
Predicate AST
      │
      ▼
Semantic Engine
      │
      ├── Symbol Resolution
      ├── Schema Awareness
      ├── Type Inference
      ├── Effective Nullability
      ├── Correlation
      └── Capability Requirements
```

---

# 403. Relation with Optimizer

```text
Semantic Predicate
      │
      ▼
Optimizer
      │
      ├── normalize
      ├── simplify
      ├── classify
      ├── move
      └── rewrite
```

---

# 404. Relation with Planner

```text
Optimized Predicate
       │
       ▼
Logical Planner
       │
       ├── Filter
       ├── Join Condition
       ├── Semi Join
       ├── Anti Join
       └── Correlated Subplan
```

---

# 405. Relation with Compiler

El Compiler recibe una estrategia ya suficientemente resuelta.

```text
Plan Predicate
      │
      ▼
Compiler
      │
      ▼
Dialect
      │
      ▼
Target SQL
```

---

# 406. Relation with Driver

No existe dependencia directa:

```text
Predicate
   X
 Driver
```

---

# 407. Relation with ORM

```text
ORM Query API
      │
      ▼
Predicate Builder
      │
      ▼
Predicate AST
```

El ORM no tendrá un segundo Predicate System.

---

# 408. Relation with Persistence

Optimistic locking podrá generar:

```text
id = :id
AND
version = :expectedVersion
```

mediante Predicate AST.

---

# 409. Optimistic locking example

```text
AndPredicate
├── Comparison(id = p1)
└── Comparison(version = p2)
```

Luego el Persistence Engine usa el Query Engine normal.

---

# 410. Soft delete example

```text
NullPredicate
├── deleted_at
└── IS_NULL
```

inyectado por ORM scope.

---

# 411. Multitenancy example

```text
ComparisonPredicate
├── tenant_id
├── EQUAL
└── Parameter(tenantId)
```

inyectado por integración opcional.

---

# 412. Authorization example

Una policy de acceso por ownership podría producir:

```text
resource.owner_id = current_user_id
```

como Predicate AST si esa integración decide realizar enforcement a nivel de query.

---

# 413. Security caution

No todas las reglas de Authorization deben convertirse automáticamente en predicates de base de datos.

La integración deberá decidir explícitamente cuándo es semánticamente válido.

---

# 414. Master architectural formula

```text
Predicate
=
Immutable Logical Intent
+
Expression Operands
+
Three-Valued Semantics
```

---

# 415. Resolved predicate

```text
Resolved Predicate
=
Predicate
+
Resolved Symbols
+
Resolved Types
+
Effective Nullability
+
Truth Domain
+
Dependencies
+
Correlation
+
Volatility
+
Capability Requirements
```

---

# 416. Optimized predicate

```text
Optimized Predicate
=
Resolved Predicate
+
Semantics-preserving Transformations
```

---

# 417. Planned predicate

```text
Planned Predicate
=
Optimized Predicate
+
Execution Strategy
+
Logical Placement
+
Target Capability Decisions
```

---

# 418. Compiled predicate

```text
Compiled Predicate
=
Planned Predicate
+
Dialect Syntax
+
Binding Layout
```

---

# 419. Separation final

```text
Predicate
    │
    ├── knows logical intent
    ├── knows operands
    └── knows structural options

Predicate does NOT know
    │
    ├── SQL syntax
    ├── PDO
    ├── Connection
    ├── Driver
    ├── current Platform
    ├── current Tenant
    ├── parameter values
    ├── execution result
    └── ORM state
```

---

# 420. Principio de lógica

> VoltStack deberá razonar sobre predicates utilizando la semántica lógica de las bases de datos, incluyendo `UNKNOWN`, y nunca utilizar la lógica booleana de PHP como sustituto.

---

# 421. Principio de portabilidad

> La portabilidad de un predicate se determinará por su semántica y por las capabilities disponibles, no por la existencia de una keyword aparentemente similar en varios motores.

---

# 422. Principio de optimización

> Ninguna optimización de predicates será válida únicamente porque produzca SQL más corto; deberá demostrar que preserva tipos, NULL semantics, three-valued logic, volatility y evaluation semantics.

---

# 423. Principio de seguridad

> Los valores dinámicos pertenecen al Binding System; los identifiers pertenecen al Identifier System; los operadores pertenecen al Semantic Operator Model; ninguno deberá convertirse en SQL mediante concatenación arbitraria.

---

# 424. Principio de extensibilidad

> Una extensión deberá agregar nueva semántica de predicate, no simplemente un mecanismo para insertar operadores SQL desconocidos dentro del core.

---

# 425. Regla maestra final

El Query Predicate System de VoltStack deberá seguir:

```text
Semantic Predicate
        +
Immutable AST
        +
Three-Valued Logic
        +
External Semantic Metadata
        +
Typed Operators
        +
Capability Requirements
        +
Safe Normalization
        +
Semantics-Preserving Optimization
        +
Planner-selected Strategies
        +
Dialect Compilation
```

---

# 426. Resultado arquitectónico

Con `Expression System` y `Predicate System`, VoltStack obtiene:

```text
                  Query AST
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
 Expression System         Predicate System
        │                         │
        │                  Three-Valued Logic
        │                         │
        └────────────┬────────────┘
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
                     │
                     ▼
                  Dialect
```

Esta frontera permitirá que consultas Laravel-like como:

```php
DB::table('users')
    ->where('active', true)
    ->whereNotNull('email')
    ->whereIn('country', ['MX', 'US'])
    ->get();
```

sean internamente transformadas en un modelo:

```text
semantic
typed
immutable
optimizable
portable
capability-aware
secure
```

sin introducir SQL específico de MySQL, MariaDB, PostgreSQL o SQLite dentro del Query Builder.

---

# 427. Próximo documento

El siguiente documento de la arquitectura será:

```text
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

y deberá formalizar la frontera:

```text
Application Values
        │
        ▼
Query Parameters
        │
        ▼
Binding Set
        │
        ▼
Semantic Type Resolution
        │
        ▼
Binding Plan
        │
        ▼
Compiled Parameters
        │
        ▼
Driver Binding
```

incluyendo especialmente:

```text
ParameterId
Scalar Parameters
Collection Parameters
Parameter Type Hints
Type Inference
BindingSet
BindingPlan
Placeholder Allocation
Named vs Positional Parameters
Parameter Expansion
IN Collection Binding
Native Array Binding
Value Conversion
Sensitive Parameter Redaction
Parameter Fingerprints
Binding Validation
Driver-neutral Binding
Prepared Statement Integration
Persistent Runtime Safety
```

manteniendo la regla:

```text
Query Parameter
≠
Runtime Value
≠
SQL Placeholder
≠
Driver Binding
```

---

# 428. Estado final

Con este documento queda definida la segunda pieza fundamental del Query AST:

```text
Expression System
      +
Predicate System
```

preparando la arquitectura para separar completamente:

```text
Query Structure
        │
        ▼
Semantic Values / Conditions
        │
        ▼
Parameters and Bindings
        │
        ▼
Type System
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
        │
        ▼
Execution
```

sin permitir que los detalles de SQL, drivers, conexiones, ORM o runtime contaminen el modelo semántico de consultas.