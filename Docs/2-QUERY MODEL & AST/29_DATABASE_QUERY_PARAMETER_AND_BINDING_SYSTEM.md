# 29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md

# VoltStack Quantum Database
## Query Parameter and Binding System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 29 — Query Parameter and Binding System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Parameters / Bindings  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de parámetros y bindings de:

```text
VoltStack/Quantum/Database
```

Su responsabilidad es separar rigurosamente:

```text
Query Parameter
≠
Runtime Value
≠
Semantic Type
≠
Database Representation
≠
SQL Placeholder
≠
Compiled Binding
≠
Native Driver Binding
```

Esta separación será obligatoria en toda la arquitectura de Database.

---

# 2. Problema arquitectónico

Una implementación simple suele mezclar:

```php
$sql = 'SELECT * FROM users WHERE email = ?';

$statement->execute([$email]);
```

Esto funciona para casos pequeños, pero mezcla implícitamente:

- estructura de consulta;
- posición del parámetro;
- valor;
- tipo;
- conversión;
- placeholder;
- estrategia de expansión;
- binding nativo;
- seguridad;
- redacción;
- cacheabilidad.

VoltStack separará cada una de estas responsabilidades.

---

# 3. Regla maestra

> Un Query Parameter representa una entrada semántica de una consulta; no representa su valor runtime, su placeholder SQL ni la operación nativa utilizada para enviarlo al servidor.

Formalmente:

```text
Parameter
=
Semantic Input Identity
+
Expected Shape
+
Type Information
+
Binding Semantics
```

No:

```text
Parameter
=
$value
```

ni:

```text
Parameter
=
"?"
```

---

# 4. Arquitectura general

```text
Application
    │
    │ values
    ▼
BindingSet
    │
    ├──────────────┐
    │              │
    ▼              ▼
Query AST      Runtime Values
    │              │
    │              │
    ▼              │
Parameter Nodes    │
    │              │
    └───────┬──────┘
            ▼
     Semantic Analysis
            │
            ├── Type Inference
            ├── Shape Validation
            ├── Nullability
            └── Capability Requirements
            │
            ▼
         Planner
            │
            ├── Scalar Binding
            ├── Collection Expansion
            ├── Native Array
            ├── VALUES Relation
            └── Platform Strategy
            │
            ▼
         Compiler
            │
            ├── Placeholder Allocation
            └── Binding Plan
            │
            ▼
       CompiledQuery
            │
            ▼
         Executor
            │
            ▼
      Value Conversion
            │
            ▼
       Driver Binding
            │
            ▼
     Native Statement
```

---

# 5. Separación fundamental

La arquitectura distinguirá al menos:

```text
ParameterDefinition
ParameterExpression
ParameterId
BindingSet
BoundValue
ParameterType
ParameterShape
BindingStrategy
BindingPlan
CompiledParameter
Placeholder
DriverBinding
```

Cada concepto tendrá una responsabilidad diferente.

---

# 6. ParameterId

Todo parámetro tendrá una identidad estable dentro de la consulta.

Conceptualmente:

```php
final readonly class ParameterId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
p1
p2
minimumAge
email
tenantId
expectedVersion
```

---

# 7. ParameterId ≠ placeholder

La identidad:

```text
ParameterId("email")
```

no significa necesariamente:

```text
:email
```

El Compiler podría producir:

```text
?
```

o:

```text
$1
```

o:

```text
:vs_1
```

según Dialect y Driver Binding Profile.

---

# 8. ParameterExpressionNode

Dentro del AST, un parámetro aparecerá como una expresión.

```text
ParameterExpressionNode
├── parameterId
├── shape
└── optional declared type
```

No contendrá normalmente el valor runtime.

---

# 9. Ejemplo

Consulta:

```php
DB::table('users')
    ->where('email', $email);
```

AST:

```text
ComparisonPredicate
├── ColumnReference(users.email)
├── EQUAL
└── ParameterExpression(p1)
```

Bindings:

```text
BindingSet
└── p1 → $email
```

---

# 10. AST independiente de valores

La estructura:

```text
users.email = p1
```

puede reutilizarse con:

```text
p1 = alice@example.com
```

o:

```text
p1 = bob@example.com
```

sin modificar el AST.

---

# 11. Beneficio

Esto permite:

- compiled query cache;
- deterministic fingerprints;
- prepared statement reuse;
- redacción de valores;
- separación de tipos;
- ejecución repetida;
- batch execution;
- persistent runtime safety.

---

# 12. BindingSet

`BindingSet` representa los valores runtime asociados a los ParameterId de una ejecución.

Conceptualmente:

```php
interface BindingSet
{
    public function has(ParameterId $id): bool;

    public function get(ParameterId $id): BoundValue;
}
```

---

# 13. BindingSet scope

`BindingSet` pertenece a:

```text
query execution
```

No a:

```text
application singleton
```

---

# 14. BindingSet lifetime

Normalmente:

```text
Execution Operation
```

o menor.

Nunca deberá sobrevivir accidentalmente entre requests persistentes.

---

# 15. BoundValue

Podrá modelarse:

```text
BoundValue
├── value
├── explicitType?
├── sensitivity
├── provenance?
└── conversionHints?
```

---

# 16. Runtime value separation

El valor:

```text
42
```

no contiene por sí mismo toda la información necesaria para saber si representa:

```text
Integer
UserId
Enum backing value
Unix timestamp
Decimal
JSON scalar
JSON document
Boolean
```

---

# 17. Type System integration

El Parameter System consumirá el Type System definido posteriormente.

Flujo:

```text
Runtime Value
      +
Parameter Semantic Context
      +
Explicit Type Hint
      +
Schema Metadata
      │
      ▼
Resolved Database Type
```

---

# 18. Type resolution precedence

Conceptualmente:

```text
Explicit Parameter Type
        │
        ▼
Semantic Context Inference
        │
        ▼
Mapping Metadata
        │
        ▼
Safe Runtime Inference
        │
        ▼
Fallback / Error
```

---

# 19. Explicit type wins

Si el programador declara:

```text
UuidType
```

VoltStack no deberá cambiarlo silenciosamente a:

```text
StringType
```

sólo porque el runtime value sea un string PHP.

---

# 20. Semantic inference

Ejemplo:

```text
users.id = :p1
```

Si:

```text
users.id → UserIdType
```

entonces:

```text
p1 → UserIdType
```

podrá inferirse.

---

# 21. Bidirectional constraints

La inferencia podrá utilizar restricciones entre operands:

```text
Column(Integer)
=
Parameter(Unknown)
```

produce:

```text
Parameter(Integer)
```

---

# 22. Parameter type inference ≠ PHP type inference

No asumir:

```text
is_string($value)
→ VARCHAR
```

como regla universal.

---

# 23. PHP enums

Ejemplo:

```php
Status::ACTIVE
```

podría mapearse mediante:

```text
EnumType
→ backing value
→ database representation
```

---

# 24. Value objects

Ejemplo:

```php
UserId
Money
EmailAddress
Uuid
```

podrán utilizar tipos registrados.

---

# 25. ParameterShape

VoltStack distinguirá la forma del parámetro.

Conceptualmente:

```text
SCALAR
COLLECTION
TUPLE
TUPLE_COLLECTION
STRUCTURED
PLATFORM_SPECIFIC
```

---

# 26. Scalar parameter

Ejemplo:

```text
age >= :p1
```

```text
p1
shape = SCALAR
```

---

# 27. Collection parameter

Ejemplo:

```php
->whereIn('id', $ids)
```

podrá representarse:

```text
InPredicate
├── Column(id)
└── CollectionParameter(p1)
```

con:

```text
p1
shape = COLLECTION
```

---

# 28. Why collection parameters matter

Sin una abstracción de collection parameter:

```text
[1, 2, 3]
```

se transforma demasiado pronto en:

```text
:p1, :p2, :p3
```

acoplando el AST al tamaño runtime de la colección.

VoltStack evitará ese acoplamiento.

---

# 29. Collection semantic model

```text
CollectionParameter
├── ParameterId
├── elementType
├── cardinality: runtime
└── semantics
```

---

# 30. Collection value remains external

El AST no almacena:

```text
[1, 2, 3]
```

Sólo conoce:

```text
CollectionParameter(p1)
```

---

# 31. Collection binding strategies

El Planner podrá seleccionar:

```text
EXPANDED_PARAMETERS
NATIVE_ARRAY
VALUES_RELATION
TEMPORARY_RELATION
CHUNKED_PREDICATE
PLATFORM_SPECIFIC
```

---

# 32. Strategy ownership

```text
AST
→ semantic collection intent

Planner
→ binding/execution strategy

Compiler
→ placeholder layout

Driver
→ native binding
```

---

# 33. Expanded parameters

Ejemplo:

```text
p1 = [10, 20, 30]
```

puede planearse como:

```text
IN (?, ?, ?)
```

---

# 34. Expansion generates compiled parameters

Conceptualmente:

```text
Semantic Parameter
p1

        │ expansion
        ▼

Compiled Parameters
p1[0]
p1[1]
p1[2]
```

---

# 35. Semantic identity preserved

Aunque existan tres compiled parameters, todos podrán conservar:

```text
sourceParameterId = p1
```

para diagnostics y binding.

---

# 36. Native array strategy

En targets apropiados:

```text
CollectionParameter
```

podría enviarse como una sola representación nativa.

Esto requiere:

```text
Platform Capability
+
Driver Capability
+
Type Support
```

---

# 37. No fake native arrays

Que PostgreSQL soporte arrays no significa automáticamente que:

```text
PDO pgsql
```

pueda bindear cualquier PHP array directamente.

Driver y Platform capabilities deberán combinarse.

---

# 38. VALUES relation

Una colección grande podría convertirse conceptualmente en:

```text
VALUES relation
```

y utilizarse como input relacional.

---

# 39. Temporary relation

Para colecciones extremadamente grandes, una estrategia futura podría utilizar:

```text
temporary table
```

o equivalente.

Esto implica:

- transaction semantics;
- connection affinity;
- cleanup;
- session state;
- pooling implications.

Por ello pertenece al Planner/Execution infrastructure.

---

# 40. Empty collection

`CollectionParameter` con cardinalidad cero no deberá llegar ingenuamente a:

```text
IN ()
```

La semántica se resolverá antes.

---

# 41. Collection cardinality

El sistema podrá clasificar:

```text
EMPTY
SINGLE
SMALL
MEDIUM
LARGE
VERY_LARGE
UNKNOWN
```

para selección de estrategia.

---

# 42. No hardcoded universal threshold

Los límites podrán depender de:

- platform;
- driver;
- max parameters;
- query size;
- policy;
- configured limits;
- workload.

---

# 43. Parameter limits

Las plataformas pueden imponer límites sobre:

```text
number of bind parameters
statement size
packet size
expression depth
```

El Planner deberá consultarlos mediante capabilities.

---

# 44. Tuple parameter

Ejemplo conceptual:

```text
(a, b) = :tuple
```

donde:

```text
tuple
=
(valueA, valueB)
```

podrá representarse mediante `TupleParameter`.

---

# 45. Tuple collection

Ejemplo:

```text
(a, b) IN ((1,2), (3,4))
```

podrá utilizar:

```text
TupleCollectionParameter
```

---

# 46. Arity

Tuple bindings deberán validar:

```text
expected arity
=
runtime arity
```

---

# 47. Structured parameters

Algunas features futuras podrán requerir parámetros estructurados.

No deberán introducirse como arrays PHP ambiguos.

---

# 48. ParameterDefinition

Podrá existir una descripción inmutable:

```text
ParameterDefinition
├── id
├── shape
├── declaredType?
├── nullable
├── sensitivity
├── expansionPolicy
└── metadata
```

---

# 49. ParameterRegistry

Durante Query construction podrá existir una registry local de parámetros.

No será global.

---

# 50. ParameterRegistry scope

```text
Query Construction Context
```

---

# 51. Parameter allocation

El Builder podrá asignar:

```text
p1
p2
p3
```

determinísticamente.

---

# 52. Named parameters

También podrá permitir:

```php
->where('tenant_id', '=', parameter('tenantId'))
```

para APIs avanzadas.

---

# 53. Named semantic parameters

Un parámetro llamado:

```text
tenantId
```

sigue sin implicar placeholder:

```text
:tenantId
```

---

# 54. Duplicate semantic parameter

Una consulta podrá utilizar el mismo ParameterId varias veces:

```text
created_at >= :date
OR
updated_at >= :date
```

semánticamente:

```text
ParameterId(date)
```

es uno.

---

# 55. Driver placeholder limitation

Algunos drivers/modos pueden requerir placeholders físicos separados.

Entonces:

```text
ParameterId(date)
```

podrá producir:

```text
CompiledParameter(date#1)
CompiledParameter(date#2)
```

sin duplicar el valor semántico.

---

# 56. PlaceholderAllocator

El Compiler utilizará:

```text
PlaceholderAllocator
```

para crear placeholders físicos.

---

# 57. Placeholder styles

Conceptualmente:

```text
POSITIONAL_QUESTION
NUMBERED_POSITIONAL
NAMED
DRIVER_DEFINED
```

---

# 58. Examples

```text
?
```

```text
$1
$2
```

```text
:vs_1
:vs_2
```

---

# 59. Placeholder strategy ownership

El estilo final depende de:

```text
Dialect
+
Driver Binding Profile
+
Compiler Strategy
```

No del AST.

---

# 60. BindingProfile

Podrá existir:

```text
SqlBindingProfile
```

que describa necesidades del target de compilación.

Ejemplo:

```text
placeholderStyle
supportsRepeatedNamedParameter
supportsNativeArrayBinding
supportsLobStreaming
booleanBindingMode
```

---

# 61. Dialect vs Driver

El Dialect puede conocer la forma sintáctica aceptada.

El Driver conoce cómo realizar el binding nativo.

Ninguno deberá asumir automáticamente las capacidades del otro.

---

# 62. CompiledParameter

Después de Planning/Compilation:

```text
CompiledParameter
├── compiledId
├── sourceParameterId
├── placeholder
├── semanticType
├── databaseType
├── bindingMode
├── sourcePath?
├── sensitivity
└── position
```

---

# 63. Compiled parameter example

```text
CompiledParameter
├── id: cp1
├── source: p1
├── placeholder: ?
├── semanticType: IntegerType
├── bindingMode: SCALAR
└── position: 1
```

---

# 64. Expanded collection example

```text
p1 = [10,20,30]
```

Binding plan:

```text
cp1 → p1[0] → ?
cp2 → p1[1] → ?
cp3 → p1[2] → ?
```

---

# 65. BindingPlan

El resultado de la planificación de bindings podrá ser:

```text
BindingPlan
├── parameterPlans
├── expansionPlans
├── placeholderRequirements
├── conversionRequirements
├── capabilityRequirements
└── shapeFingerprint
```

---

# 66. BindingPlan ≠ BindingSet

```text
BindingPlan
=
how values will be bound

BindingSet
=
which runtime values are supplied
```

---

# 67. BindingPlan immutability

Será inmutable una vez publicado.

---

# 68. BindingPlan cacheability

Podrá cachearse cuando dependa únicamente de:

```text
Query Shape
Type Resolution
Dialect
Driver Binding Profile
Capabilities
Compiler Version
```

---

# 69. Runtime cardinality caveat

Si la estrategia depende del tamaño de una colección runtime, podrán existir:

```text
BindingPlanTemplate
```

y:

```text
ResolvedBindingPlan
```

---

# 70. Two-stage binding planning

Conceptualmente:

```text
Static Query
    │
    ▼
BindingPlanTemplate
    │
    + runtime binding shape
    ▼
ResolvedBindingPlan
```

---

# 71. Why two stages

Permite reutilizar:

```text
id IN collection(p1)
```

con colecciones de tamaños diferentes.

---

# 72. BindingShape

El sistema podrá derivar:

```text
BindingShape
```

sin incluir valores.

Ejemplo:

```text
p1: scalar integer
p2: collection<string>[3]
```

---

# 73. Shape fingerprint

```text
BindingShapeFingerprint
```

podrá incluir:

- parameter shapes;
- collection cardinalities cuando afecten SQL;
- resolved types;
- expansion strategies.

Nunca valores sensibles.

---

# 74. Query fingerprint hierarchy

VoltStack deberá distinguir:

```text
SemanticQueryFingerprint
CompiledQueryFingerprint
BindingShapeFingerprint
ExecutionFingerprint
```

---

# 75. Semantic query fingerprint

Representa la estructura/intención de la consulta.

No contiene valores.

---

# 76. Compiled query fingerprint

Puede depender de:

```text
Query Fingerprint
Dialect Fingerprint
Capability Fingerprint
Compiler Version
Binding Profile
Binding Shape
```

---

# 77. Execution fingerprint

Si se necesita para tracing interno, nunca deberá incluir valores secretos directamente.

---

# 78. Binding validation

Antes de ejecutar deberá verificarse:

```text
required parameter exists
shape matches
type compatible
nullability valid
collection constraints valid
tuple arity valid
conversion possible
```

---

# 79. Missing binding

Ejemplo:

```text
AST requires p1
BindingSet has no p1
```

deberá producir:

```text
MissingQueryParameterException
```

antes de invocar al Driver.

---

# 80. Extra bindings

Política recomendada:

```text
STRICT
```

por defecto.

Un binding no utilizado podrá producir error o warning según API explícita.

---

# 81. Why reject extras

Ayuda a detectar:

- typos;
- stale code;
- wrong query reuse;
- accidental sensitive values;
- incorrect named parameters.

---

# 82. Nullability

El Parameter System deberá distinguir:

```text
parameter may accept NULL
```

de:

```text
predicate semantics when NULL is supplied
```

---

# 83. Example

```text
age >= :p1
```

aunque `p1` técnicamente acepte NULL a nivel de binding, la consulta puede producir `UNKNOWN`.

Eso pertenece a Predicate Semantic Analysis.

---

# 84. Null binding

El binding de `NULL` deberá conservar el tipo semántico esperado cuando sea conocido.

---

# 85. Typed NULL

Conceptualmente:

```text
BoundValue
├── value: null
└── type: UuidType
```

es diferente de un NULL completamente sin contexto.

---

# 86. Ambiguous NULL

Si no existe contexto suficiente:

```text
NULL
+
Unknown Type
```

puede requerir:

- explicit type;
- platform fallback;
- error.

---

# 87. Value conversion pipeline

```text
PHP / Domain Value
       │
       ▼
Semantic Type
       │
       ▼
Database Type Mapping
       │
       ▼
Database Representation
       │
       ▼
Driver Binding Value
       │
       ▼
Native Client
```

---

# 88. Example — UUID

```text
Uuid Object
   │
   ▼
UuidType
   │
   ▼
Platform Mapping
   │
   ├── native UUID
   └── string/binary representation
   │
   ▼
Driver Binding
```

---

# 89. Example — enum

```text
Status::ACTIVE
      │
      ▼
EnumType
      │
      ▼
"active"
      │
      ▼
Driver binding
```

---

# 90. Example — DateTime

```text
DateTimeImmutable
      │
      ▼
DateTimeType
      │
      ▼
Normalized DB representation
      │
      ▼
Driver binding
```

La conversión no dependerá de locale accidental.

---

# 91. Example — decimal

```text
Money / Decimal
      │
      ▼
DecimalType
      │
      ▼
Exact decimal representation
```

No convertir automáticamente a float si puede perder precisión.

---

# 92. Example — JSON

```text
Domain value
    │
    ▼
JsonType
    │
    ▼
JSON Encoder
    │
    ▼
Database Representation
    │
    ▼
Driver Binding
```

---

# 93. JSON errors

Errores de serialización deberán ocurrir antes de enviar SQL cuando sea posible.

---

# 94. SQL NULL vs JSON null

El Binding System deberá poder preservar la diferencia entre:

```text
SQL NULL
```

y:

```text
JSON "null"
```

---

# 95. Boolean binding

La representación física de:

```text
true
false
```

puede variar.

El Query AST conserva `BooleanType`.

La conversión final utiliza Platform/Driver mapping.

---

# 96. Binary data

Los parámetros binarios deberán usar un tipo explícito.

No asumir que todo PHP string es texto.

---

# 97. LOB values

Podrán requerir:

```text
LOB
BINARY_LOB
CHARACTER_LOB
STREAM
```

---

# 98. Stream binding

Un stream no deberá materializarse automáticamente en memoria si el Driver soporta streaming seguro.

---

# 99. Driver capability

Ejemplo:

```text
driver.binding.lob_stream
```

---

# 100. Stream ownership

El sistema deberá definir:

```text
who opens
who reads
who closes
when it can be retried
whether it is rewindable
```

---

# 101. Retry implications

Un stream consumido puede hacer que una query deje de ser automáticamente retryable.

---

# 102. Binding replayability

Podrá existir:

```text
BindingReplayability
├── REPLAYABLE
├── REWINDABLE
├── SINGLE_USE
└── UNKNOWN
```

---

# 103. Resilience integration

El Retry System deberá consultar la replayability de los bindings antes de repetir una operación.

---

# 104. No retry ownership

El Parameter System describe replayability.

No ejecuta retries.

---

# 105. Sensitive parameters

Cada parameter/bound value podrá clasificarse:

```text
PUBLIC
NORMAL
SENSITIVE
SECRET
CREDENTIAL
TOKEN
PERSONAL_DATA
CUSTOM
```

según el modelo final de seguridad.

---

# 106. Default safety

Los valores de parámetros no deberán aparecer en:

- exceptions;
- logs;
- telemetry labels;
- profiler output;
- debug toolbar;
- traces;

por defecto.

---

# 107. Redacted rendering

Ejemplo:

```text
p1 = [REDACTED]
```

o:

```text
p1
type: String
length: 32
value: hidden
```

---

# 108. Debug opt-in

Un entorno local podrá permitir visualización limitada de valores bajo política explícita.

---

# 109. Never interpolate for debugging

No construir:

```text
SELECT ...
WHERE password = 'secret'
```

para mostrar una query “completa”.

---

# 110. Query diagnostics

Mostrar:

```text
SQL:
SELECT ... WHERE email = ?

Bindings:
1:
  source: p1
  type: EmailType
  value: [REDACTED]
```

---

# 111. Parameter provenance

Podrá registrarse:

```text
USER_INPUT
APPLICATION
ORM
TENANT
AUTHORIZATION
SYSTEM
GENERATED
MIGRATION
EXTENSION
```

---

# 112. Provenance ≠ trust

Que un valor sea:

```text
SYSTEM
```

no significa que deba interpolarse.

Todos los valores continúan usando binding normal cuando sea posible.

---

# 113. Identifier ≠ parameter

No deberá permitirse:

```text
SELECT * FROM ?
```

esperando bindear un nombre de tabla.

---

# 114. Identifier pipeline

```text
Dynamic Identifier
      │
      ▼
Identifier Value Object
      │
      ▼
Validation
      │
      ▼
Semantic Resolution
      │
      ▼
Dialect Quoting
```

---

# 115. Parameter pipeline

```text
Dynamic Value
      │
      ▼
Parameter
      │
      ▼
Binding
```

Son pipelines diferentes.

---

# 116. SQL keywords ≠ parameters

Tampoco:

```text
ORDER BY age ?
```

con:

```text
ASC
```

como value binding ordinario.

---

# 117. Structural values

Dirección de orden, operadores, identifiers, keywords y syntax options deberán modelarse estructuralmente.

---

# 118. Raw SQL parameters

`RawExpression` y `RawPredicate` deberán seguir utilizando:

```text
bindings
```

para valores.

---

# 119. Raw SQL example

Permitido:

```php
raw(
    'LOWER(email) = ?',
    [$email],
)
```

conceptualmente convertido a un modelo seguro de raw fragment + parameters.

---

# 120. Unsafe raw SQL

No recomendado:

```php
raw("LOWER(email) = '$email'")
```

---

# 121. Raw parameter registration

Los parámetros raw deberán integrarse al mismo:

```text
Parameter Registry
BindingSet
BindingPlan
CompiledQuery
Driver Binding
```

cuando sea posible.

---

# 122. Parameter namespace

Subqueries requieren evitar colisiones.

Ejemplo:

```text
Outer:
p1

Subquery:
p1
```

no deberá causar ambigüedad.

---

# 123. ParameterScope

Podrá existir:

```text
ParameterScope
```

con identidad estructural.

---

# 124. Scoped identity

Internamente:

```text
query0:p1
query0.subquery1:p1
```

podrán ser diferentes aunque sus nombres locales coincidan.

---

# 125. User-facing names

La API podrá mantener nombres amigables mientras el AST usa IDs canónicos.

---

# 126. Parameter canonicalization

Durante normalization podrán convertirse a:

```text
p1
p2
p3
...
```

para fingerprints deterministas.

---

# 127. Original name metadata

Podrá conservarse sólo para diagnostics.

---

# 128. Parameter reuse across subqueries

Si el usuario desea compartir explícitamente el mismo parámetro:

```text
tenantId
```

entre outer query y subquery, deberá existir una forma explícita.

---

# 129. No accidental capture

Subqueries no deberán capturar parámetros por nombre accidentalmente.

---

# 130. Parameter scope ≠ symbol scope

Aunque relacionados, deberán distinguirse:

```text
Symbol Scope
```

para columnas/tables/aliases,

de:

```text
Parameter Scope
```

para inputs de consulta.

---

# 131. Query template

La separación AST/Binding permite:

```text
QueryTemplate
```

---

# 132. QueryTemplate concept

```text
QueryTemplate
├── Query AST
├── Parameter Definitions
├── Semantic Metadata
└── Binding Requirements
```

---

# 133. Execution

```text
QueryTemplate
+
BindingSet
→ Query Execution
```

---

# 134. Benefits

Permite:

- repeated execution;
- prepared operations;
- batch operations;
- cached compilation;
- repositories reutilizables.

---

# 135. QueryTemplate immutability

Un template publicado deberá ser inmutable.

---

# 136. No bound template singleton

No almacenar:

```text
QueryTemplate + current user values
```

en un singleton persistente.

---

# 137. PreparedQuery concept

Podrá existir una API futura:

```text
PreparedQuery
```

pero no deberá confundirse con native prepared statement.

---

# 138. PreparedQuery ≠ NativePreparedStatement

```text
PreparedQuery
=
framework-level reusable query plan/template

NativePreparedStatement
=
driver/native server resource
```

---

# 139. Different lifetimes

```text
PreparedQuery
→ potentially application-safe immutable

NativePreparedStatement
→ physical connection scoped
```

---

# 140. Critical pooling rule

Un native prepared statement pertenece a una physical connection.

Nunca deberá reutilizarse en otra conexión como si fuera portable.

---

# 141. Statement cache

Un futuro prepared statement cache deberá estar asociado a:

```text
PhysicalConnection
+
CompiledQueryFingerprint
+
Connection Generation
```

---

# 142. Statement cache reset

El reset del Connection State deberá decidir si esos statements pueden sobrevivir.

---

# 143. Parameter type metadata

Cada parameter podrá tener:

```text
Declared Type
Inferred Type
Resolved Type
Database Type
Driver Binding Type
```

---

# 144. Type stages

```text
DeclaredType
      │
      ▼
SemanticType
      │
      ▼
PlatformDatabaseType
      │
      ▼
DriverBindingType
```

---

# 145. No premature driver type

El Query Builder nunca deberá elegir:

```text
PDO::PARAM_INT
```

---

# 146. Correct

```text
IntegerType
```

se mantiene hasta la frontera apropiada.

---

# 147. PDO mapping

Sólo el adapter PDO decidirá finalmente si utiliza:

```text
PDO::PARAM_INT
PDO::PARAM_STR
PDO::PARAM_BOOL
PDO::PARAM_LOB
PDO::PARAM_NULL
```

u otra estrategia.

---

# 148. PDO is implementation detail

Ninguna API superior deberá exponer constantes PDO como contrato core.

---

# 149. DriverBindingType

Podrá existir una abstracción neutral:

```text
INTEGER
STRING
BOOLEAN
BINARY
LOB
NULL
DRIVER_NATIVE
```

---

# 150. Native type hints

Un driver avanzado podrá utilizar metadata adicional.

Debe mantenerse encapsulada en su adapter.

---

# 151. Conversion ownership

El Type System convierte:

```text
Domain Value
→ Database Representation
```

El Driver Binding adapter convierte:

```text
Database Representation
→ Native Client Binding
```

---

# 152. Driver does not perform domain mapping

El Driver no deberá conocer:

```text
Money
UserId
Domain Enum
Entity
Value Object
```

---

# 153. Entity binding

No deberá permitirse bindear directamente una Entity como parámetro genérico.

---

# 154. ORM convenience

El ORM puede convertir:

```text
User entity
```

a:

```text
User identifier
```

antes de llegar al Query Parameter System cuando metadata lo permita.

---

# 155. Entity composite IDs

Para IDs compuestos, el ORM podrá producir:

```text
TupleParameter
```

o predicates múltiples.

---

# 156. Binding errors

Jerarquía conceptual:

```text
QueryBindingException
├── MissingQueryParameterException
├── UnexpectedQueryParameterException
├── InvalidParameterShapeException
├── InvalidParameterTypeException
├── ParameterTypeInferenceException
├── ParameterConversionException
├── ParameterExpansionException
├── ParameterLimitExceededException
├── InvalidTupleBindingException
├── InvalidCollectionBindingException
├── NonReplayableBindingException
└── DriverBindingException
```

---

# 157. Error boundary

Debe distinguirse:

```text
semantic parameter error
binding plan error
value conversion error
driver binding error
native execution error
```

---

# 158. Conversion before acquisition

Siempre que sea razonable, conversiones que puedan fallar deberán realizarse antes de adquirir una physical connection.

---

# 159. Benefit

Evita retener recursos mientras:

- JSON falla al serializar;
- enum es inválido;
- UUID no parsea;
- collection shape es incorrecta.

---

# 160. Caveat

Algunas conversiones pueden depender de:

```text
Resolved Platform
```

y por tanto de información obtenida después de resolver el target.

Aun así no necesitan necesariamente adquirir una nueva conexión si el target ya está resuelto/cacheado.

---

# 161. Execution preparation pipeline

```text
Query
+
BindingSet
      │
      ▼
Binding Validation
      │
      ▼
Type Resolution
      │
      ▼
Strategy Resolution
      │
      ▼
Value Conversion
      │
      ▼
Resolved Binding Plan
      │
      ▼
Acquire Connection
      │
      ▼
Prepare Statement
      │
      ▼
Driver Bind
      │
      ▼
Execute
```

---

# 162. Planning order caveat

La arquitectura final podrá intercalar algunas fases cuando:

- capabilities dependan del resolved connection;
- statement preparation afecte binding;
- native array support sea driver-specific.

Pero las responsabilidades permanecerán separadas.

---

# 163. Parameter normalization

La normalization podrá:

- canonicalizar IDs;
- validar shape;
- mergear referencias equivalentes;
- ordenar metadata;
- normalizar type hints.

No tocará runtime values.

---

# 164. Binding normalization

Un `BindingSet` podrá normalizar:

- iterable → collection abstraction;
- enum → preserve semantic object until Type conversion;
- scalar wrappers;
- sensitivity metadata.

---

# 165. Do not consume generators early

Un iterable podría ser:

```text
Generator
```

y consumirlo puede ser observable.

Por ello collection normalization deberá conocer replayability.

---

# 166. CollectionSource

Podrá modelarse:

```text
CollectionSource
├── MaterializedCollection
├── ReplayableIterable
├── SinglePassIterable
└── LazyCollection
```

---

# 167. Collection planning implications

Una estrategia que necesita conocer cardinalidad antes de ejecución puede requerir materialización.

---

# 168. Resource governance

Materializar colecciones deberá respetar:

```text
memory limits
max collection size
max parameter count
```

---

# 169. Huge collections

Una colección demasiado grande no deberá provocar automáticamente millones de placeholders.

---

# 170. Planner response

Podrá:

- elegir otra estrategia;
- chunk;
- use temporary relation;
- reject with diagnostic.

---

# 171. Chunking semantics

Dividir una query no siempre preserva:

- ordering;
- limit;
- offset;
- transaction semantics;
- locking;
- aggregation.

Por tanto chunking no será una solución genérica automática.

---

# 172. Parameter expansion phase

VoltStack deberá evitar realizar expansión irreversible demasiado pronto.

Preferencia:

```text
Semantic Collection Parameter
        │
        ▼
Planner Strategy
        │
        ▼
Resolved Binding Shape
        │
        ▼
Compiler Expansion
```

---

# 173. Placeholder allocation order

Deberá ser determinista.

Ejemplo:

```text
AST traversal order
+
planned binding order
→ stable placeholders
```

---

# 174. Why deterministic

Ayuda a:

- cache;
- testing;
- debugging;
- statement reuse;
- fingerprints.

---

# 175. Named placeholder generation

Los nombres generados deberán ser:

- válidos;
- deterministas;
- libres de colisiones;
- no sensibles.

---

# 176. Do not derive from secret values

Nunca:

```text
:email_alice_example_com
```

---

# 177. Positional bindings

Para:

```text
?
```

el BindingPlan deberá mantener correspondencia exacta:

```text
position 1 → p1
position 2 → p2
```

---

# 178. Repeated parameters

Si:

```text
p1
```

aparece tres veces y el driver usa positional placeholders:

```text
position 1 → p1
position 2 → p1
position 3 → p1
```

---

# 179. Named binding reuse

Sólo podrá reutilizar el mismo placeholder si:

```text
Dialect syntax
+
Driver behavior
```

lo soportan de manera confiable.

---

# 180. No driver assumptions in compiler core

La decisión vendrá del `SqlBindingProfile`.

---

# 181. Parameter ordering

El orden semántico y el orden físico pueden diferir.

---

# 182. Semantic order

Relacionado con AST.

---

# 183. Physical order

Relacionado con SQL compilado.

---

# 184. Mapping required

Por ello `CompiledParameter` mantiene:

```text
sourceParameterId
```

---

# 185. Parameter occurrence

Podrá existir:

```text
ParameterOccurrence
```

para representar cada uso estructural de un parámetro.

---

# 186. Occurrence example

```text
p1 occurrence #1
p1 occurrence #2
```

---

# 187. Occurrence metadata

Puede ayudar en:

- diagnostics;
- source location;
- placeholder allocation;
- type constraint aggregation.

---

# 188. Type constraints from multiple occurrences

Si el mismo parámetro aparece:

```text
users.id = :p1
AND
orders.user_id = :p1
```

los dos contexts deben producir tipos compatibles.

---

# 189. Type conflict

Si un mismo ParameterId se utiliza como:

```text
Integer
```

y:

```text
DateTime
```

deberá producir:

```text
ConflictingParameterTypeException
```

---

# 190. Constraint aggregation

Conceptualmente:

```text
ParameterTypeConstraintSet
```

recopila todas las restricciones del parámetro.

---

# 191. Resolution

```text
Occurrence Constraints
      │
      ▼
Type Constraint Solver
      │
      ▼
Resolved Parameter Type
```

---

# 192. No arbitrary first occurrence wins

Incorrecto:

```text
first occurrence says string
→ use string everywhere
```

---

# 193. Nullability constraints

También podrán agregarse.

---

# 194. Collection element constraints

Ejemplo:

```text
id IN collection(p1)
```

produce:

```text
element type of p1
compatible with id type
```

---

# 195. Tuple constraints

Cada posición tendrá su propia restricción.

---

# 196. Parameter metadata externality

Metadata semántica resuelta deberá vivir fuera del AST.

---

# 197. ParameterSemanticInfo

Conceptualmente:

```text
ParameterSemanticInfo
├── id
├── shape
├── resolvedType
├── nullable
├── constraints
├── occurrences
├── sensitivity
├── portability
└── capabilityRequirements
```

---

# 198. QuerySemanticModel integration

```text
QuerySemanticModel
└── ParameterSemanticTable
    ├── p1 → info
    ├── p2 → info
    └── p3 → info
```

---

# 199. ParameterTable immutability

Después de Semantic Analysis deberá ser inmutable.

---

# 200. BindingPlan integration

```text
ParameterSemanticTable
+
BindingShape
+
Capabilities
+
BindingProfile
      │
      ▼
BindingPlanner
```

---

# 201. BindingPlanner

Responsabilidades:

- choose expansion strategy;
- enforce limits;
- map semantic parameters to compiled parameters;
- determine placeholder requirements;
- calculate binding shape;
- expose conversion requirements.

---

# 202. BindingPlanner must not

No deberá:

- execute SQL;
- own Connection;
- mutate AST;
- access EntityManager;
- bind PDO directly.

---

# 203. PlaceholderAllocator responsibility

Sólo asignar placeholders conforme al plan.

---

# 204. ValueConverter responsibility

Sólo convertir valores mediante Type System/Platform mapping.

---

# 205. DriverBinder responsibility

Sólo realizar binding contra NativeStatement.

---

# 206. Separation diagram

```text
ParameterSemanticInfo
        │
        ▼
   BindingPlanner
        │
        ▼
    BindingPlan
        │
        ▼
      Compiler
        │
        ├── SQL
        └── CompiledParameters
                │
                ▼
          ValueConverter
                │
                ▼
          ConvertedValues
                │
                ▼
           DriverBinder
                │
                ▼
        NativeStatement
```

---

# 207. CompiledQuery

El resultado del Compiler podrá incluir:

```text
CompiledQuery
├── sql
├── parameters
├── bindingPlan
├── queryFingerprint
├── compiledFingerprint
└── executionMetadata
```

---

# 208. CompiledQuery does not contain values

Por defecto:

```text
CompiledQuery
```

no deberá contener runtime values.

---

# 209. BoundCompiledQuery

Si se requiere una representación preparada para ejecución, podrá existir:

```text
BoundCompiledQuery
├── CompiledQuery
└── ConvertedBindingSet
```

con lifetime de operación.

---

# 210. Cache safety

Nunca cachear:

```text
BoundCompiledQuery
```

globalmente.

---

# 211. ConvertedBindingSet

Puede contener representaciones ya convertidas.

Es:

```text
operation-scoped
```

y potencialmente sensible.

---

# 212. Sensitive lifecycle

Deberá liberarse/referenciarse durante el menor tiempo razonable.

---

# 213. Secret values

Credentials de conexión no deberán pasar por Query Parameter System.

Son responsabilidad del Connection Credential System.

---

# 214. Query secrets

Tokens/passwords almacenados como datos de aplicación sí pueden ser query parameters y deberán marcarse sensibles.

---

# 215. Hashing/encryption

Database Parameter System no deberá decidir automáticamente:

```text
hash password
encrypt field
```

Eso pertenece a capas superiores/type converters explícitos.

---

# 216. Deterministic encryption caveat

Cualquier tipo cifrado que necesite búsquedas deberá declarar explícitamente su semántica.

No asumir que encrypted values son comparables.

---

# 217. Parameter logging

Permitido por defecto:

```text
parameter count
parameter types
parameter shapes
collection cardinality bucket
conversion duration
binding duration
```

---

# 218. Parameter logging prohibited by default

Evitar:

```text
raw value
full collection contents
secret token
password
personal data
```

---

# 219. Telemetry cardinality

No utilizar ParameterId arbitrarios del usuario como labels de alta cardinalidad sin normalización.

---

# 220. Collection telemetry

Preferir:

```text
collection_size_bucket = 10-99
```

sobre:

```text
collection_values = [...]
```

---

# 221. Binding performance telemetry

Podrá medir:

```text
binding.plan.duration
binding.validation.duration
binding.conversion.duration
binding.driver.duration
binding.parameter_count
binding.expanded_parameter_count
```

---

# 222. Prepared statement telemetry

Separada de:

```text
query compilation
```

y:

```text
driver binding
```

---

# 223. Performance principles

Hot path deberá minimizar:

- reflection;
- allocations;
- repeated type resolution;
- repeated conversion metadata lookup;
- parameter map rebuilding.

---

# 224. Compiled metadata

Type conversion paths podrán precompilarse cuando sean estables.

---

# 225. ConversionPlan

Podrá existir:

```text
ParameterConversionPlan
```

precalculado por tipo/target.

---

# 226. ConversionPlan safety

No deberá contener runtime values.

---

# 227. Collection conversion

Idealmente:

```text
element conversion plan
```

se reutiliza para todos los elementos.

---

# 228. Batch conversion

Podrá optimizarse sin alterar semántica.

---

# 229. Conversion errors

Deberán indicar:

```text
ParameterId
Expected Type
Actual Runtime Type
Conversion Stage
```

sin exponer necesariamente el valor.

---

# 230. Persistent runtime model

Safe application/worker scoped:

```text
Parameter descriptors
Type descriptors
Binding strategy descriptors
Frozen registries
Compiled binding plan templates
Conversion plans
```

---

# 231. Operation scoped

Nunca compartir entre requests:

```text
BindingSet
BoundValue
ConvertedBindingSet
ResolvedBindingPlan with request-specific shape when sensitive
NativeStatement bindings
```

---

# 232. No static bindings

Prohibido:

```php
static $bindings = [];
```

como current request binding storage.

---

# 233. FrankenPHP

Request A:

```text
p1 = user A
```

cleanup.

Request B:

```text
p1 = user B
```

Nunca deberá observar el valor de A.

---

# 234. RoadRunner

Mismo requisito.

---

# 235. OpenSwoole

Dos coroutines podrán ejecutar el mismo QueryTemplate con diferentes BindingSets simultáneamente.

---

# 236. Concurrency formula

```text
Shared Immutable QueryTemplate
+
Independent BindingSet A
+
Independent BindingSet B
=
Safe Concurrent Execution
```

---

# 237. Incorrect concurrency

```text
QueryTemplate
└── mutable currentBindings
```

está prohibido.

---

# 238. Parameter factory

Podrá existir:

```text
ParameterFactory
```

para construcción consistente.

Debe ser stateless.

---

# 239. BindingSetBuilder

Una API mutable de construcción puede existir localmente:

```text
BindingSetBuilder
```

pero deberá producir:

```text
ImmutableBindingSet
```

antes de ejecución.

---

# 240. Builder mutable ≠ model mutable

La ergonomía de construcción no justifica mutabilidad del modelo publicado.

---

# 241. ORM parameter integration

ORM:

```text
Repository
→ Entity Query
→ Query AST
→ Parameter AST
→ BindingSet
```

reutilizará el mismo sistema.

---

# 242. Persistence parameter integration

Updates generados por UnitOfWork:

```text
UPDATE users
SET name = :newName
WHERE id = :id
AND version = :version
```

utilizarán el mismo Parameter System.

---

# 243. No ORM-specific binding engine

No existirán:

```text
OrmBindingSystem
QueryBuilderBindingSystem
MigrationBindingSystem
```

duplicados para la misma semántica.

---

# 244. Schema parameter distinction

DDL no siempre permite parameters en posiciones donde DML sí.

Schema Compiler deberá modelar sus reglas por separado.

---

# 245. DDL literals

No deberán forzarse artificialmente como runtime query parameters si el motor requiere literal estructural.

Pero cualquier valor externo deberá validarse/normalizarse de forma segura.

---

# 246. Migration values

Data migrations sí utilizarán Query Parameter System cuando ejecuten DML.

---

# 247. Default values

Schema default:

```text
DEFAULT 0
```

no es lo mismo que Query Parameter.

---

# 248. LiteralExpression vs ParameterExpression

```text
LiteralExpression
=
structural compile-time literal

ParameterExpression
=
runtime execution input
```

---

# 249. Constant folding

El Optimizer podrá foldar literals.

No deberá foldar runtime parameters como si sus valores fueran parte permanente del query.

---

# 250. Parameter-aware optimization

Algunas optimizaciones runtime podrían usar binding shape o valores bajo una fase explícita de specialization.

---

# 251. Runtime specialization

Si se implementa:

```text
Generic Plan
+
Binding Characteristics
→ Specialized Plan
```

deberá mantenerse separado del semantic AST original.

---

# 252. Value-sensitive planning

Usar valores concretos para planificación puede:

- afectar cache;
- filtrar secretos a diagnostics;
- causar plan explosion.

No será comportamiento V1 por defecto.

---

# 253. Binding specialization V1

V1 podrá considerar principalmente:

```text
shape
cardinality
type
null presence where semantically necessary
```

sin utilizar valores concretos arbitrariamente.

---

# 254. NULL-sensitive specialization

Ejemplo:

```text
where('deleted_at', null)
```

debe convertirse en `NullPredicate` durante construcción cuando la API expresa esa intención.

No dejar que el binding planner cambie:

```text
=
```

por:

```text
IS NULL
```

basándose tardíamente en el valor.

---

# 255. Important invariant

La semántica de la consulta no deberá cambiar silenciosamente según el valor runtime de un parámetro.

---

# 256. Exception

Sólo APIs explícitamente diseñadas como:

```text
optional filters
dynamic query construction
```

podrán modificar estructura antes de crear el Query AST final.

---

# 257. Optional parameters

No confundir:

```text
optional query parameter
```

con:

```text
nullable parameter
```

---

# 258. Optional query input

Si un filtro no existe:

```text
no predicate
```

es diferente de:

```text
predicate with NULL parameter
```

---

# 259. Query Builder responsibility

Resolver optional filter semantics durante construcción.

---

# 260. Parameter defaults

Un `QueryTemplate` avanzado podría declarar defaults.

Pero deberán ser explícitos.

---

# 261. No implicit default null

Un parámetro faltante no se convertirá automáticamente en:

```text
NULL
```

---

# 262. Default value security

Defaults sensibles deberán seguir las mismas políticas de redacción.

---

# 263. Parameter aliases

Evitar aliases runtime ambiguos.

Preferir ParameterId estable.

---

# 264. User-defined names

Deberán validarse:

- length;
- charset;
- reserved internal prefix;
- uniqueness within scope.

---

# 265. Internal parameter prefix

VoltStack podrá reservar:

```text
__vs_
```

o equivalente para parámetros generados internamente.

---

# 266. Generated parameters

Ejemplos:

- tenant filters;
- optimistic lock version;
- ORM discriminator;
- soft-delete timestamp;
- system scopes.

---

# 267. Collision safety

User parameters no deberán colisionar con system-generated parameters.

---

# 268. ParameterOrigin

Podrá existir:

```text
USER
APPLICATION
ORM
PERSISTENCE
TENANT
AUTHORIZATION
SYSTEM
EXTENSION
```

---

# 269. Parameter security policy

Origen no cambia las reglas básicas:

```text
all dynamic values use safe binding when possible
```

---

# 270. Driver binding API

Conceptualmente:

```php
interface NativeParameterBinder
{
    public function bind(
        NativeStatement $statement,
        CompiledParameter $parameter,
        mixed $databaseValue,
    ): void;
}
```

---

# 271. Binder specialization

Cada driver podrá proporcionar:

```text
PdoParameterBinder
NativePgSqlParameterBinder
Sqlite3ParameterBinder
...
```

---

# 272. No PDO above driver boundary

Sólo:

```text
Driver/Pdo/*
```

deberá conocer PDO-specific binding semantics.

---

# 273. bindValue vs bindParam

Detalles como:

```text
PDOStatement::bindValue()
PDOStatement::bindParam()
```

son decisiones del PDO Driver Adapter.

---

# 274. Reference semantics

El core no deberá depender de APIs de binding por referencia.

---

# 275. Driver-specific bugs/quirks

Se encapsularán mediante:

```text
Driver Binding Profile
Driver Adapter
Capability Rules
```

no mediante condicionales distribuidos.

---

# 276. BindingProfile resolution

```text
Driver
+
Driver Version
+
Native Client
+
Configuration
      │
      ▼
SqlBindingProfile
```

---

# 277. Platform contribution

Algunas representaciones de valores dependen de Platform.

Por tanto:

```text
Platform Type Mapping
+
Driver Binding Profile
```

participan en la preparación final.

---

# 278. ResolvedDatabaseTarget

El Binding Planner podrá recibir:

```text
ResolvedDatabaseTarget
├── Driver
├── Dialect
├── Platform
└── EffectiveCapabilities
```

sin acoplarse a vendor conditionals.

---

# 279. Capability examples

```text
driver.binding.native_boolean
driver.binding.binary
driver.binding.lob_stream
driver.binding.native_array
driver.binding.repeated_named_parameter

query.parameter.max_count

query.parameter.collection_array
query.parameter.tuple
```

---

# 280. Capability ownership

`max_count` podría derivarse de Platform/Driver/context.

El EffectiveCapabilitySet resuelve el valor final.

---

# 281. No capability optimism

Si una capacidad no se conoce:

```text
UNKNOWN
```

deberá tratarse conservadoramente.

---

# 282. Parameter count enforcement

Antes de preparar SQL, verificar:

```text
compiled parameter count
<=
effective maximum
```

---

# 283. Fallback

Si excede:

```text
Planner
→ alternative strategy
```

si existe.

Si no:

```text
ParameterLimitExceededException
```

---

# 284. Security — parameter explosion

Un atacante podría intentar enviar:

```text
1000000 IDs
```

a un endpoint.

El sistema deberá soportar límites antes de generar millones de AST nodes/placeholders.

---

# 285. Collection parameter advantage

Un solo:

```text
CollectionParameter
```

permite detectar cardinalidad y aplicar governance antes de expansión.

---

# 286. Security — memory

La expansión deberá considerar:

```text
estimated memory
```

y límites de recursos.

---

# 287. Security — logs

Nunca loggear la colección completa automáticamente.

---

# 288. Security — error messages

Preferir:

```text
Parameter p3 exceeded maximum collection cardinality.
Received: 250000
Maximum: 10000
```

sin mostrar elementos.

---

# 289. Security — binary values

No serializar BLOBs en diagnostics.

---

# 290. Security — credentials

No confundir Query Parameters con Connection Credentials.

---

# 291. Binding events

Telemetry/Event integration podrá observar:

```text
BindingPlanningStarted
BindingPlanningCompleted
BindingConversionFailed
DriverBindingFailed
```

si se considera útil.

---

# 292. Event payload safety

Eventos no deberán incluir raw values por defecto.

---

# 293. Metrics

Ejemplos:

```text
database.binding.parameters
database.binding.expanded_parameters
database.binding.collections
database.binding.conversion_duration
database.binding.plan_cache_hit
database.binding.failure
```

---

# 294. High cardinality prevention

No usar:

```text
parameter_name
user_id
email
raw value
```

como metric labels sin política.

---

# 295. Debug toolbar

Podrá mostrar:

```text
Parameters: 4
Expanded: 12
Types:
  Integer: 8
  String: 4

Sensitive values hidden.
```

---

# 296. Testing strategy

El sistema deberá tener:

```text
Parameter AST Tests
BindingSet Tests
Type Inference Tests
Constraint Resolution Tests
Collection Tests
Tuple Tests
Placeholder Tests
BindingPlan Tests
Conversion Tests
Driver Binding Tests
Security Tests
Cache Tests
Persistent Runtime Tests
Concurrency Tests
Cross-platform Tests
```

---

# 297. Scalar tests

Cubrir:

```text
integer
string
boolean
null
float
decimal
binary
date/time
uuid
enum
value object
JSON
```

---

# 298. Collection tests

Cubrir:

```text
empty
single
small
large
too large
nullable elements
mixed types
generator
single-pass iterable
tuple collections
```

---

# 299. Type conflict tests

Ejemplo:

```text
same p1
→ Integer constraint
→ DateTime constraint
```

debe fallar determinísticamente.

---

# 300. Placeholder tests

Cubrir:

```text
?
numbered
named
repeated semantic parameter
subquery scopes
collection expansion
raw fragments
```

---

# 301. Driver conformance

Cada Driver deberá demostrar:

- scalar binding;
- NULL;
- binary;
- boolean;
- integer;
- large values;
- LOB si declara soporte;
- repeated parameters;
- cancellation interaction;
- cleanup after bind failure.

---

# 302. Cross-platform semantics

Los tests deberán ejecutarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

para el portable core.

---

# 303. Persistent worker test

```text
Request A:
p1 = secret-A

cleanup

Request B:
p1 = secret-B

assert:
no reference/value/diagnostic from A remains
```

---

# 304. Concurrent template test

```text
Shared QueryTemplate

Coroutine A:
p1 = 1

Coroutine B:
p1 = 2

assert:
independent binding/execution
```

---

# 305. Cache test

Mismos:

```text
AST
types
target
binding shape
```

con valores distintos deberán poder reutilizar el mismo compiled artifact cuando corresponda.

---

# 306. Collection cache test

Diferentes cardinalidades deberán:

```text
share semantic fingerprint
```

pero podrán:

```text
use different compiled binding shapes
```

---

# 307. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Parameter\
```

---

# 308. Proposed structure

```text
Query/
└── Parameter/
    ├── Contract/
    │   ├── Parameter.php
    │   ├── BindingSet.php
    │   ├── BindingPlanner.php
    │   ├── ValueConverter.php
    │   └── ParameterBinder.php
    │
    ├── Identity/
    │   ├── ParameterId.php
    │   ├── CompiledParameterId.php
    │   └── ParameterScopeId.php
    │
    ├── Definition/
    │   ├── ParameterDefinition.php
    │   ├── ParameterShape.php
    │   └── ParameterOrigin.php
    │
    ├── Ast/
    │   ├── ParameterExpressionNode.php
    │   ├── CollectionParameterExpressionNode.php
    │   └── TupleParameterExpressionNode.php
    │
    ├── Binding/
    │   ├── BoundValue.php
    │   ├── ImmutableBindingSet.php
    │   ├── BindingSetBuilder.php
    │   ├── BindingShape.php
    │   └── BindingReplayability.php
    │
    ├── Semantic/
    │   ├── ParameterSemanticInfo.php
    │   ├── ParameterSemanticTable.php
    │   ├── ParameterTypeConstraint.php
    │   ├── ParameterTypeConstraintSet.php
    │   └── ParameterTypeResolver.php
    │
    ├── Planning/
    │   ├── BindingPlan.php
    │   ├── BindingPlanTemplate.php
    │   ├── ResolvedBindingPlan.php
    │   ├── ParameterBindingPlan.php
    │   ├── CollectionBindingStrategy.php
    │   └── BindingPlanner.php
    │
    ├── Compilation/
    │   ├── CompiledParameter.php
    │   ├── Placeholder.php
    │   ├── PlaceholderAllocator.php
    │   ├── PlaceholderStyle.php
    │   └── BindingShapeFingerprint.php
    │
    ├── Conversion/
    │   ├── ParameterConversionPlan.php
    │   ├── ConvertedBindingSet.php
    │   └── ValueConverter.php
    │
    ├── Collection/
    │   ├── CollectionSource.php
    │   ├── MaterializedCollection.php
    │   ├── ReplayableIterable.php
    │   └── SinglePassIterable.php
    │
    ├── Tuple/
    │   ├── TupleShape.php
    │   └── TupleBindingValidator.php
    │
    ├── Security/
    │   ├── ParameterSensitivity.php
    │   └── ParameterRedactor.php
    │
    ├── Diagnostics/
    │   └── ParameterDiagnosticRenderer.php
    │
    └── Exception/
```

---

# 309. Driver-side structure

Separadamente:

```text
Driver/
├── Contract/
│   └── NativeParameterBinder.php
│
├── Binding/
│   ├── DriverBindingProfile.php
│   └── DriverBindingType.php
│
└── Pdo/
    ├── Binding/
    │   └── PdoParameterBinder.php
    ├── MySql/
    ├── PostgreSql/
    └── Sqlite/
```

---

# 310. Type-side structure

El Type System deberá permanecer separado:

```text
Type/
├── TypeRegistry
├── TypeResolver
├── DatabaseValueConverter
├── PlatformTypeMapping
└── ...
```

Parameter System consume Type System.

No lo reemplaza.

---

# 311. Dependency direction

```text
Query AST
    │
    ▼
Parameter Model
    │
    ▼
Semantic Analysis
    │
    ▼
Type System
    │
    ▼
Binding Planner
    │
    ▼
Compiler
    │
    ▼
Execution
    │
    ▼
Driver Binding
```

---

# 312. Forbidden dependencies

Nunca:

```text
Parameter AST
→ PDO

Parameter AST
→ Connection

Parameter AST
→ EntityManager

Parameter AST
→ current tenant

Parameter AST
→ FrankenPHP
```

---

# 313. DB-PARAM-001

Todo parámetro tendrá identidad semántica independiente del placeholder SQL.

---

# 314. DB-PARAM-002

`ParameterId` no será un placeholder.

---

# 315. DB-PARAM-003

El AST no almacenará runtime values por defecto.

---

# 316. DB-PARAM-004

Los runtime values vivirán en `BindingSet`.

---

# 317. DB-PARAM-005

`BindingSet` será execution-scoped.

---

# 318. DB-PARAM-006

`BindingSet` no se almacenará en singletons.

---

# 319. DB-PARAM-007

Los parámetros podrán tener shapes explícitos.

---

# 320. DB-PARAM-008

Scalar y Collection parameters serán conceptos diferentes.

---

# 321. DB-PARAM-009

Una CollectionParameter no se expandirá durante construcción del AST.

---

# 322. DB-PARAM-010

La estrategia de expansión pertenecerá al Planner.

---

# 323. DB-PARAM-011

El Compiler asignará placeholders según el BindingPlan.

---

# 324. DB-PARAM-012

El Driver realizará únicamente el binding nativo.

---

# 325. DB-PARAM-013

El Query Builder nunca utilizará constantes PDO.

---

# 326. DB-PARAM-014

Los tipos de parámetros serán semánticos hasta la frontera apropiada.

---

# 327. DB-PARAM-015

La inferencia de tipo no dependerá exclusivamente del PHP runtime type.

---

# 328. DB-PARAM-016

Los explicit type hints tendrán prioridad sobre inferencia segura.

---

# 329. DB-PARAM-017

Las restricciones de todas las occurrences del mismo ParameterId deberán unificarse.

---

# 330. DB-PARAM-018

Los conflictos de tipo no se resolverán mediante first-match-wins.

---

# 331. DB-PARAM-019

Un NULL podrá conservar información de tipo.

---

# 332. DB-PARAM-020

Un parámetro faltante no se convertirá implícitamente en NULL.

---

# 333. DB-PARAM-021

Optional parameter y nullable parameter serán conceptos distintos.

---

# 334. DB-PARAM-022

Identifiers nunca se bindearán como query values ordinarios.

---

# 335. DB-PARAM-023

Operators nunca se bindearán como query values ordinarios.

---

# 336. DB-PARAM-024

Keywords estructurales no serán runtime value parameters.

---

# 337. DB-PARAM-025

Los raw fragments deberán usar bindings seguros para valores.

---

# 338. DB-PARAM-026

Los ParameterId de subqueries deberán estar libres de colisiones.

---

# 339. DB-PARAM-027

Parameter Scope y Symbol Scope permanecerán separados.

---

# 340. DB-PARAM-028

El mismo ParameterId podrá aparecer múltiples veces.

---

# 341. DB-PARAM-029

Un ParameterId podrá producir múltiples placeholders físicos.

---

# 342. DB-PARAM-030

La repetición física dependerá del BindingProfile.

---

# 343. DB-PARAM-031

Placeholder allocation será determinista.

---

# 344. DB-PARAM-032

Los nombres de placeholders no contendrán valores sensibles.

---

# 345. DB-PARAM-033

`BindingPlan` no contendrá runtime values.

---

# 346. DB-PARAM-034

`CompiledQuery` no contendrá runtime values por defecto.

---

# 347. DB-PARAM-035

Un artifact con valores convertidos será operation-scoped.

---

# 348. DB-PARAM-036

Semantic fingerprint no contendrá valores runtime.

---

# 349. DB-PARAM-037

BindingShapeFingerprint no contendrá valores runtime.

---

# 350. DB-PARAM-038

La cardinalidad podrá formar parte del binding shape cuando afecte SQL.

---

# 351. DB-PARAM-039

La expansión de una colección respetará límites efectivos del target.

---

# 352. DB-PARAM-040

VoltStack no generará un número ilimitado de placeholders.

---

# 353. DB-PARAM-041

Huge collections pasarán por Resource Governance.

---

# 354. DB-PARAM-042

La estrategia de colección será capability-aware.

---

# 355. DB-PARAM-043

Native array support requerirá Platform y Driver support compatible.

---

# 356. DB-PARAM-044

Tuple parameters validarán arity.

---

# 357. DB-PARAM-045

Tuple collection parameters validarán arity de todos sus elementos.

---

# 358. DB-PARAM-046

Mixed collection types no se aceptarán silenciosamente si violan el tipo esperado.

---

# 359. DB-PARAM-047

Single-pass iterables no se consumirán múltiples veces accidentalmente.

---

# 360. DB-PARAM-048

Binding replayability será explícita cuando afecte retries.

---

# 361. DB-PARAM-049

El Parameter System no ejecutará retries.

---

# 362. DB-PARAM-050

El Type System realizará Domain Value → Database Representation.

---

# 363. DB-PARAM-051

El Driver adapter realizará Database Representation → Native Binding.

---

# 364. DB-PARAM-052

El Driver no conocerá Value Objects de dominio.

---

# 365. DB-PARAM-053

Las Entities no se bindearán directamente en el core.

---

# 366. DB-PARAM-054

El ORM resolverá Entity → Identifier antes del binding cuando corresponda.

---

# 367. DB-PARAM-055

La conversión Decimal no perderá precisión silenciosamente.

---

# 368. DB-PARAM-056

DateTime conversion será determinista e independiente de locale accidental.

---

# 369. DB-PARAM-057

SQL NULL y JSON null permanecerán diferenciados.

---

# 370. DB-PARAM-058

Text y Binary values permanecerán diferenciados.

---

# 371. DB-PARAM-059

LOB streaming será capability-aware.

---

# 372. DB-PARAM-060

El ownership de streams será explícito.

---

# 373. DB-PARAM-061

Los valores sensibles estarán ocultos por defecto.

---

# 374. DB-PARAM-062

Las queries no se interpolarán con valores para logging.

---

# 375. DB-PARAM-063

Telemetry no utilizará raw parameter values como labels.

---

# 376. DB-PARAM-064

Los errores de binding identificarán el ParameterId sin necesidad de revelar el valor.

---

# 377. DB-PARAM-065

Conversiones fallables se ejecutarán antes de retener recursos físicos cuando sea posible.

---

# 378. DB-PARAM-066

El Parameter System será independiente del ORM.

---

# 379. DB-PARAM-067

Query Builder, ORM y Persistence Engine reutilizarán el mismo sistema.

---

# 380. DB-PARAM-068

Native prepared statements permanecerán physical-connection-scoped.

---

# 381. DB-PARAM-069

Framework PreparedQuery y NativePreparedStatement serán conceptos distintos.

---

# 382. DB-PARAM-070

Un native prepared statement no podrá reutilizarse en otra physical connection.

---

# 383. DB-PARAM-071

Statement caches deberán considerar connection generation.

---

# 384. DB-PARAM-072

Connection reset determinará si native prepared statements pueden sobrevivir.

---

# 385. DB-PARAM-073

Parameter registries runtime estarán congeladas cuando corresponda.

---

# 386. DB-PARAM-074

No existirá una global mutable current BindingSet.

---

# 387. DB-PARAM-075

El sistema será seguro para FrankenPHP persistent workers.

---

# 388. DB-PARAM-076

El sistema será adaptable a RoadRunner.

---

# 389. DB-PARAM-077

El sistema soportará ejecución concurrente segura bajo OpenSwoole.

---

# 390. DB-PARAM-078

Un QueryTemplate immutable podrá ejecutarse concurrentemente con BindingSets independientes.

---

# 391. DB-PARAM-079

Parameter normalization será determinista.

---

# 392. DB-PARAM-080

Parameter normalization no inspeccionará valores para cambiar semántica de query.

---

# 393. DB-PARAM-081

Un parámetro NULL no transformará tardíamente `=` en `IS NULL`.

---

# 394. DB-PARAM-082

La estructura final de la consulta se decidirá antes del binding.

---

# 395. DB-PARAM-083

La specialization runtime no mutará el Query AST original.

---

# 396. DB-PARAM-084

Capability checks usarán EffectiveCapabilitySet.

---

# 397. DB-PARAM-085

No existirán vendor conditionals en Parameter AST o Binding core.

---

# 398. DB-PARAM-086

Driver quirks estarán encapsulados en Driver Binding adapters/profiles.

---

# 399. DB-PARAM-087

PDO permanecerá detrás del Driver boundary.

---

# 400. DB-PARAM-088

Los bindings de sistema estarán sujetos a las mismas reglas de seguridad que los bindings de usuario.

---

# 401. DB-PARAM-089

El sistema podrá explicar cómo un semantic parameter se convirtió en compiled parameters.

---

# 402. DB-PARAM-090

Los diagnostics de parámetros serán redacted by default.

---

# 403. Anti-pattern — value inside AST

Incorrecto:

```text
ParameterNode
├── id: p1
└── value: "alice@example.com"
```

Correcto:

```text
AST:
ParameterNode(p1)

Execution:
BindingSet
└── p1 → "alice@example.com"
```

---

# 404. Anti-pattern — placeholder inside AST

Incorrecto:

```text
ParameterNode("?")
```

Correcto:

```text
ParameterNode(p1)
```

y posteriormente:

```text
Compiler
→ Placeholder(?)
```

---

# 405. Anti-pattern — PDO type in Builder

Incorrecto:

```php
$query->where(
    'age',
    18,
    PDO::PARAM_INT,
);
```

como contrato core.

Correcto:

```text
IntegerType
→ Platform mapping
→ Driver binding
```

---

# 406. Anti-pattern — collection expansion in Builder

Incorrecto:

```text
whereIn([1,2,3])
→ immediately create p1,p2,p3
```

Correcto:

```text
CollectionParameter(p1)
→ Planner
→ binding strategy
```

---

# 407. Anti-pattern — bind identifier

Incorrecto:

```text
SELECT * FROM ?
```

Correcto:

```text
Identifier
→ validate
→ resolve
→ quote
```

---

# 408. Anti-pattern — logging interpolated SQL

Incorrecto:

```text
SELECT * FROM users
WHERE password = 'secret123'
```

Correcto:

```text
SQL:
SELECT * FROM users
WHERE password = ?

Bindings:
1: [REDACTED]
```

---

# 409. Anti-pattern — mutable prepared query

Incorrecto:

```text
PreparedQuery
└── currentBindings
```

Correcto:

```text
Immutable QueryTemplate
+
Independent BindingSet
```

---

# 410. Anti-pattern — first type wins

Incorrecto:

```text
p1 first seen as string
→ StringType
```

Correcto:

```text
all occurrences
→ constraint set
→ type resolution
```

---

# 411. Anti-pattern — PHP array ambiguity

Incorrecto:

```text
array
→ maybe JSON?
→ maybe IN collection?
→ maybe tuple?
```

Correcto:

```text
ParameterShape
+
Semantic Type
```

determinan intención.

---

# 412. Anti-pattern — silent parameter truncation

Nunca reducir una colección automáticamente:

```text
10000 values
→ first 1000
```

para cumplir un límite.

Debe elegirse estrategia alternativa o fallar.

---

# 413. Anti-pattern — automatic float decimal

Incorrecto:

```text
Decimal
→ float
```

sin garantía.

---

# 414. Anti-pattern — native statement global cache

Incorrecto:

```text
global statement cache
```

Correcto:

```text
PhysicalConnection
└── optional statement cache
```

---

# 415. Example complete pipeline

Aplicación:

```php
$query = DB::table('users')
    ->where('status', $status)
    ->whereIn('id', $ids);
```

Construcción:

```text
Query AST
└── AndPredicate
    ├── ComparisonPredicate
    │   ├── Column(status)
    │   ├── EQUAL
    │   └── Parameter(p1)
    │
    └── InPredicate
        ├── Column(id)
        └── CollectionParameter(p2)
```

Runtime:

```text
BindingSet
├── p1 → Status::ACTIVE
└── p2 → [10,20,30]
```

---

# 416. Semantic phase

```text
p1
├── shape: SCALAR
└── inferred type: StatusType

p2
├── shape: COLLECTION
└── element type: UserIdType
```

---

# 417. Planning phase

Supongamos que el target decide:

```text
p1 → scalar parameter
p2 → expanded parameters
```

Binding Plan:

```text
p1
└── SCALAR

p2
└── EXPANDED_PARAMETERS
    ├── element 0
    ├── element 1
    └── element 2
```

---

# 418. Compilation phase

SQL:

```sql
SELECT *
FROM users
WHERE status = ?
AND id IN (?, ?, ?)
```

Compiled parameters:

```text
cp1 → source p1
cp2 → source p2[0]
cp3 → source p2[1]
cp4 → source p2[2]
```

---

# 419. Conversion phase

```text
Status::ACTIVE
→ StatusType
→ "active"

UserId(10)
→ integer 10
...
```

---

# 420. Driver phase

```text
cp1
→ native string binding

cp2
→ native integer binding

cp3
→ native integer binding

cp4
→ native integer binding
```

---

# 421. Execution phase

```text
NativeStatement
      │
      ▼
Driver Execute
      │
      ▼
NativeResult
      │
      ▼
VoltStack Result
```

Parameter System termina antes de interpretar resultados.

---

# 422. Example repeated parameter

Semantic query:

```text
created_at >= p1
OR
updated_at >= p1
```

Semantic table:

```text
p1
├── DateTime constraint from created_at
└── DateTime constraint from updated_at
```

Compiled positional target:

```sql
WHERE created_at >= ?
OR updated_at >= ?
```

Mapping:

```text
cp1 → p1
cp2 → p1
```

BindingSet:

```text
p1 → DateTimeImmutable(...)
```

El valor semántico existe una sola vez.

---

# 423. Example conflicting parameter

```text
age = p1
AND
created_at = p1
```

Si:

```text
age → IntegerType
created_at → DateTimeType
```

resultado:

```text
ConflictingParameterTypeException

Parameter:
p1

Constraints:
IntegerType
DateTimeType
```

No se ejecuta SQL.

---

# 424. Example collection limit

```text
p1:
collection cardinality = 70000

effective parameter maximum:
65535
```

Planner:

```text
Expanded Parameters
→ rejected
```

Luego puede evaluar:

```text
Native Array?
VALUES Relation?
Temporary Relation?
Chunk Strategy?
```

Si ninguna estrategia preserva semántica:

```text
ParameterLimitExceededException
```

---

# 425. Example sensitive parameter

```text
password_hash = p1
```

Diagnostics:

```text
Parameter p1
Type: PasswordHashType
Sensitivity: SECRET
Value: [REDACTED]
```

Nunca:

```text
Value: "$2y$..."
```

por defecto.

---

# 426. Example persistent worker

```text
FrankenPHP Worker
│
├── Request A
│   ├── Shared QueryTemplate Q1
│   └── BindingSet {p1=A}
│
├── Cleanup
│
└── Request B
    ├── Shared QueryTemplate Q1
    └── BindingSet {p1=B}
```

Permitido compartir:

```text
Q1
```

Prohibido compartir:

```text
BindingSet A
```

---

# 427. Parameter architecture formula

```text
Query Parameter
=
Identity
+
Shape
+
Semantic Constraints
```

---

# 428. Runtime binding formula

```text
Runtime Binding
=
ParameterId
+
Runtime Value
+
Optional Explicit Type
+
Sensitivity
```

---

# 429. Semantic parameter formula

```text
Resolved Parameter
=
Parameter Definition
+
Occurrence Constraints
+
Resolved Type
+
Nullability
+
Capability Requirements
```

---

# 430. Binding planning formula

```text
Binding Plan
=
Resolved Parameters
+
Runtime Binding Shape
+
Effective Capabilities
+
Dialect Requirements
+
Driver Binding Profile
+
Resource Limits
```

---

# 431. Compilation formula

```text
Compiled Parameters
=
Binding Plan
+
Placeholder Allocation
+
Physical Parameter Layout
```

---

# 432. Conversion formula

```text
Database Binding Value
=
Runtime Value
+
Semantic Type
+
Platform Type Mapping
+
Conversion Plan
```

---

# 433. Native binding formula

```text
Native Driver Binding
=
Compiled Parameter
+
Database Binding Value
+
Driver Binding Profile
```

---

# 434. Complete formula

```text
Application Value
        │
        ▼
     BindingSet
        │
        ▼
Semantic Parameter
        │
        ▼
   Type Resolution
        │
        ▼
   Binding Planner
        │
        ▼
    Binding Plan
        │
        ▼
      Compiler
        │
        ▼
Compiled Parameter
        │
        ▼
 Value Conversion
        │
        ▼
 Database Value
        │
        ▼
  Driver Binding
        │
        ▼
 Native Statement
```

---

# 435. Critical boundary

```text
Query Parameter
≠
Runtime Value
≠
Database Value
≠
Placeholder
≠
Compiled Parameter
≠
Native Binding
```

---

# 436. Public DX

La complejidad interna no deberá contaminar la API normal.

El desarrollador podrá continuar escribiendo:

```php
DB::table('users')
    ->where('email', $email)
    ->whereIn('id', $ids)
    ->get();
```

VoltStack internamente realizará:

```text
Parameter creation
Type inference
Binding validation
Collection planning
Value conversion
Placeholder allocation
Driver binding
```

---

# 437. Advanced DX

Cuando sea necesario, podrá ofrecer:

```php
$query->bind(
    'minimum',
    $amount,
    type: DecimalType::class,
);
```

o una API equivalente fuertemente tipada.

---

# 438. Simple API first

La API avanzada no deberá ser obligatoria para consultas comunes.

Objetivo:

```text
Laravel-like DX
+
Doctrine-like type rigor
+
VoltStack semantic architecture
```

---

# 439. Security principle

> Todo valor dinámico deberá permanecer como dato hasta alcanzar la frontera de binding; nunca deberá convertirse accidentalmente en estructura SQL.

---

# 440. Type principle

> El tipo de un parámetro será determinado por su intención semántica y contexto de consulta, no únicamente por la representación PHP del valor recibido.

---

# 441. Collection principle

> Una colección es una entrada semántica de cardinalidad runtime; el número de placeholders SQL será una decisión posterior del Planner y Compiler.

---

# 442. Compilation principle

> El Compiler conoce placeholders y layout físico de parámetros, pero no será propietario de los valores runtime.

---

# 443. Driver principle

> El Driver conoce cómo enviar un valor ya convertido al cliente nativo; no conoce la intención de negocio ni el modelo de dominio que originó ese valor.

---

# 444. Runtime principle

> Query templates podrán reutilizarse entre ejecuciones y workers únicamente porque los valores runtime permanecerán fuera de los artifacts compartidos.

---

# 445. Cache principle

> Los caches de Query/AST/Plan/CompiledQuery deberán depender de estructura, tipos, capabilities y binding shape cuando corresponda, nunca de secretos o valores runtime ordinarios.

---

# 446. Diagnostic principle

> VoltStack deberá poder explicar cómo un parámetro semántico terminó convertido en uno o varios bindings físicos sin revelar necesariamente su valor.

---

# 447. Extensibility principle

Nuevos tipos y drivers podrán extender:

```text
Type Resolution
Value Conversion
Binding Strategy
Driver Binding
```

mediante contratos explícitos.

No deberán modificar arbitrariamente el Parameter AST core.

---

# 448. V1 implementation priority

## Phase 1

```text
ParameterId
ParameterExpressionNode
ParameterDefinition
ParameterShape
ImmutableBindingSet
BoundValue
ParameterSemanticInfo
Scalar Binding
PlaceholderAllocator
CompiledParameter
Basic Value Conversion
PDO Parameter Binding
```

---

# 449. V1 Phase 2

```text
CollectionParameter
Collection Binding
Expanded Parameters
BindingShape
BindingPlan
Parameter Limits
Collection Validation
```

---

# 450. V1 Phase 3

```text
Repeated Parameters
Subquery Parameter Scopes
Type Constraint Aggregation
Sensitive Parameter Redaction
Binding Diagnostics
Compiled Query Integration
```

---

# 451. V2

```text
Tuple Parameters
Tuple Collections
Native Arrays
LOB Streaming
Binding Replayability
PreparedQuery
Statement Cache Integration
Advanced Collection Strategies
```

---

# 452. V3

```text
VALUES relation strategy
Temporary relation strategy
Async binding
Advanced native types
Runtime specialization
Adaptive binding planning
Bulk parameter pipelines
```

---

# 453. Architectural result

Con este sistema:

```text
Query AST
```

permanece:

```text
immutable
semantic
value-independent
driver-independent
```

mientras:

```text
BindingSet
```

permanece:

```text
runtime
scoped
sensitive
execution-specific
```

y:

```text
CompiledQuery
```

permanece:

```text
target-specific
value-independent
cacheable when appropriate
```

---

# 454. Final architecture

```text
                     Query Template
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
         Query AST              Parameter Model
             │                         │
             └────────────┬────────────┘
                          ▼
                  Semantic Analysis
                          │
                          ▼
                 Parameter Type Table
                          │
                          ▼
                     Query Planner
                          │
                          ▼
                    Binding Planner
                          │
                          ▼
                       Compiler
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
             SQL              Compiled Parameters
                                       │
                                       │
Runtime Values                         │
     │                                 │
     ▼                                 │
 BindingSet                            │
     │                                 │
     └──────────────┬──────────────────┘
                    ▼
             Value Conversion
                    │
                    ▼
             Driver Binding
                    │
                    ▼
             Native Statement
                    │
                    ▼
                 Database
```

---

# 455. Regla maestra final

El sistema completo deberá mantener permanentemente:

```text
Semantic Parameter
        ≠
Runtime Binding
        ≠
Converted Database Value
        ≠
SQL Placeholder
        ≠
Compiled Parameter
        ≠
Native Driver Binding
```

Esta separación constituye una de las fronteras arquitectónicas fundamentales de `VoltStack/Quantum/Database`.

---

# 456. Próximo documento

El siguiente documento será:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

y deberá formalizar:

```text
Query Type
        │
        ├── Scalar Types
        ├── Numeric Types
        ├── String Types
        ├── Boolean
        ├── Binary
        ├── Date/Time
        ├── UUID
        ├── Enum
        ├── JSON
        ├── Array
        ├── Tuple
        ├── Null
        ├── Unknown
        ├── Value Objects
        └── Extension Types
```

además de la relación:

```text
PHP / Domain Type
        │
        ▼
Query Semantic Type
        │
        ▼
Platform Database Type
        │
        ▼
Database Representation
        │
        ▼
Driver Binding Type
```

manteniendo:

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

# 457. Estado final

Con los documentos:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

queda establecida la base semántica para:

```text
Expression
+
Predicate
+
Parameter
+
Runtime Binding
```

El siguiente paso será proporcionarles un sistema de tipos común mediante:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

preparando posteriormente:

```text
31_DATABASE_QUERY_METADATA_SYSTEM.md
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
```

antes de entrar al bloque completo del **Semantic Query Engine**.