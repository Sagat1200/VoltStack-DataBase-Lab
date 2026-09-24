# 273_DATABASE_JSON_QUERY_SYSTEM.md

# VoltStack Quantum Database
## JSON Query System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 273 — JSON Query System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md`  
**Siguiente documento:** `274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **JSON Query System** de VoltStack Database.

El sistema permitirá consultar, proyectar, comparar, ordenar, agregar y modificar estructuras JSON almacenadas en bases de datos compatibles, manteniendo una API semántica común para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin permitir que las diferencias sintácticas entre:

```text
JSON_EXTRACT(...)
->>
#>>
json_extract(...)
JSON_VALUE(...)
JSON_QUERY(...)
jsonb_*
```

se propaguen hacia:

```text
Application
Model API
Repository
ORM
Query Builder
```

La regla central será:

> **JSON Query en VoltStack se modela como una extensión tipada del Query AST; las rutas, valores, predicados y mutaciones JSON se expresan semánticamente y sólo el compiler de plataforma decide su representación SQL.**

Por tanto:

```text
Application
    ↓
JSON Query API
    ↓
JSON AST
    ↓
Semantic Analysis
    ↓
Type Resolution
    ↓
Capability Resolution
    ↓
Query Planner
    ↓
Platform Compiler
    ↓
Native JSON Operations
```

y nunca:

```text
Application
    ↓
vendor JSON syntax
```

---

# 2. Objetivos

El sistema deberá proporcionar:

- tipo lógico JSON;
- JSON document expressions;
- rutas JSON tipadas;
- extracción de valores;
- extracción de objetos;
- extracción de arrays;
- predicates;
- existence checks;
- containment;
- comparación;
- búsqueda en arrays;
- longitud;
- tipos internos JSON;
- ordenamiento;
- agregaciones;
- mutaciones;
- JSON Set;
- JSON Remove;
- JSON Replace;
- JSON Merge;
- JSON Patch;
- proyecciones;
- integración ORM;
- integración Query Builder;
- integración con Type System;
- índices;
- generated columns;
- optimización;
- capabilities;
- seguridad;
- telemetría;
- Multitenancy;
- Sharding;
- extensibilidad.

---

# 3. Regla semántica fundamental

VoltStack distinguirá obligatoriamente:

```text
SQL NULL
≠
JSON null
≠
missing JSON path
```

Ésta será una de las invariantes más importantes del sistema.

---

# 4. Ejemplo

Documento:

```json
{
  "name": "VoltStack",
  "description": null
}
```

Tenemos:

```text
$.name
        → JSON string "VoltStack"

$.description
        → JSON null

$.version
        → MISSING
```

Si la columna SQL completa contiene:

```sql
NULL
```

tenemos un cuarto contexto:

```text
SQL NULL document
```

---

# 5. Modelo conceptual

```text
SQL Value
│
├── SQL NULL
│
└── JSON Document
     │
     ├── Object
     ├── Array
     ├── String
     ├── Number
     ├── Boolean
     ├── JSON null
     └── Missing Path
```

`Missing Path` no es un valor JSON almacenado.

Representa:

```text
absence of a requested path
```

---

# 6. JSON Query ≠ String Query

Aunque ciertos motores almacenen JSON mediante representaciones textuales:

```text
JSON Document
≠
ordinary string
```

VoltStack mantendrá la distinción lógica.

---

# 7. JSON Query ≠ PHP Array Access

La expresión:

```php
$data['user']['name']
```

es acceso a memoria PHP.

Una expresión:

```php
->json('data', '$.user.name')
```

representa una operación ejecutable por la base de datos.

---

# 8. JSON Query ≠ JSON Serialization

```text
JSON Query
≠
json_encode()
≠
json_decode()
```

La serialización pertenece al Value Conversion System.

---

# 9. JSON Query ≠ JSON Type System

El documento:

```text
161_DATABASE_JSON_TYPE_SYSTEM.md
```

define cómo los valores JSON son representados y convertidos.

Este documento define:

```text
how JSON values are queried and manipulated
```

---

# 10. Integración arquitectónica

```text
JSON Type System
       │
       ▼
JSON Query AST
       │
       ▼
Semantic Query Engine
       │
       ▼
Query Planner
       │
       ▼
Optimizer
       │
       ▼
Platform Compiler
```

---

# 11. Arquitectura general

```text
Application
    │
    ▼
Query Builder / ORM
    │
    ▼
JSON Query API
    │
    ▼
JSON Expressions
    │
    ▼
JSON AST
    │
    ▼
JSON Semantic Analyzer
    │
    ▼
JSON Type Resolver
    │
    ▼
Capability Resolver
    │
    ▼
Query Planner
    │
    ▼
Optimizer
    │
    ▼
Platform SQL Compiler
    │
    ▼
Database JSON Engine
```

---

# 12. Principio de reutilización

JSON Query no tendrá:

```text
JSON Executor
```

independiente.

Reutilizará:

```text
Query Engine
SQL Compiler
Execution Engine
Result System
Hydration System
Type System
```

---

# 13. JSON AST

El Query AST incorporará una familia:

```text
JsonExpression
```

---

# 14. Jerarquía conceptual

```text
ExpressionNode
└── JsonExpression
    ├── JsonDocumentExpression
    ├── JsonPathExpression
    ├── JsonExtractExpression
    ├── JsonExistsExpression
    ├── JsonContainsExpression
    ├── JsonTypeExpression
    ├── JsonLengthExpression
    ├── JsonSetExpression
    ├── JsonReplaceExpression
    ├── JsonRemoveExpression
    ├── JsonMergeExpression
    └── JsonArrayExpression
```

---

# 15. JSON Document Expression

Representará una fuente JSON:

```php
Json::document('metadata');
```

---

# 16. Source

Puede ser:

```text
column
expression
parameter
subquery projection
computed JSON expression
```

---

# 17. JSON Path

Las rutas tendrán representación propia:

```text
JsonPath
```

---

# 18. JsonPath ≠ raw string

Aunque la API pueda aceptar:

```php
'$.user.address.city'
```

internamente deberá producir:

```text
JsonPath AST
```

---

# 19. JsonPath AST

Ejemplo:

```text
$
└── Property("user")
    └── Property("address")
        └── Property("city")
```

---

# 20. Path segments

Conceptualmente:

```text
JsonPathSegment
├── RootSegment
├── PropertySegment
├── ArrayIndexSegment
├── WildcardSegment
└── PlatformExtensionSegment
```

---

# 21. Ejemplo property path

```text
$.customer.name
```

se representa:

```text
Root
 ↓
Property(customer)
 ↓
Property(name)
```

---

# 22. Array index

```text
$.items[0].name
```

---

# 23. AST

```text
Root
 ↓
Property(items)
 ↓
ArrayIndex(0)
 ↓
Property(name)
```

---

# 24. Path builder

Podrá existir:

```php
$path = JsonPath::root()
    ->property('items')
    ->index(0)
    ->property('name');
```

---

# 25. Ventaja

Evita depender de sintaxis específica de:

```text
MySQL JSON Path
PostgreSQL operators
SQLite JSON Path
```

---

# 26. Dynamic property names

Un nombre:

```php
$property = $request->input('property');
```

no deberá concatenarse directamente a una ruta SQL.

---

# 27. Safe dynamic path

Se construirá mediante:

```php
JsonPath::root()->property($property);
```

con validación apropiada.

---

# 28. JSON Path security

```text
User Input
    ↓
Path Parser / Builder
    ↓
Validated JsonPath
    ↓
Platform Compiler
```

---

# 29. Raw JSON paths

Podrán existir como escape hatch:

```php
JsonPath::raw(...)
```

pero serán:

```text
explicit
platform-aware
restricted
auditable
```

---

# 30. JSON extraction

Operación:

```text
JsonExtractExpression
```

---

# 31. Ejemplo API

```php
$query->selectJson(
    column: 'metadata',
    path: '$.customer.name',
    alias: 'customer_name'
);
```

---

# 32. Fluent expression

También:

```php
Json::column('metadata')
    ->path('$.customer.name');
```

---

# 33. JSON extraction result

La extracción puede solicitar:

```text
JSON value
scalar value
typed scalar
```

---

# 34. Extraction modes

```php
enum JsonExtractionMode
{
    case JSON;
    case SCALAR;
    case TYPED;
}
```

---

# 35. JSON extraction

Ejemplo:

```json
{
  "age": 42
}
```

Modo:

```text
JSON
```

produce conceptualmente:

```text
JSON number 42
```

---

# 36. Scalar extraction

Modo:

```text
SCALAR
```

produce un valor SQL/PHP apropiado según contrato.

---

# 37. Typed extraction

Ejemplo:

```php
Json::column('metadata')
    ->path('$.age')
    ->asInteger();
```

---

# 38. Typed extraction ≠ blind cast

El Type System deberá validar compatibilidad.

---

# 39. JSON scalar types

Modelo lógico:

```text
JSON_STRING
JSON_NUMBER
JSON_INTEGER
JSON_FLOAT
JSON_BOOLEAN
JSON_NULL
JSON_OBJECT
JSON_ARRAY
```

---

# 40. JSON type inspection

API:

```php
Json::column('metadata')
    ->path('$.items')
    ->type();
```

---

# 41. JsonTypeExpression

Permitirá predicates como:

```php
$query->whereJsonType(
    'metadata',
    '$.items',
    JsonValueType::ARRAY
);
```

---

# 42. Existence

Debe existir una operación explícita:

```text
JsonExistsExpression
```

---

# 43. Example

```php
$query->whereJsonPathExists(
    'metadata',
    '$.customer.email'
);
```

---

# 44. Missing ≠ null

Documento:

```json
{
  "email": null
}
```

Entonces:

```text
path exists = true
value = JSON null
```

---

# 45. Missing example

```json
{}
```

Entonces:

```text
path exists = false
```

---

# 46. Critical rule

Nunca implementar:

```text
path exists
```

como:

```text
extracted value IS NOT NULL
```

si eso colapsa:

```text
JSON null
```

con:

```text
missing
```

---

# 47. Null predicates

La API deberá distinguir:

```php
whereJsonPathMissing(...)
whereJsonPathExists(...)
whereJsonNull(...)
whereJsonNotNull(...)
```

de:

```php
whereNull('metadata')
```

---

# 48. Four-state reasoning

Para una consulta concreta pueden existir:

```text
SQL NULL document
MISSING path
JSON null
non-null JSON value
```

---

# 49. JSON state model

```php
enum JsonPathState
{
    case DOCUMENT_SQL_NULL;
    case MISSING;
    case JSON_NULL;
    case VALUE;
}
```

No necesariamente se materializará este enum en todas las consultas, pero la semántica deberá conservarlo.

---

# 50. Equality

Ejemplo:

```php
$query->whereJson(
    'metadata',
    '$.status',
    '=',
    'active'
);
```

---

# 51. Typed comparison

Internamente:

```text
JsonExtract
   ↓
Typed Value
   ↓
ComparisonPredicate
```

---

# 52. String vs number

JSON:

```json
{
  "a": "10",
  "b": 10
}
```

VoltStack no deberá asumir:

```text
$.a == $.b
```

---

# 53. Type-aware semantics

Comparaciones deberán respetar el tipo lógico solicitado.

---

# 54. Numeric comparison

```php
$query->whereJsonNumber(
    'metadata',
    '$.price',
    '>',
    100
);
```

---

# 55. Boolean comparison

```php
$query->whereJsonBoolean(
    'metadata',
    '$.active',
    true
);
```

---

# 56. Generic API

También podrá existir:

```php
$query->whereJson(
    expression: Json::column('metadata')->path('$.price')->asDecimal(),
    operator: '>',
    value: 100
);
```

---

# 57. Parameter binding

El valor:

```text
100
```

será parameter.

No SQL literal concatenado.

---

# 58. JSON containment

Operación:

```text
JsonContainsExpression
```

---

# 59. Example

Documento:

```json
{
  "roles": ["admin", "editor"]
}
```

Consulta:

```php
$query->whereJsonContains(
    'metadata',
    '$.roles',
    'admin'
);
```

---

# 60. Containment semantics

VoltStack deberá definir explícitamente qué significa:

```text
contains
```

para:

```text
arrays
objects
scalars
```

---

# 61. Array membership

Será preferible distinguir:

```text
JSON containment
```

de:

```text
array element membership
```

cuando la plataforma tenga semánticas diferentes.

---

# 62. API

```php
$query->whereJsonArrayContains(
    'metadata',
    '$.roles',
    'admin'
);
```

---

# 63. Object containment

Ejemplo:

```json
{
  "settings": {
    "theme": "dark",
    "language": "es"
  }
}
```

Consulta conceptual:

```text
settings contains {"theme":"dark"}
```

---

# 64. Portability

La semántica exacta de containment puede diferir entre motores.

Por ello se resolverá mediante:

```text
JsonContainmentSemantics
```

---

# 65. Strict mode

Si una plataforma no puede conservar la semántica:

```text
fail
```

en lugar de aproximarla silenciosamente.

---

# 66. Array length

```php
$query->whereJsonLength(
    'metadata',
    '$.roles',
    '>',
    2
);
```

---

# 67. JsonLengthExpression

Debe distinguir:

```text
array length
object member count
string length
```

si la API los permite.

---

# 68. Generic JSON length

No deberá asumir automáticamente la misma semántica para todos los tipos.

---

# 69. Array index access

```php
Json::column('metadata')
    ->path('$.items[0]');
```

---

# 70. Negative indexes

No serán considerados portables por default.

Se expondrán únicamente mediante capability o extensión.

---

# 71. Wildcards

Rutas como:

```text
$.items[*].name
```

requieren capability explícita.

---

# 72. Wildcard result

Puede producir:

```text
multiple values
```

por lo que su tipo lógico es diferente de scalar extraction.

---

# 73. JsonSequence

Conceptualmente:

```text
JsonSequence<T>
```

podrá representar resultados multi-valued.

---

# 74. Scalar path ≠ multi-valued path

El Semantic Analyzer deberá conocer esta diferencia cuando sea inferible.

---

# 75. JSON arrays as rows

Algunas plataformas permiten expandir arrays.

Conceptualmente:

```text
JSON array
    ↓
table-like row source
```

---

# 76. JsonTableExpression

VoltStack podrá modelar:

```text
JsonTableExpression
```

como capacidad avanzada.

---

# 77. Example

JSON:

```json
{
  "items": [
    {"sku": "A", "qty": 2},
    {"sku": "B", "qty": 3}
  ]
}
```

Podrá transformarse conceptualmente en:

```text
sku | qty
----+----
A   | 2
B   | 3
```

---

# 78. JSON table ≠ hydration

Es una transformación del Query Engine.

---

# 79. Platform support

Su disponibilidad dependerá de capabilities.

---

# 80. JSON predicates

Jerarquía conceptual:

```text
JsonPredicate
├── JsonExistsPredicate
├── JsonMissingPredicate
├── JsonNullPredicate
├── JsonComparisonPredicate
├── JsonContainsPredicate
├── JsonArrayContainsPredicate
├── JsonTypePredicate
└── JsonLengthPredicate
```

---

# 81. JSON projection

Ejemplo:

```php
$query
    ->select('id')
    ->selectJson('metadata', '$.customer.name', 'customer_name');
```

---

# 82. Result shape

Podrá hidratar:

```text
scalar
tuple
DTO
projection
```

---

# 83. Partial JSON projection ≠ entity field value

Si `metadata` es un atributo completo de la entidad, seleccionar únicamente:

```text
$.customer.name
```

no significa que `metadata` esté completamente loaded.

---

# 84. LoadedFieldMask

El Hydration System deberá conservar:

```text
partial projection
```

sin marcar el JSON completo como loaded.

---

# 85. ORM example

```php
$users = User::query()
    ->whereJson(
        'preferences',
        '$.theme',
        '=',
        'dark'
    )
    ->get();
```

---

# 86. Repository example

```php
$users = $repository
    ->query()
    ->whereJson(
        field: 'preferences',
        path: JsonPath::parse('$.theme'),
        operator: Comparison::EQUAL,
        value: 'dark'
    )
    ->get();
```

---

# 87. Unified engine

Ambos:

```text
Model API
Repository
```

producen:

```text
same JSON AST
```

---

# 88. JSON mutation

El sistema deberá soportar mutaciones declarativas.

---

# 89. JsonSet

Ejemplo:

```php
$query->updateJson(
    'metadata',
    Json::set('$.status', 'active')
);
```

---

# 90. Semantics

`SET` conceptualmente:

```text
path exists
    → replace

path missing
    → create
```

cuando la operación solicitada tenga esa semántica.

---

# 91. JsonReplace

```php
Json::replace(
    '$.status',
    'active'
);
```

Conceptualmente:

```text
replace only if path exists
```

---

# 92. JsonInsert

Puede existir:

```php
Json::insert(
    '$.status',
    'active'
);
```

con:

```text
insert only if missing
```

si la plataforma puede preservar la operación.

---

# 93. JsonRemove

```php
Json::remove('$.legacy');
```

---

# 94. JsonMerge

```php
Json::merge($document);
```

---

# 95. Merge semantics

Debe distinguirse:

```text
merge preserve
merge patch
deep merge
shallow merge
```

---

# 96. No ambiguous merge

Una API genérica:

```php
Json::merge(...)
```

deberá requerir strategy o tener una semántica canónica claramente documentada.

---

# 97. Merge strategy

```php
enum JsonMergeStrategy
{
    case PATCH;
    case PRESERVE;
    case SHALLOW;
    case CUSTOM;
}
```

---

# 98. JSON Patch

Podrá modelarse:

```text
JsonPatch
```

como secuencia de operaciones.

---

# 99. Example

```php
JsonPatch::create()
    ->set('$.name', 'VoltStack')
    ->remove('$.legacy')
    ->set('$.version', 1);
```

---

# 100. Patch ≠ RFC guarantee

Si se adopta un estándar específico posteriormente deberá declararse explícitamente.

No se llamará RFC-compatible sin implementar sus garantías.

---

# 101. Atomic JSON update

Cuando el motor soporte una única expresión SQL equivalente:

```text
multiple JSON mutations
```

podrán compilarse a una operación atómica a nivel statement.

---

# 102. Statement atomicity ≠ transaction atomicity

Se mantienen las reglas de Transactions.

---

# 103. Concurrent JSON mutation

Problema:

```text
read document
modify PHP array
write entire document
```

puede producir lost updates.

---

# 104. Database-side mutation

Cuando sea posible:

```text
UPDATE ... SET json = JsonSet(json,...)
```

reduce ciertas ventanas de read-modify-write.

---

# 105. But

```text
Database-side JSON mutation
≠
automatic concurrency safety
```

---

# 106. Optimistic locking

Puede seguir siendo necesario.

---

# 107. JSON mutation + ORM

Si una actualización JSON se realiza fuera del managed entity state:

```text
IdentityMap entity
```

puede quedar stale.

---

# 108. Rule

Bulk/direct JSON mutations deberán seguir las reglas de:

```text
Persistence Consistency
Bulk Update
IdentityMap invalidation
```

---

# 109. No invisible synchronization

VoltStack no fingirá que una entidad managed fue actualizada si el ORM no puede demostrarlo.

---

# 110. JSON arrays mutation

Operaciones potenciales:

```text
append
prepend
insert at index
remove at index
replace at index
```

---

# 111. Portability

Cada una requerirá capability.

---

# 112. JSON object mutation

Operaciones:

```text
set property
remove property
rename property
merge object
```

---

# 113. Rename

Puede compilarse conceptualmente como:

```text
read old
set new
remove old
```

pero sólo si la plataforma puede preservar la semántica requerida.

---

# 114. Mutation planner

Será responsable de determinar:

```text
native operation
composed operation
unsupported operation
```

---

# 115. Capability model

Integración con:

```text
DATABASE_PLATFORM_CAPABILITY_SYSTEM
```

---

# 116. JSON capabilities

Ejemplos:

```text
supportsJsonType()
supportsJsonExtraction()
supportsJsonScalarExtraction()
supportsJsonPathExists()
supportsJsonContainment()
supportsJsonArrayMembership()
supportsJsonMutation()
supportsJsonSet()
supportsJsonRemove()
supportsJsonMerge()
supportsJsonArrayMutation()
supportsJsonAggregation()
supportsJsonTable()
supportsJsonIndexing()
supportsJsonGeneratedColumnIndexing()
```

---

# 117. Capability status

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 118. UNKNOWN ≠ SUPPORTED

---

# 119. Native JSON type

VoltStack distinguirá:

```text
logical JSON capability
```

de:

```text
native binary JSON storage
```

---

# 120. MySQL

Puede disponer de tipo JSON y funciones/operadores nativos.

---

# 121. MariaDB

Tendrá su propio capability descriptor.

No se asumirá:

```text
MariaDB JSON semantics == MySQL JSON semantics
```

---

# 122. PostgreSQL

Debe distinguir:

```text
json
jsonb
```

---

# 123. JSON ≠ JSONB

VoltStack podrá mapear ambos al tipo lógico JSON cuando sea apropiado, pero conservará metadata física relevante.

---

# 124. PostgreSQL capabilities

JSONB puede proporcionar capacidades/indexación diferentes a JSON.

---

# 125. SQLite

Puede almacenar JSON como representación compatible y proporcionar funciones JSON dependiendo de las capacidades de la instalación.

---

# 126. No assumption

No:

```text
SQLite → every JSON feature supported
```

---

# 127. Capability discovery

El Platform Capability System determinará soporte efectivo.

---

# 128. Platform compiler

Jerarquía:

```text
JsonCompiler
├── MySQLJsonCompiler
├── MariaDBJsonCompiler
├── PostgreSQLJsonCompiler
└── SQLiteJsonCompiler
```

---

# 129. JsonCompiler ≠ SQL executor

Produce representación SQL.

No ejecuta.

---

# 130. Compilation example

Intent:

```text
Extract metadata.customer.name as string
```

puede convertirse a diferentes formas físicas.

La aplicación nunca necesita conocerlas.

---

# 131. JSON indexing

El Schema System deberá poder describir estrategias de indexación JSON.

---

# 132. Problem

No todas las plataformas soportan:

```text
index arbitrary JSON path
```

de la misma manera.

---

# 133. Logical definition

Conceptualmente:

```php
$table->jsonIndex(
    column: 'metadata',
    path: '$.customer.id',
    type: 'string',
);
```

---

# 134. JsonIndexDefinition

No será necesariamente un índice físico único universal.

---

# 135. Planner

Podrá traducirlo a:

```text
expression index
generated column + index
JSON-specific index
GIN-like strategy
platform-specific mechanism
```

---

# 136. Schema portability

La definición deberá indicar:

```text
semantic intent
```

y no vendor DDL.

---

# 137. Generated columns

En algunas plataformas podrá ser útil:

```text
JSON path
   ↓
generated column
   ↓
ordinary index
```

---

# 138. Generated column ≠ JSON query requirement

Es una estrategia física de optimización.

---

# 139. Index metadata

Podrá registrar:

```text
source JSON column
path
logical type
index strategy
platform strategy
generated column
capabilities
```

---

# 140. Index selection

El Query Planner podrá conocer:

```text
available JSON path indexes
```

sin generar DDL.

---

# 141. Query Optimizer

Podrá analizar:

```text
duplicate extraction
predicate pushdown
indexable paths
constant paths
typed comparisons
unnecessary JSON materialization
projection minimization
```

---

# 142. Duplicate extraction

Consulta:

```text
SELECT json_path(...)
WHERE json_path(...) = ?
ORDER BY json_path(...)
```

puede permitir reutilización lógica/física cuando sea segura.

---

# 143. Predicate pushdown

JSON predicates seguirán siendo predicates normales del Query Engine.

---

# 144. Constant paths

Una ruta estática puede permitir mejores planes que una ruta dinámica.

---

# 145. Dynamic paths

Pueden:

```text
disable index usage
reduce portability
increase compilation complexity
```

---

# 146. Policy

El Resource/Performance system podrá:

```text
allow
warn
reject
```

dynamic paths según contexto.

---

# 147. JSON ordering

Ordenar por:

```text
$.priority
```

requiere definir el tipo.

---

# 148. Bad API

```php
orderByJson('metadata', '$.priority');
```

sin semántica de tipo puede ser ambiguo.

---

# 149. Better

```php
orderBy(
    Json::column('metadata')
        ->path('$.priority')
        ->asInteger()
);
```

---

# 150. String ordering

Dependerá de:

```text
collation
database semantics
```

---

# 151. Numeric ordering

Debe ser numérico, no lexical.

---

# 152. JSON aggregate

VoltStack podrá soportar:

```text
JSON_ARRAY_AGG
JSON_OBJECT_AGG
```

semánticamente mediante:

```text
JsonArrayAggregateExpression
JsonObjectAggregateExpression
```

---

# 153. API

```php
$query->selectJsonArrayAggregate('tags');
```

---

# 154. Aggregate ordering

Cuando el orden interno sea significativo deberá declararse explícitamente.

---

# 155. Duplicate object keys

JSON object aggregation debe definir qué ocurre con:

```text
duplicate keys
```

---

# 156. No silent assumption

Si plataformas difieren:

```text
STRICT profile → reject unsupported equivalence
```

---

# 157. JSON construction

El Query AST podrá incluir:

```text
JsonObjectExpression
JsonArrayExpression
```

---

# 158. Example

```php
Json::object([
    'id' => Column::of('id'),
    'name' => Column::of('name'),
]);
```

---

# 159. Result

La base de datos puede producir un documento JSON.

---

# 160. JSON construction ≠ serialization layer

El resultado sigue siendo un valor de Query Engine.

No una respuesta HTTP.

---

# 161. JSON + joins

JSON extraction podrá participar en:

```text
join predicates
```

si la plataforma y policy lo permiten.

---

# 162. Example

```text
orders.metadata.customer_id
=
customers.id
```

---

# 163. Performance warning

Joins sobre paths JSON no indexados pueden ser costosos.

---

# 164. Semantic analyzer

Deberá validar:

```text
JSON source type
path validity
path cardinality
expected result type
operator compatibility
mutation compatibility
capability requirements
```

---

# 165. Type inference

Ejemplo:

```php
Json::column('metadata')
    ->path('$.age')
    ->asInteger()
```

produce:

```text
Integer Query Type
```

---

# 166. Unknown JSON path type

Si no existe metadata:

```text
JsonValue
```

podrá permanecer como tipo lógico genérico.

---

# 167. Schema-aware JSON metadata

Opcionalmente podrá definirse:

```text
known JSON shape
```

---

# 168. JsonShape

Ejemplo conceptual:

```php
JsonShape::object([
    'name' => JsonField::string(),
    'age' => JsonField::integer(),
    'roles' => JsonField::array(JsonField::string()),
]);
```

---

# 169. JsonShape ≠ database schema

Describe estructura lógica dentro de un documento JSON.

---

# 170. Optional shape metadata

No será obligatorio para utilizar JSON.

---

# 171. Benefits

Permite:

```text
type inference
path validation
IDE tooling
index suggestions
static diagnostics
```

---

# 172. Unknown fields

Policy configurable:

```text
ALLOW
WARN
REJECT
```

cuando existe JsonShape.

---

# 173. Shape evolution

Debe ser independiente de schema migration física.

---

# 174. JSON shape versioning

Puede integrarse posteriormente con:

```text
History/Versioning
Application migrations
```

sin convertirlo automáticamente en DB schema migration.

---

# 175. JSON + Pagination

Compatible con:

```text
199_DATABASE_PAGINATION_SYSTEM.md
```

---

# 176. Example

```php
User::query()
    ->whereJson('preferences', '$.theme', '=', 'dark')
    ->orderBy(
        Json::column('preferences')
            ->path('$.priority')
            ->asInteger()
    )
    ->paginate(50);
```

---

# 177. Deterministic order

JSON ordering deberá utilizar tie-breaker cuando sea necesario.

---

# 178. JSON + Cursor Pagination

Compatible si:

```text
JSON ordering expression
```

es:

```text
deterministic
typed
cursor-serializable
stable enough
supported
```

---

# 179. Cursor boundary

Podrá incluir:

```text
typed extracted JSON value
+
tie-breaker
```

---

# 180. Example

```text
(priority=5, id=123)
```

---

# 181. Missing/null in cursor ordering

Debe conservar:

```text
MISSING
JSON_NULL
SQL_NULL
VALUE
```

según la semántica efectiva.

---

# 182. Critical

No codificar todos como:

```text
null
```

si altera ordering.

---

# 183. JSON + Chunk Processing

Podrá utilizar predicates JSON.

---

# 184. Mutation warning

Si el procesamiento modifica el mismo JSON path utilizado como continuation/order:

```text
skip/duplicate risk
```

---

# 185. JSON + Lazy Collection

No cambia las reglas de resource ownership.

---

# 186. JSON + Bulk Update

Ejemplo:

```php
User::query()
    ->whereJson('preferences', '$.legacy', '=', true)
    ->bulkUpdate([
        'preferences' => Json::remove('$.legacy'),
    ]);
```

---

# 187. Bulk mutation

Debe preservar las reglas de:

```text
Bulk Update
Persistence Consistency
IdentityMap
Cache Invalidation
```

---

# 188. JSON + Result Cache

Cache identity deberá incluir:

```text
JSON AST
path
types
parameters
metadata generation
platform semantics
```

cuando afecten el resultado.

---

# 189. JSON + Entity Cache

Una mutación parcial de JSON puede invalidar:

```text
whole entity cache entry
```

si no existe granularidad segura.

---

# 190. Conservative invalidation

Será preferible invalidar de más antes que conservar una entidad stale.

---

# 191. JSON + Multitenancy

JSON nunca será sustituto automático de tenant isolation.

---

# 192. Bad design

```json
{
  "tenant_id": 42
}
```

no convierte automáticamente:

```text
metadata.tenant_id
```

en tenant boundary.

---

# 193. Tenant scope

Debe provenir del:

```text
Tenant Query Context
```

---

# 194. JSON tenant predicates

Sólo podrán utilizarse como parte de aislamiento si una estrategia explícita lo define y valida.

---

# 195. JSON + Sharding

Un shard key dentro de JSON presenta problemas adicionales.

---

# 196. Shard routing

Para poder utilizar:

```text
metadata.customer_id
```

como shard key, Partition Routing deberá poder resolverlo antes de adquirir conexiones.

---

# 197. Runtime extraction is too late

No sirve:

```text
execute query
 ↓
read JSON
 ↓
discover shard
```

para routing.

---

# 198. Therefore

JSON-based shard keys requerirán:

```text
known query parameter
generated/extracted column
explicit routing context
or specialized resolver
```

---

# 199. JSON + Soft Delete

Soft Delete seguirá siendo predicate independiente.

No deberá ocultarse dentro de JSON por default.

---

# 200. JSON + Temporal Data

JSON fields podrán participar en temporal records normalmente.

---

# 201. JSON + History

History system deberá tratar una modificación parcial JSON como cambio de estado persistente.

---

# 202. JSON diff

Podrá existir en capas de history/audit, pero:

```text
JSON Query System
≠
JSON Diff System
```

---

# 203. JSON + Data Retention

Campos internos podrán tener reglas de retención específicas mediante extensiones.

---

# 204. Partial erasure

Ejemplo:

```text
remove $.personal.phone
```

podría utilizar JsonRemove.

Pero Data Retention continúa gobernando la política.

---

# 205. JSON + Full-Text Search

Un valor JSON textual podrá ser searchable si:

```text
platform capability
index strategy
search document metadata
```

lo permiten.

---

# 206. JSON Query ≠ Full-Text Search

Buscar:

```text
$.description = "database"
```

no equivale a:

```text
full-text(description, "database")
```

---

# 207. Security architecture

Integración con:

```text
226_DATABASE_SECURITY_ARCHITECTURE
227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM
228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM
231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM
233_DATABASE_QUERY_AUDIT_SYSTEM
```

---

# 208. JSON path injection

Es una superficie distinta de SQL injection.

---

# 209. Example risk

Nunca:

```php
$sql = "JSON_EXTRACT(metadata, '$.$userPath')";
```

---

# 210. Correct architecture

```text
User Path
   ↓
JsonPath Parser
   ↓
Validated AST
   ↓
Platform Compiler
```

---

# 211. JSON value injection

Valores deberán utilizar parameter binding.

---

# 212. JSON document parameters

Documentos completos también deberán pasar por:

```text
JSON Type
Value Conversion
Parameter Binding
```

---

# 213. Depth limits

Un documento externo podrá limitar:

```text
max nesting
max size
max keys
max array elements
```

---

# 214. Path limits

```text
max path depth
max wildcard count
max dynamic segments
```

---

# 215. Query complexity

También:

```text
max JSON predicates
max containment depth
max mutation operations
```

---

# 216. Sensitive JSON paths

Metadata podrá clasificar:

```text
$.public
$.internal
$.credentials
$.personal
```

---

# 217. Path-level access policy

Podrá existir:

```text
JsonPathAccessPolicy
```

---

# 218. Example

```text
READ_ALLOWED
WRITE_ALLOWED
SEARCH_ALLOWED
AUDIT_REDACTED
DENIED
```

---

# 219. Sensitive Data Protection

Debugging no deberá imprimir documentos JSON completos por default.

---

# 220. Redaction

Ejemplo:

```json
{
  "name": "Alice",
  "token": "[REDACTED]"
}
```

---

# 221. Audit

Podrá registrar:

```text
JSON operation
column
path fingerprint/classification
mutation type
actor
tenant
```

sin almacenar valores sensibles.

---

# 222. Telemetry

Métricas:

```text
database_json_queries_total
database_json_predicates_total
database_json_mutations_total
database_json_query_duration
database_json_failures_total
database_json_index_usage_total
```

---

# 223. Bounded dimensions

Permitidas:

```text
platform
operation
result status
capability status
```

---

# 224. Forbidden labels

No:

```text
raw JSON document
customer value
arbitrary JSON path
tenant ID
```

---

# 225. Query Profiler

Podrá mostrar:

```text
JSON predicates
paths count
indexed paths
unindexed paths
mutation operations
execution time
```

---

# 226. Debug information

Ejemplo:

```text
JSON Query
────────────────────────────

Column:
  metadata

Path:
  $.customer.id

Operation:
  comparison

Logical Type:
  integer

Operator:
  =

Platform:
  PostgreSQL

Physical Type:
  jsonb

Index:
  customer_id_json_idx

Index Strategy:
  expression

Capability:
  supported
```

---

# 227. Explain

```php
$query->explainJson();
```

deberá describir intención semántica.

---

# 228. Explain ≠ SQL dump

---

# 229. Performance model

Principales factores:

```text
document size
path depth
predicate count
index availability
physical JSON representation
mutation frequency
projection size
aggregation
array expansion
```

---

# 230. Whole document materialization

Evitar cuando sólo se requiere:

```text
one scalar path
```

si la plataforma permite extracción server-side.

---

# 231. Index-aware planner

Podrá detectar:

```text
frequently queried unindexed path
```

para diagnostics.

No creará índices automáticamente.

---

# 232. Index recommendation ≠ schema mutation

---

# 233. Large JSON documents

Resource Governance podrá limitar:

```text
maximum projected JSON size
maximum aggregate JSON size
maximum mutation document size
```

---

# 234. JSON aggregation risk

Una agregación puede producir un documento enorme.

Debe respetar memory/result budgets.

---

# 235. Result streaming

JSON documents grandes podrán seguir las reglas del Streaming Result System.

---

# 236. Streaming ≠ incremental JSON parser

No se prometerá parsing incremental del contenido salvo implementación específica.

---

# 237. Persistent Runtime

Compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 238. Immutable shared state

Puede compartirse:

```text
JsonShape metadata
compiled JsonPath definitions
platform capability descriptors
immutable compiler metadata
```

---

# 239. Scoped state

Debe ser scoped:

```text
JSON query parameters
tenant
authorization
resource budgets
temporary planner state
diagnostics
```

---

# 240. No static current path

Nunca:

```php
static ?JsonPath $currentPath;
```

---

# 241. No static tenant JSON policy

---

# 242. Worker reset

Eliminar:

```text
temporary JSON query context
sensitive values
diagnostics
parameter references
tenant bindings
```

---

# 243. Error model

```text
DatabaseException
└── JsonQueryException
    ├── JsonPathException
    ├── JsonPathSyntaxException
    ├── JsonTypeException
    ├── JsonComparisonException
    ├── JsonContainmentException
    ├── JsonMutationException
    ├── JsonCapabilityException
    ├── JsonCompilationException
    ├── JsonIndexException
    ├── JsonSecurityException
    ├── JsonResourceException
    └── JsonDistributionException
```

---

# 244. JsonPathException

Ruta inválida o no resoluble.

---

# 245. JsonTypeException

Operación incompatible con el tipo esperado.

---

# 246. JsonContainmentException

Semántica de containment no representable.

---

# 247. JsonMutationException

Mutación inválida/no soportada.

---

# 248. JsonCapabilityException

Capability requerida ausente.

---

# 249. JsonIndexException

Problema con estrategia/index metadata.

---

# 250. JsonSecurityException

Path/value/operation bloqueada por security policy.

---

# 251. JsonResourceException

Límite de:

```text
size
depth
complexity
```

excedido.

---

# 252. Testing architecture

Requerirá:

```text
unit tests
AST tests
path parser tests
semantic tests
type tests
compiler tests
platform conformance tests
integration tests
security tests
performance tests
persistent runtime tests
```

---

# 253. Null semantics tests

Corpus:

```json
{
  "present": "value",
  "json_null": null
}
```

Probar:

```text
$.present
$.json_null
$.missing
SQL NULL document
```

---

# 254. Mandatory assertion

Los cuatro estados no deberán colapsarse accidentalmente.

---

# 255. Path tests

Probar:

```text
nested properties
array indexes
escaped property names
Unicode
wildcards
invalid paths
deep paths
```

---

# 256. Type tests

Probar:

```text
"10"
10
10.5
true
false
null
[]
{}
```

---

# 257. Comparison tests

Especialmente:

```text
"10" vs 10
false vs 0
JSON null vs SQL NULL
missing vs JSON null
```

---

# 258. Containment tests

Arrays:

```json
["admin", "editor"]
```

Objects:

```json
{"theme":"dark"}
```

Nested objects.

---

# 259. Mutation tests

Probar:

```text
set existing
set missing
replace existing
replace missing
insert existing
insert missing
remove existing
remove missing
merge
multiple operations
```

---

# 260. Concurrency tests

Dos transactions modificando distintos/same paths.

No asumir merge automático.

---

# 261. ORM consistency tests

Una direct JSON update no deberá dejar una managed entity marcada falsamente como synchronized.

---

# 262. Platform conformance

Ejecutar corpus equivalente en:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 263. Portability tests

Validar semántica, no igualdad de SQL.

---

# 264. Security tests

Probar:

```text
malicious paths
quotes
SQL fragments
wildcards
extreme nesting
huge documents
Unicode edge cases
```

---

# 265. Parameter tests

Los valores JSON no deberán convertirse en SQL ejecutable.

---

# 266. Index tests

Comprobar:

```text
index metadata resolution
generated-column strategy
expression-index strategy
capability failure
```

---

# 267. Pagination tests

Probar ordering por:

```text
string path
integer path
nullable path
missing path
```

---

# 268. Cursor tests

Especialmente:

```text
MISSING
JSON_NULL
SQL_NULL
VALUE
```

---

# 269. Multitenancy tests

JSON predicates nunca deberán eliminar Tenant Query Context.

---

# 270. Sharding tests

Validar:

```text
JSON shard-key routing
unresolvable shard key
multi-shard queries
```

---

# 271. Persistent runtime tests

Request A:

```text
tenant A
path $.internal.a
```

Request B:

```text
tenant B
path $.public.b
```

No deberá existir leakage.

---

# 272. Proposed directory structure

```text
src/Quantum/Database/Json/
│
├── Contract/
│   ├── JsonCompiler.php
│   ├── JsonPathParser.php
│   ├── JsonCapabilityResolver.php
│   ├── JsonSemanticAnalyzer.php
│   └── JsonMutationPlanner.php
│
├── AST/
│   ├── JsonExpression.php
│   ├── JsonDocumentExpression.php
│   ├── JsonExtractExpression.php
│   ├── JsonExistsExpression.php
│   ├── JsonContainsExpression.php
│   ├── JsonTypeExpression.php
│   ├── JsonLengthExpression.php
│   ├── JsonSetExpression.php
│   ├── JsonInsertExpression.php
│   ├── JsonReplaceExpression.php
│   ├── JsonRemoveExpression.php
│   ├── JsonMergeExpression.php
│   ├── JsonObjectExpression.php
│   ├── JsonArrayExpression.php
│   └── JsonTableExpression.php
│
├── Path/
│   ├── JsonPath.php
│   ├── JsonPathSegment.php
│   ├── RootSegment.php
│   ├── PropertySegment.php
│   ├── ArrayIndexSegment.php
│   ├── WildcardSegment.php
│   ├── JsonPathParser.php
│   └── JsonPathState.php
│
├── Type/
│   ├── JsonValueType.php
│   ├── JsonExtractionMode.php
│   ├── JsonSequence.php
│   └── JsonTypeResolver.php
│
├── Predicate/
│   ├── JsonPredicate.php
│   ├── JsonExistsPredicate.php
│   ├── JsonMissingPredicate.php
│   ├── JsonNullPredicate.php
│   ├── JsonComparisonPredicate.php
│   ├── JsonContainsPredicate.php
│   ├── JsonArrayContainsPredicate.php
│   ├── JsonTypePredicate.php
│   └── JsonLengthPredicate.php
│
├── Mutation/
│   ├── JsonMutation.php
│   ├── JsonSet.php
│   ├── JsonInsert.php
│   ├── JsonReplace.php
│   ├── JsonRemove.php
│   ├── JsonMerge.php
│   ├── JsonPatch.php
│   ├── JsonMergeStrategy.php
│   └── JsonMutationPlan.php
│
├── Shape/
│   ├── JsonShape.php
│   ├── JsonField.php
│   ├── JsonObjectShape.php
│   ├── JsonArrayShape.php
│   ├── JsonScalarShape.php
│   └── JsonShapeRegistry.php
│
├── Index/
│   ├── JsonIndexDefinition.php
│   ├── JsonIndexMetadata.php
│   ├── JsonIndexStrategy.php
│   └── JsonIndexResolver.php
│
├── Capability/
│   ├── JsonCapability.php
│   ├── JsonCapabilities.php
│   ├── JsonCapabilityStatus.php
│   └── DefaultJsonCapabilityResolver.php
│
├── Semantic/
│   ├── JsonSemanticAnalyzer.php
│   ├── JsonSemanticProfile.php
│   ├── JsonContainmentSemantics.php
│   └── JsonValidationResult.php
│
├── Planning/
│   ├── JsonQueryPlan.php
│   ├── JsonMutationPlan.php
│   └── DefaultJsonMutationPlanner.php
│
├── Compiler/
│   ├── AbstractJsonCompiler.php
│   ├── MySQLJsonCompiler.php
│   ├── MariaDBJsonCompiler.php
│   ├── PostgreSQLJsonCompiler.php
│   └── SQLiteJsonCompiler.php
│
├── Security/
│   ├── JsonSecurityPolicy.php
│   ├── JsonPathAccessPolicy.php
│   ├── JsonInputLimits.php
│   └── JsonRedactor.php
│
├── Diagnostics/
│   ├── JsonDiagnostics.php
│   ├── JsonExplain.php
│   └── JsonPlanFormatter.php
│
├── Telemetry/
│   └── JsonTelemetry.php
│
└── Exception/
    ├── JsonQueryException.php
    ├── JsonPathException.php
    ├── JsonPathSyntaxException.php
    ├── JsonTypeException.php
    ├── JsonComparisonException.php
    ├── JsonContainmentException.php
    ├── JsonMutationException.php
    ├── JsonCapabilityException.php
    ├── JsonCompilationException.php
    ├── JsonIndexException.php
    ├── JsonSecurityException.php
    ├── JsonResourceException.php
    └── JsonDistributionException.php
```

---

# 273. Architectural invariants

## DB-JSON-001

JSON Query será una extensión del Query Engine.

## DB-JSON-002

JSON Query no tendrá un Execution Engine independiente.

## DB-JSON-003

JSON Query reutilizará Query AST.

## DB-JSON-004

JSON Query reutilizará Semantic Engine.

## DB-JSON-005

JSON Query reutilizará Query Planner.

## DB-JSON-006

JSON Query reutilizará Optimizer.

## DB-JSON-007

JSON Query reutilizará SQL Compiler.

## DB-JSON-008

JSON Query reutilizará Execution Engine.

## DB-JSON-009

JSON Query reutilizará Type System.

## DB-JSON-010

JSON Query será distinto de JSON Type System.

## DB-JSON-011

JSON Query será distinto de JSON serialization.

## DB-JSON-012

JSON Query será distinto de PHP array access.

## DB-JSON-013

JSON logical value será distinto de ordinary string.

## DB-JSON-014

SQL NULL será distinto de JSON null.

## DB-JSON-015

JSON null será distinto de missing path.

## DB-JSON-016

Missing path será distinto de SQL NULL.

## DB-JSON-017

Existence no será inferida mediante `IS NOT NULL` cuando colapse estados.

## DB-JSON-018

JSON path tendrá representación AST.

## DB-JSON-019

JSON path no será raw SQL.

## DB-JSON-020

Dynamic paths serán validados.

## DB-JSON-021

Raw paths serán escape hatch explícito.

## DB-JSON-022

Raw paths serán auditable/restricted.

## DB-JSON-023

JSON extraction tendrá result type explícito.

## DB-JSON-024

JSON extraction será distinta de scalar extraction.

## DB-JSON-025

Typed extraction será validada por Type System.

## DB-JSON-026

JSON string `"10"` será distinto de JSON number `10`.

## DB-JSON-027

Boolean será distinto de numeric zero/one salvo conversión explícita.

## DB-JSON-028

JSON predicates serán typed.

## DB-JSON-029

JSON values serán parameterized.

## DB-JSON-030

Containment tendrá semántica explícita.

## DB-JSON-031

Array membership será distinguible de generic containment.

## DB-JSON-032

Unsupported containment no se degradará silenciosamente.

## DB-JSON-033

JSON length tendrá semántica dependiente del tipo.

## DB-JSON-034

Negative array indexes requerirán capability.

## DB-JSON-035

Wildcards requerirán capability.

## DB-JSON-036

Multi-valued path será distinto de scalar path.

## DB-JSON-037

JSON table será distinto de Hydration.

## DB-JSON-038

JSON projection podrá producir scalar/tuple/DTO/projection.

## DB-JSON-039

Partial JSON projection no marcará el documento completo como loaded.

## DB-JSON-040

Model API y Repository producirán el mismo JSON AST.

## DB-JSON-041

JSON mutation será declarativa.

## DB-JSON-042

JsonSet será distinto de JsonReplace.

## DB-JSON-043

JsonReplace será distinto de JsonInsert.

## DB-JSON-044

JsonRemove tendrá semántica explícita.

## DB-JSON-045

JsonMerge requerirá semántica definida.

## DB-JSON-046

No existirá merge ambiguo.

## DB-JSON-047

JSON Patch será distinto de transaction.

## DB-JSON-048

Statement atomicity será distinta de transaction atomicity.

## DB-JSON-049

Database-side JSON mutation no garantizará concurrency safety.

## DB-JSON-050

Optimistic locking podrá seguir siendo necesario.

## DB-JSON-051

Direct JSON mutation podrá invalidar ORM state.

## DB-JSON-052

IdentityMap no será actualizado ficticiamente.

## DB-JSON-053

JSON array mutations serán capability-aware.

## DB-JSON-054

JSON object mutations serán capability-aware.

## DB-JSON-055

Mutation Planner no ejecutará SQL.

## DB-JSON-056

Capabilities gobernarán operaciones JSON.

## DB-JSON-057

UNKNOWN capability será distinta de SUPPORTED.

## DB-JSON-058

Database version será distinta de capability.

## DB-JSON-059

Logical JSON será distinto de native binary JSON.

## DB-JSON-060

MySQL será distinto de MariaDB.

## DB-JSON-061

PostgreSQL JSON será distinto de JSONB.

## DB-JSON-062

SQLite JSON capabilities serán descubiertas.

## DB-JSON-063

No se asumirá feature availability por nombre del motor.

## DB-JSON-064

JsonCompiler será platform-specific.

## DB-JSON-065

JsonCompiler no ejecutará queries.

## DB-JSON-066

Driver no interpretará JSON AST.

## DB-JSON-067

Connection no conocerá JsonShape.

## DB-JSON-068

ORM no generará vendor JSON syntax.

## DB-JSON-069

JSON indexing será descrito semánticamente.

## DB-JSON-070

JsonIndexDefinition no implicará una única estrategia física.

## DB-JSON-071

Generated columns podrán ser estrategia física.

## DB-JSON-072

Generated columns no serán requeridas universalmente.

## DB-JSON-073

Query Planner podrá consumir JSON index metadata.

## DB-JSON-074

Query Planner no creará índices.

## DB-JSON-075

Optimizer podrá reutilizar JSON extraction.

## DB-JSON-076

Optimizer preservará semántica NULL/MISSING.

## DB-JSON-077

Dynamic paths podrán afectar indexabilidad.

## DB-JSON-078

JSON ordering será typed.

## DB-JSON-079

Numeric ordering no utilizará semántica lexical.

## DB-JSON-080

String ordering respetará collation efectiva.

## DB-JSON-081

JSON aggregation tendrá resource budgets.

## DB-JSON-082

Duplicate object key semantics serán explícitas.

## DB-JSON-083

JSON construction será distinta de HTTP serialization.

## DB-JSON-084

JSON expressions podrán participar en joins.

## DB-JSON-085

JSON joins estarán sujetos al Performance System.

## DB-JSON-086

Semantic Analyzer validará source JSON.

## DB-JSON-087

Semantic Analyzer validará path.

## DB-JSON-088

Semantic Analyzer validará result cardinality cuando sea conocida.

## DB-JSON-089

Semantic Analyzer validará operator compatibility.

## DB-JSON-090

Type inference podrá utilizar JsonShape.

## DB-JSON-091

JsonShape será opcional.

## DB-JSON-092

JsonShape será distinto de database schema.

## DB-JSON-093

JsonShape podrá mejorar diagnostics.

## DB-JSON-094

JsonShape evolution será distinta de physical schema migration.

## DB-JSON-095

JSON Query será compatible con Pagination.

## DB-JSON-096

JSON ordering requerirá deterministic tie-breakers cuando corresponda.

## DB-JSON-097

JSON Query podrá ser compatible con Cursor Pagination.

## DB-JSON-098

Cursor JSON values serán typed.

## DB-JSON-099

Cursor no colapsará MISSING/JSON_NULL/SQL_NULL si afecta orden.

## DB-JSON-100

JSON Query será compatible con Chunk Processing.

## DB-JSON-101

Mutar ordering path durante chunking será tratado como riesgo.

## DB-JSON-102

JSON Query será compatible con Lazy Collection.

## DB-JSON-103

Bulk JSON mutations seguirán Bulk Update semantics.

## DB-JSON-104

Bulk JSON mutation no sincronizará mágicamente IdentityMap.

## DB-JSON-105

Result Cache identity incluirá JSON semantics relevantes.

## DB-JSON-106

Entity Cache será invalidado conservadoramente cuando sea necesario.

## DB-JSON-107

JSON será distinto de tenant isolation.

## DB-JSON-108

Tenant Query Context no será sustituido por JSON fields.

## DB-JSON-109

JSON tenant predicates requerirán estrategia explícita.

## DB-JSON-110

JSON shard key deberá resolverse antes de conexión.

## DB-JSON-111

Runtime JSON extraction no podrá decidir shard tardíamente.

## DB-JSON-112

JSON shard routing podrá utilizar explicit routing context.

## DB-JSON-113

Soft Delete será independiente de JSON.

## DB-JSON-114

Temporal semantics serán independientes de JSON.

## DB-JSON-115

History registrará JSON state changes según su política.

## DB-JSON-116

JSON Query será distinto de JSON Diff.

## DB-JSON-117

Retention policy será distinta de JSON mutation.

## DB-JSON-118

JsonRemove podrá ser mecanismo de ejecución de una retention action.

## DB-JSON-119

JSON Query será distinto de Full-Text Search.

## DB-JSON-120

JSON text equality será distinta de full-text matching.

## DB-JSON-121

JSON path será untrusted input cuando provenga externamente.

## DB-JSON-122

JSON path injection será tratada explícitamente.

## DB-JSON-123

JSON values no serán concatenados en SQL.

## DB-JSON-124

JSON documents usarán Value Conversion.

## DB-JSON-125

JSON documents tendrán size/depth limits.

## DB-JSON-126

JSON paths tendrán complexity limits.

## DB-JSON-127

JSON mutation count podrá limitarse.

## DB-JSON-128

Sensitive JSON paths podrán clasificarse.

## DB-JSON-129

Path-level security podrá aplicarse.

## DB-JSON-130

Debugging no imprimirá sensitive JSON por default.

## DB-JSON-131

Audit no almacenará sensitive JSON values por default.

## DB-JSON-132

Telemetry tendrá bounded dimensions.

## DB-JSON-133

Raw JSON paths no serán telemetry labels.

## DB-JSON-134

Raw JSON documents no serán telemetry labels.

## DB-JSON-135

Query Profiler reconocerá JSON operations.

## DB-JSON-136

JSON Explain describirá semántica.

## DB-JSON-137

JSON Explain será distinto de SQL dump.

## DB-JSON-138

Large JSON documents estarán sujetos a Resource Governance.

## DB-JSON-139

JSON aggregates estarán sujetos a result-size limits.

## DB-JSON-140

Partial extraction será preferible a whole-document materialization cuando sea posible.

## DB-JSON-141

Index recommendations no modificarán schema automáticamente.

## DB-JSON-142

JSON streaming seguirá Streaming Result semantics.

## DB-JSON-143

Streaming Result no implicará incremental JSON parsing.

## DB-JSON-144

Persistent runtime JSON state será scoped.

## DB-JSON-145

Immutable JsonShape metadata podrá compartirse.

## DB-JSON-146

Compiled immutable JsonPath podrá compartirse cuando sea context-free.

## DB-JSON-147

Current JsonPath nunca será static mutable state.

## DB-JSON-148

Tenant JSON security policy nunca será static mutable state.

## DB-JSON-149

Worker reset eliminará temporary JSON state.

## DB-JSON-150

Sensitive JSON parameters no sobrevivirán scopes.

## DB-JSON-151

JSON errors tendrán jerarquía específica.

## DB-JSON-152

Path errors serán distintos de type errors.

## DB-JSON-153

Type errors serán distintos de capability errors.

## DB-JSON-154

Mutation errors serán distintos de compilation errors.

## DB-JSON-155

Security errors serán distintos de resource errors.

## DB-JSON-156

Platform conformance será testeada.

## DB-JSON-157

Portability tests compararán semántica, no SQL.

## DB-JSON-158

NULL/MISSING semantics serán testeadas obligatoriamente.

## DB-JSON-159

Path parser tendrá dedicated tests.

## DB-JSON-160

JSON scalar types serán testeados.

## DB-JSON-161

Containment semantics serán testeadas.

## DB-JSON-162

Mutation semantics serán testeadas.

## DB-JSON-163

Concurrency effects serán testeados.

## DB-JSON-164

ORM consistency después de direct JSON mutation será testeada.

## DB-JSON-165

Security path injection será testeada.

## DB-JSON-166

Resource exhaustion será testeada.

## DB-JSON-167

JSON index strategies serán testeadas.

## DB-JSON-168

Pagination con JSON ordering será testeada.

## DB-JSON-169

Cursor null/missing semantics serán testeadas.

## DB-JSON-170

Tenant isolation será testeada.

## DB-JSON-171

Shard routing será testeado.

## DB-JSON-172

Persistent runtime isolation será testeada.

## DB-JSON-173

JSON AST será immutable o tratado como immutable después de planificación.

## DB-JSON-174

Planning será deterministic bajo el mismo contexto.

## DB-JSON-175

No se fabricará portabilidad mediante coerciones silenciosas.

## DB-JSON-176

Platform-specific JSON extensions estarán aisladas.

## DB-JSON-177

Portable JSON core no dependerá de vendor syntax.

## DB-JSON-178

JSON API será independiente de HTTP.

## DB-JSON-179

JSON API será independiente de URL/query-string semantics.

## DB-JSON-180

VoltStack preservará explícitamente la diferencia entre valor, null y ausencia.

---

# 274. Modelo formal

Sea:

```text
D
```

un documento JSON.

Sea:

```text
P = (s₁, s₂, ..., sₙ)
```

una ruta JSON.

Definimos:

```text
Resolve(D,P)
```

como la resolución semántica de `P` sobre `D`.

El resultado pertenece a:

```text
JsonResolution =
    Missing
  | JsonNull
  | JsonValue(V)
```

Si el documento SQL completo es NULL:

```text
DocumentState = SqlNull
```

Por tanto:

```text
SqlNull
≠ Missing
≠ JsonNull
≠ JsonValue(V)
```

---

# 275. Existencia formal

```text
Exists(D,P) = true
```

cuando:

```text
Resolve(D,P) ∈ {JsonNull, JsonValue(V)}
```

y:

```text
Exists(D,P) = false
```

cuando:

```text
Resolve(D,P) = Missing
```

---

# 276. JSON null formal

```text
IsJsonNull(D,P)
```

es verdadero únicamente cuando:

```text
Resolve(D,P) = JsonNull
```

No cuando:

```text
Resolve(D,P) = Missing
```

---

# 277. Typed extraction

Sea:

```text
T
```

un tipo lógico solicitado.

```text
Extract_T(D,P)
```

sólo será válido cuando la resolución pueda convertirse a `T` conforme al Type System.

---

# 278. Mutation model

Una mutación:

```text
M(D,P,V)
```

produce:

```text
D'
```

sin implicar:

```text
TransactionCommit
```

ni:

```text
ORMStateSynchronized
```

---

# 279. Query planning model

```text
Plan =
f(
    JsonAST,
    TypeMetadata,
    JsonShape?,
    PlatformCapabilities,
    IndexMetadata,
    QueryContext,
    TenantContext,
    DistributionContext,
    ResourcePolicy
)
```

---

# 280. Arquitectura conceptual final

```text
                         Application
                              │
                              ▼
                  Model / Repository API
                              │
                              ▼
                        Query Builder
                              │
                              ▼
                         JSON API
                              │
                              ▼
                         JSON AST
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
          JsonPath         JsonShape        JSON Type
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                    Semantic Analyzer
                              │
                              ▼
                       Type Resolver
                              │
                              ▼
                   Capability Resolver
                              │
                              ▼
                       Query Planner
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
        Index Metadata    Tenant Context   Shard Routing
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                         Optimizer
                              │
                              ▼
                     Platform Compiler
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
       MySQL              PostgreSQL             SQLite
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                       Execution Engine
                              │
                              ▼
                         Result System
                              │
                              ▼
                     Type Conversion
                              │
                              ▼
                 Hydration / Projection
```

---

# 281. Query flow

```text
Application JSON Query
        │
        ▼
Query Builder
        │
        ▼
JSON AST
        │
        ▼
Path Validation
        │
        ▼
Semantic Validation
        │
        ▼
Type Resolution
        │
        ▼
Security Scope
        │
        ▼
Tenant Scope
        │
        ▼
Shard Resolution
        │
        ▼
Capability Analysis
        │
        ▼
Index Analysis
        │
        ▼
Query Planning
        │
        ▼
Optimization
        │
        ▼
Platform Compilation
        │
        ▼
Prepared Execution
        │
        ▼
Result Conversion
        │
        ▼
Hydration / Projection
```

---

# 282. Mutation flow

```text
Application
    │
    ▼
JsonPatch / JsonMutation
    │
    ▼
JSON Mutation AST
    │
    ▼
Semantic Validation
    │
    ▼
Security Validation
    │
    ▼
Capability Resolution
    │
    ▼
Mutation Planner
    │
    ▼
Update Query AST
    │
    ▼
Query Planner
    │
    ▼
Platform Compiler
    │
    ▼
Execution Engine
    │
    ▼
Persistence Consistency
    │
    ├── Cache Invalidation
    ├── IdentityMap considerations
    └── Transaction outcome
```

---

# 283. Example: Query

```php
$users = User::query()
    ->whereJson(
        expression: Json::column('preferences')
            ->path('$.notifications.email')
            ->asBoolean(),
        operator: '=',
        value: true,
    )
    ->orderBy(
        Json::column('preferences')
            ->path('$.priority')
            ->asInteger()
    )
    ->orderBy('id')
    ->paginate(50);
```

---

# 284. Internal representation

```text
SelectQuery
│
├── From(User)
│
├── Predicate
│    └── Comparison
│         ├── JsonExtract
│         │    ├── preferences
│         │    ├── $.notifications.email
│         │    └── BOOLEAN
│         ├── =
│         └── Parameter(true)
│
├── Order
│    ├── JsonExtract($.priority, INTEGER)
│    └── id
│
└── Pagination
```

---

# 285. Example: mutation

```php
User::query()
    ->where('id', 42)
    ->updateJson(
        'preferences',
        JsonPatch::create()
            ->set('$.theme', 'dark')
            ->set('$.notifications.email', true)
            ->remove('$.legacy')
    );
```

---

# 286. Internal mutation

```text
UpdateQuery
│
├── Target(User)
├── Predicate(id = 42)
└── Assignment
     └── JsonPatchExpression
          ├── Set($.theme, "dark")
          ├── Set($.notifications.email, true)
          └── Remove($.legacy)
```

---

# 287. Platform isolation

The application does not know whether the compiler ultimately uses:

```text
MySQL JSON functions
MariaDB JSON functions
PostgreSQL operators/functions
SQLite JSON functions
```

---

# 288. Portable core

```text
JSON Query Core
├── JsonPath
├── Extraction
├── Typed Comparison
├── Existence
├── Null Semantics
├── Containment
├── Length
├── Mutation
├── Construction
└── Aggregation
```

---

# 289. Native extensions

```text
JSON Query Extensions
├── MySQL
├── MariaDB
├── PostgreSQL
└── SQLite
```

---

# 290. Native extension rule

Platform-specific functionality deberá vivir fuera del portable core.

---

# 291. Architectural decision

VoltStack implementará JSON Query como una **extensión semántica y tipada del Query Engine**, no como una colección de helpers que concatenan funciones SQL específicas del proveedor.

La arquitectura será:

```text
JSON Intent
    ↓
JSON API
    ↓
JsonPath + JSON AST
    ↓
Semantic Analysis
    ↓
Type Resolution
    ↓
Capability Resolution
    ↓
Index Analysis
    ↓
Query Planning
    ↓
Optimization
    ↓
Platform Compilation
    ↓
Execution Engine
```

La API pública podrá mantener una experiencia simple:

```php
User::query()
    ->whereJson(
        'preferences',
        '$.theme',
        '=',
        'dark'
    )
    ->get();
```

mientras internamente VoltStack conservará:

```text
typed paths
typed values
AST
capabilities
platform isolation
security
index awareness
null semantics
distribution awareness
persistent-runtime safety
```

La regla principal será:

> **El desarrollador describe qué parte de un documento JSON desea consultar o modificar; nunca necesita describir qué operador o función SQL utiliza físicamente el motor de base de datos.**

Y la invariante semántica principal será:

> **SQL NULL, JSON null y una ruta JSON inexistente son estados diferentes. VoltStack deberá preservar esa diferencia desde el Query AST hasta la compilación, ejecución, cursor, hidratación y resultado.**

Finalmente:

> **VoltStack no sacrificará semántica para aparentar portabilidad. Cuando MySQL, MariaDB, PostgreSQL o SQLite no puedan representar correctamente una operación JSON, la limitación será explícita mediante capabilities, diagnostics o errores de planificación.**

---

# 292. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
✓ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
✓ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
✓ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 293. Siguiente documento

```text
274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
```

El siguiente documento definirá la extensión geoespacial de VoltStack Database, incluyendo:

```text
geographic types
geometry types
Point
LineString
Polygon
MultiPoint
MultiLineString
MultiPolygon
GeometryCollection
coordinate systems
SRID
latitude/longitude
geography vs geometry
spatial predicates
contains
within
intersects
touches
overlaps
crosses
disjoint
distance
nearest-neighbor search
bounding boxes
spatial indexes
coordinate transformation
platform capabilities
MySQL spatial support
MariaDB spatial support
PostgreSQL/PostGIS integration
SQLite spatial extensions
ORM mapping
Query AST
Spatial AST
Type System integration
Query Builder
compiler integration
pagination
cursor pagination
sharding
multitenancy
security
resource governance
telemetry
testing
```

manteniendo como principio:

> **Una coordenada no posee significado geoespacial completo sin un sistema de referencia; VoltStack no deberá comparar, medir ni transformar geometrías ignorando silenciosamente su SRID o sistema de coordenadas.**