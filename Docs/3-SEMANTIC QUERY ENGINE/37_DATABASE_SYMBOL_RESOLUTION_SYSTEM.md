# 37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md

# VoltStack Quantum Database
## Symbol Resolution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 37 — Symbol Resolution System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el **Symbol Resolution System** de:

```text
VoltStack/Quantum/Database
```

Su responsabilidad es transformar referencias estructurales presentes en el Query AST en identidades semánticas inequívocas.

Ejemplo:

```sql
SELECT u.name
FROM users AS u
```

El AST puede contener:

```text
ColumnReference
├── qualifier = "u"
└── name = "name"
```

pero todavía no sabe qué representa realmente `u`.

El Symbol Resolution System deberá convertirlo conceptualmente en:

```text
ColumnReference
        │
        ▼
Qualifier "u"
        │
        ▼
RelationSymbol R1
        │
        ▼
ColumnSymbol C4
```

sin modificar el AST.

---

# 2. Objetivo central

El sistema responderá preguntas como:

```text
¿Qué significa este identificador?
¿En qué scope existe?
¿A qué símbolo hace referencia?
¿Es local o externo?
¿Está ocultando otro símbolo?
¿Es ambiguo?
¿Existe?
¿Qué namespace debe consultarse?
```

---

# 3. Regla maestra

> Todo identificador semánticamente relevante utilizado por una consulta deberá resolverse contra un scope y namespace explícitos antes de que las fases posteriores dependan de él.

---

# 4. Separación fundamental

VoltStack mantendrá:

```text
Identifier
≠
Symbol
≠
Reference
≠
Definition
≠
Scope
≠
Namespace
≠
Schema Object
```

---

# 5. Identifier

Un:

```text
Identifier
```

es una representación estructurada de un nombre.

Ejemplos:

```text
users
email
u
orders
total
```

Puede contener:

```text
Identifier
├── canonicalValue
├── originalValue
├── quotingMode
└── normalizationMetadata
```

No representa por sí mismo un objeto semántico.

---

# 6. Definition

Una:

```text
Definition
```

introduce un nombre dentro de un scope.

Ejemplo:

```sql
FROM users AS u
```

introduce:

```text
RelationDefinition("u")
```

---

# 7. Reference

Una:

```text
Reference
```

utiliza un nombre previamente disponible.

Ejemplo:

```sql
SELECT u.email
```

contiene una referencia a:

```text
u
```

---

# 8. Symbol

Un:

```text
Symbol
```

es la identidad semántica asociada a una definición.

Ejemplo:

```text
RelationSymbol
├── id = R1
├── name = u
├── namespace = RELATION
├── scope = Q1
└── origin = users
```

---

# 9. Regla de identidad

Dos symbols con el mismo nombre no son necesariamente el mismo símbolo.

Ejemplo:

```text
Scope Q1 → u → Symbol R1
Scope Q2 → u → Symbol R7
```

Por tanto:

```text
Symbol identity ≠ Symbol name
```

---

# 10. SymbolId

Cada símbolo tendrá una identidad estable durante el análisis:

```text
SymbolId
```

Ejemplos conceptuales:

```text
REL:R1
COL:C8
CTE:C2
PAR:P1
WIN:W4
PROJ:A7
```

---

# 11. SymbolId no depende de memoria

Nunca utilizar:

```text
spl_object_id()
memory address
random UUID generated during analysis
```

como identidad semántica estable.

---

# 12. Identidad determinista

Siempre que sea posible:

```text
SymbolId
=
Query Structure
+
Scope Identity
+
Definition Position
+
Symbol Kind
```

---

# 13. SymbolKind

VoltStack podrá reconocer inicialmente:

```text
RELATION
COLUMN
CTE
PROJECTION
PARAMETER
WINDOW
FUNCTION
TYPE
SEQUENCE
VARIABLE
EXTENSION
```

No todos necesariamente utilizarán exactamente el mismo registry.

---

# 14. Scope

Un:

```text
SemanticScope
```

define el entorno de visibilidad en el que pueden introducirse y resolverse símbolos.

---

# 15. Ejemplo simple

```sql
SELECT u.name
FROM users u
```

produce conceptualmente:

```text
Scope Q1
└── RELATION
    └── u → R1
```

---

# 16. ScopeId

Cada scope tendrá:

```text
ScopeId
```

determinista dentro del Query Artifact.

Ejemplos:

```text
Q1
Q1.SUB1
Q1.CTE1
Q1.SET2
```

La implementación podrá utilizar IDs compactos internamente.

---

# 17. Scope kinds

Podrán existir:

```text
QUERY
SUBQUERY
CTE
DERIVED_RELATION
SET_BRANCH
PROJECTION
AGGREGATE
WINDOW
DML
RETURNING
EXTENSION
```

---

# 18. Scope no equivale a AST node

Un scope normalmente está asociado a una estructura AST, pero:

```text
Scope ≠ AST Node
```

Algunos nodes no crean scope.

Algunas construcciones pueden requerir varios namespaces dentro del mismo scope.

---

# 19. Scope Graph

La estructura completa será:

```text
SemanticScopeGraph
```

Ejemplo:

```text
Q1
├── SUBQUERY Q2
│   └── SUBQUERY Q3
├── CTE Q4
└── DERIVED Q5
```

---

# 20. No mutable parent pointers en AST

La relación:

```text
child scope → parent scope
```

pertenecerá al:

```text
SemanticScopeGraph
```

No se añadirá como estado mutable a los AST nodes.

---

# 21. Scope edge

Conceptualmente:

```text
ScopeEdge
├── child
├── parent
├── kind
└── visibilityPolicy
```

---

# 22. Scope edge kinds

Ejemplos:

```text
LEXICAL_PARENT
CORRELATION_PARENT
CTE_PARENT
DERIVED_RELATION_PARENT
SET_OPERATION_PARENT
DML_PARENT
EXTENSION
```

---

# 23. Scope visibility

No todos los símbolos del parent necesariamente serán visibles desde el child.

Por tanto:

```text
parent scope
```

no implica:

```text
inherit everything
```

---

# 24. VisibilityPolicy

Cada relación de scopes podrá declarar:

```text
VisibilityPolicy
```

Ejemplo:

```text
relations       = visible
parameters      = visible
projectionAlias = not visible
windows         = not visible
```

---

# 25. Namespace

Un:

```text
SymbolNamespace
```

evita mezclar categorías de símbolos incompatibles.

---

# 26. Namespaces iniciales

```text
RELATION
COLUMN
CTE
PROJECTION
PARAMETER
WINDOW
FUNCTION
TYPE
SEQUENCE
EXTENSION
```

---

# 27. Namespaces separados

Una query podría contener simultáneamente:

```text
relation alias: total
projection alias: total
parameter: total
```

sin que necesariamente exista conflicto.

---

# 28. Namespace rule

La colisión se evalúa dentro del namespace correspondiente.

No mediante:

```text
one giant string → symbol map
```

---

# 29. Namespace table

Conceptualmente:

```text
Scope Q1
├── RELATION
│   ├── u
│   └── o
│
├── PROJECTION
│   └── total
│
├── PARAMETER
│   └── minimum
│
└── WINDOW
    └── monthly
```

---

# 30. SymbolTable

La tabla semántica principal será:

```text
SemanticSymbolTable
```

---

# 31. SymbolTable responsibilities

Permitirá:

```text
SymbolId → Symbol
ScopeId + Namespace + Name → candidate symbols
Definition NodeId → SymbolId
Reference NodeId → resolution result
```

---

# 32. SymbolTable no resuelve schema por sí sola

La SymbolTable sabe:

```text
u → relation symbol
```

pero no necesariamente:

```text
u → physical schema public.users
```

Eso se profundizará en:

```text
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
```

---

# 33. Symbol descriptor

Conceptualmente:

```php
interface SemanticSymbol
{
    public function id(): SymbolId;

    public function kind(): SymbolKind;

    public function scopeId(): ScopeId;

    public function namespace(): SymbolNamespace;

    public function declaredName(): Identifier;

    public function origin(): SymbolOrigin;
}
```

---

# 34. SymbolOrigin

Permitirá saber de dónde surgió un símbolo.

Ejemplos:

```text
FROM_RELATION
RELATION_ALIAS
CTE_DEFINITION
DERIVED_RELATION
PROJECTION_ALIAS
PARAMETER_DEFINITION
WINDOW_DEFINITION
SCHEMA_COLUMN
EXTENSION
```

---

# 35. RelationSymbol

Representa una relación visible.

Ejemplo:

```text
RelationSymbol
├── id
├── scopeId
├── alias
├── relationSourceId
├── origin
└── visibility
```

---

# 36. RelationSymbol ≠ SchemaTable

Un `RelationSymbol` puede representar:

```text
physical table
CTE
derived query
VALUES relation
table function
set operation
extension relation
```

Por ello:

```text
RelationSymbol ≠ Table
```

---

# 37. ColumnSymbol

Representa una columna semánticamente visible.

Conceptualmente:

```text
ColumnSymbol
├── id
├── owningRelationSymbol?
├── declaredName
├── outputName
├── origin
└── sourceDefinition
```

---

# 38. ColumnSymbol puede ser derivado

Ejemplo:

```sql
SELECT price * quantity AS total
```

`total` puede generar un símbolo de proyección que no corresponde directamente a una columna física.

---

# 39. CteSymbol

Representará una definición:

```sql
WITH active_users AS (...)
```

como:

```text
CteSymbol
├── id
├── name = active_users
├── scope
├── queryBlock
├── recursive
└── outputRelation?
```

---

# 40. ProjectionSymbol

Ejemplo:

```sql
SELECT SUM(total) AS revenue
```

produce:

```text
ProjectionSymbol
└── revenue
```

---

# 41. ParameterSymbol

Los parámetros tendrán namespace propio.

Ejemplo:

```text
:id
:status
```

Podrán estar representados también por `ParameterId`.

La integración deberá evitar duplicar identidades innecesariamente.

---

# 42. ParameterId vs SymbolId

Preferencia:

```text
ParameterId
```

continúa siendo identidad primaria del parámetro.

El Symbol Resolution System podrá mantener:

```text
Parameter Symbol Reference
→ ParameterId
```

sin crear una segunda identidad incompatible.

---

# 43. WindowSymbol

Ejemplo:

```sql
WINDOW monthly AS (...)
```

produce:

```text
WindowSymbol
└── monthly
```

---

# 44. FunctionSymbol

Las funciones podrán utilizar un mecanismo de resolución relacionado, pero:

```text
query-local symbol lookup
```

y:

```text
FunctionRegistry lookup
```

son operaciones diferentes.

---

# 45. Function namespace

El namespace `FUNCTION` podrá representar nombres visibles/importados, pero las signatures pertenecen al:

```text
FunctionRegistry
```

---

# 46. Type symbols

De manera similar:

```text
TypeSymbol
```

puede identificar una referencia semántica a un tipo, mientras:

```text
QueryTypeRegistry
```

mantiene los descriptors.

---

# 47. Definitions

Toda construcción que introduzca un símbolo producirá conceptualmente:

```text
SymbolDefinition
```

---

# 48. SymbolDefinition

```text
SymbolDefinition
├── definitionId
├── nodeId
├── scopeId
├── namespace
├── identifier
├── symbolKind
├── visibility
└── origin
```

---

# 49. Definition discovery

El sistema recorrerá el AST para identificar definiciones antes de resolver referencias que dependan de ellas.

---

# 50. Registration pipeline

```text
AST
 │
 ▼
Definition Discovery
 │
 ▼
SymbolDefinition
 │
 ▼
Conflict Validation
 │
 ▼
Symbol Creation
 │
 ▼
Namespace Registration
```

---

# 51. Duplicate definition

Ejemplo:

```sql
FROM users u
JOIN orders u ON ...
```

introduce dos aliases:

```text
u
u
```

en el mismo namespace/scope.

Resultado:

```text
DuplicateSymbolDefinitionException
```

---

# 52. Duplicate rules

Las reglas dependerán del namespace.

No habrá una única regla universal.

---

# 53. Case sensitivity

La comparación de identifiers dependerá de:

```text
IdentifierSemantics
```

y del contexto de Dialect/Platform cuando corresponda.

---

# 54. Original vs canonical name

Un Identifier podrá conservar:

```text
original = "UserName"
canonical = "username"
```

cuando las reglas aplicables así lo determinen.

---

# 55. Quoted identifiers

Debe distinguirse:

```text
users
```

de un identifier quoted cuando la plataforma tenga semántica diferente.

---

# 56. No strtolower global

Nunca implementar:

```php
$name = strtolower($identifier);
```

como política universal.

---

# 57. IdentifierCanonicalizer

La normalización de nombres utilizará:

```text
IdentifierCanonicalizer
```

configurado mediante semántica explícita.

---

# 58. Canonical key

Conceptualmente:

```text
SymbolLookupKey
=
ScopeId
+
Namespace
+
CanonicalIdentifier
```

---

# 59. Registration ordering

El registro será determinista.

Ejemplo:

```text
AST definition traversal order
```

podrá determinar IDs internos estables.

---

# 60. Reference model

Toda referencia semántica podrá representarse mediante:

```text
SymbolReference
```

---

# 61. SymbolReference

Conceptualmente:

```text
SymbolReference
├── nodeId
├── referenceKind
├── identifier
├── qualifier?
├── originatingScope
└── expectedNamespace
```

---

# 62. ReferenceKind

Ejemplos:

```text
RELATION_REFERENCE
COLUMN_REFERENCE
CTE_REFERENCE
PROJECTION_REFERENCE
WINDOW_REFERENCE
PARAMETER_REFERENCE
FUNCTION_REFERENCE
TYPE_REFERENCE
EXTENSION_REFERENCE
```

---

# 63. Resolution result

Toda resolución devolverá:

```text
SymbolResolutionResult
```

---

# 64. Estados

```text
RESOLVED
AMBIGUOUS
UNKNOWN
DEFERRED
INVALID
```

---

# 65. Resolved

```text
ResolvedSymbolReference
├── referenceId
├── symbolId
├── scopeDistance
├── resolutionPath
└── correlation?
```

---

# 66. Ambiguous

```text
AmbiguousSymbolReference
├── reference
├── candidates
└── diagnostic
```

---

# 67. Unknown

```text
UnknownSymbolReference
├── reference
├── searchedScopes
├── namespace
└── diagnostic
```

---

# 68. Deferred

Sólo cuando existe una dependencia semántica pendiente conocida.

Ejemplo:

```text
derived relation output columns not finalized yet
```

---

# 69. Invalid

Para referencias cuyo formato/contexto no permite resolución semántica válida.

---

# 70. Resolution path

Cada resultado podrá conservar:

```text
ResolutionPath
```

Ejemplo:

```text
Q3
→ Q2
→ Q1
→ RELATION:u
```

---

# 71. Scope distance

Conceptualmente:

```text
0 = current scope
1 = direct parent
2 = grandparent
...
```

---

# 72. Local reference

```text
scopeDistance = 0
```

---

# 73. Outer reference

```text
scopeDistance > 0
```

---

# 74. Correlated reference

Una referencia externa dentro de una subquery puede clasificarse como:

```text
CORRELATED
```

si su uso crea dependencia con el outer query.

---

# 75. Correlation no es simple lookup

No toda consulta a parent scope implica necesariamente la misma semántica.

El sistema registrará:

```text
OuterSymbolDependency
```

para análisis posterior.

---

# 76. Qualified column resolution

Ejemplo:

```sql
u.email
```

Pipeline:

```text
"u"
 │
 ▼
resolve RELATION namespace
 │
 ▼
RelationSymbol R1
 │
 ▼
resolve "email" against relation output
 │
 ▼
ColumnSymbol C7
```

---

# 77. Qualified reference

La resolución deberá ocurrir en dos etapas:

```text
qualifier resolution
+
member resolution
```

---

# 78. Member namespace

Una relación tendrá conceptualmente:

```text
RelationOutputNamespace
```

para sus columnas visibles.

---

# 79. Relation output

```text
RelationOutput
├── column 1
├── column 2
├── column 3
└── ...
```

Cada columna tendrá identidad semántica.

---

# 80. Unqualified column resolution

Ejemplo:

```sql
SELECT id
FROM users u
JOIN orders o ON ...
```

Debe buscar:

```text
users.id
orders.id
```

---

# 81. Unqualified unique match

Si sólo existe:

```text
users.id
```

resultado:

```text
RESOLVED
```

---

# 82. Unqualified ambiguous match

Si existen:

```text
users.id
orders.id
```

resultado:

```text
AMBIGUOUS
```

---

# 83. No first-match

Nunca:

```text
first relation containing "id" wins
```

---

# 84. CandidateSet

La resolución producirá primero:

```text
SymbolCandidateSet
```

---

# 85. Candidate pipeline

```text
Reference
   │
   ▼
Candidate Discovery
   │
   ▼
Visibility Filtering
   │
   ▼
Namespace Filtering
   │
   ▼
Qualification Filtering
   │
   ▼
Semantic Eligibility
   │
   ▼
CandidateSet
```

---

# 86. Candidate count

```text
0 → UNKNOWN
1 → RESOLVED
>1 → AMBIGUOUS
```

salvo reglas especializadas.

---

# 87. Visibility before ambiguity

Los candidatos invisibles no deberán generar falsa ambigüedad.

---

# 88. Qualification

Una referencia calificada restringirá candidatos.

Ejemplo:

```text
u.id
```

no deberá considerar:

```text
o.id
```

después de resolver `u`.

---

# 89. Alias precedence

Cuando una relación posee alias:

```sql
FROM users AS u
```

el nombre físico:

```text
users
```

no necesariamente permanecerá visible como qualifier.

---

# 90. Alias visibility policy

Esto será determinado explícitamente por:

```text
RelationAliasVisibilityPolicy
```

---

# 91. No platform assumptions scattered

Las diferencias de semántica deberán centralizarse.

No:

```php
if ($platform === 'mysql') { ... }
```

dentro de cada resolver.

---

# 92. Projection aliases

La visibilidad de aliases de proyección depende del contexto.

Ejemplo:

```sql
SELECT price * quantity AS total
ORDER BY total
```

puede permitir:

```text
ORDER BY → projection alias total
```

---

# 93. WHERE visibility

El mismo alias:

```text
total
```

puede no estar visible en:

```text
WHERE
```

según las reglas semánticas elegidas/target.

---

# 94. ClauseResolutionPolicy

VoltStack utilizará:

```text
ClauseResolutionPolicy
```

para definir qué namespaces son visibles desde:

```text
SELECT
WHERE
JOIN_ON
GROUP_BY
HAVING
ORDER_BY
WINDOW
RETURNING
```

---

# 95. ResolutionContext

Cada lookup recibirá:

```text
SymbolResolutionContext
```

---

# 96. Contenido

```text
SymbolResolutionContext
├── scopeId
├── clauseContext
├── expectedNamespace
├── qualification
├── visibilityPolicy
├── identifierSemantics
└── semanticMode
```

---

# 97. No current clause global

Nunca:

```php
SymbolResolver::$currentClause
```

---

# 98. Scope lookup algorithm

Conceptualmente:

```text
current scope
     │
     ├── match found uniquely → resolve
     │
     ├── ambiguity → error
     │
     └── no match
            │
            ▼
      parent visibility?
            │
       yes ─┴─ no
       │        │
       ▼        ▼
    parent    unknown
```

---

# 99. Nearest scope wins

Cuando shadowing sea permitido:

```text
nearest visible scope
```

tendrá precedencia.

---

# 100. Ejemplo shadowing

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM archived_users u
    WHERE u.id = 10
)
```

Dentro de la subquery:

```text
u
```

resuelve a:

```text
archived_users
```

---

# 101. Outer u oculto

El `u` externo continúa existiendo, pero queda shadowed para referencias no calificadas por scope identity.

---

# 102. Shadow record

El sistema podrá registrar:

```text
ShadowRelation
├── innerSymbol
└── shadowedOuterSymbol
```

para diagnostics/debug.

---

# 103. Explicit outer access

VoltStack no inventará sintaxis especial para acceder a un símbolo shadowed.

Si SQL semantics no permiten distinguirlo:

```text
no access
```

---

# 104. Correlated example

```sql
SELECT u.id
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

---

# 105. Inner scope

```text
Scope Q2
├── local relation o
└── parent Q1
```

---

# 106. Resolution

```text
o.user_id
→ Q2
→ RelationSymbol(o)

u.id
→ Q2 no u
→ parent Q1
→ RelationSymbol(u)
```

---

# 107. Correlation record

```text
CorrelationReference
├── innerScope = Q2
├── outerScope = Q1
├── symbol = users.id
└── distance = 1
```

---

# 108. Correlation set

Cada subquery podrá producir:

```text
CorrelationDependencySet
```

---

# 109. Multiple levels

```text
Q3
→ Q2
→ Q1
```

Si Q3 referencia Q1:

```text
distance = 2
```

---

# 110. Scope boundary restrictions

Algunos scope edges podrán bloquear determinados lookups.

Ejemplo conceptual:

```text
Derived table
```

puede no tener acceso automático a outer relations salvo que la semántica LATERAL lo permita.

---

# 111. Lateral visibility

Para:

```text
LATERAL
```

se utilizará una policy explícita:

```text
LateralVisibilityPolicy
```

---

# 112. No implicit lateral behavior

Una derived relation normal no deberá convertirse accidentalmente en correlated relation.

---

# 113. CTE scope

Los CTE tendrán reglas de visibilidad específicas.

---

# 114. Example

```sql
WITH a AS (...),
     b AS (SELECT * FROM a)
SELECT * FROM b
```

`b` puede ver:

```text
a
```

si la política de CTE ordering lo permite.

---

# 115. CTE forward reference

Ejemplo:

```sql
WITH a AS (SELECT * FROM b),
     b AS (...)
```

podrá ser:

```text
invalid
```

salvo semántica explícita que permita dicha dependencia.

---

# 116. Recursive CTE

Para:

```sql
WITH RECURSIVE tree AS (...)
```

`tree` puede ser visible dentro de su propio cuerpo bajo reglas especiales.

---

# 117. Recursive symbol registration

Por ello:

```text
CTE symbol registration
```

puede ocurrir antes del análisis completo de su body.

---

# 118. Recursive reference ≠ ordinary reference

Debe registrarse:

```text
RecursiveCteReference
```

o metadata semántica equivalente.

---

# 119. CTE dependency graph

Las resoluciones alimentarán:

```text
CteDependencyGraph
```

---

# 120. Projection scope

Una projection podrá generar symbols después de analizar sus expressions.

---

# 121. Projection name

Puede provenir de:

```text
explicit alias
column name
derived/generated name
anonymous output
```

---

# 122. Anonymous projection

No toda proyección necesita ser referenciable por nombre.

---

# 123. Projection Symbol eligibility

Sólo outputs nombrados y visibles deberán registrarse en namespaces que permitan lookup.

---

# 124. Duplicate projection names

Podrán ser válidos en algunos result sets:

```sql
SELECT u.id, o.id
```

pero pueden ser ambiguos para:

```text
ORDER BY id
```

---

# 125. Output naming ≠ lookup uniqueness

Por tanto:

```text
duplicate output labels
```

no necesariamente implican query inválida.

---

# 126. Positional references

Construcciones como:

```sql
ORDER BY 1
```

no deberán fingirse como SymbolReference textual.

---

# 127. PositionReference

Se modelará separadamente:

```text
ProjectionPositionReference
```

---

# 128. Wildcards

```text
*
u.*
```

requieren resolución especializada.

---

# 129. WildcardReference

Conceptualmente:

```text
WildcardReference
├── qualifier?
└── scope
```

---

# 130. Unqualified wildcard

```text
*
```

expande relaciones visibles según reglas de projection.

---

# 131. Qualified wildcard

```text
u.*
```

primero resuelve:

```text
u → RelationSymbol
```

y después obtiene su output relation.

---

# 132. Wildcard expansion timing

La expansión deberá ocurrir cuando exista información suficiente sobre los outputs de las relaciones.

---

# 133. No AST mutation requirement

La expansión podrá producir un artifact normalizado/semántico derivado.

No es necesario modificar el AST original.

---

# 134. Deferred wildcard

Si:

```text
u
```

es una derived relation cuyo output todavía no está resuelto:

```text
wildcard resolution = DEFERRED
```

---

# 135. Ambiguous qualifier

Aunque raro, si las reglas permiten múltiples candidatos con mismo qualifier:

```text
AMBIGUOUS_RELATION_REFERENCE
```

---

# 136. Schema-qualified references

Ejemplo:

```text
public.users
```

no debe confundirse automáticamente con:

```text
relationAlias.column
```

---

# 137. Identifier path

Se utilizará una representación estructurada:

```text
IdentifierPath
```

---

# 138. IdentifierPath

Ejemplos:

```text
users
u.email
public.users
catalog.public.users
```

---

# 139. Path interpretation

La interpretación dependerá de:

```text
ReferenceKind
+
ClauseContext
+
ResolutionPolicy
```

No simplemente del número de segmentos.

---

# 140. NameResolver

No deberá existir un parser ad hoc como:

```php
explode('.', $name);
```

en la capa semántica.

El AST deberá haber estructurado previamente los identifiers.

---

# 141. Relation name resolution

Una relación no aliased:

```sql
FROM users
```

podrá introducir un relation symbol visible como:

```text
users
```

según la policy correspondiente.

---

# 142. Schema-aware relation reference

La búsqueda conceptual será:

```text
query-local symbols
        │
        ▼
CTE namespace
        │
        ▼
relation definitions
        │
        ▼
schema-aware resolver
```

según contexto.

---

# 143. Local definitions before physical schema

Si existe:

```text
CTE users
```

y también tabla física:

```text
users
```

la política deberá determinar precedencia.

---

# 144. CTE shadowing table

Comúnmente:

```text
CTE users
```

puede ocultar:

```text
physical users table
```

dentro del scope aplicable.

---

# 145. Explicit physical qualification

El usuario podrá necesitar una referencia más explícita si quiere la tabla física.

Esto dependerá de las reglas de Platform/Dialect.

---

# 146. Schema-aware resolution boundary

El Symbol Resolver no deberá contener toda la lógica del schema.

Delegará a:

```text
SchemaAwareQueryResolver
```

---

# 147. Interface conceptual

```php
interface SchemaAwareQueryResolverInterface
{
    public function resolveRelation(
        RelationReference $reference,
        SchemaResolutionContext $context,
    ): SchemaRelationResolutionResult;
}
```

---

# 148. Query-local resolution first

La resolución deberá priorizar correctamente:

```text
query-local semantic definitions
```

antes de consultar el schema cuando la semántica lo requiera.

---

# 149. No database query

`SchemaAwareQueryResolver` trabajará sobre:

```text
SchemaView
```

No sobre una conexión activa.

---

# 150. Symbol resolution phases

La resolución completa podrá dividirse en:

```text
1. Scope discovery
2. Definition discovery
3. Symbol registration
4. Local reference resolution
5. Schema relation resolution
6. Relation output construction
7. Column resolution
8. Outer reference resolution
9. Deferred resolution
10. Final verification
```

---

# 151. Multi-pass necessity

Esto evita exigir que:

```text
everything is already known
```

durante el primer recorrido.

---

# 152. Deferred queue

Podrá existir:

```text
DeferredSymbolResolutionQueue
```

---

# 153. Deferred entry

```text
DeferredResolution
├── referenceId
├── reason
├── dependency
├── retryPhase
└── attempts
```

---

# 154. Deferred reason

Ejemplos:

```text
WAITING_FOR_DERIVED_RELATION_OUTPUT
WAITING_FOR_CTE_OUTPUT
WAITING_FOR_SET_OUTPUT
WAITING_FOR_EXTENSION_SYMBOLS
```

---

# 155. No arbitrary retry

Cada deferred resolution deberá declarar:

```text
what evidence is missing
```

---

# 156. Retry only on dependency change

No:

```text
try again until it works
```

Sino:

```text
dependency finalized
→ retry affected references
```

---

# 157. Unresolved final state

Antes de crear `SemanticQueryArtifact`:

```text
required unresolved references = 0
```

---

# 158. Optional unresolved references

Sólo extensiones explícitamente diseñadas para late resolution podrán conservar referencias especiales, si las fases posteriores entienden formalmente ese estado.

V1 deberá ser conservador.

---

# 159. Recommended V1 rule

```text
No unresolved core symbols leave Semantic Analysis.
```

---

# 160. SymbolResolutionTable

El resultado de todas las referencias se almacenará en:

```text
SymbolResolutionTable
```

---

# 161. Mapping

Conceptualmente:

```text
ReferenceNodeId → ResolvedSymbolReference
```

---

# 162. AST remains clean

Ejemplo:

```text
AST Node N15
```

continúa conteniendo:

```text
u.id
```

mientras:

```text
SymbolResolutionTable[N15]
```

contiene:

```text
ColumnSymbol C8
```

---

# 163. Benefits

Esto permite:

- AST reusable;
- multiple target analyses;
- no mutable annotations;
- cacheable semantic artifacts;
- concurrent analysis;
- clean phase separation.

---

# 164. Resolution provenance

Cada resolution podrá registrar:

```text
ResolutionProvenance
```

---

# 165. Provenance fields

```text
originatingScope
resolvedScope
namespace
lookupPath
qualificationMode
policy
schemaFallbackUsed
extensionUsed
```

---

# 166. Provenance usefulness

Será útil para:

- diagnostics;
- debugger;
- explain semantic;
- optimizer;
- extension debugging.

---

# 167. Symbol visibility

Un símbolo podrá declarar:

```text
LOCAL
INHERITABLE
CORRELATABLE
EXPORTABLE
RECURSIVE
INTERNAL
```

como características, no necesariamente como un único enum excluyente.

---

# 168. VisibilityDescriptor

Preferir composición:

```text
VisibilityDescriptor
├── local
├── childScopes
├── correlation
├── export
└── recursiveSelfReference
```

---

# 169. Relation output visibility

Las columnas de una relación pueden ser visibles:

```text
through relation qualification
```

y quizá también:

```text
through unqualified lookup
```

según scope.

---

# 170. Hidden columns

El modelo permitirá:

```text
hidden/system columns
```

sin asumir que todas son seleccionables automáticamente.

---

# 171. Extension columns

Extensiones podrán contribuir columns virtuales mediante contratos explícitos.

---

# 172. Extension symbols

El sistema permitirá:

```text
ExtensionSymbol
```

sin obligar al core a conocer cada categoría futura.

---

# 173. SymbolKindId

Para extensibilidad podrá preferirse:

```text
SymbolKindId
```

sobre un enum cerrado universal.

Ejemplos:

```text
core.relation
core.column
core.cte
vendor.postgresql.some_symbol
extension.search.vector
```

---

# 174. Core optimized kinds

Los símbolos core podrán seguir utilizando representaciones especializadas y eficientes.

---

# 175. Extension descriptor

```text
SymbolKindDescriptor
├── id
├── namespace
├── definitionPolicy
├── lookupPolicy
├── visibilityPolicy
├── fingerprintPolicy
└── diagnosticFormatter
```

---

# 176. Frozen registry

El:

```text
SymbolKindRegistry
```

se configurará durante bootstrap y se congelará.

---

# 177. No runtime arbitrary registration

No permitir:

```text
request A registers symbol kind
request B sees it
```

---

# 178. Symbol resolver architecture

```text
                 SymbolResolutionCoordinator
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ScopeResolver    DefinitionResolver   ReferenceResolver
          │                │                │
          ▼                ▼                ▼
      ScopeGraph       SymbolTable      ResolutionTable
                           │                │
                           └───────┬────────┘
                                   ▼
                         Semantic Symbol Model
```

---

# 179. Coordinator responsibility

El:

```text
SymbolResolutionCoordinator
```

coordina fases.

No debe implementar todas las reglas.

---

# 180. Specialized resolvers

Ejemplos:

```text
RelationSymbolResolver
ColumnSymbolResolver
CteSymbolResolver
ProjectionSymbolResolver
WindowSymbolResolver
ParameterSymbolResolver
WildcardResolver
OuterReferenceResolver
ExtensionSymbolResolver
```

---

# 181. Resolver registry

Podrá existir:

```text
SymbolResolverRegistry
```

especializado por:

```text
ReferenceKind
```

---

# 182. No service locator

El registry no será:

```text
get(any class name)
```

Sino:

```text
ReferenceKind → typed resolver descriptor
```

---

# 183. Resolution pipeline

```text
Reference Node
      │
      ▼
Reference Classifier
      │
      ▼
Resolution Context
      │
      ▼
Specialized Resolver
      │
      ▼
Candidate Discovery
      │
      ▼
Visibility Rules
      │
      ▼
Ambiguity Rules
      │
      ▼
Resolution Result
      │
      ▼
SymbolResolutionTable
```

---

# 184. Relation resolution pipeline

```text
RelationReference
      │
      ▼
CTE / Local Relation Lookup
      │
      ├── found → RelationSymbol
      │
      └── not found
              │
              ▼
      SchemaAwareResolver
              │
              ▼
        RelationSymbol
```

---

# 185. Column resolution pipeline

```text
ColumnReference
      │
      ├── qualified?
      │       │
      │       ├── yes
      │       │    ▼
      │       │ Resolve qualifier
      │       │    ▼
      │       │ Relation output lookup
      │       │
      │       └── no
      │            ▼
      │       Visible relation outputs
      │            ▼
      │       Candidate collection
      │
      ▼
Unique / Ambiguous / Unknown
```

---

# 186. Diagnostic architecture

Errores de resolución deberán utilizar códigos estables.

---

# 187. Suggested codes

```text
DB-SYM-001 UNKNOWN_SYMBOL
DB-SYM-002 AMBIGUOUS_SYMBOL
DB-SYM-003 DUPLICATE_SYMBOL
DB-SYM-004 INVALID_QUALIFIER
DB-SYM-005 UNKNOWN_RELATION
DB-SYM-006 UNKNOWN_COLUMN
DB-SYM-007 AMBIGUOUS_COLUMN
DB-SYM-008 UNKNOWN_CTE
DB-SYM-009 INVALID_OUTER_REFERENCE
DB-SYM-010 INVALID_SCOPE_REFERENCE
DB-SYM-011 ILLEGAL_SHADOWING
DB-SYM-012 UNRESOLVED_DEFERRED_REFERENCE
DB-SYM-013 INVALID_RECURSIVE_REFERENCE
DB-SYM-014 UNKNOWN_WINDOW
DB-SYM-015 INVALID_PROJECTION_REFERENCE
```

---

# 188. Unknown column diagnostic

Ejemplo:

```text
DB-SYM-006

Unknown column "emial" on relation "users".
```

Podrá incluir:

```text
Did you mean "email"?
```

---

# 189. Suggestion engine

Las sugerencias deberán:

- ser bounded;
- no cambiar semántica;
- no auto-corregir;
- no consultar recursos externos.

---

# 190. Typo distance

Podrá utilizarse una métrica simple como:

```text
Levenshtein distance
```

sobre un conjunto pequeño de candidatos visibles.

---

# 191. Diagnostic candidate limit

No calcular sugerencias sobre miles de columnas sin límite.

---

# 192. Ambiguous column diagnostic

Ejemplo:

```text
DB-SYM-007

Column "id" is ambiguous.

Candidates:
- users.id
- orders.id

Qualify the column explicitly.
```

---

# 193. Duplicate alias diagnostic

```text
DB-SYM-003

Relation alias "u" is defined more than once in the same query scope.
```

---

# 194. Invalid outer reference

```text
DB-SYM-009

Relation "u" exists in an outer scope but is not visible from this derived relation.

Consider using a supported lateral relation when appropriate.
```

---

# 195. Source locations

Si están disponibles:

```text
definition location
reference location
conflicting definition location
```

deberán incluirse en diagnostics.

---

# 196. Security

Los nombres de schema pueden ser sensibles en ciertos entornos.

La política diagnóstica deberá permitir:

```text
schema identifier redaction
```

cuando corresponda.

---

# 197. No runtime values

El Symbol Resolution System no necesita:

```text
parameter runtime values
```

---

# 198. No credentials

Tampoco:

```text
database credentials
```

---

# 199. No active connection

Tampoco:

```text
PDO
NativeConnection
ConnectionLease
```

---

# 200. No EntityManager

Tampoco deberá depender de:

```text
EntityManager
UnitOfWork
IdentityMap
```

---

# 201. ORM integration

ORM puede proporcionar:

```text
ORM Metadata View
```

que posteriormente ayude a construir Query AST o semantic metadata.

Pero Symbol Resolution seguirá siendo independiente del ORM runtime.

---

# 202. Schema integration

El único acceso requerido será mediante interfaces read-only como:

```text
SchemaView
RelationDescriptor
ColumnDescriptor
```

---

# 203. Persistent runtime safety

Todos los estados mutables:

```text
ScopeGraphBuilder
SymbolTableBuilder
ResolutionTableBuilder
DeferredResolutionQueue
CandidateWorkspace
```

serán:

```text
operation-scoped
```

---

# 204. Shared immutable objects

Podrán compartirse:

```text
SymbolKindRegistry
IdentifierSemantics
ClauseResolutionPolicies
ResolverDescriptors
```

si están congelados.

---

# 205. Forbidden global state

No existirán:

```text
SymbolResolver::$currentScope
SymbolResolver::$currentQuery
SymbolResolver::$currentAlias
SymbolResolver::$currentTenant
SymbolResolver::$currentSchema
```

---

# 206. Concurrent queries

Debe ser seguro:

```text
Fiber A → Query A → Scope Q1
Fiber B → Query B → Scope Q1
```

Aunque ambos utilicen internamente IDs locales como `Q1`.

Su identidad pertenece al artifact/operation correspondiente.

---

# 207. No cross-operation identity assumption

`Q1` de una consulta no es igual a `Q1` de otra consulta.

---

# 208. Operation identity

Cuando sea necesario para diagnostics internos:

```text
QueryOperationId + ScopeId
```

puede identificar globalmente durante una operación.

---

# 209. Symbol cache

La resolución query-local normalmente será suficientemente barata y ligada al artifact.

No se recomienda un:

```text
global SymbolResolutionCache
```

independiente.

---

# 210. Semantic artifact caching

El resultado completo podrá cachearse como parte del:

```text
SemanticQueryArtifact
```

si su fingerprint es válido.

---

# 211. Schema changes

Si cambia el schema relevante:

```text
semantic artifact
```

deberá invalidarse mediante fingerprints/versiones apropiadas.

---

# 212. Identifier semantics changes

Si cambia:

```text
target platform identifier semantics
```

también puede ser necesaria invalidación.

---

# 213. Resolution fingerprint

Conceptualmente:

```text
SymbolResolutionFingerprint
=
QueryStructureFingerprint
+
ScopePolicyFingerprint
+
IdentifierSemanticsFingerprint
+
SchemaSemanticFingerprint
+
ExtensionSymbolRegistryFingerprint
```

---

# 214. Runtime values excluded

No incluir:

```text
parameter values
request IDs
trace IDs
```

---

# 215. Query labels excluded

Un:

```text
QueryLabel
```

observacional tampoco deberá modificar symbol resolution.

---

# 216. Resolution determinism

Mismo:

```text
AST
+
Scope Policy
+
Identifier Semantics
+
Schema View
+
Extension Registry
```

debe producir:

```text
same symbol resolution
```

---

# 217. Deterministic ambiguity ordering

Los candidatos mostrados en diagnostics deberán ordenarse establemente.

---

# 218. Candidate ranking

No deberá utilizar heurística para resolver ambigüedad real.

La heurística sólo podrá ayudar a:

```text
diagnostic suggestions
```

---

# 219. Explicit ambiguity principle

Si semánticamente existen dos candidatos válidos:

```text
AMBIGUOUS
```

No:

```text
probably this one
```

---

# 220. Resolution complexity

Para evitar:

```text
every column lookup
×
every relation
×
every scope
```

se utilizarán índices internos.

---

# 221. Scope namespace index

Conceptualmente:

```text
ScopeNamespaceIndex
```

mapea:

```text
CanonicalIdentifier
→ candidate SymbolIds
```

---

# 222. Relation column index

```text
RelationColumnIndex
```

mapea:

```text
RelationSymbolId
+
CanonicalColumnName
→ ColumnSymbolId(s)
```

---

# 223. Unqualified column index

Podrá construirse:

```text
VisibleColumnIndex
```

para un scope:

```text
column name
→ visible candidates
```

---

# 224. Lazy index

Estos índices podrán crearse lazy si mejora memoria.

---

# 225. Complexity target

Lookup normal deberá aproximarse a:

```text
O(1)
```

o:

```text
O(scope depth)
```

en casos de outer lookup.

---

# 226. Scope depth budget

El Query Processing Budget podrá limitar:

```text
maximum scope depth
```

---

# 227. Symbol count budget

También:

```text
maximum symbols
```

---

# 228. Candidate count budget

Y:

```text
maximum candidate expansion
```

para queries generadas dinámicamente.

---

# 229. Wildcard expansion budget

Especialmente:

```text
SELECT *
```

sobre relaciones extremadamente amplias.

---

# 230. Failure on budget

Debe producir un diagnostic controlado:

```text
DB-SYM-BUDGET-EXCEEDED
```

No memory exhaustion.

---

# 231. Testing strategy

El sistema deberá probarse por capas.

---

# 232. Scope tests

Casos:

```text
root scope
nested subquery
multiple nested subqueries
derived relation
CTE
recursive CTE
set operation
DML
```

---

# 233. Namespace tests

Verificar:

```text
same name / different namespace
same name / same namespace
```

---

# 234. Relation alias tests

```text
unaliased relation
aliased relation
duplicate alias
shadowed alias
outer alias
```

---

# 235. Column tests

```text
qualified column
unqualified unique column
unqualified ambiguous column
unknown column
qualified unknown column
```

---

# 236. Correlation tests

```text
direct outer reference
two-level outer reference
shadowed outer reference
blocked outer reference
lateral access
```

---

# 237. CTE tests

```text
previous CTE reference
invalid forward reference
recursive self reference
invalid recursion
shadow physical table
```

---

# 238. Projection tests

```text
explicit alias
duplicate aliases
ORDER BY alias
WHERE alias visibility
position reference
```

---

# 239. Wildcard tests

```text
*
u.*
unknown qualifier.*
derived relation.*
deferred wildcard
```

---

# 240. Identifier tests

```text
quoted
unquoted
case-sensitive
case-insensitive
schema-qualified
catalog-qualified
```

---

# 241. Persistent runtime tests

```text
Query A defines alias u
RESET
Query B does not define u
```

Query B nunca deberá ver el `u` de A.

---

# 242. Concurrent resolution tests

Ejecutar múltiples analyses simultáneamente con:

```text
same alias names
different schemas
different targets
different extension registries
```

sin contaminación.

---

# 243. Determinism tests

La misma consulta deberá producir los mismos:

```text
ScopeIds
SymbolIds
ResolutionResults
Fingerprints
```

cuando los inputs semánticos sean iguales.

---

# 244. Architecture tests

Deberán impedir imports hacia:

```text
PDO
NativeConnection
ConnectionLease
Transaction
EntityManager
UnitOfWork
IdentityMap
HTTP Request
Session
Service Container
```

---

# 245. Property-based tests

Podrán generar:

```text
nested scopes
aliases
column sets
shadowing structures
```

para comprobar invariants.

---

# 246. Fuzzing

Especialmente útil para:

```text
deep scope graphs
large alias sets
duplicate names
ambiguous columns
wildcard expansion
```

---

# 247. Invariants

## DB-SYM-001

Todo Symbol tendrá `SymbolId`.

## DB-SYM-002

Todo Symbol pertenecerá a un scope o registry explícitamente definido.

## DB-SYM-003

Todo Symbol tendrá namespace.

## DB-SYM-004

Symbol identity no dependerá únicamente del nombre.

## DB-SYM-005

Symbol IDs serán deterministas dentro del artifact.

## DB-SYM-006

AST nodes no almacenarán mutable resolved symbols.

## DB-SYM-007

Scope relations vivirán fuera del AST.

## DB-SYM-008

Todo scope tendrá `ScopeId`.

## DB-SYM-009

Todo scope tendrá kind.

## DB-SYM-010

Todo child scope tendrá relación explícita con su parent cuando exista.

## DB-SYM-011

Parent scope no implica visibilidad universal.

## DB-SYM-012

Visibility será explícita.

## DB-SYM-013

Namespaces serán independientes.

## DB-SYM-014

No existirá un único global string-symbol map.

## DB-SYM-015

Definitions se registrarán antes de referencias dependientes cuando corresponda.

## DB-SYM-016

Duplicate definitions se validarán por namespace.

## DB-SYM-017

Forward references deberán ser explícitamente soportadas.

## DB-SYM-018

CTE recursion tendrá reglas explícitas.

## DB-SYM-019

Toda referencia tendrá originating scope.

## DB-SYM-020

Toda referencia tendrá expected namespace o resolution strategy explícita.

## DB-SYM-021

Toda resolución terminará en un estado conocido.

## DB-SYM-022

Core references no podrán quedar unresolved después de finalization.

## DB-SYM-023

Deferred resolution tendrá causa explícita.

## DB-SYM-024

Deferred resolution tendrá dependencia explícita.

## DB-SYM-025

No existirán retry loops arbitrarios.

## DB-SYM-026

Ambigüedad real producirá error.

## DB-SYM-027

No existirá first-match resolution.

## DB-SYM-028

Nearest scope podrá ganar sólo bajo reglas de shadowing válidas.

## DB-SYM-029

Shadowing será explícito.

## DB-SYM-030

Outer lookup respetará scope boundaries.

## DB-SYM-031

Correlated references serán identificables.

## DB-SYM-032

Correlation depth será derivable.

## DB-SYM-033

Derived relations no tendrán acceso outer implícito.

## DB-SYM-034

Lateral visibility será explícita.

## DB-SYM-035

Qualified column lookup resolverá primero el qualifier.

## DB-SYM-036

Qualified column lookup sólo consultará la relación resuelta.

## DB-SYM-037

Unqualified lookup considerará únicamente relations visibles.

## DB-SYM-038

Invisible candidates no producirán falsa ambigüedad.

## DB-SYM-039

Relation alias visibility será política explícita.

## DB-SYM-040

Projection alias visibility será clause-aware.

## DB-SYM-041

Window name visibility será explícita.

## DB-SYM-042

Position references no serán falsos symbol names.

## DB-SYM-043

Wildcard references tendrán resolución especializada.

## DB-SYM-044

Wildcard expansion requerirá relation outputs conocidos.

## DB-SYM-045

Wildcard expansion podrá diferirse de manera controlada.

## DB-SYM-046

Schema-qualified names no se interpretarán mediante heurísticas frágiles.

## DB-SYM-047

Identifier paths estarán estructurados.

## DB-SYM-048

Identifier canonicalization será centralizada.

## DB-SYM-049

No se utilizará lowercase universal.

## DB-SYM-050

Quoted identifiers conservarán su semántica.

## DB-SYM-051

Query-local definitions podrán preceder schema lookup según policy.

## DB-SYM-052

Schema lookup utilizará `SchemaView`.

## DB-SYM-053

Symbol Resolution no abrirá conexiones.

## DB-SYM-054

Symbol Resolution no ejecutará SQL.

## DB-SYM-055

Symbol Resolution no dependerá del Driver.

## DB-SYM-056

Symbol Resolution no dependerá del Executor.

## DB-SYM-057

Symbol Resolution no dependerá del ORM runtime.

## DB-SYM-058

Symbol Resolution no dependerá del EntityManager.

## DB-SYM-059

Symbol Resolution no dependerá del UnitOfWork.

## DB-SYM-060

Symbol Resolution no dependerá del HTTP Request.

## DB-SYM-061

Symbol Resolution no utilizará Service Container como locator.

## DB-SYM-062

Todo mutable resolution state será operation-scoped.

## DB-SYM-063

Shared registries serán immutable/frozen.

## DB-SYM-064

No existirá global current scope.

## DB-SYM-065

No existirá global current symbol table.

## DB-SYM-066

No existirá global current tenant.

## DB-SYM-067

No existirá global current schema.

## DB-SYM-068

Queries concurrentes no compartirán mutable resolution state.

## DB-SYM-069

Symbol IDs locales de queries distintas no implicarán identidad global.

## DB-SYM-070

Resolution results serán deterministas.

## DB-SYM-071

Candidate ordering será determinista.

## DB-SYM-072

Diagnostics tendrán códigos estables.

## DB-SYM-073

Diagnostics de ambigüedad mostrarán candidatos relevantes.

## DB-SYM-074

Diagnostics no expondrán runtime parameter values.

## DB-SYM-075

Diagnostics no expondrán credentials.

## DB-SYM-076

Suggestion engine no auto-corregirá consultas.

## DB-SYM-077

Suggestion engine será bounded.

## DB-SYM-078

Symbol resolution podrá ejecutarse offline.

## DB-SYM-079

La mayoría de symbol resolution tests no requerirán DB real.

## DB-SYM-080

Schema changes deberán invalidar artifacts dependientes.

## DB-SYM-081

Identifier semantic changes podrán invalidar artifacts.

## DB-SYM-082

Runtime values no participarán en resolution fingerprint.

## DB-SYM-083

Trace IDs no participarán en resolution fingerprint.

## DB-SYM-084

Query labels observacionales no participarán en resolution fingerprint.

## DB-SYM-085

Extension symbols deberán registrarse mediante contracts.

## DB-SYM-086

Extension symbol kinds tendrán identidad namespaced.

## DB-SYM-087

Extension registries se congelarán después de bootstrap.

## DB-SYM-088

Extensions no registrarán symbol kinds globales durante requests.

## DB-SYM-089

Resolver registry será especializado.

## DB-SYM-090

Resolver registry no será service locator.

## DB-SYM-091

Candidate lookup deberá utilizar índices cuando sea apropiado.

## DB-SYM-092

Scope depth estará sujeto a resource governance.

## DB-SYM-093

Symbol count estará sujeto a resource governance.

## DB-SYM-094

Wildcard expansion estará sujeta a resource governance.

## DB-SYM-095

Budget failure será controlado.

## DB-SYM-096

Budget failure no producirá artifact válido parcial.

## DB-SYM-097

SymbolResolutionTable será immutable al finalizar.

## DB-SYM-098

ScopeGraph será immutable al finalizar.

## DB-SYM-099

SymbolTable será immutable al finalizar.

## DB-SYM-100

Las fases posteriores consumirán identidades semánticas resueltas y no repetirán name resolution.

---

# 248. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Semantic\Symbol\
```

---

# 249. Estructura propuesta

```text
Query/
└── Semantic/
    └── Symbol/
        ├── Contract/
        │   ├── SymbolInterface.php
        │   ├── SymbolResolverInterface.php
        │   ├── SymbolDefinitionResolverInterface.php
        │   └── SymbolKindDescriptorInterface.php
        │
        ├── Identity/
        │   ├── SymbolId.php
        │   ├── ScopeId.php
        │   ├── SymbolKindId.php
        │   └── ReferenceId.php
        │
        ├── Scope/
        │   ├── SemanticScope.php
        │   ├── SemanticScopeGraph.php
        │   ├── SemanticScopeGraphBuilder.php
        │   ├── ScopeKind.php
        │   ├── ScopeEdge.php
        │   ├── ScopeEdgeKind.php
        │   └── ScopeVisibilityPolicy.php
        │
        ├── Namespace/
        │   ├── SymbolNamespace.php
        │   ├── ScopeNamespaceIndex.php
        │   └── NamespaceVisibilityPolicy.php
        │
        ├── Definition/
        │   ├── SymbolDefinition.php
        │   ├── SymbolDefinitionId.php
        │   ├── DefinitionDiscovery.php
        │   └── DefinitionRegistry.php
        │
        ├── Symbol/
        │   ├── RelationSymbol.php
        │   ├── ColumnSymbol.php
        │   ├── CteSymbol.php
        │   ├── ProjectionSymbol.php
        │   ├── WindowSymbol.php
        │   ├── ParameterSymbol.php
        │   └── ExtensionSymbol.php
        │
        ├── Reference/
        │   ├── SymbolReference.php
        │   ├── RelationReference.php
        │   ├── ColumnReference.php
        │   ├── CteReference.php
        │   ├── ProjectionReference.php
        │   ├── WindowReference.php
        │   ├── WildcardReference.php
        │   └── ProjectionPositionReference.php
        │
        ├── Resolution/
        │   ├── SymbolResolutionCoordinator.php
        │   ├── SymbolResolutionContext.php
        │   ├── SymbolResolutionResult.php
        │   ├── ResolvedSymbolReference.php
        │   ├── AmbiguousSymbolReference.php
        │   ├── UnknownSymbolReference.php
        │   ├── DeferredSymbolReference.php
        │   ├── SymbolCandidateSet.php
        │   ├── SymbolResolutionTable.php
        │   ├── DeferredSymbolResolutionQueue.php
        │   └── ResolutionProvenance.php
        │
        ├── Resolver/
        │   ├── RelationSymbolResolver.php
        │   ├── ColumnSymbolResolver.php
        │   ├── CteSymbolResolver.php
        │   ├── ProjectionSymbolResolver.php
        │   ├── WindowSymbolResolver.php
        │   ├── ParameterSymbolResolver.php
        │   ├── WildcardResolver.php
        │   ├── OuterReferenceResolver.php
        │   └── ExtensionSymbolResolver.php
        │
        ├── Identifier/
        │   ├── Identifier.php
        │   ├── IdentifierPath.php
        │   ├── IdentifierCanonicalizer.php
        │   ├── IdentifierSemantics.php
        │   └── SymbolLookupKey.php
        │
        ├── Visibility/
        │   ├── VisibilityDescriptor.php
        │   ├── ClauseResolutionPolicy.php
        │   ├── RelationAliasVisibilityPolicy.php
        │   └── LateralVisibilityPolicy.php
        │
        ├── Correlation/
        │   ├── CorrelationReference.php
        │   ├── CorrelationDependencySet.php
        │   └── OuterSymbolDependency.php
        │
        ├── Extension/
        │   ├── SymbolKindDescriptor.php
        │   ├── SymbolKindRegistry.php
        │   └── ExtensionSymbolResolver.php
        │
        ├── Diagnostic/
        │   ├── SymbolDiagnostic.php
        │   ├── SymbolSuggestionEngine.php
        │   └── SymbolDiagnosticCode.php
        │
        └── Exception/
            ├── SymbolResolutionException.php
            ├── UnknownSymbolException.php
            ├── AmbiguousSymbolException.php
            ├── DuplicateSymbolDefinitionException.php
            ├── InvalidOuterReferenceException.php
            └── UnresolvedDeferredReferenceException.php
```

---

# 250. Public/internal boundary

La mayor parte de este sistema será:

```text
internal
```

El usuario normalmente interactuará mediante APIs como:

```php
DB::table('users as u')
    ->where('u.active', true)
    ->get();
```

sin manipular manualmente `SymbolId`.

---

# 251. Extension API

Sólo contracts seleccionados deberán ser públicos para extensiones:

```text
SymbolKindDescriptor
SymbolResolverExtension
SymbolDefinitionContribution
SymbolResolutionContribution
```

---

# 252. No leaking internals

No exponer públicamente builders mutables como:

```text
SemanticScopeGraphBuilder
SymbolTableBuilder
DeferredResolutionQueue
```

---

# 253. Integración con Query Type System

Una vez resuelta:

```text
u.id
→ ColumnSymbol C7
```

el Query Type System podrá consultar:

```text
C7
→ semantic type
```

---

# 254. Integración con Predicate System

Un predicate:

```text
u.id = :id
```

se convertirá semánticamente en:

```text
ColumnSymbol C7
EQUAL
ParameterId P1
```

sin cambiar el AST.

---

# 255. Integración con Constraint System

Después podrá derivarse:

```text
C7 = P1
```

como semantic constraint.

---

# 256. Integración con Join Resolution

Ejemplo:

```text
o.user_id = u.id
```

se convierte en:

```text
ColumnSymbol C11
=
ColumnSymbol C7
```

permitiendo reconocer join keys.

---

# 257. Integración con Semantic Graph

Los symbols serán identidades fundamentales para nodos/edges del:

```text
SemanticQueryGraph
```

---

# 258. Integración con Optimizer

El Optimizer no deberá volver a interpretar:

```text
"u.id"
```

como string.

Recibirá:

```text
ColumnSymbolId
```

o semantic references equivalentes.

---

# 259. Integración con Compiler

El Compiler podrá usar el semantic artifact para conocer qué identidad representa una referencia.

Pero:

```text
Compiler
```

no deberá repetir name resolution.

---

# 260. Integración con Schema-Aware Resolution

La siguiente capa profundizará:

```text
RelationSymbol
        │
        ▼
Schema Relation
        │
        ▼
Schema Columns
        │
        ▼
Column Symbols
```

---

# 261. Arquitectura consolidada

```text
                     Query AST
                        │
                        ▼
                Scope Discovery
                        │
                        ▼
                 Scope Graph
                        │
                        ▼
              Definition Discovery
                        │
                        ▼
               Symbol Registration
                        │
                        ▼
                   Symbol Table
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
     Local Resolution       Schema-Aware Resolution
            │                       │
            └───────────┬───────────┘
                        ▼
                Relation Outputs
                        │
                        ▼
                Column Resolution
                        │
                        ▼
              Outer Scope Resolution
                        │
                        ▼
              Correlation Detection
                        │
                        ▼
              Deferred Resolution
                        │
                        ▼
              Resolution Validation
                        │
                        ▼
              SymbolResolutionTable
                        │
                        ▼
              Semantic Query Engine
```

---

# 262. Fórmula del sistema

```text
Resolved Symbol
=
Reference
+
Originating Scope
+
Namespace
+
Identifier Semantics
+
Visibility Rules
+
Available Definitions
+
Schema View
+
Resolution Policy
```

---

# 263. Fórmula de columna calificada

```text
Resolved Qualified Column
=
Resolve(Qualifier → RelationSymbol)
+
Resolve(Column → RelationOutput)
```

---

# 264. Fórmula de columna no calificada

```text
Unqualified Column Candidates
=
Union(
    matching columns
    from all visible relation outputs
)
```

Entonces:

```text
0 candidates → UNKNOWN
1 candidate  → RESOLVED
N candidates → AMBIGUOUS
```

---

# 265. Fórmula de outer resolution

```text
Outer Resolution
=
Local Lookup
+
Allowed Parent Traversal
+
Namespace Visibility
+
Shadowing Rules
+
Correlation Rules
```

---

# 266. Fórmula de seguridad runtime

```text
Safe Symbol Resolution
=
Immutable Query AST
+
Operation-Scoped Mutable Workspace
+
Frozen Registries
+
Explicit Scope Graph
+
Explicit Visibility
+
No Ambient State
+
Deterministic Lookup
+
Bounded Resource Usage
```

---

# 267. Resultado arquitectónico

Antes de Symbol Resolution:

```text
u.id
o.total
status
monthly
active_users
```

son nombres estructurados.

Después:

```text
u.id
→ ColumnSymbol C7

o.total
→ ColumnSymbol C12

status
→ ParameterId P3

monthly
→ WindowSymbol W1

active_users
→ CteSymbol CTE2
```

Las siguientes fases dejan de razonar principalmente sobre strings.

Razonan sobre:

```text
semantic identities
```

---

# 268. Decisión final

VoltStack adoptará un **Symbol Resolution System basado en scopes, namespaces e identidades semánticas explícitas**, separado completamente del AST mutable, SQL Compiler, Driver, Connection y ORM runtime.

La arquitectura fundamental será:

```text
Name
  │
  ▼
Identifier
  │
  ▼
Reference
  │
  ▼
Scope + Namespace
  │
  ▼
Candidate Resolution
  │
  ├── 0 ──► Unknown
  │
  ├── 1 ──► Resolved Symbol
  │
  └── N ──► Ambiguous
```

con resolución externa:

```text
Current Scope
     │
     ▼
Parent Scope
     │
     ▼
Correlation
```

únicamente cuando las reglas de visibilidad lo permitan.

---

# 269. Invariante central

> Después del Symbol Resolution System, ninguna fase semántica posterior deberá necesitar adivinar qué representa un nombre utilizado por la consulta.

---

# 270. Próximo documento

```text
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
```

definirá cómo VoltStack conecta las identidades semánticas resueltas con el modelo estructural del schema:

```text
Relation Reference
        │
        ▼
Semantic Relation
        │
        ▼
Schema View
        │
        ▼
Schema Relation Descriptor
        │
        ├── Columns
        ├── Types
        ├── Nullability
        ├── Primary Keys
        ├── Unique Keys
        ├── Foreign Keys
        ├── Constraints
        └── Capabilities
        │
        ▼
Schema-Aware Semantic Information
```

sin que el Query Engine abra conexiones, ejecute introspection SQL ni quede acoplado al motor físico de base de datos.