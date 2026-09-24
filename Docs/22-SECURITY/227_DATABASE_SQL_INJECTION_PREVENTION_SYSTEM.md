# 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md

# VoltStack Quantum Database
## SQL Injection Prevention System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 227 — SQL Injection Prevention System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `226_DATABASE_SECURITY_ARCHITECTURE.md`  
**Siguiente documento:** `228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de prevención de **SQL Injection** de `VoltStack/Quantum/Database`.

El sistema deberá impedir que información externa o no confiable modifique accidental o maliciosamente la estructura de una consulta SQL.

La defensa no dependerá únicamente de prepared statements.

VoltStack utilizará una arquitectura basada en:

```text
Typed Query API
      ↓
Query Model
      ↓
Query AST
      ↓
Semantic Validation
      ↓
Identifier Validation
      ↓
SQL Compiler
      ↓
Prepared Statement
      ↓
Typed Parameter Binding
      ↓
Driver
      ↓
Database
```

Regla central:

> **En VoltStack ningún dato externo deberá convertirse en estructura SQL simplemente por ser un string. Los valores serán parámetros y la estructura SQL deberá originarse exclusivamente en objetos semánticos, metadata validada o escape hatches explícitos sujetos a política.**

---

# 2. SQL Injection

SQL Injection ocurre cuando datos que deberían ser tratados como información terminan siendo interpretados como estructura SQL.

Ejemplo vulnerable:

```php
$sql = "SELECT * FROM users WHERE email = '$email'";
```

Si:

```text
$email =
' OR 1=1 --
```

la información proporcionada deja de comportarse únicamente como valor.

Conceptualmente:

```text
Expected:

SQL Structure + Data

Actual:

SQL Structure + Attacker-Controlled SQL Structure
```

VoltStack deberá impedir esta transición.

---

# 3. Principio Data ≠ Code

La distinción fundamental será:

```text
DATA
≠
SQL STRUCTURE
```

Ejemplos de datos:

```text
email
name
age
UUID
date
JSON value
search term
numeric amount
```

Ejemplos de estructura SQL:

```text
table
column
operator
JOIN
ORDER BY
GROUP BY
CTE
function
direction
alias
schema
constraint
```

---

# 4. Valores y estructura

Una consulta segura puede conceptualizarse como:

```text
Query Structure
+
Bound Values
```

No como:

```text
SQL String
+
String Concatenation
```

---

# 5. Modelo de seguridad

```text
Application Input
      │
      ▼
   UNTRUSTED
      │
      ▼
Input Resolution
      │
      ├─────────────┐
      │             │
      ▼             ▼
    Value       Structural Request
      │             │
      ▼             ▼
 Parameter      Allowlist/Metadata
      │             │
      │             ▼
      │        Semantic Object
      │             │
      └──────┬──────┘
             ▼
          Query AST
             │
             ▼
     Semantic Validation
             │
             ▼
         Compiler
             │
      ┌──────┴──────┐
      ▼             ▼
Identifier      Placeholder
 Quoting           SQL
      │             │
      └──────┬──────┘
             ▼
       Prepared Statement
             │
             ▼
       Parameter Binding
             │
             ▼
           Driver
```

---

# 6. SQL injection surfaces

VoltStack deberá proteger al menos:

```text
WHERE values
INSERT values
UPDATE values
DELETE predicates
table names
column names
aliases
ORDER BY
GROUP BY
HAVING
JOIN conditions
JOIN targets
operators
functions
LIMIT/OFFSET
CTEs
subqueries
UNION
JSON paths
full-text expressions
schema identifiers
DDL
migration definitions
savepoints
raw SQL
raw expressions
driver options
```

---

# 7. Prepared statements

Prepared statements serán una defensa fundamental para valores.

Ejemplo conceptual:

```sql
SELECT *
FROM users
WHERE email = ?
```

Bindings:

```text
[1] => attacker-controlled value
```

La base de datos recibe por separado:

```text
SQL structure
```

y:

```text
parameter values
```

---

# 8. Prepared Statements ≠ Complete SQL Injection Defense

Prepared statements no pueden representar de forma general:

```text
table identifiers
column identifiers
operators
keywords
ORDER BY direction
SQL functions
```

Por tanto:

```text
Prepared Statements
≠
Complete SQL Injection Prevention
```

---

# 9. Parameter model

VoltStack deberá representar parámetros mediante objetos tipados.

```php
final readonly class QueryParameter
{
    public function __construct(
        public ParameterId $id,
        public mixed $value,
        public DatabaseTypeId $type,
        public ParameterMode $mode,
    ) {}
}
```

---

# 10. Parameter identity

Los parámetros deberán utilizar identificadores internos.

Ejemplo:

```text
ParameterId(p1)
ParameterId(p2)
ParameterId(p3)
```

El Compiler decidirá su representación física.

---

# 11. Placeholder strategy

Dependiendo de la plataforma:

```text
?
```

o:

```text
$1
$2
$3
```

u otra representación compatible.

El Query Builder no necesita conocerla.

---

# 12. Query Builder

Ejemplo:

```php
User::query()
    ->where('email', '=', $email)
    ->where('status', '=', $status);
```

Internamente:

```text
ComparisonPredicate
├── ColumnReference(email)
├── EqualityOperator
└── ParameterReference(p1)

ComparisonPredicate
├── ColumnReference(status)
├── EqualityOperator
└── ParameterReference(p2)
```

---

# 13. SQL Compiler

Posteriormente:

```text
Query AST
   ↓
PostgreSQLCompiler
   ↓
SELECT ...
WHERE "email" = $1
AND "status" = $2
```

Bindings permanecen separados.

---

# 14. No interpolation

El Compiler no deberá producir:

```php
$sql = "... WHERE email = '{$value}'";
```

---

# 15. No manual escaping

VoltStack no deberá utilizar como defensa principal:

```php
addslashes($value);
```

ni mecanismos equivalentes.

---

# 16. Escaping ≠ Binding

```text
Escaping
≠
Parameter Binding
```

El escaping depende de:

```text
database
encoding
SQL mode
context
driver
```

y puede ser extremadamente fácil de implementar incorrectamente.

---

# 17. Typed parameter binding

El Type System deberá participar antes del binding.

```text
PHP Value
    ↓
VoltStack Type
    ↓
Canonical Database Value
    ↓
Binding Type
    ↓
Driver Parameter
```

---

# 18. Example

```php
$query->where('active', true);
```

puede convertirse conceptualmente en:

```text
PHP bool
↓
BooleanType
↓
canonical DB representation
↓
driver binding
```

sin interpolar:

```text
TRUE
1
'1'
```

manualmente.

---

# 19. NULL

`NULL` deberá representarse semánticamente.

Por ejemplo:

```php
$query->whereNull('deleted_at');
```

produce:

```text
IsNullPredicate
└── ColumnReference(deleted_at)
```

No:

```php
$query->whereRaw(
    "deleted_at IS " . $input
);
```

---

# 20. IN predicates

Ejemplo:

```php
$query->whereIn('id', $ids);
```

deberá producir:

```text
InPredicate
├── ColumnReference(id)
└── ParameterList
    ├── p1
    ├── p2
    └── p3
```

No:

```php
implode(',', $ids)
```

insertado directamente en SQL.

---

# 21. Empty IN

El caso:

```php
whereIn('id', [])
```

deberá resolverse semánticamente.

Por ejemplo:

```text
AlwaysFalsePredicate
```

y no mediante SQL inválido o concatenación especial insegura.

---

# 22. Large IN

Listas extremadamente grandes estarán sujetas además a:

```text
resource governance
parameter count limits
platform capabilities
```

Seguridad sintáctica no implica seguridad de recursos.

---

# 23. LIKE

Esto:

```php
$query->where('name', 'LIKE', $search);
```

deberá bindear `$search`.

---

# 24. LIKE wildcard semantics

Existe una distinción importante:

```text
SQL Injection
≠
LIKE Wildcard Injection
```

Un usuario podría introducir:

```text
%
_
```

que no rompe la estructura SQL, pero sí modifica la semántica del patrón.

---

# 25. LIKE policies

Podrán existir modos:

```php
enum LikePatternMode
{
    case RAW_PATTERN;
    case LITERAL;
    case PREFIX;
    case SUFFIX;
    case CONTAINS;
}
```

---

# 26. Literal LIKE

Por ejemplo:

```php
$query->whereLike(
    'name',
    $input,
    LikePatternMode::LITERAL
);
```

deberá escapar semánticamente los caracteres wildcard según dialecto.

---

# 27. Identifier injection

Una segunda categoría crítica es:

```text
Identifier Injection
```

Ejemplo vulnerable:

```php
$order = $_GET['sort'];

$sql = "SELECT * FROM users ORDER BY $order";
```

Prepared statements no solucionan este problema.

---

# 28. Identifier object model

VoltStack deberá utilizar:

```php
final readonly class Identifier
{
    public function __construct(
        public IdentifierValue $value,
        public IdentifierKind $kind,
    ) {}
}
```

---

# 29. Identifier kinds

```php
enum IdentifierKind
{
    case TABLE;
    case COLUMN;
    case SCHEMA;
    case ALIAS;
    case INDEX;
    case CONSTRAINT;
    case SAVEPOINT;
}
```

---

# 30. ColumnReference

Las consultas deberían utilizar un nodo más específico:

```php
final readonly class ColumnReference implements ExpressionNode
{
    public function __construct(
        public RelationReference $relation,
        public ColumnMetadataId $column,
    ) {}
}
```

cuando metadata esté disponible.

---

# 31. Metadata resolution

Ejemplo externo:

```text
sort=created
```

podría resolverse mediante:

```text
created
↓
AllowedSortRegistry
↓
User.created_at
↓
ColumnMetadataId
↓
ColumnReference
```

---

# 32. Never direct identifier input

No:

```php
$query->orderBy($_GET['sort']);
```

si el argumento se interpreta como identificador arbitrario proveniente de input externo.

---

# 33. API distinction

La API deberá diferenciar entre:

```text
developer-authored identifier
```

y:

```text
untrusted external identifier request
```

---

# 34. Developer identifier

Código estático como:

```php
$query->orderBy('created_at');
```

puede resolverse durante Query Model construction mediante metadata.

---

# 35. External identifier

Para input externo:

```php
$sort = $allowedSorts->resolve(
    $request->input('sort')
);
```

---

# 36. Allowlist

Ejemplo:

```php
$allowedSorts = AllowedSorts::define([
    'name' => UserFields::NAME,
    'created' => UserFields::CREATED_AT,
]);
```

---

# 37. Allowlist ≠ String Sanitization

Preferir:

```text
external alias
→
known semantic field
```

sobre:

```text
remove dangerous characters
→
hope identifier is safe
```

---

# 38. Identifier quoting

El Compiler deberá quote identifiers según plataforma.

Conceptualmente:

```text
Identifier
↓
IdentifierQuoter
↓
QuotedIdentifier
```

---

# 39. Platform examples

Podría producir:

```sql
"created_at"
```

o:

```sql
`created_at`
```

según plataforma.

---

# 40. Quoting ≠ Validation

El sistema deberá validar la estructura antes de quote.

No deberá intentar convertir cualquier string arbitrario en identificador seguro simplemente rodeándolo con delimitadores.

---

# 41. Qualified identifiers

Una referencia:

```text
users.email
```

no deberá ser tratada necesariamente como un único string.

Preferir:

```text
QualifiedIdentifier
├── TableIdentifier(users)
└── ColumnIdentifier(email)
```

---

# 42. Dot injection

Esto evita problemas donde:

```text
schema.table
```

o:

```text
table.column
```

se interpreten ambiguamente.

---

# 43. Alias security

Aliases deberán utilizar nodos estructurados.

```text
AliasIdentifier
```

No strings SQL libres.

---

# 44. ORDER BY security

`ORDER BY` tiene dos componentes:

```text
expression
direction
```

Ambos deberán ser semánticos.

---

# 45. SortDirection

```php
enum SortDirection
{
    case ASC;
    case DESC;
}
```

---

# 46. External direction

Input:

```text
direction=desc
```

deberá resolverse:

```text
"desc"
↓
SortDirection::DESC
```

No concatenarse.

---

# 47. NULL ordering

También:

```php
enum NullOrdering
{
    case FIRST;
    case LAST;
    case PLATFORM_DEFAULT;
}
```

---

# 48. Dynamic ORDER BY

Patrón seguro:

```text
External Sort Name
       ↓
AllowedSortResolver
       ↓
OrderExpression
       ↓
SortDirection enum
       ↓
OrderByNode
```

---

# 49. GROUP BY

Misma regla:

```text
External Field
↓
Allowed Group Resolver
↓
Expression Node
```

---

# 50. HAVING

Los valores de HAVING deberán parametrizarse igual que WHERE.

---

# 51. Operators

Nunca:

```php
$query->where(
    'age',
    $_GET['operator'],
    $value
);
```

si cualquier string puede convertirse en SQL.

---

# 52. Operator enum

```php
enum ComparisonOperator
{
    case EQ;
    case NEQ;
    case GT;
    case GTE;
    case LT;
    case LTE;
    case LIKE;
}
```

---

# 53. Operator resolution

```text
"greater_than"
↓
AllowedOperatorResolver
↓
ComparisonOperator::GT
```

---

# 54. Operator ≠ Raw SQL

Un operador es una entidad semántica conocida por el Compiler.

---

# 55. JOIN security

Dynamic JOINs también representan estructura SQL.

Nunca:

```php
$query->join($_GET['table'], ...);
```

sin resolución estructural.

---

# 56. JoinTarget

```php
final readonly class JoinTarget
{
    public function __construct(
        public RelationReference $relation,
        public JoinAlias $alias,
    ) {}
}
```

---

# 57. JoinType

```php
enum JoinType
{
    case INNER;
    case LEFT;
    case RIGHT;
    case FULL;
    case CROSS;
}
```

sujeto a capabilities de plataforma.

---

# 58. Join predicates

Se expresarán como Predicate AST.

```text
JoinPredicate
├── ColumnReference(users.role_id)
├── EqualityOperator
└── ColumnReference(roles.id)
```

---

# 59. Relationship JOINs

Cuando ORM genere JOINs, deberán derivarse de:

```text
RelationshipMetadata
```

no de strings arbitrarios.

---

# 60. Subquery security

Una subquery deberá ser:

```text
QueryModel
```

o:

```text
SubqueryNode
```

No:

```text
string SQL
```

por default.

---

# 61. Example

```php
$query->whereExists(
    Order::query()
        ->whereColumn('orders.user_id', 'users.id')
);
```

conceptualmente produce:

```text
ExistsPredicate
└── SubqueryNode
```

---

# 62. CTE security

CTEs deberán modelarse mediante:

```text
CommonTableExpressionNode
```

con:

```text
name
query
columns
recursive flag
```

estructurados.

---

# 63. CTE names

Los nombres serán identifiers.

No parámetros.

---

# 64. Recursive CTE

Además de injection prevention, deberá aplicarse resource governance cuando corresponda.

---

# 65. UNION

`UNION`, `UNION ALL`, `INTERSECT`, etc. serán operaciones semánticas.

No strings dinámicos.

---

# 66. SetOperationType

```php
enum SetOperationType
{
    case UNION;
    case UNION_ALL;
    case INTERSECT;
    case EXCEPT;
}
```

según capabilities.

---

# 67. Function calls

Funciones SQL deberán representarse mediante nodos.

```text
FunctionExpression
├── FunctionId
└── Arguments
```

---

# 68. Function Registry

Funciones conocidas podrán registrarse:

```text
COUNT
SUM
AVG
LOWER
UPPER
COALESCE
```

---

# 69. External function names

No permitir:

```text
?function=<arbitrary SQL>
```

---

# 70. Dynamic function resolution

Si existe una API externa:

```text
"average"
↓
AllowedFunctionResolver
↓
FunctionId(AVG)
```

---

# 71. Function Registry security

Custom functions serán registradas durante bootstrap.

No desde input de request.

---

# 72. JSON query security

JSON paths pueden convertirse en otra superficie de injection.

---

# 73. JSON path

Preferir:

```php
new JsonPath([
    JsonPathSegment::key('profile'),
    JsonPathSegment::key('city'),
]);
```

sobre concatenación SQL.

---

# 74. External JSON paths

Deberán:

```text
parse
validate
bound depth
resolve semantics
compile
```

---

# 75. JSON path limits

Podrán limitarse:

```text
path depth
segment count
segment length
wildcards
recursive descent
```

según API.

---

# 76. JSON Path ≠ SQL Fragment

Incluso si una plataforma representa JSON path mediante string SQL, el Query Engine deberá mantenerlo como objeto semántico.

---

# 77. Full-text search

Los términos de búsqueda serán valores.

La configuración estructural:

```text
language
mode
fields
ranking
```

deberá resolverse mediante enums/metadata.

---

# 78. LIMIT

`LIMIT` deberá recibir un valor numérico validado.

---

# 79. LIMIT binding

Algunas plataformas permiten bindearlo; otras pueden requerir representación literal.

Cuando deba compilarse como literal:

```text
validated integer
```

deberá provenir de un tipo numérico interno, nunca del string externo original.

---

# 80. OFFSET

Misma regla.

---

# 81. Integer normalization

```text
"50"
↓
ValidatedInteger
↓
50
↓
Compiler
```

No:

```text
"50; DROP TABLE..."
↓
SQL
```

---

# 82. Pagination

El Pagination System deberá producir:

```text
PageSize
Offset
```

tipados.

---

# 83. Cursor pagination

Los cursores externos deberán decodificarse a:

```text
typed boundary values
```

y nunca convertirse en SQL.

---

# 84. Cursor integrity

Firma válida tampoco convierte el cursor en SQL.

Continúa siendo:

```text
semantic boundary data
```

---

# 85. Chunk processing

Continuation boundaries deberán reutilizar el mismo modelo semántico.

---

# 86. Bulk operations

Bulk Insert/Update/Delete deberán usar Query Models.

No construir SQL concatenando datasets.

---

# 87. Bulk Insert

Conceptualmente:

```text
BulkInsertModel
├── TargetRelation
├── Columns
└── Rows
    ├── ParameterSet
    ├── ParameterSet
    └── ParameterSet
```

---

# 88. Bulk Update

Los nuevos valores serán parámetros.

---

# 89. Bulk Delete

Predicates serán Query AST.

---

# 90. Import

Datos importados deberán entrar como:

```text
decoded values
↓
validated values
↓
typed values
↓
parameters
```

Nunca:

```text
CSV field
↓
SQL concatenation
```

---

# 91. Export

Export no crea normalmente SQL desde contenido exportado, pero sus filtros dinámicos seguirán las mismas reglas.

---

# 92. Schema Builder

DDL también puede sufrir injection si utiliza identifiers dinámicos.

---

# 93. Schema AST

Preferir:

```text
CreateTableNode
├── TableIdentifier
├── ColumnDefinitions
├── IndexDefinitions
└── Constraints
```

---

# 94. Table creation

No:

```php
DB::statement(
    "CREATE TABLE " . $_POST['name']
);
```

---

# 95. Migration identifiers

Los identifiers definidos por código de migration deberán validarse estructuralmente.

---

# 96. Migration ≠ Trusted String Boundary

Aunque migrations sean código privilegiado, no significa que el Compiler deba abandonar sus invariantes.

---

# 97. Savepoint names

Deben utilizar:

```text
SavepointIdentifier
```

y no SQL libre.

---

# 98. Raw SQL architecture

VoltStack necesitará SQL crudo para casos avanzados.

Por tanto:

```text
Raw SQL
≠
Forbidden
```

Pero:

```text
Raw SQL
=
Explicit Security Boundary
```

---

# 99. RawExpression

Propuesta:

```php
final readonly class RawExpression implements ExpressionNode
{
    public function __construct(
        public TrustedSqlFragment $sql,
        public ParameterBag $parameters,
        public RawExpressionOrigin $origin,
    ) {}
}
```

---

# 100. TrustedSqlFragment

No deberá poder construirse accidentalmente desde cualquier string en APIs externas.

---

# 101. Factory

Podría requerir:

```php
Raw::sql(
    'COALESCE(last_login_at, created_at)'
);
```

haciendo visible el escape hatch.

---

# 102. Raw parameters

Incluso SQL raw deberá permitir:

```php
Raw::sql(
    'score > ?',
    [$score]
);
```

en lugar de interpolación.

---

# 103. Raw SQL rule

```text
Raw Structure
+
Bound Values
```

continúa siendo preferible a:

```text
Raw Structure With Interpolated Values
```

---

# 104. Raw SQL policy

Podrán existir políticas:

```php
enum RawSqlPolicy
{
    case ALLOW;
    case ALLOW_WITH_AUDIT;
    case WARN;
    case DENY;
}
```

---

# 105. Production policy

Aplicaciones de alta seguridad podrían utilizar:

```text
DENY
```

para determinados contextos.

---

# 106. Raw origin

```php
enum RawExpressionOrigin
{
    case FRAMEWORK;
    case APPLICATION;
    case EXTENSION;
}
```

---

# 107. Origin ≠ Safe

`APPLICATION` no implica automáticamente seguridad.

Sirve para diagnóstico/policy.

---

# 108. Raw SQL audit

Podrá registrar:

```text
query fingerprint
origin
call-site fingerprint
operation type
security context reference
```

sin registrar valores sensibles.

---

# 109. Unsafe interpolation detection

En desarrollo, VoltStack podrá detectar patrones sospechosos en raw SQL.

Pero:

> **La detección heurística nunca será la defensa principal.**

---

# 110. Why heuristic detection is insufficient

No puede determinar de forma confiable si:

```php
Raw::sql($sql);
```

contiene datos no confiables.

La seguridad debe provenir de arquitectura/API/policy.

---

# 111. Second-order SQL injection

VoltStack deberá contemplar:

```text
Second-Order Injection
```

---

# 112. Definition

Ocurre cuando un valor inicialmente almacenado como dato posteriormente se recupera y se utiliza incorrectamente como estructura SQL.

Ejemplo:

```text
User Input
↓
Stored Safely
↓
Loaded Later
↓
Concatenated Into SQL Structure
↓
Injection
```

---

# 113. Stored Data ≠ Trusted Data

Regla:

> **Que un valor provenga de la base de datos no lo convierte automáticamente en estructura SQL confiable.**

---

# 114. Example

Si la DB contiene:

```text
sort_expression = "name; DROP TABLE users"
```

esto sigue siendo data.

No debe pasar directamente a:

```text
ORDER BY
```

---

# 115. Stored configuration

Configuración dinámica almacenada deberá resolverse mediante:

```text
stable alias
↓
registry
↓
semantic object
```

---

# 116. Metadata trust

VoltStack distinguirá:

```text
Framework-compiled metadata
```

de:

```text
arbitrary database content
```

---

# 117. Metadata poisoning

Si metadata externa puede modificarse, deberá verificarse antes de convertirla en estructura SQL.

---

# 118. Identifier Registry

Podrá existir:

```php
interface IdentifierRegistry
{
    public function resolveColumn(
        EntityTypeId $entity,
        ExternalFieldAlias $alias
    ): ColumnReference;
}
```

---

# 119. Dynamic filtering APIs

Una API externa podría recibir:

```json
{
  "field": "email",
  "operator": "equals",
  "value": "user@example.com"
}
```

Debe transformarse así:

```text
field
↓
Field Resolver
↓
ColumnReference

operator
↓
Operator Resolver
↓
ComparisonOperator

value
↓
Type Conversion
↓
QueryParameter
```

---

# 120. Never

```text
field + operator + value
↓
SQL string concatenation
```

---

# 121. Query DSL

Si VoltStack implementa un Query DSL externo en el futuro, deberá tener:

```text
Lexer
↓
Parser
↓
AST
↓
Semantic Analyzer
↓
Security Validation
↓
Query AST
```

No:

```text
DSL string
↓
SQL string replacement
```

---

# 122. VoltStack Query Language

El futuro VoltStack Query Language deberá mantener la misma frontera.

```text
VQL
≠
SQL macro language
```

---

# 123. VQL identifiers

Deberán resolverse contra:

```text
entity metadata
field registry
relationship metadata
capabilities
authorization scope
```

---

# 124. Query AST safety

Una vez construido correctamente, el Query AST deberá garantizar que:

```text
value nodes
```

no puedan transformarse accidentalmente en:

```text
structural SQL nodes
```

---

# 125. Type-level separation

Preferir tipos distintos:

```text
ParameterValue
ColumnIdentifier
TableIdentifier
Operator
SortDirection
FunctionId
RawExpression
```

---

# 126. Avoid generic StringExpression

Evitar una clase universal como:

```php
new StringExpression($value);
```

si puede significar indistintamente:

```text
identifier
value
SQL fragment
function
```

---

# 127. AST node contracts

Ejemplo:

```php
interface ValueExpression extends ExpressionNode {}

interface StructuralExpression extends ExpressionNode {}

interface IdentifierExpression extends StructuralExpression {}
```

---

# 128. Semantic analyzer

Deberá verificar que cada nodo aparezca en un contexto permitido.

Ejemplo:

```text
ORDER BY
```

acepta una expresión ordenable.

No un arbitrary statement node.

---

# 129. AST provenance

Opcionalmente, nodos sensibles podrán conservar:

```text
origin
trust classification
```

para policy/diagnóstico.

---

# 130. Compiler invariant

El Compiler deberá asumir un AST validado, pero continuar preservando separación estructural.

---

# 131. Compiler shall not concatenate values

Nunca:

```php
$sql .= (string) $parameter->value;
```

---

# 132. Compiler output

Propuesta:

```php
final readonly class CompiledQuery
{
    public function __construct(
        public CompiledSql $sql,
        public CompiledParameterBag $parameters,
        public QueryFingerprint $fingerprint,
        public CompilationMetadata $metadata,
    ) {}
}
```

---

# 133. CompiledSql

Debe contener estructura SQL.

No valores interpolados salvo literales internos seguros requeridos por el dialecto.

---

# 134. Internal literals

Algunos valores estructurales conocidos pueden compilarse directamente.

Ejemplo:

```text
ASC
DESC
NULL
TRUE
FALSE
```

cuando provienen de enums/nodos internos.

---

# 135. Internal literal ≠ User String

La diferencia es el origen tipado.

---

# 136. Parameter bag

```php
final readonly class CompiledParameterBag
{
    /** @var list<CompiledParameter> */
    public array $parameters;
}
```

---

# 137. CompiledParameter

```php
final readonly class CompiledParameter
{
    public function __construct(
        public ParameterPosition $position,
        public mixed $value,
        public DriverBindingType $type,
    ) {}
}
```

---

# 138. Driver binding

El Driver deberá utilizar las APIs nativas apropiadas para binding.

---

# 139. Driver shall not re-interpolate

Prohibido:

```text
Compiled SQL + Parameters
↓
Driver String Replacement
↓
Final SQL
```

como estrategia general.

---

# 140. Emulated prepares

Algunos drivers/plataformas pueden utilizar prepared statements emulados.

VoltStack deberá considerar esta capacidad explícitamente.

---

# 141. Native vs emulated prepares

Capability model:

```php
$platform->capabilities()
    ->preparedStatements()
    ->mode();
```

podría distinguir:

```text
NATIVE
EMULATED
HYBRID
UNSUPPORTED
UNKNOWN
```

---

# 142. Emulation security

Una implementación emulada deberá demostrar equivalencia segura.

No deberá asumirse segura simplemente por llamarse "prepared".

---

# 143. PDO

Si un driver oficial utiliza PDO, la configuración de prepares deberá ser definida por el adapter correspondiente.

El Query Engine no deberá depender de detalles PDO.

---

# 144. Encoding

Connection encoding deberá configurarse mediante APIs del driver/plataforma.

No mediante input SQL concatenado.

---

# 145. Character set

Problemas de charset pueden afectar escaping.

Otra razón para evitar manual escaping.

---

# 146. Connection initialization

Opciones como charset deberán formar parte de:

```text
Connection Configuration
```

no SQL generado desde request input.

---

# 147. SQL modes

SQL modes pueden modificar interpretación.

El Platform/Dialect deberá conocer aquellos que afecten compilación o seguridad.

---

# 148. Platform capability

```text
Version
≠
Effective SQL Mode
≠
Capability
```

---

# 149. MySQL

El MySQL Compiler deberá utilizar las reglas de su plataforma.

---

# 150. MariaDB

MariaDB tendrá compiler/capabilities independientes cuando sus semánticas diverjan.

---

# 151. PostgreSQL

Usará su estrategia de placeholders/identifiers mediante el Compiler/Driver correspondiente.

---

# 152. SQLite

Mantendrá la misma separación entre:

```text
structure
parameters
```

aunque su deployment común sea local.

---

# 153. Stored procedures

Invocaciones a procedimientos deberán modelar:

```text
procedure identifier
parameters
```

por separado.

---

# 154. Procedure name

No deberá ser un valor externo arbitrario.

---

# 155. Procedure parameters

Deberán bindearse cuando la API/plataforma lo permita.

---

# 156. Stored procedure ≠ Injection immunity

Un procedimiento almacenado puede contener SQL dinámico vulnerable internamente.

VoltStack solo puede garantizar la seguridad de su frontera de llamada.

---

# 157. Database functions

Mismo principio.

---

# 158. Dynamic SQL inside database

Queda fuera del control completo del Query Compiler.

Deberá documentarse como:

```text
External Database Security Boundary
```

---

# 159. Multi-statements

Los drivers oficiales deberían deshabilitar multi-statements por default cuando no sean necesarios y la plataforma lo permita.

---

# 160. Why

Reduce impacto potencial de determinadas vulnerabilidades y evita APIs ambiguas.

---

# 161. Multi-statement requirement

Si una operación administrativa requiere múltiples statements:

```text
CompiledCommandBatch
```

deberá ser preferible a un string concatenado.

---

# 162. Comment tokens

VoltStack no deberá intentar defenderse eliminando:

```text
--
#
/*
*/
```

de valores.

Correct parameter binding hace innecesaria esa estrategia.

---

# 163. Keyword blacklists

No usar como defensa primaria:

```text
DROP
DELETE
UNION
SELECT
OR
```

---

# 164. Why blacklists fail

Palabras SQL pueden:

```text
appear legitimately in data
vary by dialect
be encoded differently
be bypassed
```

---

# 165. Sanitization

La arquitectura deberá evitar el término ambiguo:

```text
sanitize SQL
```

cuando realmente se necesite:

```text
parse
validate
resolve
parameterize
quote identifiers
```

---

# 166. Query security pipeline

```text
Query Request
     ↓
Input Classification
     ↓
Semantic Resolution
     ↓
Query AST Construction
     ↓
AST Validation
     ↓
SQL Injection Analysis
     ↓
Security Policy
     ↓
Compiler
     ↓
CompiledQuery
     ↓
Prepared Statement
     ↓
Typed Binding
     ↓
Execution
```

---

# 167. SQLInjectionAnalyzer

Propuesta:

```php
interface SqlInjectionAnalyzer
{
    public function analyze(
        QueryModel $query,
        QuerySecurityContext $context
    ): SqlInjectionAnalysis;
}
```

---

# 168. Analysis result

```php
final readonly class SqlInjectionAnalysis
{
    public function __construct(
        public SqlInjectionSafety $safety,
        public array $findings,
        public SecurityEvidence $evidence,
    ) {}
}
```

---

# 169. Safety states

```php
enum SqlInjectionSafety
{
    case SAFE;
    case SAFE_WITH_RAW_BOUNDARY;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 170. UNKNOWN ≠ SAFE

Si una operación requiere demostrar seguridad:

```text
UNKNOWN
→
DENY
```

por default.

---

# 171. Findings

Ejemplos:

```text
RAW_SQL_PRESENT
UNRESOLVED_IDENTIFIER
UNTRUSTED_IDENTIFIER
UNSAFE_INTERPOLATION
UNKNOWN_OPERATOR
UNREGISTERED_FUNCTION
UNSAFE_JSON_PATH
UNBOUND_VALUE
UNSUPPORTED_PARAMETER_CONTEXT
```

---

# 172. Finding severity

```php
enum SecurityFindingSeverity
{
    case INFO;
    case WARNING;
    case HIGH;
    case CRITICAL;
}
```

---

# 173. Static structural safety

La mayoría de queries normales deberían poder demostrar seguridad sin análisis heurístico costoso porque fueron construidas mediante nodos tipados.

---

# 174. Fast path

```text
Typed AST
+
No Raw Nodes
+
Validated Metadata
+
Supported Compiler
=
Structurally Safe Fast Path
```

---

# 175. Security proof metadata

El Query Model podría almacenar:

```text
ValidationGeneration
MetadataGeneration
SecurityAnalysisGeneration
```

para evitar trabajo redundante de forma segura.

---

# 176. Generation mismatch

Si cambia metadata/security policy:

```text
cached analysis
```

puede dejar de ser reutilizable.

---

# 177. Compiled Query Cache

Solo deberá almacenar consultas compiladas que preserven:

```text
parameter separation
platform identity
metadata generation
compiler generation
```

---

# 178. Cache ≠ Interpolation

Compiled Query Cache nunca deberá crear una versión SQL por cada valor interpolado.

---

# 179. Query fingerprint

Deberá basarse en estructura semántica.

Ejemplo:

```text
email = ?
```

no necesita incorporar el email completo al fingerprint.

---

# 180. Security-sensitive fingerprint

Raw SQL structure sí deberá influir en el fingerprint.

---

# 181. Query Result Cache

Los parámetros deberán formar parte de la identidad semántica del resultado, pero no necesariamente aparecer en texto legible.

---

# 182. Telemetry

No registrar valores completos por default.

Preferir:

```text
query fingerprint
parameter count
parameter type IDs
raw-boundary presence
```

---

# 183. Debug mode

Incluso debug deberá aplicar redaction.

---

# 184. SQL preview

Una herramienta de desarrollo puede mostrar:

```sql
SELECT * FROM users WHERE email = ?
```

Bindings:

```text
p1 = [REDACTED]
```

---

# 185. Interpolated SQL preview

No debería ser la representación canónica.

Si existe únicamente para diagnóstico, deberá estar claramente marcada:

```text
NON_EXECUTABLE_DEBUG_REPRESENTATION
```

---

# 186. Debug preview ≠ Executable SQL

Nunca reutilizar el SQL interpolado de debug para ejecución.

---

# 187. Error handling

Excepciones específicas:

```text
SqlInjectionPreventionException
├── UnsafeSqlExpressionException
├── UnsafeIdentifierException
├── UnresolvedIdentifierException
├── UnsafeRawSqlException
├── UnsafeOperatorException
├── UnsafeFunctionException
├── UnsafeJsonPathException
├── ParameterizationException
└── SqlInjectionSafetyUnknownException
```

---

# 188. Error messages

En desarrollo:

```text
Dynamic ORDER BY field "foo" is not registered as an allowed sort.
```

En producción externa podría reducirse a:

```text
Invalid query input.
```

---

# 189. Never echo payload unnecessarily

Una excepción no deberá imprimir el payload completo si puede contener:

```text
credentials
PII
attack payloads
sensitive business data
```

---

# 190. Audit

Intentos rechazados podrán generar:

```text
DatabaseSecurityAuditEvent
```

según policy.

---

# 191. Audit event

Ejemplo:

```text
event:
    database.query.security_rejected

reason:
    UNTRUSTED_IDENTIFIER

query:
    fingerprint

principal:
    reference

tenant:
    safe reference
```

---

# 192. Attack ≠ Invalid Input

No toda query rechazada representa un ataque.

Por tanto telemetry deberá utilizar:

```text
security violation
```

o:

```text
unsafe query request
```

sin afirmar automáticamente intención maliciosa.

---

# 193. Query events

`QueryExecuting` deberá ocurrir únicamente después de que la query haya pasado los controles obligatorios.

---

# 194. Rejected query event

Queries rechazadas deberán usar un evento distinto:

```text
QuerySecurityRejected
```

---

# 195. No execution event for rejected query

Esto evita métricas falsas.

---

# 196. Extension nodes

Custom Query AST nodes deberán declarar cómo se compilan.

---

# 197. Custom compiler security

Una extensión que registra:

```text
CustomExpression
```

deberá mantener:

```text
values → parameters
identifiers → structural quoting
```

---

# 198. Extension contract

Conceptualmente:

```php
interface SecureQueryNodeCompiler
{
    public function compile(
        QueryNode $node,
        CompilationContext $context
    ): SqlFragment;
}
```

---

# 199. Extension ≠ Automatic Trust

Custom compilers forman parte del trusted computing base.

---

# 200. Extension conformance

VoltStack deberá proporcionar tests de conformidad.

---

# 201. Compiler conformance tests

Cada compiler oficial deberá superar casos con:

```text
quotes
semicolons
comments
unicode
null bytes
control characters
wildcards
long strings
nested JSON
```

como valores.

---

# 202. Identifier conformance

Identifiers deberán probarse con:

```text
quotes
dots
spaces
reserved words
unicode
delimiter characters
control characters
empty names
oversized names
```

---

# 203. Fuzzing

Targets prioritarios:

```text
IdentifierParser
ExternalFieldResolver
OperatorResolver
JsonPathParser
RawExpressionParser
Query DSL Parser
ParameterBinder
```

---

# 204. Property test

Una propiedad importante:

> Para cualquier valor `V` aceptado como `QueryParameter`, modificar `V` nunca deberá modificar la estructura del SQL compilado, excepto donde su tipo semántico determine legítimamente una forma estructural distinta previamente definida.

---

# 205. Structural fingerprint property

Para:

```text
Q(value=A)
Q(value=B)
```

si únicamente cambia un parámetro normal:

```text
StructuralFingerprint(QA)
=
StructuralFingerprint(QB)
```

---

# 206. Example property

```text
email = "alice@example.com"
```

y:

```text
email = "' OR 1=1 --"
```

deben producir la misma estructura:

```sql
WHERE email = ?
```

---

# 207. Parameter count property

El segundo payload deberá seguir siendo exactamente un parámetro.

---

# 208. SQL structure immutability

Formalmente:

Sea:

```text
S(Q)
```

la estructura SQL compilada y:

```text
V(Q)
```

el conjunto de valores.

Para una query parametrizable:

```text
S(Q, v1) = S(Q, v2)
```

para cualesquiera valores válidos `v1`, `v2` del mismo contexto estructural.

---

# 209. Structural inputs

Cuando cambia legítimamente:

```text
SortDirection
```

la estructura puede cambiar:

```text
ASC
→
DESC
```

pero el valor deberá provenir de:

```text
validated semantic enum
```

no del string externo.

---

# 210. Identifier safety formalization

Sea:

```text
E
```

un alias externo.

Debe existir:

```text
ResolveIdentifier(E) → I
```

donde `I` es un Identifier válido conocido.

Si no existe:

```text
ResolveIdentifier(E) = UNKNOWN
```

entonces:

```text
Reject(E)
```

---

# 211. No sanitize fallback

Nunca:

```text
ResolveIdentifier(E) = UNKNOWN
↓
sanitize(E)
↓
use anyway
```

---

# 212. Raw boundary formalization

Una query:

```text
Q
```

con raw node:

```text
R
```

deberá satisfacer:

```text
RawAllowed(R, Context, Policy)
=
ALLOW
```

antes de compilarse.

---

# 213. Raw values

Incluso dentro de `R`:

```text
external values
```

deberán permanecer parámetros.

---

# 214. Threat: concatenated ORDER BY

Ataque:

```text
sort=name DESC; DELETE FROM users
```

Defensa:

```text
External sort
↓
AllowedSortResolver
↓
UNKNOWN
↓
REJECT
```

---

# 215. Threat: malicious operator

Ataque:

```text
operator="= 1 OR 1=1 --"
```

Defensa:

```text
OperatorResolver
↓
UNKNOWN
↓
REJECT
```

---

# 216. Threat: table injection

Ataque:

```text
table="users; DROP TABLE orders"
```

Defensa:

```text
No public arbitrary table-string API
+
Identifier Resolution
```

---

# 217. Threat: LIKE wildcard abuse

Input:

```text
%
```

No es necesariamente SQL Injection.

Pero una API de búsqueda literal deberá convertirlo en patrón literal según policy.

---

# 218. Threat: JSON path injection

Input externo no deberá concatenarse:

```text
JSON_EXTRACT(data, '$." + input + "')
```

Debe pasar por `JsonPath`.

---

# 219. Threat: second-order injection

Un string recuperado de DB continúa siendo un value hasta que un resolver autorizado lo transforme en un semantic object.

---

# 220. Threat: extension compiler

Un compiler defectuoso puede reintroducir injection.

Por eso los custom compilers requieren conformance testing.

---

# 221. Threat: debug execution

SQL interpolado para debug nunca deberá poder enviarse al Executor.

---

# 222. Threat: stored query templates

Templates SQL almacenados en DB deberán considerarse raw SQL.

No trusted metadata por default.

---

# 223. Stored SQL policy

Podrá existir una feature explícita:

```text
StoredQueryDefinition
```

pero requerirá:

```text
trusted source
version
integrity
policy
parameter schema
```

---

# 224. Stored Query ≠ Ordinary Data

Solo una conversión explícita y autorizada podrá elevar data almacenada a definición ejecutable.

---

# 225. Security context

```php
final readonly class SqlInjectionSecurityContext
{
    public function __construct(
        public QueryOrigin $origin,
        public SecurityPrincipalReference $principal,
        public DatabaseOperation $operation,
        public RawSqlPolicy $rawSqlPolicy,
        public SecurityPolicyGeneration $generation,
    ) {}
}
```

---

# 226. QueryOrigin

```php
enum QueryOrigin
{
    case FRAMEWORK;
    case APPLICATION;
    case ORM;
    case MIGRATION;
    case ADMINISTRATION;
    case EXTENSION;
}
```

---

# 227. Origin does not bypass safety

Incluso:

```text
FRAMEWORK
```

deberá preservar AST/compiler invariants.

---

# 228. Migration origin

Puede permitir operaciones estructurales adicionales, pero no convertir values en SQL strings.

---

# 229. Administration origin

Igualmente.

---

# 230. Persistent runtime

SQL injection security state deberá ser:

```text
operation scoped
```

---

# 231. No static raw policy

No:

```php
static $allowRaw = true;
```

---

# 232. No request leakage

Si Request A habilita una policy especial:

```text
Request B
```

no deberá heredarla.

---

# 233. Immutable registries

Registries como:

```text
OperatorRegistry
FunctionRegistry
IdentifierMetadataRegistry
```

podrán ser worker-shared si son:

```text
immutable
compiled
request-independent
```

---

# 234. Dynamic registration

No permitir registrar funciones/operadores desde request input.

---

# 235. Tenant context

Tenant identifiers tampoco deberán concatenarse.

Ejemplo incorrecto:

```php
$table = 'tenant_' . $_GET['tenant'];
```

---

# 236. Tenant physical routing

Si una arquitectura utiliza schema/database/table por tenant:

```text
TenantContext
↓
TenantStorageResolver
↓
Validated Physical Resource
↓
Identifier
```

---

# 237. Tenant ID ≠ Physical Identifier

El tenant ID lógico no deberá convertirse automáticamente en table/schema name.

---

# 238. Sharding

Shard identifiers se resolverán mediante topology metadata.

No desde strings arbitrarios.

---

# 239. Replica routing

Endpoint selection no participa en SQL structure y deberá permanecer fuera del SQL compiler.

---

# 240. Security + cache

Compiled Query Cache podrá reutilizar estructura segura.

---

# 241. Cache key

Debe incluir cuando corresponda:

```text
query structural fingerprint
compiler generation
platform
dialect
metadata generation
```

---

# 242. Raw cache

Queries con RawExpression podrán cachearse únicamente si su estructura raw forma parte del fingerprint y la policy lo permite.

---

# 243. Security metadata cache

Allowed fields/functions/operators podrán precompilarse.

---

# 244. Cache poisoning

Un input externo no deberá poder insertar nuevas definiciones en registries/cache de metadata.

---

# 245. Observability

Métricas:

```text
database.sql_security.query_checked
database.sql_security.query_rejected
database.sql_security.raw_boundary
database.sql_security.identifier_rejected
database.sql_security.parameterization_failure
```

---

# 246. No payload labels

Nunca:

```text
payload="' OR 1=1 --"
```

como metric label.

---

# 247. Trace attributes

Preferir:

```text
db.query.fingerprint
db.operation
db.raw_boundary
db.security.result
db.security.finding_code
```

---

# 248. Sampling

Security rejection events podrán tener reglas diferentes de sampling, pero siempre bounded.

---

# 249. Logging

Un log seguro:

```text
Database query rejected:
reason=UNTRUSTED_IDENTIFIER
fingerprint=...
```

---

# 250. Unsafe log

Evitar:

```text
Attacker payload: [full raw payload]
SQL: [full query with secrets]
```

por default.

---

# 251. Developer diagnostics

En desarrollo, un diagnostic puede mostrar:

```text
Field "foo" cannot be used for sorting.

Allowed aliases:
- name
- created
- status
```

sin exponer internals innecesarios.

---

# 252. Production diagnostics

Podrá reducirse a:

```text
Invalid sort field.
```

---

# 253. Security testing matrix

| Superficie | Test principal |
|---|---|
| WHERE | parameter preservation |
| INSERT | value binding |
| UPDATE | value binding |
| DELETE | predicate binding |
| IN | list parameterization |
| LIKE | binding + wildcard semantics |
| ORDER BY | allowlist resolution |
| GROUP BY | identifier resolution |
| JOIN | relation resolution |
| operators | enum resolution |
| functions | registry resolution |
| JSON paths | typed parsing |
| CTE | structural AST |
| subquery | QueryModel composition |
| DDL | typed identifiers |
| raw SQL | policy + bindings |
| stored data | second-order protection |
| extension | compiler conformance |

---

# 254. Attack corpus

Testing podrá incluir strings como:

```text
'
"
`
;
--
#
/*
*/
OR 1=1
UNION SELECT
DROP TABLE
${...}
\0
Unicode control characters
very long strings
nested quotes
```

Siempre como datos de test, no como lógica de detección primaria.

---

# 255. Expected property

Todos estos payloads utilizados como valores deberán permanecer:

```text
parameters
```

---

# 256. No SQL keyword inspection requirement

Una query segura no necesita determinar si un parámetro contiene:

```text
DROP TABLE
```

porque el valor no será interpretado como estructura SQL.

---

# 257. Developer Experience

La API segura deberá ser natural:

```php
User::query()
    ->where('email', $email)
    ->orderBy(UserFields::CREATED_AT)
    ->limit(50)
    ->get();
```

---

# 258. Dynamic API

```php
$sort = $sortResolver->resolve(
    $request->input('sort')
);

User::query()
    ->orderBy($sort)
    ->get();
```

---

# 259. Explicit unsafe API

```php
$query->selectRaw(
    Raw::sql('COUNT(DISTINCT user_id)')
);
```

deberá ser visualmente distinta.

---

# 260. Security ergonomics rule

> **VoltStack deberá hacer que la forma más corta y natural de construir una consulta sea también la forma segura.**

---

# 261. Proposed namespaces

```text
VoltStack\Quantum\Database\Security\SqlInjection
VoltStack\Quantum\Database\Query\Identifier
VoltStack\Quantum\Database\Query\Parameter
VoltStack\Quantum\Database\Query\Raw
```

---

# 262. Directory structure

```text
src/Quantum/Database/
│
├── Security/
│   └── SqlInjection/
│       ├── Contract/
│       │   ├── SqlInjectionAnalyzer.php
│       │   ├── IdentifierSecurityPolicy.php
│       │   └── RawSqlSecurityPolicy.php
│       │
│       ├── Analysis/
│       │   ├── SqlInjectionAnalysis.php
│       │   ├── SqlInjectionSafety.php
│       │   ├── SqlInjectionFinding.php
│       │   └── SecurityFindingSeverity.php
│       │
│       ├── Identifier/
│       │   ├── IdentifierValidator.php
│       │   ├── ExternalIdentifierResolver.php
│       │   └── IdentifierSecurity.php
│       │
│       ├── Raw/
│       │   ├── RawSqlPolicy.php
│       │   ├── RawSqlSecurity.php
│       │   ├── RawExpressionOrigin.php
│       │   └── TrustedSqlFragment.php
│       │
│       ├── Context/
│       │   ├── SqlInjectionSecurityContext.php
│       │   └── QueryOrigin.php
│       │
│       ├── Diagnostic/
│       │   ├── SqlInjectionDiagnostic.php
│       │   └── SqlInjectionDiagnosticCode.php
│       │
│       ├── Telemetry/
│       │   └── SqlInjectionSecurityTelemetry.php
│       │
│       ├── Testing/
│       │   ├── SqlInjectionPayloadCorpus.php
│       │   ├── SqlInjectionAssertions.php
│       │   └── CompilerSecurityConformanceSuite.php
│       │
│       └── Exception/
│           ├── SqlInjectionPreventionException.php
│           ├── UnsafeSqlExpressionException.php
│           ├── UnsafeIdentifierException.php
│           ├── UnresolvedIdentifierException.php
│           ├── UnsafeRawSqlException.php
│           ├── UnsafeOperatorException.php
│           ├── UnsafeFunctionException.php
│           ├── UnsafeJsonPathException.php
│           ├── ParameterizationException.php
│           └── SqlInjectionSafetyUnknownException.php
│
├── Query/
│   ├── Identifier/
│   │   ├── Identifier.php
│   │   ├── IdentifierKind.php
│   │   ├── QualifiedIdentifier.php
│   │   └── ColumnReference.php
│   │
│   ├── Parameter/
│   │   ├── QueryParameter.php
│   │   ├── ParameterId.php
│   │   └── ParameterBag.php
│   │
│   └── Raw/
│       ├── RawExpression.php
│       └── Raw.php
│
└── Compiler/
    └── Security/
        ├── IdentifierQuoter.php
        └── SecureCompilationContext.php
```

---

# 263. Core interfaces

```php
interface SqlInjectionAnalyzer
{
    public function analyze(
        QueryModel $query,
        SqlInjectionSecurityContext $context
    ): SqlInjectionAnalysis;
}
```

```php
interface IdentifierValidator
{
    public function validate(
        Identifier $identifier,
        IdentifierContext $context
    ): IdentifierValidationResult;
}
```

```php
interface ExternalIdentifierResolver
{
    public function resolve(
        string $externalAlias,
        IdentifierResolutionContext $context
    ): ResolvedIdentifier;
}
```

---

# 264. Integration with Query Pipeline

```text
Query Builder
     ↓
Query Model
     ↓
Normalization
     ↓
Validation
     ↓
Semantic Analysis
     ↓
SQL Injection Security
     ↓
Optimization
     ↓
Planning
     ↓
Compilation
     ↓
Execution
```

---

# 265. Security before optimizer

Un AST estructuralmente inseguro no deberá entrar al Optimizer normal.

---

# 266. Optimizer invariant

Optimizer tampoco deberá convertir valores en estructura SQL.

---

# 267. Rewrite rules

Una rewrite:

```text
Predicate A
↓
Predicate B
```

deberá preservar parameter identity/semantics.

---

# 268. Planner invariant

Planner opera sobre nodos semánticos seguros.

No crea SQL strings.

---

# 269. Compiler invariant

Compiler es el único subsistema que transforma estructura semántica en SQL.

---

# 270. Executor invariant

Executor no concatena parámetros.

---

# 271. Driver invariant

Driver preserva binding.

---

# 272. Architectural invariants

## DB-SQLSEC-001
Data nunca será SQL structure implícitamente.

## DB-SQLSEC-002
External strings serán untrusted por default.

## DB-SQLSEC-003
Query values serán parámetros cuando la plataforma lo permita.

## DB-SQLSEC-004
Prepared statements serán la ruta normal.

## DB-SQLSEC-005
Prepared statements no serán considerados defensa completa.

## DB-SQLSEC-006
Identifiers no serán parámetros.

## DB-SQLSEC-007
Identifiers serán objetos semánticos.

## DB-SQLSEC-008
Dynamic external identifiers requerirán resolución.

## DB-SQLSEC-009
Unknown external identifier será rechazado.

## DB-SQLSEC-010
Identifier sanitization no sustituirá allowlist resolution.

## DB-SQLSEC-011
Identifier quoting será responsabilidad del Compiler/Dialect.

## DB-SQLSEC-012
Quoting no sustituirá validation.

## DB-SQLSEC-013
Quoting no sustituirá authorization.

## DB-SQLSEC-014
Qualified identifiers serán estructurados.

## DB-SQLSEC-015
Aliases serán identifiers.

## DB-SQLSEC-016
ORDER BY expressions serán semánticas.

## DB-SQLSEC-017
Sort direction será enum/value semántico.

## DB-SQLSEC-018
External sort fields requerirán allowlist.

## DB-SQLSEC-019
GROUP BY fields serán estructurados.

## DB-SQLSEC-020
HAVING values serán parametrizados.

## DB-SQLSEC-021
Operators serán semantic enums/objects.

## DB-SQLSEC-022
External operators requerirán resolver.

## DB-SQLSEC-023
Unknown operator será rechazado.

## DB-SQLSEC-024
JOIN target será estructural.

## DB-SQLSEC-025
JOIN type será semántico.

## DB-SQLSEC-026
JOIN predicates serán AST.

## DB-SQLSEC-027
ORM relationship JOINs derivarán de metadata.

## DB-SQLSEC-028
Subqueries serán Query Models.

## DB-SQLSEC-029
CTEs serán AST nodes.

## DB-SQLSEC-030
CTE names serán identifiers.

## DB-SQLSEC-031
Set operations serán semantic enums.

## DB-SQLSEC-032
Functions serán registry-backed.

## DB-SQLSEC-033
External arbitrary function names serán rechazados.

## DB-SQLSEC-034
JSON paths serán objetos semánticos.

## DB-SQLSEC-035
External JSON paths serán parseados/validados.

## DB-SQLSEC-036
JSON paths tendrán límites de recursos.

## DB-SQLSEC-037
Full-text terms serán values.

## DB-SQLSEC-038
Full-text configuration será estructural.

## DB-SQLSEC-039
LIMIT será typed integer.

## DB-SQLSEC-040
OFFSET será typed integer.

## DB-SQLSEC-041
External numeric input será validado antes de compilation.

## DB-SQLSEC-042
Cursor values nunca serán SQL.

## DB-SQLSEC-043
Chunk continuation nunca será SQL.

## DB-SQLSEC-044
Bulk values serán parámetros.

## DB-SQLSEC-045
Import values serán parámetros.

## DB-SQLSEC-046
Schema Builder utilizará AST.

## DB-SQLSEC-047
DDL identifiers serán estructurados.

## DB-SQLSEC-048
Migration code no deshabilitará compiler invariants.

## DB-SQLSEC-049
Savepoint names serán identifiers.

## DB-SQLSEC-050
Raw SQL será explícito.

## DB-SQLSEC-051
Raw SQL será security boundary.

## DB-SQLSEC-052
Raw SQL podrá estar sujeto a policy.

## DB-SQLSEC-053
Raw SQL será observable.

## DB-SQLSEC-054
Raw SQL values deberán seguir usando binding.

## DB-SQLSEC-055
Raw origin no implicará seguridad.

## DB-SQLSEC-056
Heuristic payload detection no será defensa principal.

## DB-SQLSEC-057
Stored data no será trusted SQL.

## DB-SQLSEC-058
Second-order injection será parte del threat model.

## DB-SQLSEC-059
Stored configuration deberá resolverse semánticamente.

## DB-SQLSEC-060
Stored SQL templates serán raw/trusted-code boundary.

## DB-SQLSEC-061
Query AST separará values y structure por tipo.

## DB-SQLSEC-062
Generic ambiguous string nodes serán evitados.

## DB-SQLSEC-063
Semantic Analyzer verificará contextos AST.

## DB-SQLSEC-064
Compiler nunca interpolará QueryParameter values.

## DB-SQLSEC-065
CompiledQuery separará SQL y bindings.

## DB-SQLSEC-066
CompiledSql no contendrá parámetros interpolados normales.

## DB-SQLSEC-067
Internal structural literals provendrán de tipos conocidos.

## DB-SQLSEC-068
Driver nunca reconstruirá SQL mediante string replacement.

## DB-SQLSEC-069
Prepared statement capability será explícita.

## DB-SQLSEC-070
Emulated prepares no serán asumidos seguros sin contrato.

## DB-SQLSEC-071
Character encoding será connection configuration.

## DB-SQLSEC-072
Manual escaping no será defensa primaria.

## DB-SQLSEC-073
Keyword blacklists no serán defensa primaria.

## DB-SQLSEC-074
Comment stripping no será defensa primaria.

## DB-SQLSEC-075
Multi-statements estarán deshabilitados por default cuando sea viable.

## DB-SQLSEC-076
Stored procedures tendrán identifiers y parameters separados.

## DB-SQLSEC-077
Stored procedure internals estarán fuera de garantías completas del compiler.

## DB-SQLSEC-078
Database-side dynamic SQL será external security boundary.

## DB-SQLSEC-079
SQLInjectionAnalyzer no ejecutará queries.

## DB-SQLSEC-080
UNKNOWN safety no será SAFE.

## DB-SQLSEC-081
Required UNKNOWN safety fallará cerrada.

## DB-SQLSEC-082
Typed AST sin raw nodes tendrá fast path.

## DB-SQLSEC-083
Security analysis podrá versionarse por generation.

## DB-SQLSEC-084
Generation mismatch invalidará análisis reutilizado cuando corresponda.

## DB-SQLSEC-085
Compiled Query Cache preservará parameter separation.

## DB-SQLSEC-086
Compiled Query Cache key incluirá compiler/platform generations necesarias.

## DB-SQLSEC-087
Query fingerprints serán estructurales.

## DB-SQLSEC-088
Normal parameter values no deberán aparecer en structural fingerprint.

## DB-SQLSEC-089
Raw structure sí afectará fingerprint.

## DB-SQLSEC-090
Telemetry no registrará parámetros completos por default.

## DB-SQLSEC-091
Debug aplicará redaction.

## DB-SQLSEC-092
Interpolated debug SQL nunca será executable representation.

## DB-SQLSEC-093
Security exceptions minimizarán payload leakage.

## DB-SQLSEC-094
Rejected queries podrán auditarse.

## DB-SQLSEC-095
Invalid input no será automáticamente clasificado como ataque.

## DB-SQLSEC-096
Rejected query no emitirá QueryExecuting.

## DB-SQLSEC-097
Custom Query Nodes deberán preservar value/structure separation.

## DB-SQLSEC-098
Custom compilers serán trusted components.

## DB-SQLSEC-099
Custom compilers tendrán conformance tests.

## DB-SQLSEC-100
Official compilers tendrán security conformance tests.

## DB-SQLSEC-101
Identifier parser tendrá fuzz tests.

## DB-SQLSEC-102
Operator resolver tendrá fuzz tests.

## DB-SQLSEC-103
JSON path parser tendrá fuzz tests.

## DB-SQLSEC-104
Query DSL parser tendrá fuzz tests cuando exista.

## DB-SQLSEC-105
Parameter values no cambiarán estructura SQL.

## DB-SQLSEC-106
Malicious-looking value seguirá siendo un parámetro.

## DB-SQLSEC-107
Structural external input deberá elevarse mediante resolver explícito.

## DB-SQLSEC-108
Unknown structural input nunca tendrá sanitize-and-use fallback.

## DB-SQLSEC-109
Raw nodes requerirán policy permitida.

## DB-SQLSEC-110
LIKE wildcard semantics serán distintas de SQL Injection.

## DB-SQLSEC-111
LIKE literal mode escapará wildcards según dialecto.

## DB-SQLSEC-112
Tenant ID lógico no será physical identifier automáticamente.

## DB-SQLSEC-113
Tenant physical resources vendrán de trusted resolver.

## DB-SQLSEC-114
Shard identifiers vendrán de topology metadata.

## DB-SQLSEC-115
Endpoint routing nunca será SQL structure.

## DB-SQLSEC-116
Security metadata cache no será writable desde request input.

## DB-SQLSEC-117
Metric labels no contendrán attack payloads.

## DB-SQLSEC-118
Logs no expondrán SQL+secrets por default.

## DB-SQLSEC-119
Safe API será más ergonómica que raw API.

## DB-SQLSEC-120
Raw escape hatch será visualmente explícito.

## DB-SQLSEC-121
Security registries serán immutable durante normal request processing.

## DB-SQLSEC-122
Request input no registrará operadores.

## DB-SQLSEC-123
Request input no registrará functions.

## DB-SQLSEC-124
Request input no registrará compilers.

## DB-SQLSEC-125
Request-specific raw policy no será static global state.

## DB-SQLSEC-126
Persistent workers resetearán security context.

## DB-SQLSEC-127
ORM utilizará el mismo Query AST seguro.

## DB-SQLSEC-128
Repository utilizará el mismo Query AST seguro.

## DB-SQLSEC-129
Model API utilizará el mismo Query AST seguro.

## DB-SQLSEC-130
Schema utilizará identifiers seguros.

## DB-SQLSEC-131
Migration utilizará compiler estructural.

## DB-SQLSEC-132
Pagination utilizará numeric/value objects.

## DB-SQLSEC-133
Cursor pagination reutilizará typed boundaries.

## DB-SQLSEC-134
Chunk processing reutilizará typed boundaries.

## DB-SQLSEC-135
Lazy collection no creará SQL propio.

## DB-SQLSEC-136
Bulk Insert no creará SQL mediante dataset concatenation.

## DB-SQLSEC-137
Bulk Update no interpolará nuevos valores.

## DB-SQLSEC-138
Bulk Delete utilizará predicates AST.

## DB-SQLSEC-139
Import no convertirá filas en SQL strings.

## DB-SQLSEC-140
Export filters utilizarán Query AST.

## DB-SQLSEC-141
Optimizer preservará parameter semantics.

## DB-SQLSEC-142
Planner no generará SQL.

## DB-SQLSEC-143
Compiler será única capa normal de generación SQL.

## DB-SQLSEC-144
Executor no reinterpretará strings como SQL structure.

## DB-SQLSEC-145
Driver recibirá CompiledQuery/statement+bindings.

## DB-SQLSEC-146
Compiler security será capability-aware.

## DB-SQLSEC-147
Version no equivaldrá a safe prepare capability.

## DB-SQLSEC-148
MySQL y MariaDB tendrán análisis independiente cuando corresponda.

## DB-SQLSEC-149
PostgreSQL compiler preservará las mismas invariantes.

## DB-SQLSEC-150
SQLite compiler preservará las mismas invariantes.

## DB-SQLSEC-151
Application input será data hasta resolución explícita.

## DB-SQLSEC-152
Database-loaded data seguirá siendo data.

## DB-SQLSEC-153
Signed cursor data seguirá siendo data.

## DB-SQLSEC-154
Authenticated input seguirá siendo untrusted SQL structure.

## DB-SQLSEC-155
Authorized input seguirá requiriendo safe SQL construction.

## DB-SQLSEC-156
Authorization no sustituirá SQL injection prevention.

## DB-SQLSEC-157
SQL injection prevention no sustituirá authorization.

## DB-SQLSEC-158
Validation no sustituirá binding.

## DB-SQLSEC-159
Binding no sustituirá identifier resolution.

## DB-SQLSEC-160
Identifier resolution no sustituirá resource governance.

## DB-SQLSEC-161
Security será defense in depth.

## DB-SQLSEC-162
No habrá SQL injection bypass implícito para framework internals.

## DB-SQLSEC-163
No habrá SQL injection bypass implícito para migrations.

## DB-SQLSEC-164
No habrá SQL injection bypass implícito para admin operations.

## DB-SQLSEC-165
Raw SQL será la única frontera intencional para SQL libre.

## DB-SQLSEC-166
Raw SQL nunca será producido automáticamente desde user input.

## DB-SQLSEC-167
Raw SQL providers podrán restringirse por environment/policy.

## DB-SQLSEC-168
Unsafe query detection será explicable.

## DB-SQLSEC-169
Security findings tendrán códigos estables.

## DB-SQLSEC-170
Security failures serán testeables determinísticamente.

---

# 273. Anti-patterns

## Anti-pattern 1 — String concatenation

```php
$sql = "SELECT * FROM users WHERE email = '$email'";
```

### Correcto

```php
User::query()
    ->where('email', $email)
    ->first();
```

---

## Anti-pattern 2 — Dynamic ORDER BY

```php
$sql .= ' ORDER BY ' . $_GET['sort'];
```

### Correcto

```php
$sort = $allowedSorts->resolve(
    $request->input('sort')
);

$query->orderBy($sort);
```

---

## Anti-pattern 3 — Operator concatenation

```php
$sql .= " age {$operator} ?";
```

### Correcto

```php
$operator = $operatorResolver->resolve(
    $externalOperator
);
```

---

## Anti-pattern 4 — IN concatenation

```php
$sql .= ' WHERE id IN (' . implode(',', $ids) . ')';
```

### Correcto

```php
$query->whereIn('id', $ids);
```

---

## Anti-pattern 5 — Raw user input

```php
$query->whereRaw(
    $request->input('filter')
);
```

### Correcto

Parsear/resolver el filtro a AST.

---

## Anti-pattern 6 — Stored data trusted

```php
$query->orderByRaw(
    $userPreference->sortExpression
);
```

### Correcto

```text
stored alias
↓
AllowedSortResolver
↓
OrderExpression
```

---

## Anti-pattern 7 — Table by tenant string

```php
$table = 'tenant_' . $tenantId;
```

### Correcto

```text
TenantContext
↓
PhysicalStorageResolver
↓
Validated TableIdentifier
```

---

## Anti-pattern 8 — Debug SQL execution

```php
$executor->execute(
    $debugger->interpolate($query)
);
```

### Correcto

El Executor solo acepta la representación compilada ejecutable.

---

# 274. Security guarantee boundaries

VoltStack podrá garantizar la seguridad estructural de:

```text
Query Builder
ORM-generated queries
Schema Builder
Migration Compiler
Bulk Engine
Pagination
Cursor Pagination
Chunk Processing
Lazy traversal
```

cuando se utilicen sus APIs seguras.

---

# 275. Reduced guarantee boundary

Cuando se utiliza:

```text
Raw SQL
```

VoltStack ya no puede demostrar completamente la seguridad de la estructura proporcionada por el desarrollador.

Sin embargo aún podrá garantizar:

```text
bound raw parameters
connection security
execution policy
resource policy
audit
```

según contexto.

---

# 276. External boundaries

VoltStack tampoco puede garantizar internamente la seguridad de:

```text
stored procedure implementation
database triggers containing dynamic SQL
external SQL proxies
custom database extensions
malicious custom drivers
malicious custom compilers
```

---

# 277. Explicit trust elevation

La arquitectura deberá hacer visible cuándo ocurre:

```text
Data
↓
Trusted SQL Structure
```

Esta transición nunca deberá ser accidental.

---

# 278. Security equation

Para una consulta normal:

```text
SafeQuery
=
TypedStructure
∧ ValidIdentifiers
∧ BoundValues
∧ ValidOperators
∧ ValidFunctions
∧ SecureCompiler
∧ SecureDriverBinding
```

Si existe raw SQL:

```text
SafeRawQuery
=
ExplicitRawBoundary
∧ PolicyAllows
∧ DeveloperTrustedStructure
∧ BoundExternalValues
∧ SecureCompilerIntegration
∧ SecureDriverBinding
```

---

# 279. Layer responsibilities

```text
Application
    classify business input

Query Input Security
    resolve external query capabilities

Query Builder
    build typed AST

Semantic Analyzer
    validate meaning

SQL Injection Prevention
    enforce structure/data separation

Compiler
    generate dialect SQL

Parameter Binder
    bind typed values

Driver
    execute statement safely

Database
    interpret compiled SQL
```

---

# 280. Relation with document 228

Existe una separación importante:

```text
227 SQL Injection Prevention
```

responde:

> ¿Cómo impedimos que data se convierta en SQL?

Mientras:

```text
228 Query Input Security
```

responderá:

> ¿Qué capacidades de consulta permitimos que un consumidor externo solicite?

Por ejemplo, una query puede estar perfectamente protegida contra SQL Injection y aun permitir:

```text
expensive filters
forbidden fields
huge limits
deep relationship traversal
sensitive sorting
complex query DoS
```

Por eso ambos sistemas son independientes pero complementarios.

---

# 281. Regla maestra

> **VoltStack nunca intentará determinar si un valor “parece SQL malicioso” para decidir si es seguro. La arquitectura hará que los valores permanezcan valores y que únicamente estructuras semánticas explícitamente reconocidas puedan convertirse en SQL.**

En forma compacta:

```text
Values
→ Parameters

Identifiers
→ Resolvers + Metadata + Quoting

Operators
→ Enums

Functions
→ Registry

Query Structure
→ AST

SQL Generation
→ Compiler

Execution
→ Prepared Statements

Raw SQL
→ Explicit Security Boundary
```

---

# 282. Resultado arquitectónico

La ruta segura completa será:

```text
External Input
       ↓
Classification
       ↓
┌──────┴─────────────┐
│                    │
▼                    ▼
Value           Structural Request
│                    │
▼                    ▼
Type System      Resolver/Allowlist
│                    │
▼                    ▼
Parameter       Semantic AST Node
│                    │
└──────────┬─────────┘
           ▼
        Query AST
           ↓
   Semantic Validation
           ↓
 SQL Injection Analysis
           ↓
        Compiler
           ↓
 ┌─────────┴──────────┐
 ▼                    ▼
SQL Structure       Bindings
 ▼                    ▼
Identifier         Typed
 Quoting          Parameters
 └─────────┬──────────┘
           ▼
    Prepared Statement
           ↓
         Driver
           ↓
        Database
```

Esto permite que VoltStack ofrezca una API similar en ergonomía a Laravel:

```php
User::query()
    ->where('email', $email)
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->limit(50)
    ->get();
```

mientras internamente utiliza una arquitectura considerablemente más estricta:

```text
Model API
↓
Query Builder
↓
Typed Query Model
↓
AST
↓
Semantic Security
↓
Compiler
↓
CompiledQuery
├── SQL Structure
└── Typed Bindings
↓
Prepared Statement
↓
Driver
```

La comodidad de la API pública no deberá reducir las garantías internas.

---

# 283. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
○ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
○ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
○ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
○ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
○ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
○ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 284. Siguiente documento

```text
228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
```

El siguiente documento deberá extender esta arquitectura para definir cómo VoltStack controla **qué capacidades de consulta puede solicitar un consumidor externo**, incluyendo:

```text
Dynamic Filter Security
Allowed Field System
Allowed Operator System
Dynamic Sort Security
Dynamic Relationship Filtering
Field Selection Security
Projection Security
Query Complexity Limits
Relationship Depth Limits
Filter Count Limits
IN List Limits
Pagination Limits
Aggregation Security
Grouping Security
Search Security
JSON Query Input Security
Query DSL Security
Query Input Normalization
Query Input Validation
Authorization-aware Fields
Sensitive Fields
Query Cost Budgets
Resource Exhaustion Prevention
Query Input Policy
Security Diagnostics
Testing
Persistent Runtime Isolation
```

bajo la regla:

> **Una consulta libre de SQL Injection no es necesariamente una consulta segura: VoltStack deberá controlar no solo cómo se construye SQL, sino también qué operaciones, campos, relaciones y niveles de complejidad puede solicitar cada consumidor.**