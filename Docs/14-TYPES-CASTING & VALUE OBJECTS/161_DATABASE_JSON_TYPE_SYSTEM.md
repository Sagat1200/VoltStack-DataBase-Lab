# 161_DATABASE_JSON_TYPE_SYSTEM.md

# VoltStack Quantum Database
## Database JSON Type System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 161 — Database JSON Type System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md`  
**Siguiente documento:** `162_DATABASE_DATE_TIME_TYPE_SYSTEM.md`

---

# 1. Propósito

`Database JSON Type System` define la representación lógica, tipado, conversión, persistencia, hidratación y capacidades fundamentales de datos JSON dentro de VoltStack Database.

El sistema deberá proporcionar una abstracción uniforme sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin reducir JSON a un simple `string`.

La regla central será:

> **JSON en VoltStack será un tipo lógico estructurado y no un string decorado: el framework deberá preservar la diferencia entre SQL NULL, JSON null, clave ausente, objeto, array y scalar, mientras delega a cada plataforma únicamente la representación física y las capacidades de consulta disponibles.**

Esto implica:

```text
SQL NULL
≠
JSON null
≠
MISSING
≠
{}
≠
[]
≠
""
≠
0
≠
false
```

---

# 2. Alcance

Este documento define:

- tipo lógico JSON;
- modelo de valores JSON;
- representación PHP;
- canonicalización;
- encoding/decoding;
- SQL NULL;
- JSON null;
- missing;
- objects;
- arrays;
- scalars;
- numeric semantics;
- Unicode;
- JSON document mapping;
- JSON Value Objects;
- enum integration;
- Casting integration;
- Value Conversion integration;
- parameter binding;
- hydration;
- persistence;
- dirty tracking;
- canonical equality;
- platform mapping;
- capabilities;
- schema integration;
- mutation model;
- patch model;
- indexing capabilities;
- security;
- resource governance;
- persistent runtime;
- diagnostics;
- telemetry;
- testing;
- performance.

---

# 3. Fuera de alcance principal

La arquitectura completa del lenguaje de consultas JSON se documentará posteriormente en:

```text
273_DATABASE_JSON_QUERY_SYSTEM.md
```

Por tanto, este documento define:

```text
JSON values
JSON paths
JSON capabilities
JSON mutations
JSON operation contracts
```

pero no implementa por completo:

```text
JSON query planner
JSON predicate optimizer
JSON SQL dialect compilation
```

---

# 4. Posición arquitectónica

```text
Application
    │
    ▼
PHP JSON Representation
    │
    ▼
JSON Type System
    │
    ├── Type Metadata
    ├── Value Model
    ├── Canonicalization
    ├── Validation
    └── Capability Requirements
    │
    ▼
Value Conversion
    │
    ▼
Query / Persistence
    │
    ▼
Platform
    │
    ▼
Physical JSON Representation
    │
    ▼
Database
```

---

# 5. JSON ≠ string

Aunque un Driver pueda transportar:

```text
{"name":"VoltStack"}
```

como bytes/string, semánticamente:

```text
JSON Document
≠
PHP string
```

---

# 6. Logical Type ≠ Physical Type

VoltStack tendrá un tipo lógico:

```text
JSON
```

que puede mapearse físicamente a:

```text
MySQL JSON
MariaDB JSON-compatible representation
PostgreSQL JSON
PostgreSQL JSONB
SQLite TEXT + JSON capabilities
```

según plataforma y policy.

---

# 7. JSON Type identity

Dentro del Type System:

```php
final readonly class JsonTypeId
{
    public function __construct(
        public string $value = 'json',
    ) {}
}
```

Conceptualmente:

```text
DatabaseTypeId("json")
```

seguirá siendo la identidad lógica principal.

---

# 8. JSON type descriptor

```php
final readonly class JsonTypeDescriptor
{
    public function __construct(
        public JsonStoragePreference $storage,
        public JsonRootConstraint $root,
        public JsonNumberPolicy $numbers,
        public JsonObjectPolicy $objects,
        public JsonCanonicalizationPolicy $canonicalization,
    ) {}
}
```

---

# 9. Root JSON forms

JSON admite:

```text
object
array
string
number
boolean
null
```

VoltStack deberá decidir si una columna acepta:

```text
ANY_JSON_VALUE
```

o exige una forma concreta.

---

# 10. JsonRootConstraint

```php
enum JsonRootConstraint
{
    case ANY;
    case DOCUMENT;
    case OBJECT;
    case ARRAY;
    case SCALAR;
}
```

---

# 11. DOCUMENT

VoltStack podrá definir `DOCUMENT` como:

```text
OBJECT | ARRAY
```

cuando una aplicación quiera impedir scalars en la raíz.

---

# 12. OBJECT

Ejemplo válido:

```json
{
  "theme": "dark",
  "notifications": true
}
```

---

# 13. ARRAY

Ejemplo:

```json
[
  "php",
  "rust",
  "go"
]
```

---

# 14. SCALAR

Incluye:

```text
string
number
boolean
JSON null
```

cuando la policy lo permita.

---

# 15. SQL NULL

Una columna SQL puede contener:

```text
NULL
```

Esto representa ausencia del valor SQL de la columna.

---

# 16. JSON null

Una columna JSON puede contener un documento cuyo valor es:

```json
null
```

Esto es un valor JSON válido.

---

# 17. Diferencia fundamental

```text
SQL NULL
```

y:

```text
JSON null
```

no deberán colapsarse.

---

# 18. Missing

Dentro de:

```json
{
  "name": "VoltStack"
}
```

la ruta:

```text
$.version
```

está ausente.

Eso es:

```text
MISSING
```

---

# 19. Missing ≠ JSON null

```json
{
  "version": null
}
```

contiene la ruta con valor JSON null.

Por tanto:

```text
Missing($.version)
≠
JsonNull($.version)
```

---

# 20. Four-state model

Al consultar una ruta JSON podrán existir al menos:

```text
SQL_NULL_DOCUMENT
JSON_NULL
MISSING
PRESENT_VALUE
```

---

# 21. Why this matters

Sin esta separación:

```text
"key does not exist"
```

podría confundirse con:

```text
"key exists and explicitly has no JSON value"
```

rompiendo:

- patches;
- defaults;
- validation;
- queries;
- migrations;
- dirty tracking.

---

# 22. JsonNull

VoltStack deberá poseer una representación interna inequívoca.

```php
final readonly class JsonNull
{
    public static function value(): self;
}
```

---

# 23. JsonMissing

Para operaciones internas:

```php
final readonly class JsonMissing
{
    public static function value(): self;
}
```

---

# 24. Sentinels

Conceptualmente:

```text
JsonNull
JsonMissing
```

son sentinels semánticos.

No son strings especiales.

---

# 25. Nunca usar magic strings

Prohibido:

```text
"__NULL__"
"__MISSING__"
```

---

# 26. PHP representation

VoltStack podrá soportar una representación ergonómica:

```text
JSON object → associative array
JSON array  → list
JSON string → string
JSON number → int / decimal representation
JSON bool   → bool
JSON null   → JsonNull or policy-aware null
```

---

# 27. Ambigüedad de PHP null

PHP:

```php
null
```

no distingue naturalmente:

```text
SQL NULL
```

de:

```text
JSON null
```

---

# 28. Boundary-aware null

Por ello, la conversión deberá conocer el contexto.

Ejemplo:

```php
JsonValue::null()
```

podrá expresar explícitamente JSON null.

Mientras:

```php
null
```

en el nivel de la propiedad nullable podrá representar SQL NULL.

---

# 29. JsonValue

VoltStack podrá utilizar internamente:

```php
interface JsonValue
{
    public function kind(): JsonValueKind;
}
```

---

# 30. JsonValueKind

```php
enum JsonValueKind
{
    case OBJECT;
    case ARRAY;
    case STRING;
    case NUMBER;
    case BOOLEAN;
    case NULL;
}
```

---

# 31. Internal typed tree

Arquitectónicamente, la representación más segura será:

```text
JsonObject
JsonArray
JsonString
JsonNumber
JsonBoolean
JsonNull
```

aunque la API pública permita arrays PHP por ergonomía.

---

# 32. Public representation ≠ internal representation

VoltStack podrá aceptar:

```php
[
    'theme' => 'dark',
]
```

pero normalizarlo internamente a:

```text
JsonObject
```

cuando se requiera preservar semántica precisa.

---

# 33. PHP array ambiguity

En PHP:

```php
[]
```

puede representar:

```json
[]
```

o conceptualmente:

```json
{}
```

sin metadata adicional.

---

# 34. Empty object vs empty array

JSON exige:

```text
{}
≠
[]
```

VoltStack no deberá perder esa diferencia.

---

# 35. JsonObject

```php
final readonly class JsonObject implements JsonValue
{
    public function __construct(
        public array $members,
    ) {}
}
```

---

# 36. JsonArray

```php
final readonly class JsonArray implements JsonValue
{
    public function __construct(
        public array $items,
    ) {}
}
```

---

# 37. Public wrappers

Podrán existir helpers:

```php
Json::object([]);
Json::array([]);
Json::null();
```

para eliminar ambigüedades.

---

# 38. Associative array inference

Un array PHP con keys no secuenciales:

```php
[
    'name' => 'VoltStack',
]
```

puede inferirse razonablemente como JSON object.

---

# 39. Sequential array inference

```php
[
    'php',
    'rust',
]
```

puede inferirse como JSON array.

---

# 40. Empty array policy

Para:

```php
[]
```

la metadata deberá decidir.

No deberá inferirse arbitrariamente si el tipo requiere distinción.

---

# 41. JsonObjectPolicy

```php
enum JsonObjectPolicy
{
    case TYPED_TREE;
    case ASSOCIATIVE_ARRAY;
    case STDCLASS;
}
```

---

# 42. Recommended internal policy

```text
TYPED_TREE
```

para infraestructura interna que necesite fidelidad completa.

---

# 43. Developer ergonomics

El Model API podrá ofrecer:

```php
protected function casts(): array
{
    return [
        'settings' => 'json',
    ];
}
```

---

# 44. Typed JSON property

También:

```php
private JsonDocument $settings;
```

o:

```php
private array $settings;
```

según mapping.

---

# 45. Value Object JSON mapping

Desde el documento 160:

```text
Value Object
→ explicit structured mapping
→ JSON
```

---

# 46. No arbitrary object serialization

Esto seguirá prohibido:

```php
json_encode($domainObject);
```

como persistencia implícita.

---

# 47. JSON Value Object pipeline

```text
Value Object
    ↓
ValueObjectDecomposer
    ↓
Canonical Structured Value
    ↓
JSON Type System
    ↓
JSON Encoder
    ↓
DB Transport
```

---

# 48. JSON enums

Un enum dentro de JSON deberá usar una representación explícita.

Ejemplo:

```text
OrderStatus::Paid
→ "paid"
```

mediante Enum Mapping cuando la metadata lo declare.

---

# 49. No arbitrary UnitEnum encoding

El JSON encoder base no deberá decidir:

```text
case name?
backing value?
FQCN?
```

por sí mismo.

---

# 50. JSON casting

`DATABASE_CASTING_SYSTEM` podrá utilizar:

```text
JsonCast
```

para transformar entre:

```text
JSON logical value
↔
application representation
```

---

# 51. Value Conversion

`DATABASE_VALUE_CONVERSION_SYSTEM` controla la frontera:

```text
canonical JSON value
↔
database transport representation
```

---

# 52. Layer separation

```text
Casting
    application representation

JSON Type
    logical semantics

Value Conversion
    transport conversion

Platform
    physical representation

Driver
    protocol transport
```

---

# 53. Encoding

El encoder deberá ser:

- deterministic cuando canonicalization lo requiera;
- strict;
- Unicode-safe;
- depth-limited;
- error-reporting.

---

# 54. No silent encoding failure

Nunca:

```text
json_encode(...)
→ false
→ persist ""
```

---

# 55. Encoding errors

Deberán producir:

```text
JsonEncodingException
```

---

# 56. Decoding

El decoder deberá rechazar documentos inválidos.

---

# 57. No silent invalid JSON

Nunca:

```text
invalid JSON
→ null
```

porque eso confundiría:

```text
invalid
```

con:

```text
JSON null
```

---

# 58. JsonDecodingException

Deberá conservar:

- field context;
- type;
- platform;
- operation;
- bounded diagnostic information.

---

# 59. Unicode

JSON deberá preservar Unicode correctamente.

---

# 60. No unnecessary ASCII escaping

La representación canónica podrá evitar escapes innecesarios cuando sea seguro:

```json
{"city":"Monterrey"}
```

y:

```json
{"message":"México"}
```

sin degradar Unicode.

---

# 61. Invalid UTF-8

Deberá fallar explícitamente.

No deberá sustituirse silenciosamente salvo policy opt-in.

---

# 62. Numeric problem

JSON posee un único concepto sintáctico general de número, mientras PHP y las bases manejan:

```text
int
float
decimal
big integer
```

de formas diferentes.

---

# 63. Precision rule

> **VoltStack no deberá convertir silenciosamente números JSON de precisión arbitraria a `float` cuando eso pueda alterar su valor.**

---

# 64. Example

JSON:

```text
9007199254740993
```

no deberá convertirse sin advertencia a una representación que produzca:

```text
9007199254740992
```

---

# 65. JsonNumber

Representación interna:

```php
final readonly class JsonNumber implements JsonValue
{
    public function __construct(
        public string $lexeme,
    ) {}
}
```

Esto permite preservar precisión.

---

# 66. Number policy

```php
enum JsonNumberPolicy
{
    case PHP_NATIVE;
    case LOSSLESS;
    case DECIMAL_AWARE;
}
```

---

# 67. Recommended persistence policy

Para infraestructura Database:

```text
LOSSLESS
```

será la política preferida cuando sea necesaria fidelidad.

---

# 68. Application convenience

`PHP_NATIVE` podrá utilizarse cuando el desarrollador acepte explícitamente las limitaciones.

---

# 69. Negative zero

JSON numeric canonicalization deberá definir el tratamiento de:

```text
-0
-0.0
0
0.0
```

según la política.

---

# 70. NaN

No pertenece al estándar JSON.

Por tanto:

```text
NaN
```

deberá rechazarse.

---

# 71. Infinity

Igualmente:

```text
Infinity
-Infinity
```

deberán rechazarse.

---

# 72. Canonicalization

Dos representaciones pueden ser semánticamente equivalentes:

```json
{"a":1,"b":2}
```

y:

```json
{
  "b": 2,
  "a": 1
}
```

dependiendo de la política de igualdad.

---

# 73. JSON object order

El orden de miembros de un object no deberá considerarse identidad semántica por defecto.

---

# 74. JSON array order

El orden de un array sí será significativo:

```text
[1,2]
≠
[2,1]
```

---

# 75. Canonical object representation

Para fingerprints/comparación, VoltStack podrá ordenar object keys determinísticamente.

---

# 76. Storage representation ≠ canonical comparison representation

No es obligatorio reescribir físicamente la DB cada vez que cambia el orden textual de las keys.

---

# 77. JsonCanonicalizationPolicy

```php
enum JsonCanonicalizationPolicy
{
    case STRUCTURAL;
    case CANONICAL_TEXT;
    case PLATFORM_NATIVE;
}
```

---

# 78. Recommended default

```text
STRUCTURAL
```

para dirty tracking y semantic equality.

---

# 79. Structural equality

Conceptualmente:

```text
JsonEquivalent(A, B)
```

compara:

- value kinds;
- object members sin depender de member order;
- array elements respetando order;
- scalar values;
- numeric policy.

---

# 80. Text equality

No usar:

```text
rawJsonA === rawJsonB
```

como default dirty tracking.

---

# 81. Example

```text
{"a":1,"b":2}
```

y:

```text
{"b":2,"a":1}
```

no deberían producir un UPDATE únicamente por formatting/order cuando la policy sea estructural.

---

# 82. Duplicate object keys

Input como:

```json
{"a":1,"a":2}
```

es problemático.

VoltStack deberá adoptar una política estricta.

---

# 83. Recommended duplicate-key policy

Rechazar durante parsing/canonicalization cuando pueda detectarse.

---

# 84. Why

Aceptar "last wins" silenciosamente puede ocultar:

- malformed data;
- security bugs;
- producer inconsistencies.

---

# 85. JSON path

El Type System deberá definir una representación lógica portable.

---

# 86. JsonPath

```php
final readonly class JsonPath
{
    /**
     * @param list<JsonPathSegment> $segments
     */
    public function __construct(
        public array $segments,
    ) {}
}
```

---

# 87. Path segments

```text
Root
ObjectKey("user")
ObjectKey("address")
ObjectKey("city")
ArrayIndex(0)
```

---

# 88. Path ≠ SQL string

No representar internamente la semántica únicamente como:

```text
"$.user.address.city"
```

---

# 89. Why

Las plataformas tienen sintaxis diferentes.

Por tanto:

```text
Logical JsonPath
→ Platform Compiler
→ Physical JSON path syntax
```

---

# 90. User-supplied path security

Dynamic paths deberán convertirse a segmentos validados.

Nunca concatenarse directamente al SQL.

---

# 91. Path escaping

Keys como:

```text
user.name
```

pueden ser una key literal, no dos segmentos.

El typed path model elimina esta ambigüedad.

---

# 92. JSON capability model

Cada plataforma deberá declarar capacidades.

Ejemplo:

```php
interface JsonPlatformCapabilities
{
    public function supportsNativeJsonType(): bool;

    public function supportsBinaryJsonStorage(): bool;

    public function supportsPathExtraction(): bool;

    public function supportsContainment(): bool;

    public function supportsPartialMutation(): bool;

    public function supportsJsonIndexing(): bool;

    public function supportsJsonAggregation(): bool;
}
```

---

# 93. Capability ≠ vendor check

No:

```php
if ($database === 'postgres') {
}
```

en el core JSON Type System.

---

# 94. Platform resolver

```text
JsonTypeDescriptor
        ↓
JsonPhysicalTypeResolver
        ↓
Platform Capability System
        ↓
Physical Type
```

---

# 95. MySQL

VoltStack podrá aprovechar el tipo JSON nativo cuando la versión/capabilities lo permitan.

---

# 96. MariaDB

Deberá analizarse independientemente de MySQL.

No se asumirá equivalencia completa de:

- storage;
- validation;
- functions;
- indexing;
- comparison;
- JSON behavior.

---

# 97. PostgreSQL

Deberá distinguir:

```text
JSON
```

y:

```text
JSONB
```

como representaciones físicas distintas del mismo dominio lógico JSON cuando corresponda.

---

# 98. PostgreSQL JSON

Puede ser útil cuando importa preservar más directamente la representación textual.

---

# 99. PostgreSQL JSONB

Puede ser preferible cuando importan:

- querying;
- indexing;
- normalized storage;
- containment.

---

# 100. Logical JSON ≠ PostgreSQL JSONB

`JSONB` será una decisión física/capability.

No la identidad lógica del tipo de dominio.

---

# 101. SQLite

SQLite podrá representar JSON físicamente mediante su modelo de storage disponible y capacidades JSON detectadas.

---

# 102. SQLite type affinity

No se deberá confundir:

```text
declared JSON intent
```

con:

```text
strict native JSON storage type
```

si la plataforma no ofrece esa semántica.

---

# 103. Platform differences

VoltStack deberá normalizar semántica cuando sea razonable y declarar limitaciones cuando no lo sea.

---

# 104. No fake portability

Si una operación JSON no puede conservar semántica equivalente en una plataforma:

```text
UNSUPPORTED
```

o:

```text
REQUIRES_EMULATION
```

será preferible a generar comportamiento diferente silenciosamente.

---

# 105. JSON physical strategy

```php
enum JsonStoragePreference
{
    case PORTABLE;
    case NATIVE;
    case BINARY_OPTIMIZED;
    case TEXTUAL;
    case PLATFORM_DEFAULT;
}
```

---

# 106. PORTABLE

Favorece comportamiento común entre plataformas.

---

# 107. NATIVE

Favorece el tipo JSON nativo cuando existe.

---

# 108. BINARY_OPTIMIZED

Puede resolver a representaciones como:

```text
JSONB
```

cuando la plataforma lo soporte.

---

# 109. TEXTUAL

Puede ser útil para casos donde preservar una representación textual sea parte del requisito.

---

# 110. Schema integration

Ejemplo:

```php
$table->json('settings');
```

deberá producir:

```text
Logical JSON Column
```

no SQL directo.

---

# 111. Schema model

```text
ColumnDefinition
    type = JSON
    jsonOptions = ...
```

---

# 112. Schema compiler

```text
Logical JSON
     ↓
Platform JSON Resolver
     ↓
Physical JSON Declaration
```

---

# 113. Introspection

Reverse mapping:

```text
Physical Column Metadata
        ↓
Platform Type Introspector
        ↓
Logical JSON Type
        +
Physical Strategy
        +
Confidence
```

---

# 114. Introspection uncertainty

Una columna `TEXT` con datos JSON no implica automáticamente:

```text
Logical JSON
```

---

# 115. Constraint evidence

Podrá existir evidencia adicional:

```text
JSON validation constraint
```

pero aun así deberá conservarse el nivel de confianza apropiado.

---

# 116. Unknown ≠ JSON

No inferir intención únicamente porque algunas filas parezcan contener JSON.

---

# 117. Schema diff

Debe distinguir:

```text
TEXT
→ JSON

JSON
→ JSONB-like representation

JSON
→ TEXT
```

como cambios potencialmente significativos.

---

# 118. Data migration

Cambiar representación puede requerir:

```text
validation
conversion
backfill
verification
```

no únicamente `ALTER TYPE`.

---

# 119. Invalid legacy JSON

Antes de:

```text
TEXT → native JSON
```

deberá verificarse que los datos existentes sean válidos.

---

# 120. Migration safety

La Migration Safety Layer deberá considerar:

- table size;
- rewrite behavior;
- locking;
- invalid documents;
- index rebuilds;
- compatibility;
- rollback feasibility.

---

# 121. Parameter binding

Aplicación:

```php
[
    'theme' => 'dark',
]
```

se normaliza:

```text
JsonObject
```

y posteriormente:

```text
JSON transport value
```

antes del binding.

---

# 122. Driver boundary

El Driver no necesita conocer:

```text
JsonObject
```

si el Platform/Conversion layer ya ha producido la representación de transporte apropiada.

---

# 123. Query Builder

Podrá aceptar:

```php
$query->where('settings', $jsonValue);
```

cuando la operación sea semánticamente válida.

---

# 124. JSON-specific query operations

Posteriormente podrán existir:

```php
->whereJsonContains(...)
->whereJsonPath(...)
->whereJsonExists(...)
```

pero deberán converger en AST semántico.

---

# 125. Query Builder does not emit JSON SQL

No:

```text
JSON_EXTRACT(...)
@>
json_extract(...)
```

desde el Builder.

---

# 126. Semantic JSON expressions

El Query AST podrá contener:

```text
JsonExtractExpression
JsonContainsPredicate
JsonExistsPredicate
JsonTypePredicate
JsonPathExpression
```

---

# 127. Compiler responsibility

El Compiler + Platform traducirán esas expresiones a la representación física.

---

# 128. Query Type System integration

`DATABASE_QUERY_TYPE_SYSTEM` deberá comprender que:

```text
JsonExtract(path)
```

puede producir:

```text
JSON
STRING
NUMBER
BOOLEAN
NULLABLE(...)
UNKNOWN
```

según metadata/operation.

---

# 129. Type inference

El sistema del documento 39 podrá consumir:

```text
JSON operation signatures
```

para inferir tipos.

---

# 130. Unknown path type

Para JSON schemaless:

```text
$.some.dynamic.path
```

el tipo puede ser:

```text
JSON_VALUE / UNKNOWN
```

---

# 131. No fabricated scalar type

No asumir:

```text
string
```

únicamente porque una comparación lo sugiere.

---

# 132. JSON schema-aware extension

En el futuro una metadata estructural opcional podrá proporcionar tipos conocidos para paths.

---

# 133. JSON schema metadata ≠ database schema

Debe distinguirse:

```text
Database Schema
```

de:

```text
JSON Document Shape Schema
```

---

# 134. Hydration

Pipeline:

```text
DB Value
   ↓
Driver
   ↓
Value Conversion
   ↓
JSON Decode
   ↓
Canonical JSON Value
   ↓
Casting / VO Mapping
   ↓
Entity Property
```

---

# 135. Hydration error

JSON inválido observado en una representación que debería garantizar JSON válido deberá tratarse como inconsistencia.

---

# 136. No fallback to raw string

Nunca:

```text
invalid JSON
→ return raw string
```

sin policy explícita de integración legacy.

---

# 137. Result hydration

JSON columns deberán conservar su TypeDescriptor dentro del Hydration Plan.

---

# 138. Hydration Plan

```text
column settings
→ JSON Type
→ JsonDecoder
→ JsonCast
→ entity.settings
```

---

# 139. Persistence

Pipeline:

```text
Entity JSON Property
       ↓
Casting
       ↓
Canonical JSON
       ↓
Validation
       ↓
Value Conversion
       ↓
Binding
```

---

# 140. Dirty tracking

Raw text comparison será insuficiente.

---

# 141. Structural snapshot

El UnitOfWork podrá conservar:

```text
CanonicalJsonFingerprint
```

o una canonical structural snapshot.

---

# 142. JsonFingerprint

```php
final readonly class JsonFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 143. Fingerprint requirements

Deberá ser:

- deterministic;
- structural;
- policy-aware;
- collision-resistant enough for intended usage.

---

# 144. Fingerprint ≠ sole proof when unsafe

Si se utiliza un hash, una política estricta puede requerir comparación estructural cuando exista conflicto o cuando correctness lo exija.

---

# 145. Dirty formula

```text
Dirty(JSON)
=
!JsonEquivalent(
    Snapshot,
    Current
)
```

---

# 146. Example

Snapshot:

```json
{"theme":"dark","language":"es"}
```

Current:

```json
{"language":"es","theme":"dark"}
```

Resultado con structural equality:

```text
NOT DIRTY
```

---

# 147. Array example

Snapshot:

```json
["php","rust"]
```

Current:

```json
["rust","php"]
```

Resultado:

```text
DIRTY
```

---

# 148. Mutation model

VoltStack deberá distinguir:

```text
replace entire JSON document
```

de:

```text
patch JSON document
```

---

# 149. Whole replacement

```php
$model->settings = [
    'theme' => 'dark',
];
```

representa un nuevo documento.

---

# 150. Patch

Podrá existir una API:

```php
$model->settings()->set('theme', 'dark');
```

pero no deberá convertirse inmediatamente en SQL.

---

# 151. JsonPatch

```php
final readonly class JsonPatch
{
    /**
     * @param list<JsonPatchOperation> $operations
     */
    public function __construct(
        public array $operations,
    ) {}
}
```

---

# 152. Patch operations

```text
SET
REMOVE
INSERT
REPLACE
APPEND
INCREMENT
```

cuando sus semánticas estén claramente definidas.

---

# 153. Semantic patch

```text
JsonPatch
≠
SQL JSON_SET(...)
```

---

# 154. Platform compilation

```text
JsonPatch
    ↓
Planner
    ↓
Platform Capability Analysis
    ↓
native partial update
or
read/modify/write strategy
or
unsupported
```

---

# 155. No hidden read-modify-write

Si una operación requiere leer el documento antes de escribirlo, esa decisión deberá ser explícita y considerar concurrencia.

---

# 156. Lost update problem

Dos workers:

```text
Worker A changes $.theme
Worker B changes $.language
```

pueden sobrescribirse si ambos reemplazan el documento completo.

---

# 157. Partial update capability

Una plataforma con mutación JSON atómica puede reducir ese riesgo para paths independientes.

---

# 158. Concurrency still matters

Partial JSON update no elimina:

- transaction semantics;
- optimistic locking;
- concurrent mutation conflicts.

---

# 159. Optimistic locking integration

El JSON system podrá participar en:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

mediante el owner Entity.

---

# 160. Patch ordering

Operaciones:

```text
REMOVE $.a
SET $.a.b = 1
```

pueden depender del orden.

Por tanto:

```text
JsonPatch operations are ordered
```

---

# 161. Patch normalization

Solo se combinarán/reordenarán operaciones cuando pueda probarse equivalencia.

---

# 162. Path existence

Patch deberá distinguir:

```text
SET
INSERT_IF_MISSING
REPLACE_IF_PRESENT
REMOVE
```

cuando la plataforma/capability lo permita.

---

# 163. JSON null patch

```text
SET $.x = JSON null
```

no equivale a:

```text
REMOVE $.x
```

---

# 164. SQL NULL update

Igualmente:

```text
column = SQL NULL
```

no equivale a:

```text
JSON document = null
```

---

# 165. Containment capability

El Type System podrá declarar operaciones lógicas como:

```text
CONTAINS
CONTAINED_BY
HAS_KEY
HAS_ANY_KEY
HAS_ALL_KEYS
```

sin definir todavía toda la Query API.

---

# 166. Capability matrix

Cada operación podrá evaluarse como:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 167. Indexing

JSON indexing será capability-driven.

---

# 168. Indexing strategies

Pueden incluir:

```text
native JSON index
expression index
generated column + index
path index
GIN-like index
```

dependiendo de plataforma.

---

# 169. Logical index request

Schema podrá expresar:

```text
index JSON path $.customer.id
```

sin codificar una implementación específica.

---

# 170. Physical planner

Platform decide:

```text
native path index
```

o:

```text
generated/expression column/index
```

cuando sea soportado.

---

# 171. Index semantics

VoltStack deberá conocer:

- path;
- extracted logical type;
- null semantics;
- collation where relevant;
- expression determinism.

---

# 172. Indexing unknown path type

No crear un índice tipado automáticamente sin suficiente metadata.

---

# 173. Generated columns

Podrán ser estrategia física para plataformas que las soporten.

Pero:

```text
Generated Column
≠
JSON logical property
```

---

# 174. JSON object keys

Las keys son strings.

No deberán sufrir automáticamente:

- case folding;
- naming convention transformations;
- entity property normalization;

salvo mapping explícito.

---

# 175. Case sensitivity

```text
"Name"
≠
"name"
```

como JSON keys.

---

# 176. Collation

String comparison dentro de JSON puede depender de operación/plataforma.

VoltStack no deberá fingir semántica uniforme cuando la DB difiera.

---

# 177. Portability report

Diagnostics podrá indicar:

```text
JSON operation:
    containment

MySQL:
    supported

MariaDB:
    supported with semantic differences

PostgreSQL:
    supported

SQLite:
    emulated / capability-dependent
```

según capabilities concretas detectadas.

---

# 178. No vendor assumptions

La matriz se construirá desde Platform Capability System, no desde constantes hardcoded en el Query Builder.

---

# 179. JSON validation

El Type System valida:

```text
valid JSON structure
```

y type/root constraints.

---

# 180. Domain validation

Reglas como:

```text
settings.theme must be dark|light
```

pertenecen principalmente al Validation/domain layer.

---

# 181. Optional shape validation

Puede integrarse una extensión de JSON shape/schema validation.

---

# 182. Database constraints

Si la plataforma soporta constraints apropiadas, Schema podrá proyectar algunas reglas.

Pero:

```text
Application JSON validation
≠
Database constraint truth
```

---

# 183. Security — depth bombs

Input JSON puede tener nesting excesivo.

Debe existir:

```text
maxDepth
```

---

# 184. Security — document size

Debe existir:

```text
maxBytes
```

---

# 185. Security — member count

Puede existir:

```text
maxObjectMembers
maxArrayItems
```

---

# 186. Security — path depth

Dynamic JSON paths tendrán:

```text
maxPathDepth
```

---

# 187. Security — patch count

Un patch deberá tener:

```text
maxPatchOperations
```

---

# 188. JsonResourcePolicy

```php
final readonly class JsonResourcePolicy
{
    public function __construct(
        public int $maxDepth,
        public int $maxBytes,
        public int $maxObjectMembers,
        public int $maxArrayItems,
        public int $maxPathDepth,
        public int $maxPatchOperations,
    ) {}
}
```

---

# 189. Resource limits ≠ domain limits

El framework puede imponer límites de seguridad aunque el dominio tenga reglas adicionales.

---

# 190. JSON path injection

Nunca:

```php
$sql .= "$userPath";
```

---

# 191. JSON content injection

El contenido JSON seguirá pasando como parameter binding cuando sea posible.

---

# 192. Sensitive JSON

Documentos JSON pueden contener:

```text
tokens
personal information
credentials
financial data
```

Telemetry deberá aplicar redaction.

---

# 193. No raw JSON telemetry by default

Nunca registrar automáticamente documentos completos.

---

# 194. Diagnostic preview

Si se necesita:

```text
bounded
redacted
explicitly enabled
```

---

# 195. Large JSON documents

No deberán duplicarse innecesariamente en memoria.

---

# 196. Streaming

Para documentos extremadamente grandes, una extensión futura podrá soportar streaming parser/encoder.

---

# 197. V1 scope

V1 podrá priorizar:

```text
bounded in-memory JSON documents
```

con límites explícitos.

---

# 198. Canonicalization cost

Ordenar recursivamente keys puede ser costoso.

Por tanto, canonicalization deberá aplicarse únicamente cuando sea necesaria.

---

# 199. No canonical text per access

No generar un canonical JSON string cada vez que se accede a una propiedad.

---

# 200. Snapshot optimization

Podrá utilizarse:

```text
structural normalized representation
+
lazy fingerprint
```

---

# 201. Mutation-aware tracking

Si se utiliza un wrapper mutable:

```php
JsonDocument
```

podrá registrar patches/deltas.

---

# 202. Array mutation problem

Con arrays PHP normales:

```php
$model->settings['theme'] = 'dark';
```

el framework necesita detectar mutación interna.

---

# 203. Strategies

Podrán existir:

```text
SNAPSHOT_COMPARE
IMMUTABLE_DOCUMENT
TRACKED_JSON_DOCUMENT
```

---

# 204. JsonTrackingStrategy

```php
enum JsonTrackingStrategy
{
    case SNAPSHOT_COMPARE;
    case IMMUTABLE;
    case TRACKED_MUTATIONS;
}
```

---

# 205. Default

Para API ergonómica basada en arrays:

```text
SNAPSHOT_COMPARE
```

será una opción razonable.

---

# 206. High-performance option

```text
TRACKED_MUTATIONS
```

puede reducir comparaciones de documentos grandes.

---

# 207. Tracked document

```php
interface TrackedJsonDocument
{
    public function patches(): JsonPatch;

    public function isDirty(): bool;
}
```

---

# 208. Tracked state is scoped

Nunca compartir estado mutable de `TrackedJsonDocument` entre requests.

---

# 209. Cache

Type descriptors y compiled JSON plans pueden compartirse.

---

# 210. Result cache

JSON result values deberán ser tratados como datos de resultado, no como Type metadata.

---

# 211. Cache serialization

El Result Cache deberá preservar:

```text
SQL NULL
JSON null
{}
[]
```

sin colapsarlos.

---

# 212. Cache key

Query parameters JSON requieren fingerprints deterministas cuando formen parte de cache keys.

---

# 213. Object key order

Cache key para objetos deberá ser structural/canonical para evitar:

```text
{"a":1,"b":2}
```

y:

```text
{"b":2,"a":1}
```

como keys distintas cuando sean semánticamente equivalentes.

---

# 214. Arrays remain ordered

Para arrays:

```text
[1,2]
```

y:

```text
[2,1]
```

deberán producir fingerprints diferentes.

---

# 215. Persistent runtime

Compartible:

```text
JsonTypeDescriptor
Json codecs
Compiled Json Plans
Platform JSON capability metadata
```

si son immutable/stateless.

---

# 216. Scoped runtime state

```text
decoder buffers
encoder buffers
patch accumulators
tracking state
resource counters
diagnostic context
```

deberá ser scoped.

---

# 217. FrankenPHP

```text
Worker
├── Immutable JSON Type Metadata
├── Compiled JSON Plans
│
├── Request A JSON State
└── Request B JSON State
```

---

# 218. RoadRunner

Aplicará el mismo modelo.

---

# 219. OpenSwoole

Buffers y mutable trackers deberán ser coroutine-safe/scoped.

---

# 220. No static mutable decoder state

Prohibido:

```php
static $currentDocument;
```

---

# 221. Type generation

Cambios en:

- number policy;
- root constraint;
- storage preference;
- canonicalization;
- codec;
- JSON mapping;

deberán alterar el fingerprint/generation correspondiente.

---

# 222. JsonTypeFingerprint

Conceptualmente:

```text
JSON TypeId
+
Root Constraint
+
Number Policy
+
Object Policy
+
Canonicalization Policy
+
Codec Generation
+
Platform Capability Generation
```

---

# 223. Diagnostics API

Conceptualmente:

```php
Database::types()
    ->json()
    ->explain('settings');
```

---

# 224. Diagnostic example

```text
JSON TYPE

Field:
    User.settings

Logical Type:
    JSON

Root:
    OBJECT

PHP Representation:
    associative-array

Tracking:
    SNAPSHOT_COMPARE

Canonical Equality:
    STRUCTURAL

Physical Platform:
    PostgreSQL

Physical Storage:
    JSONB

Path Extraction:
    SUPPORTED

Containment:
    SUPPORTED

Partial Mutation:
    SUPPORTED

Indexing:
    SUPPORTED

SQL NULL:
    allowed

JSON null:
    allowed

Max Depth:
    64

Max Document:
    1 MiB
```

---

# 225. Portability diagnostic

```text
JSON PORTABILITY

Operation:
    PATH CONTAINMENT

Required Semantics:
    OBJECT CONTAINMENT

Current Platform:
    PostgreSQL

Status:
    SUPPORTED

Portable Baseline:
    LIMITED

Fallback:
    NONE

Migration Risk:
    MEDIUM
```

---

# 226. Telemetry

Métricas potenciales:

```text
database.json.encode.total
database.json.decode.total
database.json.encode.failure
database.json.decode.failure
database.json.bytes
database.json.depth_limit
database.json.size_limit
database.json.patch.operations
database.json.partial_update
database.json.full_replace
database.json.canonicalization
database.json.dirty_check
database.json.platform_emulation
database.json.capability_failure
```

---

# 227. Cardinality

No utilizar:

```text
JSON path
raw key
raw document
```

como labels no acotadas.

---

# 228. Tracing

JSON operations podrán anotarse dentro de spans existentes.

No crear spans por cada key/path trivial.

---

# 229. Error hierarchy

```text
DatabaseJsonException
├── JsonTypeException
├── InvalidJsonValueException
├── JsonEncodingException
├── JsonDecodingException
├── JsonUnicodeException
├── JsonNumberPrecisionException
├── JsonDuplicateKeyException
├── JsonRootTypeException
├── JsonNullSemanticsException
├── JsonPathException
├── JsonPathTypeException
├── JsonPathLimitException
├── JsonPatchException
├── JsonPatchConflictException
├── JsonResourceLimitException
├── JsonPlatformCapabilityException
├── JsonPhysicalMappingException
├── JsonSchemaCompatibilityException
├── JsonIntrospectionException
├── JsonPersistenceException
├── JsonHydrationException
├── JsonRuntimeStateException
└── JsonInvariantViolationException
```

---

# 230. Testing architecture

Debe existir:

```text
JsonTypeConformanceSuite
```

---

# 231. Core value tests

Probar:

```text
object
array
string
integer-like number
decimal-like number
boolean
JSON null
SQL NULL
empty object
empty array
```

---

# 232. Null semantics test

Verificar inequívocamente:

```text
SQL NULL
≠
JSON null
≠
MISSING
```

---

# 233. Empty structure test

```text
{}
≠
[]
```

---

# 234. Numeric tests

Incluir:

```text
0
-0
1
1.0
large integers
high precision decimals
very small decimals
exponents
```

---

# 235. Invalid numeric tests

Rechazar:

```text
NaN
Infinity
-Infinity
```

---

# 236. Unicode tests

Incluir:

- ASCII;
- español;
- caracteres multibyte;
- emoji;
- invalid UTF-8.

---

# 237. Object equality tests

```text
{"a":1,"b":2}
```

vs:

```text
{"b":2,"a":1}
```

deberán seguir la canonicalization policy.

---

# 238. Array equality tests

Order deberá conservarse.

---

# 239. Duplicate key tests

El decoder estricto deberá detectar/rechazar cuando la estrategia lo soporte.

---

# 240. Path tests

Probar:

- nested objects;
- array indexes;
- keys with dots;
- quotes;
- Unicode keys;
- missing keys;
- JSON null.

---

# 241. Patch tests

Probar:

```text
SET
REMOVE
INSERT
REPLACE
APPEND
```

y conflictos de paths.

---

# 242. Patch null tests

Distinguir:

```text
set JSON null
remove path
set SQL NULL document
```

---

# 243. Dirty tracking tests

Cambios de formatting/key order no deberán generar false dirty bajo structural equality.

---

# 244. Persistence round-trip

Para cada valor soportado:

```text
Logical JSON
→ DB
→ Logical JSON
```

deberá preservar semántica.

---

# 245. Platform matrix

Ejecutar conformance tests sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 246. Physical round-trip

Verificar que diferencias físicas no cambien el valor lógico esperado.

---

# 247. Schema tests

Probar:

- create JSON column;
- introspection;
- diff;
- storage strategy changes;
- nullable;
- indexes;
- migration from text.

---

# 248. Invalid legacy data test

`TEXT → JSON` deberá detectar documentos inválidos antes de afirmar migración segura.

---

# 249. Resource tests

Probar:

- max depth;
- max bytes;
- max members;
- max array items;
- max path depth;
- max patch operations.

---

# 250. Persistent runtime tests

Dos requests deberán compartir únicamente metadata/planes immutable.

---

# 251. Performance benchmarks

Medir:

```text
encode MB/sec
decode MB/sec
structural comparison/sec
fingerprint/sec
path compilation/sec
patch planning/sec
hydration/sec
persistence conversion/sec
```

---

# 252. Large document benchmarks

Incluir tamaños como:

```text
1 KB
10 KB
100 KB
1 MB
```

dentro de límites configurados.

---

# 253. Hot-path rules

No realizar:

- schema introspection;
- capability discovery repetida;
- reflection;
- reparsing de JsonPath compilado;
- container lookup por node;
- canonical string generation innecesaria.

---

# 254. Compiled JsonPath

Paths frecuentes podrán compilarse:

```php
final readonly class CompiledJsonPath
{
    public function __construct(
        public JsonPath $logicalPath,
        public JsonPathFingerprint $fingerprint,
    ) {}
}
```

La representación SQL específica seguirá perteneciendo al Platform/Compiler.

---

# 255. Directory structure

```text
src/Quantum/Database/Type/Json/
│
├── Contract/
│   ├── JsonCodec.php
│   ├── JsonEncoder.php
│   ├── JsonDecoder.php
│   ├── JsonCanonicalizer.php
│   ├── JsonComparator.php
│   ├── JsonFingerprintGenerator.php
│   └── JsonPlatformCapabilities.php
│
├── Type/
│   ├── JsonType.php
│   ├── JsonTypeId.php
│   ├── JsonTypeDescriptor.php
│   ├── JsonRootConstraint.php
│   ├── JsonStoragePreference.php
│   ├── JsonNumberPolicy.php
│   ├── JsonObjectPolicy.php
│   └── JsonCanonicalizationPolicy.php
│
├── Value/
│   ├── JsonValue.php
│   ├── JsonValueKind.php
│   ├── JsonObject.php
│   ├── JsonArray.php
│   ├── JsonString.php
│   ├── JsonNumber.php
│   ├── JsonBoolean.php
│   ├── JsonNull.php
│   └── JsonMissing.php
│
├── Codec/
│   ├── DefaultJsonCodec.php
│   ├── DefaultJsonEncoder.php
│   └── DefaultJsonDecoder.php
│
├── Canonicalization/
│   ├── StructuralJsonCanonicalizer.php
│   ├── CanonicalJsonComparator.php
│   └── JsonFingerprint.php
│
├── Path/
│   ├── JsonPath.php
│   ├── JsonPathSegment.php
│   ├── JsonRootSegment.php
│   ├── JsonObjectKeySegment.php
│   ├── JsonArrayIndexSegment.php
│   ├── CompiledJsonPath.php
│   └── JsonPathFingerprint.php
│
├── Patch/
│   ├── JsonPatch.php
│   ├── JsonPatchOperation.php
│   ├── JsonSetOperation.php
│   ├── JsonRemoveOperation.php
│   ├── JsonInsertOperation.php
│   ├── JsonReplaceOperation.php
│   ├── JsonAppendOperation.php
│   └── JsonPatchNormalizer.php
│
├── Tracking/
│   ├── JsonTrackingStrategy.php
│   ├── TrackedJsonDocument.php
│   ├── JsonSnapshot.php
│   └── JsonDirtyChecker.php
│
├── Platform/
│   ├── JsonPhysicalTypeResolver.php
│   ├── JsonCapabilityResolver.php
│   └── JsonOperationCapability.php
│
├── Schema/
│   ├── JsonSchemaProjector.php
│   ├── JsonSchemaCompatibilityAnalyzer.php
│   └── JsonIndexDescriptor.php
│
├── Conversion/
│   ├── JsonDatabaseValueConverter.php
│   └── JsonTransportValue.php
│
├── Policy/
│   └── JsonResourcePolicy.php
│
├── Diagnostics/
│   ├── JsonTypeExplainer.php
│   ├── JsonPortabilityReport.php
│   └── JsonDiagnosticReport.php
│
└── Exception/
    ├── DatabaseJsonException.php
    ├── InvalidJsonValueException.php
    ├── JsonEncodingException.php
    ├── JsonDecodingException.php
    ├── JsonUnicodeException.php
    ├── JsonNumberPrecisionException.php
    ├── JsonDuplicateKeyException.php
    ├── JsonRootTypeException.php
    ├── JsonPathException.php
    ├── JsonPatchException.php
    ├── JsonResourceLimitException.php
    ├── JsonPlatformCapabilityException.php
    ├── JsonSchemaCompatibilityException.php
    ├── JsonHydrationException.php
    └── JsonInvariantViolationException.php
```

---

# 256. Dependency model

Permitido:

```text
JSON Type System
      ↓
Database Type System

JSON Type System
      ↓
Value Conversion contracts

JSON Type System
      ↓
Casting contracts

JSON Type System
      ↓
Platform Capabilities
```

---

# 257. Value Object integration

```text
Value Object Mapping
        ↓
JSON Type System
```

cuando la estrategia sea JSON.

No:

```text
JSON Type System
→ arbitrary domain object discovery
```

---

# 258. Query integration

```text
Query Semantic Engine
        ↓
JSON Type Semantics
        ↓
JSON AST
        ↓
Planner
        ↓
Compiler
```

---

# 259. Schema integration

```text
Schema
  ↓
Logical JSON Type
  ↓
Platform Resolver
```

---

# 260. Driver boundary

```text
JSON logical semantics
```

no deberán filtrarse al Driver salvo los transport hints estrictamente necesarios.

---

# 261. Architectural invariants

## DB-JSON-001
JSON será un tipo lógico estructurado.

## DB-JSON-002
JSON no será tratado como string decorado.

## DB-JSON-003
Logical JSON Type será independiente de physical SQL type.

## DB-JSON-004
SQL NULL será distinto de JSON null.

## DB-JSON-005
JSON null será distinto de missing path.

## DB-JSON-006
Missing será distinto de SQL NULL.

## DB-JSON-007
Empty object será distinto de empty array.

## DB-JSON-008
Object será distinto de array.

## DB-JSON-009
Array order será significativo.

## DB-JSON-010
Object member order no será semantic identity por default.

## DB-JSON-011
JSON numbers no sufrirán silent precision loss.

## DB-JSON-012
NaN será rechazado.

## DB-JSON-013
Infinity será rechazado.

## DB-JSON-014
Invalid JSON no será convertido silenciosamente a null.

## DB-JSON-015
Encoding failure será explícito.

## DB-JSON-016
Decoding failure será explícito.

## DB-JSON-017
Invalid UTF-8 no se corregirá silenciosamente por default.

## DB-JSON-018
Duplicate object keys tendrán policy explícita.

## DB-JSON-019
Strict duplicate-key rejection será preferido.

## DB-JSON-020
PHP array ambiguity será reconocida explícitamente.

## DB-JSON-021
Empty PHP array no decidirá universalmente object vs array.

## DB-JSON-022
Internal typed representation podrá diferir de public PHP representation.

## DB-JSON-023
JSON Type System no serializará arbitrary objects.

## DB-JSON-024
Value Objects usarán explicit structured mapping.

## DB-JSON-025
Enums dentro de JSON usarán mapping explícito.

## DB-JSON-026
UnitEnum no será serializado arbitrariamente.

## DB-JSON-027
Casting será distinto de JSON logical typing.

## DB-JSON-028
Value Conversion será distinto de JSON logical typing.

## DB-JSON-029
Platform será responsable de physical representation.

## DB-JSON-030
Driver no será responsable de domain JSON semantics.

## DB-JSON-031
JsonPath será una estructura tipada.

## DB-JSON-032
JsonPath no será únicamente SQL syntax.

## DB-JSON-033
Dynamic paths serán validados.

## DB-JSON-034
User paths nunca se concatenarán directamente al SQL.

## DB-JSON-035
Capability checks sustituirán vendor conditionals.

## DB-JSON-036
MySQL y MariaDB serán plataformas independientes.

## DB-JSON-037
PostgreSQL JSON y JSONB serán physical strategies distintas.

## DB-JSON-038
Logical JSON no será equivalente a JSONB.

## DB-JSON-039
SQLite JSON intent será distinto de storage affinity.

## DB-JSON-040
Unsupported semantics no se fingirán portables.

## DB-JSON-041
Physical strategy será explícita/capability-driven.

## DB-JSON-042
Schema Builder producirá logical JSON definitions.

## DB-JSON-043
Schema Compiler resolverá physical declarations.

## DB-JSON-044
TEXT no será introspectado automáticamente como logical JSON.

## DB-JSON-045
Introspection uncertainty será preservada.

## DB-JSON-046
Schema representation changes podrán requerir data migration.

## DB-JSON-047
Legacy TEXT→JSON requerirá validación de datos.

## DB-JSON-048
Query Builder no generará vendor JSON SQL.

## DB-JSON-049
JSON query operations serán semantic AST.

## DB-JSON-050
Compiler traducirá semantic JSON operations.

## DB-JSON-051
Unknown JSON path type no será fabricado.

## DB-JSON-052
Hydration Plan conservará JSON type information.

## DB-JSON-053
Hydration no retornará raw invalid JSON por default.

## DB-JSON-054
Persistence validará JSON antes de binding cuando corresponda.

## DB-JSON-055
Dirty tracking no dependerá de raw textual equality.

## DB-JSON-056
Structural equality será default recomendado.

## DB-JSON-057
Object key reordering no producirá false dirty bajo structural policy.

## DB-JSON-058
Array reordering sí podrá producir dirty.

## DB-JSON-059
Snapshot deberá preservar JSON semantics.

## DB-JSON-060
JsonFingerprint será deterministic.

## DB-JSON-061
Hash fingerprint no reemplazará correctness cuando la policy requiera comparación exacta.

## DB-JSON-062
Whole replacement será distinto de patch.

## DB-JSON-063
JsonPatch será semantic, no SQL.

## DB-JSON-064
Patch operation ordering será significativo.

## DB-JSON-065
Patch normalization preservará equivalencia.

## DB-JSON-066
SET JSON null será distinto de REMOVE.

## DB-JSON-067
SQL NULL update será distinto de JSON null update.

## DB-JSON-068
Hidden read-modify-write no ocurrirá sin policy/planning explícito.

## DB-JSON-069
Partial update capability no eliminará concurrency concerns.

## DB-JSON-070
JSON mutation integrará optimistic locking mediante owner semantics.

## DB-JSON-071
Containment será una logical capability.

## DB-JSON-072
Indexing será capability-driven.

## DB-JSON-073
JSON logical index request será distinto de physical index implementation.

## DB-JSON-074
Generated columns serán implementation detail de schema strategy.

## DB-JSON-075
JSON keys conservarán case sensitivity.

## DB-JSON-076
Naming conventions no modificarán JSON keys automáticamente.

## DB-JSON-077
String comparison differences entre plataformas serán declaradas.

## DB-JSON-078
Portability limitations serán explícitas.

## DB-JSON-079
JSON type validation será distinta de domain validation.

## DB-JSON-080
JSON document shape schema será distinto de database schema.

## DB-JSON-081
Resource depth será limitada.

## DB-JSON-082
Document bytes serán limitables.

## DB-JSON-083
Object members serán limitables.

## DB-JSON-084
Array items serán limitables.

## DB-JSON-085
Path depth será limitable.

## DB-JSON-086
Patch operation count será limitable.

## DB-JSON-087
Resource limits serán distintos de domain validation limits.

## DB-JSON-088
Sensitive JSON no aparecerá en telemetry por default.

## DB-JSON-089
Raw JSON no será metric label.

## DB-JSON-090
Large documents no se duplicarán innecesariamente.

## DB-JSON-091
V1 podrá utilizar bounded in-memory processing.

## DB-JSON-092
Canonicalization no ocurrirá innecesariamente por property access.

## DB-JSON-093
Tracked mutation state será scoped.

## DB-JSON-094
Compiled metadata será shareable en persistent runtimes.

## DB-JSON-095
Decoder mutable state no será static/global.

## DB-JSON-096
Coroutine-local isolation será obligatoria cuando corresponda.

## DB-JSON-097
Type policy changes invalidarán fingerprints/plans.

## DB-JSON-098
Cache serialization preservará SQL NULL vs JSON null.

## DB-JSON-099
Cache serialization preservará empty object vs empty array.

## DB-JSON-100
JSON query cache fingerprints serán structural cuando corresponda.

## DB-JSON-101
No habrá schema introspection en JSON hot paths.

## DB-JSON-102
No habrá repeated capability discovery en hot paths.

## DB-JSON-103
No habrá SQL generation en JsonCodec.

## DB-JSON-104
No habrá connection access en JsonCanonicalizer.

## DB-JSON-105
No habrá EntityManager access en JsonType.

## DB-JSON-106
No habrá transaction management en JSON conversion.

## DB-JSON-107
No habrá authorization decisions en JSON Type System.

## DB-JSON-108
No habrá API serialization policy implícita en DB JSON mapping.

## DB-JSON-109
JSON object structural equality será deterministic.

## DB-JSON-110
JSON array structural equality preservará position.

## DB-JSON-111
Canonical number handling será policy-aware.

## DB-JSON-112
Large integer precision deberá preservarse bajo LOSSLESS.

## DB-JSON-113
Decimal precision deberá preservarse bajo DECIMAL_AWARE/LOSSLESS.

## DB-JSON-114
JsonMissing nunca será persistido como un JSON scalar.

## DB-JSON-115
JsonMissing será únicamente estado semántico de path/field absence.

## DB-JSON-116
JsonNull sí será un valor JSON persistible.

## DB-JSON-117
SQL NULL será gobernado por column/property nullability.

## DB-JSON-118
Root constraints serán validados.

## DB-JSON-119
Object-only field rechazará array root.

## DB-JSON-120
Array-only field rechazará object root.

## DB-JSON-121
Physical database normalization no redefinirá logical JSON semantics silenciosamente.

## DB-JSON-122
Schema diff preservará storage strategy intent.

## DB-JSON-123
Migration planner tratará representation changes como potencialmente data-affecting.

## DB-JSON-124
JSON path compilation será cacheable.

## DB-JSON-125
Compiled path cache no almacenará request mutable state.

## DB-JSON-126
JSON patch planner será independiente del SQL dialect.

## DB-JSON-127
Native partial mutation será seleccionada por capabilities.

## DB-JSON-128
Emulation deberá declarar sus concurrency implications.

## DB-JSON-129
JSON indexes deberán declarar extracted type cuando sea necesario.

## DB-JSON-130
Unknown extracted type no se convertirá en arbitrary index type.

## DB-JSON-131
Round-trip tests serán obligatorios.

## DB-JSON-132
Cross-platform conformance será obligatorio.

## DB-JSON-133
Null semantics tendrán dedicated conformance tests.

## DB-JSON-134
Precision boundaries tendrán dedicated tests.

## DB-JSON-135
Unicode boundaries tendrán dedicated tests.

## DB-JSON-136
Resource limits tendrán dedicated tests.

## DB-JSON-137
Persistent runtime isolation tendrá dedicated tests.

## DB-JSON-138
JSON mapping será deterministic.

## DB-JSON-139
JSON operations serán explainable.

## DB-JSON-140
VoltStack preservará estructura JSON independientemente de su representación física.

---

# 262. Anti-patterns

## 262.1 JSON como string

```php
private string $settings;
```

y confiar en que "contiene JSON".

**Rechazado como arquitectura tipada.**

---

# 263. SQL NULL = JSON null

```text
NULL
==
'null'
```

**Rechazado.**

---

# 264. Missing = null

```text
key absent
==
key present with null
```

**Rechazado.**

---

# 265. `{}` = `[]`

**Rechazado.**

---

# 266. Dirty tracking textual

```php
$oldJson !== $newJson
```

**Rechazado como default.**

---

# 267. PHP float para todo número JSON

**Rechazado.**

Puede perder precisión.

---

# 268. Vendor SQL en Model

```php
->whereRaw("JSON_EXTRACT(...)")
```

como API central del sistema.

**Rechazado.**

Raw SQL seguirá siendo escape hatch, no arquitectura JSON.

---

# 269. JsonPath concatenado

```php
"JSON_EXTRACT(data, '$." . $userInput . "')"
```

**Rechazado.**

---

# 270. Arbitrary object encoding

```php
json_encode($entity);
```

para persistir una columna JSON.

**Rechazado.**

---

# 271. Silent invalid JSON recovery

```text
invalid JSON
→ []
```

**Rechazado.**

---

# 272. Hidden full-document rewrite

Una operación conceptual:

```text
set $.theme
```

no deberá convertirse silenciosamente en:

```text
SELECT document
→ modify PHP array
→ UPDATE whole document
```

sin que el planner/policy conozca las implicaciones.

---

# 273. Ejemplo — Settings

Entidad:

```php
final class User
{
    #[Column(type: 'json')]
    private array $settings = [];
}
```

Mapping:

```text
User.settings
    ↓
JSON OBJECT
    ↓
STRUCTURAL equality
    ↓
SNAPSHOT_COMPARE
```

---

# 274. Persistence example

Aplicación:

```php
$user->settings = [
    'theme' => 'dark',
    'language' => 'es',
];
```

Pipeline:

```text
PHP array
    ↓
JsonCast
    ↓
JsonObject
    ↓
JsonCanonicalization
    ↓
Value Conversion
    ↓
Platform Transport
    ↓
Prepared Statement
```

---

# 275. Hydration example

DB:

```json
{"language":"es","theme":"dark"}
```

Pipeline:

```text
DB transport value
    ↓
JSON Decoder
    ↓
JsonObject
    ↓
JsonCast
    ↓
PHP associative array
```

---

# 276. Dirty tracking example

Snapshot:

```json
{
  "theme": "dark",
  "language": "es"
}
```

Application value:

```php
[
    'language' => 'es',
    'theme' => 'dark',
]
```

Canonical structural comparison:

```text
EQUIVALENT
```

Resultado:

```text
NO UPDATE
```

---

# 277. JSON null example

Documento:

```json
{
  "avatar": null
}
```

Path:

```text
$.avatar
```

Resultado:

```text
PRESENT
+
JSON_NULL
```

---

# 278. Missing example

Documento:

```json
{}
```

Path:

```text
$.avatar
```

Resultado:

```text
MISSING
```

---

# 279. SQL NULL example

Row:

```text
settings = SQL NULL
```

Resultado:

```text
NO JSON DOCUMENT
```

---

# 280. Four-state comparison

```text
settings = SQL NULL
    → SQL_NULL_DOCUMENT

settings = 'null'
    → JSON_NULL

settings = '{}'
path $.avatar
    → MISSING

settings = '{"avatar":"x"}'
path $.avatar
    → PRESENT_VALUE
```

---

# 281. Patch example

```php
$patch = JsonPatch::make()
    ->set(
        JsonPath::key('theme'),
        'dark'
    )
    ->remove(
        JsonPath::key('legacy')
    );
```

Semantic representation:

```text
JsonPatch
├── SET
│   ├── path: ["theme"]
│   └── value: "dark"
└── REMOVE
    └── path: ["legacy"]
```

No SQL todavía.

---

# 282. Planner

```text
JsonPatch
    ↓
Platform Capability Analysis
    │
    ├── Native Partial Mutation
    │
    ├── Safe Emulation
    │
    └── Unsupported
    ↓
Physical Query Plan
```

---

# 283. Value Object JSON example

```php
final readonly class Preferences
{
    public function __construct(
        public Theme $theme,
        public bool $notifications,
    ) {}
}
```

Persistent JSON:

```json
{
  "theme": "dark",
  "notifications": true
}
```

Pipeline:

```text
Preferences
    ↓
ValueObjectDecomposer
    ↓
{
    theme: Theme::Dark,
    notifications: true
}
    ↓
Enum Mapping
    ↓
{
    theme: "dark",
    notifications: true
}
    ↓
JSON Type System
```

---

# 284. Master value model

```text
                    JSON VALUE
                        │
       ┌────────────────┼────────────────┐
       │                │                │
    OBJECT            ARRAY           SCALAR
       │                │                │
       │                │       ┌────────┼─────────┐
       │                │       │        │         │
       │                │     STRING   NUMBER   BOOLEAN
       │                │                  │
       │                │               JSON NULL
       │                │
       └────────────────┴──────────────────┘

Outside JSON value domain:

SQL NULL
MISSING PATH
```

---

# 285. Master persistence model

```text
Application Value
       │
       ▼
Casting / Value Object Mapping
       │
       ▼
Canonical JSON Value
       │
       ▼
JSON Validation
       │
       ▼
Value Conversion
       │
       ▼
Platform Physical Mapping
       │
       ▼
Prepared Binding
       │
       ▼
Database
```

---

# 286. Master query model

```text
JsonPath
+
JsonOperation
+
Typed JsonValue
        │
        ▼
Semantic Query AST
        │
        ▼
Optimizer
        │
        ▼
Planner
        │
        ▼
Platform Capability Resolution
        │
        ▼
SQL Compiler
```

---

# 287. Master null formula

```text
SQL_NULL
≠
JSON_NULL
≠
MISSING
```

y:

```text
JSON_OBJECT({})
≠
JSON_ARRAY([])
```

---

# 288. Master equality formula

Para structural policy:

```text
JsonEquivalent(A, B)
iff

Kind(A) = Kind(B)

AND

StructuralContent(A)
≡
StructuralContent(B)
```

donde:

```text
Object:
    member order ignored

Array:
    element order preserved

Number:
    compared according to JsonNumberPolicy
```

---

# 289. Master platform formula

```text
LogicalJsonType
+
JsonOperation
+
PlatformCapabilities
+
MappingPolicy

→

PhysicalJsonRepresentation
+
ExecutionStrategy
```

---

# 290. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
JSON as Logical Structured Type
```

y no como simple string.

Adoptará:

```text
SQL NULL
≠
JSON null
≠
MISSING
```

como invariante fundamental.

Adoptará:

```text
Typed Logical JsonPath
```

independiente de SQL.

Adoptará:

```text
Structural Equality
```

como estrategia recomendada para dirty tracking.

Adoptará:

```text
Lossless Numeric Handling
```

en fronteras donde exista riesgo de precisión.

Adoptará:

```text
Capability-driven Platform Mapping
```

para MySQL, MariaDB, PostgreSQL y SQLite.

Adoptará:

```text
Semantic JsonPatch
```

independiente de funciones SQL específicas.

Adoptará:

```text
Bounded Resource Processing
```

para proteger workers persistentes.

Y mantendrá:

```text
JSON Logical Semantics
≠
PHP Representation
≠
Transport Representation
≠
Physical Database Representation
```

---

# 291. Regla maestra final

> **El JSON Type System de VoltStack deberá preservar la estructura y semántica del documento desde la aplicación hasta la base de datos sin permitir que las peculiaridades de PHP, del driver o del dialecto SQL redefinan silenciosamente su significado.**

La arquitectura deberá garantizar que:

```text
null
{}
[]
missing
0
false
""
```

continúen siendo valores/estados diferentes cuando el dominio JSON los diferencia.

Asimismo:

```text
MySQL JSON
MariaDB JSON representation
PostgreSQL JSON
PostgreSQL JSONB
SQLite JSON-capable storage
```

serán únicamente implementaciones físicas de:

```text
VoltStack Logical JSON Type
```

y no cinco modelos de programación diferentes para el desarrollador.

---

# 292. Siguiente documento

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura temporal completa de VoltStack Database, incluyendo:

- Date;
- Time;
- LocalDateTime;
- Instant;
- OffsetDateTime;
- ZonedDateTime;
- Duration;
- timezone;
- UTC normalization;
- timezone database;
- precision;
- fractional seconds;
- Unix timestamps;
- PHP `DateTimeInterface`;
- immutable temporal objects;
- canonical temporal representation;
- SQL `DATE`;
- `TIME`;
- `DATETIME`;
- `TIMESTAMP`;
- PostgreSQL temporal types;
- MySQL/MariaDB differences;
- SQLite temporal representation;
- daylight-saving transitions;
- ambiguous local times;
- nonexistent local times;
- leap-second policy;
- timezone conversion;
- hydration;
- persistence;
- parameter binding;
- comparison;
- ordering;
- arithmetic;
- query expressions;
- schema mapping;
- defaults;
- `CURRENT_TIMESTAMP`;
- generated temporal values;
- optimistic timestamps;
- dirty tracking;
- Value Objects;
- serialization boundaries;
- persistent runtime;
- security;
- telemetry;
- diagnostics;
- testing;
- performance;
- architectural invariants.

Regla central propuesta:

> **VoltStack no tratará fecha, hora, instante y zona horaria como variantes intercambiables de `DateTime`: cada concepto temporal tendrá semántica lógica explícita, y cualquier conversión entre tiempo local, offset, zona e instante deberá ser intencional, determinista y visible para el Type System.**