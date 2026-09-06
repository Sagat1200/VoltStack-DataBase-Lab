# 30_DATABASE_QUERY_TYPE_SYSTEM.md

# VoltStack Quantum Database
## Query Type System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 30 — Query Type System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Type System  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del **Query Type System** de:

```text
VoltStack/Quantum/Database
```

El sistema de tipos permitirá que el Query Engine comprenda el significado semántico de:

- columnas;
- parámetros;
- literales;
- expresiones;
- funciones;
- operadores;
- predicados;
- agregaciones;
- subqueries;
- tuplas;
- valores JSON;
- enums;
- UUID;
- fechas;
- tipos personalizados;
- Value Objects.

La regla fundamental será:

```text
PHP Type
≠
Domain Type
≠
Query Semantic Type
≠
Platform Database Type
≠
Database Representation
≠
Driver Binding Type
```

---

# 2. Problema arquitectónico

Una implementación simple podría asumir:

```php
is_int($value)    → INTEGER
is_string($value) → VARCHAR
is_bool($value)   → BOOLEAN
```

Esto es insuficiente.

Un `string` PHP podría representar:

```text
String
UUID
Email
Decimal
JSON
Date
DateTime
Enum backing value
Binary
IP Address
Domain Identifier
```

Del mismo modo:

```text
42
```

podría representar:

```text
Integer
UserId
OrderId
Version
Year
BitMask
Enum backing value
```

Por ello VoltStack necesita un sistema de tipos semántico independiente.

---

# 3. Regla maestra

> El Query Type describe el significado de un valor dentro de una consulta, no simplemente su representación PHP ni su representación física en la base de datos.

---

# 4. Arquitectura general

```text
PHP / Domain Value
        │
        ▼
Application / ORM Metadata
        │
        ▼
Query Semantic Type
        │
        ├── Expressions
        ├── Parameters
        ├── Columns
        ├── Functions
        ├── Operators
        └── Predicates
        │
        ▼
Semantic Analysis
        │
        ├── Type Resolution
        ├── Type Inference
        ├── Compatibility
        ├── Coercion
        └── Constraint Solving
        │
        ▼
Resolved Query Type
        │
        ▼
Platform Type Mapping
        │
        ▼
Database Type / Representation
        │
        ▼
Driver Binding Type
        │
        ▼
Native Client
```

---

# 5. Posición dentro del Query Engine

```text
Query Model
    │
    ▼
Query AST
    │
    ├── Expressions
    ├── Predicates
    └── Parameters
    │
    ▼
Query Type System
    │
    ▼
Semantic Analysis
    │
    ▼
Semantic Graph
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

El Query Type System forma parte de la infraestructura semántica.

---

# 6. Query Type ≠ Database Type

Ejemplo:

```text
Query Type:
Boolean
```

podría representarse físicamente como:

```text
PostgreSQL:
BOOLEAN

MySQL:
BOOLEAN / TINYINT semantics

SQLite:
INTEGER affinity / STRICT representation
```

El Query Engine conserva:

```text
BooleanType
```

hasta que la plataforma de destino sea conocida.

---

# 7. Query Type ≠ PHP Type

Ejemplo:

```php
$status = 'active';
```

podría tener:

```text
PHP type:
string
```

pero semánticamente:

```text
Query type:
StatusEnumType
```

---

# 8. Query Type ≠ Domain Type

Un Value Object:

```php
final readonly class UserId
{
    public function __construct(
        public int $value,
    ) {}
}
```

es un tipo de dominio.

Puede mapearse a:

```text
UserIdQueryType
```

que posteriormente puede utilizar:

```text
Integer Database Representation
```

---

# 9. Capas de tipos

VoltStack distinguirá:

```text
Layer 1
PHP Runtime Type

Layer 2
Domain Type

Layer 3
Query Semantic Type

Layer 4
Platform Database Type

Layer 5
Database Representation

Layer 6
Driver Binding Type
```

---

# 10. Ejemplo completo

```text
UserId(42)
   │
   │ Domain Type
   ▼
UserIdType
   │
   │ Query Semantic Type
   ▼
IntegerType Mapping
   │
   │ Platform Type
   ▼
BIGINT
   │
   │ Database Representation
   ▼
42
   │
   │ Driver Binding
   ▼
INTEGER / native integer
```

---

# 11. Objetivos

El Query Type System deberá proporcionar:

1. tipos semánticos;
2. inferencia;
3. resolución;
4. compatibilidad;
5. coerción;
6. promoción;
7. constraints;
8. nullability;
9. tipos compuestos;
10. tipos personalizados;
11. integración con Schema;
12. integración con ORM;
13. integración con Parameters;
14. integración con Expressions;
15. integración con Functions;
16. integración con Platform;
17. integración con Driver Binding.

---

# 12. No objetivos

El Query Type System no deberá:

- generar SQL;
- ejecutar consultas;
- abrir conexiones;
- bindear PDO;
- administrar EntityManager;
- administrar UnitOfWork;
- resolver tenants;
- gestionar transacciones;
- contener lógica de negocio.

---

# 13. QueryType

Contrato conceptual:

```php
interface QueryType
{
    public function id(): QueryTypeId;

    public function category(): QueryTypeCategory;
}
```

El contrato real deberá mantenerse pequeño.

---

# 14. QueryTypeId

Los tipos tendrán identidad estable.

Ejemplos:

```text
core.boolean
core.integer
core.big_integer
core.decimal
core.string
core.binary
core.uuid
core.date
core.datetime
core.json
core.null
core.unknown
```

Tipos personalizados:

```text
app.user_id
app.money
app.email
```

---

# 15. QueryTypeId ≠ database type name

Incorrecto:

```text
QueryTypeId("VARCHAR(255)")
```

Correcto:

```text
QueryTypeId("core.string")
```

La declaración física se resuelve después.

---

# 16. QueryTypeCategory

Categorías conceptuales:

```text
UNKNOWN
NULL
BOOLEAN
NUMERIC
STRING
BINARY
TEMPORAL
IDENTIFIER
ENUM
JSON
COLLECTION
TUPLE
STRUCTURED
DOMAIN
PLATFORM
EXTENSION
```

---

# 17. TypeDescriptor

La metadata estática podrá vivir en:

```text
TypeDescriptor
├── id
├── category
├── characteristics
├── portability
├── supportedOperations
└── extensionMetadata
```

---

# 18. Type instance vs descriptor

Deberán distinguirse:

```text
TypeDescriptor
```

de:

```text
ResolvedQueryType
```

El descriptor describe el tipo.

El resolved type puede incluir información contextual.

---

# 19. ResolvedQueryType

Conceptualmente:

```text
ResolvedQueryType
├── typeId
├── nullability
├── parameters
├── constraints
├── origin
├── certainty
└── portability
```

---

# 20. Parameterized types

Algunos tipos requieren parámetros.

Ejemplos:

```text
Decimal(precision=18, scale=2)

String(length=255)

Tuple<Integer, String>

Collection<Uuid>

Enum<Status>

Domain<UserId, Integer>
```

---

# 21. Query Type immutability

Los tipos resueltos deberán ser inmutables.

---

# 22. Query Type lifetime

Los descriptors podrán ser:

```text
application / worker safe
```

si son inmutables.

Los resolved types podrán pertenecer al:

```text
Semantic Analysis artifact
```

y también ser inmutables.

---

# 23. No current type global

Prohibido:

```php
TypeRegistry::setCurrentType(...);
```

---

# 24. TypeRegistry

VoltStack tendrá un registry especializado:

```text
QueryTypeRegistry
```

Responsabilidades:

- registrar descriptors;
- resolver IDs;
- detectar colisiones;
- validar extensiones;
- congelarse después del bootstrap.

---

# 25. No universal registry

No deberá convertirse en:

```text
DatabaseRegistry
```

para drivers, dialects, types, functions, operators, etc.

Cada dominio tendrá su registry especializado.

---

# 26. Registry lifecycle

```text
Mutable Bootstrap Registry
        │
        ▼
Extension Registration
        │
        ▼
Validation
        │
        ▼
Freeze
        │
        ▼
Immutable Runtime Registry
```

---

# 27. Built-in types

El core deberá proporcionar como mínimo:

```text
UnknownType
NullType
BooleanType

IntegerType
BigIntegerType
DecimalType
FloatType

StringType
TextType
BinaryType

DateType
TimeType
DateTimeType
DateTimeTzType
IntervalType

UuidType

JsonType

EnumType

CollectionType
TupleType

DomainType
```

No todos necesitan exponerse directamente al usuario.

---

# 28. UnknownType

`UnknownType` representa:

```text
type not yet resolved
```

No significa:

```text
mixed forever
```

---

# 29. UnknownType usage

Ejemplo:

```text
Parameter(p1)
```

antes de Semantic Analysis:

```text
type = Unknown
```

Después:

```text
users.id = p1
```

puede resolverse:

```text
p1 → UserIdType
```

---

# 30. UnknownType behavior

Deberá ser conservador.

No permitirá asumir automáticamente operaciones arbitrarias.

---

# 31. Unknown ≠ Any

VoltStack deberá evitar introducir un tipo:

```text
Any
```

que desactive el sistema semántico.

---

# 32. NullType

`NullType` representa el literal SQL NULL.

---

# 33. NullType ≠ nullable

Estos conceptos son distintos:

```text
NullType
```

y:

```text
Nullable<IntegerType>
```

---

# 34. Nullability

Preferentemente será una propiedad separada:

```text
ResolvedQueryType
├── type: IntegerType
└── nullability: NULLABLE
```

---

# 35. Nullability states

Podrán existir:

```text
NON_NULL
NULLABLE
UNKNOWN
```

---

# 36. Why UNKNOWN nullability

Durante análisis parcial puede no conocerse todavía.

---

# 37. BooleanType

Representa:

```text
TRUE
FALSE
```

y operaciones booleanas semánticas.

No define representación física.

---

# 38. SQL three-valued logic

Los predicados SQL pueden producir:

```text
TRUE
FALSE
UNKNOWN
```

por presencia de NULL.

Esto deberá modelarse semánticamente.

---

# 39. Predicate result type

Conceptualmente:

```text
BooleanType
+
SQL nullability / truth semantics
```

según expresión.

---

# 40. Integer types

El core podrá distinguir:

```text
IntegerType
BigIntegerType
```

y posiblemente:

```text
SmallIntegerType
```

si aporta valor semántico.

---

# 41. Avoid platform leakage

No introducir en el core:

```text
TINYINT
MEDIUMINT
INT8
SERIAL
```

como tipos semánticos portables.

---

# 42. Unsigned

`UNSIGNED` deberá tratarse cuidadosamente.

Puede ser:

```text
Platform Type Constraint
```

o metadata especializada.

No todos los motores lo soportan igual.

---

# 43. DecimalType

Conceptualmente:

```text
DecimalType
├── precision?
└── scale?
```

---

# 44. Decimal semantics

Debe preservar precisión exacta.

No deberá convertirse silenciosamente a `float`.

---

# 45. FloatType

Representa números de punto flotante aproximados.

---

# 46. Decimal ≠ Float

```text
Decimal
≠
Floating Point
```

Esta distinción deberá mantenerse en toda la pipeline.

---

# 47. Numeric hierarchy

Conceptualmente:

```text
Numeric
├── Integer
│   ├── Integer
│   └── BigInteger
│
├── Decimal
└── Float
```

La implementación no necesita usar herencia.

---

# 48. Numeric promotion

Ejemplo conceptual:

```text
Integer + Integer
→ Integer / promoted Integer

Integer + Decimal
→ Decimal

Decimal + Float
→ platform/semantic rule
```

Las reglas serán explícitas.

---

# 49. Overflow semantics

VoltStack no deberá inventar semántica de overflow distinta al target sin declararlo.

---

# 50. Numeric constraints

Podrán incluir:

```text
precision
scale
signedness
range
```

cuando sean relevantes.

---

# 51. StringType

Representa texto semántico.

Puede contener:

```text
length?
encoding semantics?
collation semantics?
```

pero deberá evitar acoplarse a un vendor.

---

# 52. TextType

Puede representar texto de longitud no limitada semánticamente.

---

# 53. String vs Text

La diferencia puede ser útil principalmente para:

- schema mapping;
- storage;
- constraints;
- indexing.

En Query expressions podrán compartir muchas operaciones.

---

# 54. Collation

Collation no será simplemente propiedad de `StringType`.

Puede depender de:

- column metadata;
- expression;
- platform;
- explicit query clause.

---

# 55. Character set

Igualmente:

```text
Query String Semantics
≠
Physical Character Set
```

---

# 56. BinaryType

Representa bytes.

No deberá confundirse con StringType.

---

# 57. Binary comparison

Las reglas de comparación podrán diferir de texto.

---

# 58. UuidType

`UuidType` será un tipo semántico portable.

---

# 59. UUID representation

Podrá mapearse a:

```text
PostgreSQL native UUID
CHAR/VARCHAR
BINARY
BLOB
```

según Platform/Configuration.

---

# 60. UUID semantics remain stable

Aunque la representación cambie:

```text
UuidType
```

permanece.

---

# 61. DateType

Representa:

```text
calendar date
```

sin tiempo.

---

# 62. TimeType

Representa:

```text
time of day
```

---

# 63. DateTimeType

Representa fecha y hora sin asumir automáticamente timezone-aware semantics.

---

# 64. DateTimeTzType

Representa un instante/fecha-hora con semántica de zona temporal cuando sea necesario.

---

# 65. Temporal distinctions

```text
Date
≠
Time
≠
DateTime
≠
DateTimeTz
```

---

# 66. PHP DateTime does not decide query type

Un:

```php
DateTimeImmutable
```

podría mapearse a:

```text
DateType
DateTimeType
DateTimeTzType
```

según metadata/contexto.

---

# 67. Timezone normalization

Será responsabilidad del Type Conversion System, no del AST.

---

# 68. IntervalType

Representará duraciones/intervalos semánticos cuando el target lo permita.

---

# 69. Interval capability

No todos los motores soportan intervalos con la misma semántica.

Deberá ser capability-aware.

---

# 70. JsonType

Representa un documento/valor JSON.

---

# 71. JSON ≠ String

Aunque se almacene como texto:

```text
JsonType
≠
StringType
```

---

# 72. JSON representation

Puede ser:

```text
PostgreSQL json
PostgreSQL jsonb
MySQL JSON
MariaDB JSON-compatible representation
SQLite TEXT/BLOB representation
```

sin cambiar la intención semántica.

---

# 73. Json storage strategy

Podrá existir metadata:

```text
JsonStorageStrategy
```

separada de `JsonType`.

---

# 74. JSON null

Deberá distinguirse:

```text
SQL NULL
```

de:

```text
JSON null
```

---

# 75. JSON scalar types

El sistema podrá conocer semánticamente:

```text
JSON_STRING
JSON_NUMBER
JSON_BOOLEAN
JSON_NULL
JSON_OBJECT
JSON_ARRAY
```

cuando una operación JSON lo requiera.

---

# 76. JSON path result

Una expresión JSON puede devolver:

```text
JsonType
StringType
NumericType
BooleanType
UnknownType
```

dependiendo de la operación.

---

# 77. EnumType

Conceptualmente:

```text
EnumType
├── enumClass?
├── values
├── backingType
└── storageStrategy
```

---

# 78. Enum semantics

Un enum de aplicación no equivale automáticamente a un enum nativo de base de datos.

---

# 79. Enum strategies

Podrán incluir:

```text
BACKED_VALUE
NATIVE_DATABASE_ENUM
STRING_CONSTRAINT
INTEGER_CONSTRAINT
CUSTOM
```

---

# 80. Enum compatibility

Dos enums con backing type string no serán automáticamente compatibles.

---

# 81. DomainType

Permite preservar tipos semánticos de dominio.

Ejemplo:

```text
DomainType<UserId>
├── semantic identity: app.user_id
└── underlying type: BigInteger
```

---

# 82. Domain types

Ejemplos:

```text
UserId
OrderId
TenantId
Money
EmailAddress
PhoneNumber
CountryCode
Percentage
```

---

# 83. UserId ≠ OrderId

Aunque ambos utilicen:

```text
BIGINT
```

físicamente:

```text
UserIdType
≠
OrderIdType
```

semánticamente.

---

# 84. Domain compatibility

La compatibilidad podrá definirse:

```text
STRICT
UNDERLYING_COMPATIBLE
EXPLICIT_COERCION
CUSTOM
```

---

# 85. Strict domain identifiers

Por defecto, IDs de dominios diferentes no deberían compararse accidentalmente.

Ejemplo:

```text
users.id = orders.id
```

si uno es `UserIdType` y otro `OrderIdType`, puede requerir diagnóstico.

---

# 86. MoneyType

`Money` puede requerir:

```text
amount
currency
```

por lo que podría ser:

- scalar domain type;
- tuple;
- structured type;
- mapped Value Object.

La arquitectura deberá permitir las tres estrategias.

---

# 87. CollectionType

Conceptualmente:

```text
CollectionType<T>
```

---

# 88. CollectionType ≠ database array

```text
CollectionType<Integer>
```

no significa:

```text
PostgreSQL INTEGER[]
```

automáticamente.

---

# 89. Collection semantic use

Puede utilizarse para:

```text
IN
ANY
bulk values
function arguments
array operators
```

dependiendo del contexto.

---

# 90. Collection cardinality

No forma parte necesariamente del tipo.

Pertenece normalmente al:

```text
BindingShape
```

---

# 91. TupleType

Conceptualmente:

```text
TupleType<T1, T2, ...>
```

---

# 92. Tuple example

```text
Tuple<UserIdType, StatusType>
```

---

# 93. Tuple arity

Forma parte del tipo.

---

# 94. Tuple compatibility

Requiere:

```text
same arity
+
compatible corresponding elements
```

---

# 95. StructuredType

Permitirá tipos compuestos futuros.

Ejemplos:

```text
record
row
composite
domain structure
```

---

# 96. Structured type caution

No deberá utilizarse como:

```text
mixed array
```

para evitar tipado.

---

# 97. PlatformSpecificType

Podrá existir como escape hatch controlado.

Ejemplos:

```text
PostgreSqlTsVectorType
PostgreSqlInetType
PostgreSqlRangeType
```

---

# 98. Platform-specific types

Deberán declarar:

```text
portability = PLATFORM_SPECIFIC
```

---

# 99. ExtensionType

Paquetes podrán registrar tipos personalizados.

Ejemplo:

```text
postgis.geometry
postgresql.inet
app.encrypted_email
```

---

# 100. Extension requirements

Un tipo extendido deberá poder declarar:

- QueryType descriptor;
- PHP/domain mapping;
- database representation;
- platform mappings;
- converter;
- binder requirements;
- expression/operator support;
- schema mapping;
- hydration mapping.

---

# 101. QueryTypeRegistry

Ejemplo conceptual:

```php
interface QueryTypeRegistry
{
    public function has(QueryTypeId $id): bool;

    public function get(QueryTypeId $id): QueryTypeDescriptor;
}
```

---

# 102. TypeResolver

Responsabilidad:

```text
Type Evidence
+
Context
+
Constraints
→ ResolvedQueryType
```

---

# 103. Type evidence

Puede provenir de:

```text
Explicit Type
Schema Metadata
ORM Metadata
Expression Signature
Function Signature
Operator Signature
Parameter Constraint
Literal
Domain Mapping
Runtime Value
Extension
```

---

# 104. Evidence priority

Una posible prioridad:

```text
Explicit Semantic Type
        │
        ▼
Schema / ORM Metadata
        │
        ▼
Expression / Function / Operator Constraints
        │
        ▼
Parameter Context
        │
        ▼
Safe Literal Inference
        │
        ▼
Safe Runtime Inference
        │
        ▼
Unknown / Error
```

---

# 105. Explicit type

Ejemplo:

```text
parameter p1
declared as UuidType
```

no deberá degradarse a StringType.

---

# 106. Schema metadata

Ejemplo:

```text
users.id
→ UserIdType
```

entonces:

```text
users.id = p1
```

impone:

```text
p1 compatible with UserIdType
```

---

# 107. Literal inference

Literal:

```text
42
```

puede inferirse inicialmente como:

```text
IntegerLiteralType
```

o `IntegerType`.

---

# 108. Literal refinement

El contexto podrá refinarlo.

Ejemplo:

```text
user_id = 42
```

puede tratar el literal como compatible con:

```text
UserIdType
```

sin cambiar su valor.

---

# 109. Runtime inference

Será el fallback más débil.

---

# 110. Runtime string ambiguity

Un string no permite inferir de forma segura:

```text
UUID
JSON
DateTime
Enum
Email
```

sin metadata.

---

# 111. Type inference

El sistema deberá soportar inferencia contextual.

---

# 112. Example — comparison

```text
Column<UserId>
=
Parameter<Unknown>
```

produce:

```text
Parameter<UserId>
```

---

# 113. Example — arithmetic

```text
Integer
+
Decimal
```

puede producir:

```text
Decimal
```

según reglas de promoción.

---

# 114. Example — function

```text
LOWER(String)
→ String
```

---

# 115. Example — aggregate

```text
COUNT(...)
→ Integer/BigInteger semantic result
```

según especificación semántica.

---

# 116. Example — SUM

```text
SUM(Integer)
```

puede requerir promoción para evitar overflow.

La regla deberá definirse semánticamente y luego adaptarse al Platform.

---

# 117. FunctionSignature

Las funciones deberán declarar signatures.

Conceptualmente:

```text
FunctionSignature
├── argumentTypes
├── returnTypeRule
├── variadic
├── nullabilityRule
└── capabilityRequirements
```

---

# 118. OperatorSignature

Igualmente:

```text
OperatorSignature
├── leftTypeConstraint
├── rightTypeConstraint
├── resultTypeRule
└── coercionRules
```

---

# 119. Predicate type rules

Predicados deberán validar compatibilidad.

Ejemplo:

```text
Integer = DateTime
```

deberá ser inválido salvo coerción explícita válida.

---

# 120. LIKE

Conceptualmente:

```text
LIKE(String, String)
→ Boolean
```

No aceptar automáticamente cualquier tipo.

---

# 121. IN

```text
IN(
    T,
    Collection<T-compatible>
)
→ Boolean
```

---

# 122. BETWEEN

```text
BETWEEN(
    T,
    T-compatible,
    T-compatible
)
→ Boolean
```

---

# 123. Arithmetic operators

Ejemplo:

```text
ADD(Numeric, Numeric)
→ NumericPromotion(L, R)
```

---

# 124. String concatenation

Debe modelarse como operación semántica:

```text
Concat(String, String)
→ String
```

El Dialect decidirá:

```text
||
CONCAT(...)
```

u otra sintaxis.

---

# 125. Type compatibility

VoltStack necesitará:

```text
TypeCompatibilityResolver
```

---

# 126. Compatibility outcomes

Conceptualmente:

```text
EXACT
COMPATIBLE
PROMOTABLE
COERCIBLE
PLATFORM_DEPENDENT
INCOMPATIBLE
UNKNOWN
```

---

# 127. Exact

```text
Integer
vs
Integer
→ EXACT
```

---

# 128. Compatible

Ejemplo:

```text
UserId
vs
Integer literal
```

podría ser compatible bajo una regla controlada.

---

# 129. Promotable

Ejemplo:

```text
Integer
→ Decimal
```

---

# 130. Coercible

Requiere una conversión semántica explícita o segura.

---

# 131. Platform dependent

Ejemplo de feature/tipo cuya compatibilidad sólo puede resolverse al conocer el target.

---

# 132. Incompatible

Ejemplo:

```text
DateTime
+
JsonObject
```

sin operación registrada.

---

# 133. Unknown

No existe suficiente información.

---

# 134. Coercion

`TypeCoercion` significa:

```text
T1
→
T2
```

con semántica definida.

---

# 135. Coercion ≠ arbitrary cast

No deberá utilizarse para “hacer funcionar” cualquier query.

---

# 136. Implicit coercion

Deberá ser conservadora.

---

# 137. Explicit coercion

El usuario podrá solicitar:

```text
CAST(expression AS type)
```

mediante un AST semántico.

---

# 138. CastExpression

Conceptualmente:

```text
CastExpression
├── expression
└── targetQueryType
```

No contendrá SQL como:

```text
CAST(x AS VARCHAR)
```

---

# 139. Compiler mapping

El Compiler + Platform + Dialect decidirán la declaración física del target.

---

# 140. Coercion categories

```text
IMPLICIT_SAFE
IMPLICIT_LOSSLESS
EXPLICIT_SAFE
EXPLICIT_LOSSY
PLATFORM_SPECIFIC
UNSUPPORTED
```

---

# 141. Lossy coercion

Nunca deberá ocurrir silenciosamente.

---

# 142. String → Integer

No deberá ser una coerción universal automática.

---

# 143. Integer → String

Igualmente puede cambiar semántica de comparación/ordenamiento.

No asumir.

---

# 144. Decimal → Float

Es potencialmente lossy.

---

# 145. DateTime → String

No deberá utilizarse como fallback genérico.

---

# 146. TypePromotion

La promoción es diferente de coerción.

```text
Promotion
=
choose common semantic type
```

---

# 147. Numeric promotion

Ejemplo:

```text
Integer + Decimal
→ Decimal
```

---

# 148. CommonTypeResolver

Podrá existir:

```text
CommonTypeResolver
```

para:

- CASE;
- COALESCE;
- UNION;
- VALUES;
- arrays;
- tuples;
- arithmetic.

---

# 149. CASE expression

```text
CASE
WHEN predicate THEN Integer
ELSE Decimal
END
```

requiere:

```text
CommonType(Integer, Decimal)
→ Decimal
```

---

# 150. COALESCE

```text
COALESCE(T1, T2, ...)
→ CommonType(...)
```

con nullability específica.

---

# 151. UNION

Las proyecciones correspondientes deberán ser compatibles.

---

# 152. UNION example

```text
SELECT Integer
UNION
SELECT Decimal
```

puede requerir un common type.

---

# 153. VALUES

Filas:

```text
(1, 'a')
(2, 'b')
```

producen tipos por columna.

---

# 154. Tuple common type

Para:

```text
Tuple<A, B>
Tuple<C, D>
```

resolver:

```text
Tuple<Common(A,C), Common(B,D)>
```

si es posible.

---

# 155. Null type unification

```text
CommonType(Integer, Null)
→ Nullable<Integer>
```

---

# 156. Unknown type unification

```text
CommonType(Integer, Unknown)
```

puede permanecer parcialmente resuelto hasta tener más evidencia.

---

# 157. TypeConstraint

El Semantic Engine utilizará constraints.

Conceptualmente:

```text
TypeConstraint
```

---

# 158. Constraint examples

```text
MustBeNumeric
MustBeString
MustBeComparableWith(T)
MustBeAssignableTo(T)
MustHaveCommonTypeWith(T)
MustBeCollectionOf(T)
MustBeTupleOfArity(N)
MustSupportOperator(O)
MustSupportFunction(F)
```

---

# 159. Constraint sources

Podrán provenir de:

- columns;
- operators;
- functions;
- predicates;
- assignments;
- joins;
- unions;
- insert targets;
- update targets;
- parameters.

---

# 160. Constraint graph

Conceptualmente:

```text
Expression A
     │
     ├── constraint
     ▼
Type Variable X
     │
     ├── compatible
     ▼
Type Variable Y
     │
     ▼
Resolved Type
```

---

# 161. TypeVariable

Durante Semantic Analysis podrán utilizarse:

```text
TypeVariable
```

para nodos aún no resueltos.

---

# 162. Type variable example

```text
Parameter p1
→ T1

users.id
→ UserIdType

Constraint:
T1 compatible with UserIdType

Resolve:
T1 = UserIdType
```

---

# 163. Constraint solver

Podrá existir:

```text
QueryTypeConstraintSolver
```

---

# 164. Solver responsibility

Resolver:

```text
type variables
constraints
common types
nullability
coercions
```

---

# 165. Solver must not

No deberá:

- generate SQL;
- inspect PDO;
- open connection;
- execute queries;
- hydrate entities.

---

# 166. Type certainty

Cada resolución podrá registrar:

```text
EXPLICIT
INFERRED
DERIVED
PLATFORM_RESOLVED
FALLBACK
UNKNOWN
```

---

# 167. Why certainty matters

Ayuda a:

- diagnostics;
- strict mode;
- extension debugging;
- schema mismatch detection.

---

# 168. Type origin

Podrá registrar:

```text
SCHEMA
ORM_METADATA
PARAMETER
LITERAL
FUNCTION
OPERATOR
CAST
EXTENSION
PLATFORM
```

---

# 169. Type provenance

Ejemplo:

```text
p1
Resolved Type: UserId
Origin: users.id
Certainty: INFERRED
```

---

# 170. Semantic type table

El resultado del análisis podrá contener:

```text
QueryTypeTable
```

---

# 171. QueryTypeTable

Conceptualmente:

```text
QueryTypeTable
├── AstNodeId → ResolvedQueryType
├── ParameterId → ResolvedQueryType
├── ProjectionId → ResolvedQueryType
└── SymbolId → ResolvedQueryType
```

---

# 172. AST remains immutable

No escribir:

```text
$node->resolvedType = ...
```

---

# 173. Correct

```text
AST Node
+
Semantic Type Table
```

---

# 174. Why external table

Permite:

- AST immutability;
- multiple semantic analyses;
- different platform contexts;
- easier caching;
- parallel analysis;
- clean transformations.

---

# 175. Type annotations

Si se requieren, serán artifacts separados.

---

# 176. Expression type

Cada ExpressionNode podrá recibir un resolved type externo.

---

# 177. Predicate type

Cada PredicateNode tendrá semántica booleana, pero podrá tener metadata adicional sobre nullability/truth behavior.

---

# 178. ColumnReference type

Se resolverá mediante Schema/Symbol Resolution.

---

# 179. Parameter type

Se resolverá mediante constraints.

---

# 180. Literal type

Se inferirá inicialmente y podrá refinarse.

---

# 181. Function result type

Se derivará de `FunctionSignature`.

---

# 182. Aggregate result type

Se derivará de una regla específica.

---

# 183. Subquery scalar type

Una scalar subquery deberá producir exactamente una columna semántica.

---

# 184. Scalar subquery

```text
SELECT MAX(age)
```

podrá tener:

```text
Integer?
```

dependiendo de nullability.

---

# 185. EXISTS

```text
EXISTS(subquery)
→ Boolean
```

---

# 186. IN subquery

Debe comparar:

```text
left type
```

contra:

```text
subquery projection type
```

---

# 187. Row subquery

Podrá producir `TupleType`.

---

# 188. Projection typing

Cada projection tendrá tipo resuelto.

---

# 189. Result metadata

Los tipos de proyección podrán alimentar posteriormente:

```text
Result Metadata
Hydration Plan
Scalar Conversion
ORM Hydration
```

---

# 190. Query Type System and Result System

El mismo tipo semántico puede utilizarse para:

```text
input conversion
```

y:

```text
result conversion
```

pero las pipelines serán diferentes.

---

# 191. Input pipeline

```text
Domain Value
→ Query Type
→ Database Representation
→ Driver Binding
```

---

# 192. Output pipeline

```text
Native Database Value
→ Platform Type Mapping
→ Query/Result Type
→ PHP/Domain Value
```

---

# 193. Symmetry not guaranteed

La conversión input/output puede no ser perfectamente simétrica.

Ejemplo:

```text
database timestamp
→ DateTimeImmutable
```

mientras:

```text
DateTimeInterface
→ normalized database string
```

---

# 194. Query Type vs ORM Type

Idealmente habrá una infraestructura de tipos compartida o interoperable.

Pero:

```text
Query Type
```

no deberá depender de EntityManager.

---

# 195. ORM mapping

ORM metadata puede decir:

```text
User::$id
→ UserIdType
```

y alimentar el Query Type System.

---

# 196. Query Engine independent

El Query Engine también debe funcionar sin ORM.

---

# 197. Schema integration

Schema metadata podrá declarar:

```text
Column
├── semanticType
├── nativeTypeMetadata
└── nullability
```

---

# 198. Semantic vs native schema type

Ejemplo:

```text
semantic:
UuidType

native:
CHAR(36)
```

---

# 199. Introspection

Una base existente puede proporcionar sólo:

```text
native:
VARCHAR(36)
```

No siempre puede inferirse:

```text
UuidType
```

sin mapping adicional.

---

# 200. Unknown semantic mapping

Debe conservarse honestamente.

No inventar significado de dominio.

---

# 201. PlatformTypeMapping

Contrato conceptual:

```text
Query Semantic Type
+
Resolved Platform
+
Context
→ Platform Database Type
```

---

# 202. Platform type examples

```text
UuidType
→ PostgreSQL UUID

UuidType
→ MySQL BINARY(16)

UuidType
→ SQLite BLOB
```

según configuración.

---

# 203. Type mapping context

Puede incluir:

```text
QUERY_PARAMETER
QUERY_RESULT
SCHEMA_COLUMN
CAST_TARGET
FUNCTION_ARGUMENT
MIGRATION
```

---

# 204. Mapping may differ by context

Un tipo podría necesitar distinta representación para:

- schema declaration;
- query binding;
- result hydration.

---

# 205. DatabaseRepresentation

Debe distinguirse del tipo físico declarado.

Ejemplo:

```text
Platform Type:
UUID

Database Representation:
"550e8400-e29b-41d4-a716-446655440000"
```

---

# 206. DriverBindingType

Después:

```text
STRING
```

para un driver específico.

---

# 207. Query type portability

Cada tipo podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_MAPPING
CAPABILITY_DEPENDENT
PLATFORM_SPECIFIC
EXTENSION_SPECIFIC
```

---

# 208. Portable type

Ejemplo:

```text
IntegerType
StringType
BooleanType
```

aunque su representación física varíe.

---

# 209. Capability-dependent

Ejemplo:

```text
JsonType
```

para operaciones avanzadas.

Almacenar JSON y consultar JSON son capabilities distintas.

---

# 210. Platform-specific

Ejemplo:

```text
PostgreSqlTsVectorType
```

---

# 211. Type capability requirements

Un tipo puede requerir capabilities para:

- storage;
- comparison;
- ordering;
- indexing;
- function operations;
- casts;
- binding;
- hydration.

---

# 212. Type existence ≠ operation support

Que un tipo pueda almacenarse no significa que soporte todas las operaciones.

---

# 213. Example JSON

```text
JsonType storage
→ supported

Json path query
→ maybe supported

Json containment
→ maybe supported

Json ordering
→ maybe unsupported
```

---

# 214. TypeOperationSupport

Podrá existir:

```text
TypeOperationSupport
```

para consultar compatibilidad semántica.

---

# 215. Operations

Ejemplos:

```text
EQUALITY
ORDERING
ARITHMETIC
CONCAT
PATTERN_MATCH
CONTAINMENT
INDEXING
JSON_PATH
ARRAY_ACCESS
CAST
```

---

# 216. Operator Registry integration

El Operator System podrá declarar:

```text
operator
+
type signatures
```

---

# 217. Function Registry integration

Igualmente funciones.

---

# 218. Type system should not own every function

Evitar un God Type System.

Funciones y operadores registran signatures; el Type System las resuelve.

---

# 219. Comparison semantics

Comparabilidad deberá ser explícita.

---

# 220. Equality compatibility

No implica ordering compatibility.

Ejemplo:

```text
JSON
```

puede soportar ciertas comparaciones pero no ordering portable.

---

# 221. OrderableType

Puede representarse mediante capability/trait descriptor, no necesariamente interfaz PHP.

---

# 222. NumericType

Igualmente, una característica:

```text
numeric = true
```

puede ser mejor que herencia profunda.

---

# 223. Type characteristics

Ejemplos:

```text
numeric
integral
exact
approximate
textual
binary
temporal
comparable
orderable
collection
structured
domain
```

---

# 224. Composition over inheritance

Preferir descriptors y capabilities del tipo frente a:

```text
IntegerType extends NumericType extends ScalarType...
```

si genera jerarquías rígidas.

---

# 225. TypeFamily

Podrá existir como clasificación semántica.

Ejemplos:

```text
NUMERIC
TEXTUAL
TEMPORAL
COLLECTION
```

---

# 226. TypeFamily ≠ PHP inheritance

Es metadata semántica.

---

# 227. Type aliases

Podrán existir aliases:

```text
int → core.integer
bool → core.boolean
uuid → core.uuid
```

pero separados de IDs canónicos.

---

# 228. Alias collision

Deberá detectarse durante bootstrap.

---

# 229. Custom type registration

Ejemplo conceptual:

```php
$types->register(
    new QueryTypeDescriptor(
        id: new QueryTypeId('app.user_id'),
        category: QueryTypeCategory::DOMAIN,
    ),
);
```

---

# 230. Extension lifecycle

```text
Discover Extension
      │
      ▼
Register Type Descriptor
      │
      ▼
Register Mappings
      │
      ▼
Register Converters
      │
      ▼
Register Signatures
      │
      ▼
Validate
      │
      ▼
Freeze
```

---

# 231. No runtime arbitrary registration

No registrar tipos nuevos en medio de una request.

---

# 232. Persistent runtime safety

Registries congelados serán:

```text
immutable
thread-safe
coroutine-safe
worker-safe
```

---

# 233. Type converters

Conversión deberá delegarse a servicios especializados.

---

# 234. Input converter

```text
Domain/PHP Value
→ Database Representation
```

---

# 235. Output converter

```text
Database Representation
→ PHP/Domain Value
```

---

# 236. Converter must be deterministic

Cuando sea posible.

---

# 237. Converter context

Podrá recibir:

```text
ResolvedQueryType
ResolvedPlatform
ConversionPolicy
```

sin recibir EntityManager.

---

# 238. Stateless converters

Preferidos.

---

# 239. Stateful conversion

Si una extensión necesita estado request-scoped deberá declararlo explícitamente.

---

# 240. No container lookup

Converters no deberán hacer:

```php
app(...)
```

o equivalente.

Dependencias explícitas.

---

# 241. Value validation

Antes de convertir podrá validarse:

```text
runtime value compatible with semantic type
```

---

# 242. Example UUID

```text
UuidType
+
"not-a-uuid"
→ conversion error
```

antes de Driver binding.

---

# 243. Example enum

```text
StatusType
+
"invalid"
→ EnumConversionException
```

---

# 244. Example decimal

Validar precision/scale cuando sea necesario.

---

# 245. Type validation ≠ business validation

Ejemplo:

```text
age >= 18
```

es business/domain validation.

El Type System sólo verifica que `age` sea compatible con su tipo.

---

# 246. Validation subsystem separation

```text
Quantum/Validation
≠
Database Query Type Validation
```

---

# 247. Query Type validation

Verifica:

- structural type compatibility;
- operator compatibility;
- function signatures;
- conversions;
- parameter types;
- projection compatibility.

---

# 248. Security

El Type System deberá contribuir a seguridad evitando:

- ambiguous conversions;
- unsafe casts;
- binary/text confusion;
- malformed JSON;
- invalid UUID;
- lossy silent conversion.

---

# 249. Type metadata must not contain secrets

Descriptors y resolved types nunca deberán almacenar valores runtime.

---

# 250. Error messages

Ejemplo:

```text
Query type mismatch.

Expression:
users.created_at = :p1

Expected:
DateTimeType

Received:
IntegerType

Parameter:
p1
```

---

# 251. Sensitive values

No incluir:

```text
p1 = secret-value
```

por defecto.

---

# 252. TypeMismatchException

Jerarquía conceptual:

```text
QueryTypeException
├── UnknownQueryTypeException
├── DuplicateQueryTypeException
├── TypeResolutionException
├── TypeInferenceException
├── TypeConstraintException
├── TypeMismatchException
├── TypeCompatibilityException
├── TypeCoercionException
├── LossyTypeCoercionException
├── UnsupportedTypeOperationException
├── UnsupportedPlatformTypeException
├── TypeConversionException
└── TypeExtensionException
```

---

# 253. Type diagnostics

El sistema podrá explicar:

```text
Expression:
p1

Initial Type:
Unknown

Constraint:
ComparableWith(users.id)

users.id:
UserIdType

Resolved:
UserIdType

Certainty:
INFERRED
```

---

# 254. Explain mode

Una API de diagnóstico podría producir:

```text
database:query:type-explain
```

o integrarse al query explain framework.

---

# 255. Query type fingerprint

Podrá existir:

```text
QueryTypeFingerprint
```

---

# 256. Fingerprint inputs

Puede incluir:

- type IDs;
- parameters;
- nullability;
- relevant constraints;
- extension versions.

No runtime values.

---

# 257. Compilation cache

El compiled query cache podrá depender de tipos cuando estos cambien SQL o binding.

---

# 258. Example

```text
p1 Integer
```

y:

```text
p1 Json
```

pueden requerir diferentes:

- casts;
- placeholders;
- conversions;
- bindings.

---

# 259. Type cache separation

Distinguir:

```text
Type Descriptor Cache
Semantic Type Resolution Cache
Platform Type Mapping Cache
Conversion Plan Cache
```

---

# 260. No universal type cache

Cada cache tendrá ownership y invalidation claros.

---

# 261. Type resolution cache

Sólo cachear cuando las entradas sean deterministas.

---

# 262. Platform mapping cache

Key conceptual:

```text
QueryTypeFingerprint
+
PlatformFingerprint
+
CapabilityFingerprint
+
MappingPolicyFingerprint
```

---

# 263. Extension fingerprint

Si extensiones modifican mappings:

```text
ExtensionGraphFingerprint
```

deberá participar.

---

# 264. Performance

El sistema deberá evitar:

- reflection por parámetro;
- reconstrucción de descriptors;
- resolución repetida de aliases;
- parsing repetido de enum metadata;
- conversion plan recreation.

---

# 265. Compiled type metadata

En producción podrá precompilarse:

```text
ORM Property
→ QueryTypeId
→ Platform Mapping
→ Conversion Plan
```

cuando sea seguro.

---

# 266. Reflection

Podrá utilizarse durante:

```text
metadata compilation
```

pero deberá minimizarse en hot path.

---

# 267. Type resolution complexity

La resolución común debería aproximarse a:

```text
O(1)
```

para lookups directos y reglas precompiladas.

---

# 268. Constraint solving

Consultas complejas pueden requerir análisis adicional, pero deberá ser determinista.

---

# 269. No network I/O

El Query Type Solver no deberá hacer network I/O.

---

# 270. Platform discovery

Si una decisión requiere server version/capabilities, deberá recibir un `ResolvedDatabaseTarget` o `CapabilitySnapshot`.

No abrir conexión por sí mismo.

---

# 271. Offline mode

El sistema deberá permitir análisis/compilación offline con:

```text
Target Platform
+
Target Version
+
Capability Profile
```

---

# 272. Online mode

Podrá utilizar capabilities descubiertas del servidor.

---

# 273. Same AST, different target

Ejemplo:

```text
UuidType
```

puede producir diferentes mappings para:

```text
PostgreSQL
MySQL
MariaDB
SQLite
```

sin modificar el AST.

---

# 274. Type System and Optimizer

El Optimizer podrá utilizar información de tipos para transformaciones seguras.

---

# 275. Example optimizer

```text
IntegerExpression + IntegerLiteral(0)
```

podría simplificarse cuando la semántica lo permita.

---

# 276. Type-preserving transformations

Toda optimization deberá preservar:

```text
semantic result type
```

salvo transformación explícitamente equivalente.

---

# 277. Optimizer cannot invent casts

No deberá introducir casts arbitrarios para hacer válida una consulta inválida.

---

# 278. Type System and Planner

El Planner podrá usar tipos para:

- binding strategies;
- result strategies;
- native arrays;
- tuple handling;
- JSON operations;
- casts;
- bulk operations.

---

# 279. Type System and Compiler

El Compiler recibe tipos ya resueltos.

No deberá ser el principal Type Resolver.

---

# 280. Compiler defensive validation

Podrá verificar invariantes.

Ejemplo:

```text
cannot compile unresolved required type
```

---

# 281. Type System and Driver

Driver recibe:

```text
Database Representation
+
Driver Binding Type
```

No `DomainType`.

---

# 282. Driver isolation

Driver no deberá importar:

```text
ORM\Entity
Query\Type\DomainType
Application\UserId
```

---

# 283. Type System and Hydration

Hydration podrá consumir:

```text
Result Type Metadata
```

para convertir resultados.

---

# 284. Hydration separation

Query Type System no instancia entidades.

---

# 285. ORM Hydrator

Podrá convertir:

```text
database value
→ UserId
```

usando Type conversion metadata.

---

# 286. Identity Map

No forma parte del Type System.

---

# 287. UnitOfWork

No forma parte del Type System.

---

# 288. Type System and Persistence

Persistence Planner utiliza ORM metadata para producir:

```text
Query Parameters
+
Query Types
```

---

# 289. Optimistic locking

Ejemplo:

```text
version
→ VersionType
```

puede ser un DomainType basado en integer.

---

# 290. Tenant IDs

`TenantIdType` puede ser un DomainType.

No requiere dependencia del paquete Multitenancy.

---

# 291. Authentication IDs

Igualmente:

```text
UserIdType
SessionIdType
TokenIdType
```

pueden existir sin acoplar Database a Authentication.

---

# 292. Cross-package type registration

Paquetes oficiales podrán registrar tipos mediante Extension API.

---

# 293. Query type extension point

Contrato estable y estrecho.

---

# 294. Extension cannot mutate core type

Una extensión no deberá cambiar silenciosamente:

```text
core.integer
```

por otra semántica.

---

# 295. Explicit decoration

Si se permite decoración, deberá ser declarada y validada.

---

# 296. Replacement

Reemplazar un tipo core requerirá política explícita y probablemente no estará permitido en V1.

---

# 297. Type aliases can be extended

Sin cambiar el ID canónico.

---

# 298. Platform type extensions

Ejemplo:

```text
postgresql.inet
postgresql.cidr
postgresql.range
postgis.geometry
```

---

# 299. Platform-specific function signatures

Podrán registrar tipos adicionales sin contaminar el core portable.

---

# 300. Capability integration

Ejemplo:

```text
JsonType
+
JSON_CONTAINS operation
```

requiere:

```text
query.json.contains
```

---

# 301. Capability failure

Si el tipo existe pero la operación no:

```text
CapabilityNotSupportedException
```

o un error semántico especializado deberá ocurrir antes de ejecución cuando sea posible.

---

# 302. Type error vs capability error

Distinguir:

```text
Integer LIKE Integer
→ Type Error
```

de:

```text
JsonContains(Json, Json)
on platform without support
→ Capability Error
```

---

# 303. Type error vs conversion error

Distinguir:

```text
StringType compared with DateTimeType
→ semantic type error
```

de:

```text
DateTimeType parameter receives invalid DateTime value
→ conversion error
```

---

# 304. Type error vs driver binding error

Distinguir:

```text
semantic compatibility failure
```

de:

```text
native client cannot bind converted representation
```

---

# 305. Strict mode

VoltStack podrá ofrecer:

```text
TypeStrictness
```

---

# 306. Possible modes

```text
STRICT
BALANCED
COMPATIBILITY
```

pero las diferencias deberán estar muy limitadas.

---

# 307. Strict mode

Rechaza coerciones ambiguas y unknown types importantes.

---

# 308. Balanced mode

Permite coerciones seguras bien definidas.

---

# 309. Compatibility mode

Podría permitir algunas conversiones heredadas explícitamente documentadas.

Nunca deberá desactivar seguridad.

---

# 310. Production recommendation

```text
BALANCED
```

o `STRICT` para aplicaciones sensibles.

---

# 311. No magic casting mode

No ofrecer:

```text
LOOSE_EVERYTHING
```

---

# 312. SQL implicit casts

Que el servidor realice implicit casts no significa que VoltStack deba depender de ellos.

---

# 313. Why avoid server magic

Puede variar por:

- vendor;
- version;
- collation;
- configuration;
- operator;
- index usage.

---

# 314. Explicit semantic control

VoltStack deberá conocer cuándo una coerción forma parte de la semántica.

---

# 315. Index performance

Un cast incorrecto puede impedir uso de índices.

El Planner podrá diagnosticarlo.

---

# 316. Type-aware performance diagnostics

Ejemplo:

```text
Comparison requires cast of indexed column.

Column:
users.id

From:
UserIdType

To:
StringType

Potential index impact.
```

---

# 317. Cast direction

Preferir convertir parámetro al tipo de la columna cuando sea semánticamente seguro, en vez de castear la columna.

---

# 318. Example

Preferible conceptualmente:

```text
column<UserId> = parameter<UserId>
```

sobre:

```text
CAST(column AS string) = parameter<string>
```

---

# 319. Type-aware binding

La resolución temprana del tipo del parámetro ayuda a evitar casts SQL innecesarios.

---

# 320. Result typing

El Query Type System deberá producir metadata suficiente para conocer tipos de proyección.

---

# 321. Select example

```text
SELECT
    id,
    name,
    COUNT(*) AS orders
```

Semantic projection:

```text
id     → UserIdType
name   → StringType
orders → BigIntegerType
```

---

# 322. Alias does not change type

```text
COUNT(*) AS total
```

continúa siendo el tipo semántico de COUNT.

---

# 323. Expression alias metadata

Se mantiene separada del Type System.

---

# 324. Aggregate nullability

Ejemplos:

```text
COUNT
→ non-null

SUM
→ potentially nullable

AVG
→ potentially nullable
```

según semántica.

---

# 325. MIN/MAX

Result type generalmente deriva del input, con nullability ajustada.

---

# 326. CASE nullability

Dependerá de todas sus branches y ELSE.

---

# 327. COALESCE nullability

Puede reducir nullability cuando existe un argumento garantizado non-null.

---

# 328. Arithmetic nullability

Si cualquier operand puede ser NULL:

```text
result may be NULL
```

según semántica SQL.

---

# 329. Function null propagation

Las signatures deberán poder declarar reglas como:

```text
PROPAGATE
NEVER_NULL
ALWAYS_NULLABLE
CUSTOM
```

---

# 330. Type signature generics

El sistema podrá soportar signatures conceptuales:

```text
MAX<T: Orderable>(T) → T
```

---

# 331. Generic signatures

No requieren generics PHP nativos.

Pueden modelarse mediante descriptors.

---

# 332. Example COALESCE

```text
COALESCE<T>(T?, T, ...) → T
```

---

# 333. Example IN

```text
IN<T>(T, Collection<T>) → Boolean
```

---

# 334. Example equality

```text
EQUAL<T: Comparable>(T, T) → Boolean
```

---

# 335. Example arithmetic

```text
ADD<N: Numeric>(N1, N2)
→ Promote(N1, N2)
```

---

# 336. Type variables in signatures

Podrán reutilizar el Constraint Solver.

---

# 337. Function overloads

El sistema podrá soportar múltiples signatures.

---

# 338. Overload resolution

Se basará en:

- exact match;
- safe promotion;
- explicit coercion;
- specificity;
- deterministic priority.

---

# 339. Ambiguous overload

Debe producir error.

No elegir arbitrariamente.

---

# 340. Vendor function overloads

Podrán registrarse mediante Platform extensions.

---

# 341. Query type normalization

Tipos equivalentes deberán tener representación canónica.

---

# 342. Example

Evitar:

```text
StringType(length=null)
StringType()
UnboundedStringType
```

representando accidentalmente lo mismo.

---

# 343. Canonical type factory

Podrá existir:

```text
QueryTypeFactory
```

o interning para tipos comunes inmutables.

---

# 344. Type interning

Opcional para reducir allocations.

No deberá cambiar semántica.

---

# 345. Structural equality

Dos Query Types serán equivalentes por estructura semántica, no necesariamente por identidad de objeto.

---

# 346. Type equality

Conceptualmente:

```text
typeId
+
parameters
+
relevant semantic constraints
```

---

# 347. Native metadata excluded

Metadata puramente física no deberá contaminar equality semántica.

---

# 348. Type hashing

Debe ser determinista.

---

# 349. Serialization

Si se serializan tipos para caches:

- formato versionado;
- sin objetos PHP arbitrarios;
- sin closures;
- sin runtime state;
- sin secrets.

---

# 350. Compiled metadata

Preferir representaciones seguras y explícitas.

---

# 351. Type system runtime safety

Safe shared state:

```text
Frozen Type Registry
Type Descriptors
Compatibility Rules
Coercion Rules
Function Signatures
Operator Signatures
Compiled Conversion Metadata
```

---

# 352. Request-scoped state

```text
Type Constraint Graph
Semantic Type Table
Resolution Trace
Temporary Type Variables
```

deberá pertenecer a la operación/análisis.

---

# 353. Persistent worker rule

Nada del análisis de Request A deberá quedar asociado al análisis de Request B.

---

# 354. FrankenPHP example

```text
Worker
│
├── Request A
│   └── Query Type Context A
│
├── Dispose
│
└── Request B
    └── Query Type Context B
```

---

# 355. OpenSwoole example

```text
Shared:
Frozen QueryTypeRegistry

Coroutine A:
TypeResolutionContext A

Coroutine B:
TypeResolutionContext B
```

---

# 356. No mutable current context

Prohibido:

```php
TypeSystem::$currentContext
```

---

# 357. TypeResolutionContext

Conceptualmente:

```text
TypeResolutionContext
├── schema metadata
├── symbol table
├── parameter table
├── function signatures
├── operator signatures
├── capability snapshot
├── strictness
└── extension context
```

---

# 358. Context ≠ service locator

Debe contener datos/contexto de análisis, no servicios arbitrarios.

---

# 359. TypeSystem service

Podrá actuar como facade/orchestrator:

```text
QueryTypeSystem
├── Resolver
├── ConstraintSolver
├── CompatibilityResolver
├── CommonTypeResolver
└── CoercionResolver
```

---

# 360. Avoid God TypeSystem

Cada responsabilidad deberá permanecer separada.

---

# 361. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Type\
```

---

# 362. Proposed structure

```text
Query/
└── Type/
    ├── Contract/
    │   ├── QueryType.php
    │   ├── TypeResolver.php
    │   ├── TypeCompatibilityResolver.php
    │   ├── TypeCoercionResolver.php
    │   └── CommonTypeResolver.php
    │
    ├── Identity/
    │   └── QueryTypeId.php
    │
    ├── Descriptor/
    │   ├── QueryTypeDescriptor.php
    │   ├── QueryTypeCategory.php
    │   ├── TypeCharacteristic.php
    │   └── TypePortability.php
    │
    ├── Core/
    │   ├── UnknownType.php
    │   ├── NullType.php
    │   ├── BooleanType.php
    │   ├── IntegerType.php
    │   ├── BigIntegerType.php
    │   ├── DecimalType.php
    │   ├── FloatType.php
    │   ├── StringType.php
    │   ├── TextType.php
    │   ├── BinaryType.php
    │   ├── DateType.php
    │   ├── TimeType.php
    │   ├── DateTimeType.php
    │   ├── DateTimeTzType.php
    │   ├── IntervalType.php
    │   ├── UuidType.php
    │   ├── JsonType.php
    │   ├── EnumType.php
    │   ├── CollectionType.php
    │   ├── TupleType.php
    │   └── DomainType.php
    │
    ├── Resolution/
    │   ├── ResolvedQueryType.php
    │   ├── TypeResolutionContext.php
    │   ├── TypeResolutionResult.php
    │   ├── TypeCertainty.php
    │   └── TypeOrigin.php
    │
    ├── Constraint/
    │   ├── TypeVariable.php
    │   ├── TypeConstraint.php
    │   ├── TypeConstraintSet.php
    │   ├── TypeConstraintGraph.php
    │   └── QueryTypeConstraintSolver.php
    │
    ├── Compatibility/
    │   ├── TypeCompatibility.php
    │   ├── TypeCompatibilityResult.php
    │   └── DefaultTypeCompatibilityResolver.php
    │
    ├── Coercion/
    │   ├── TypeCoercion.php
    │   ├── TypeCoercionKind.php
    │   ├── TypeCoercionPlan.php
    │   └── DefaultTypeCoercionResolver.php
    │
    ├── Promotion/
    │   ├── TypePromotionRule.php
    │   ├── NumericPromotionRule.php
    │   └── CommonTypeResolver.php
    │
    ├── Signature/
    │   ├── FunctionSignature.php
    │   ├── OperatorSignature.php
    │   ├── TypeSignatureParameter.php
    │   ├── TypeVariableSignature.php
    │   └── SignatureResolver.php
    │
    ├── Table/
    │   └── QueryTypeTable.php
    │
    ├── Registry/
    │   ├── QueryTypeRegistry.php
    │   └── FrozenQueryTypeRegistry.php
    │
    ├── Fingerprint/
    │   └── QueryTypeFingerprint.php
    │
    ├── Diagnostics/
    │   ├── TypeResolutionTrace.php
    │   └── TypeDiagnosticRenderer.php
    │
    └── Exception/
```

---

# 363. Platform mapping namespace

Separadamente:

```text
Type/
├── Mapping/
│   ├── PlatformTypeMapper.php
│   ├── PlatformTypeMapping.php
│   ├── TypeMappingContext.php
│   └── TypeMappingRegistry.php
│
└── Conversion/
    ├── DatabaseValueConverter.php
    ├── InputValueConverter.php
    ├── OutputValueConverter.php
    └── ConversionPlan.php
```

La ubicación final podrá ser:

```text
Database/Type/
```

si posteriormente se decide compartir esta infraestructura entre Query, Schema, ORM y Hydration.

---

# 364. Important architectural decision

Aunque este documento se denomina:

```text
DATABASE_QUERY_TYPE_SYSTEM
```

no significa que todos los tipos deban vivir físicamente bajo:

```text
Query\Type
```

Si Schema, ORM, Hydration y Query comparten el mismo concepto semántico, la implementación deberá elevar los contratos comunes a:

```text
VoltStack\Quantum\Database\Type
```

manteniendo:

```text
Query-specific type analysis
```

dentro de:

```text
VoltStack\Quantum\Database\Query\Type
```

---

# 365. Recommended separation

```text
Database\Type
│
├── Core Semantic Types
├── Registry
├── Platform Mapping
├── Value Conversion
└── Extension Types

Database\Query\Type
│
├── Type Inference
├── Constraint Solver
├── Expression Typing
├── Predicate Typing
├── Function Signatures
├── Operator Signatures
└── Query Type Table
```

Esta separación evita duplicación futura.

---

# 366. Dependency graph

```text
                Database\Type
                     ▲
          ┌──────────┼──────────┐
          │          │          │
        Query      Schema      ORM
          │                     │
          ▼                     ▼
 Query\Type Analysis        Metadata
          │
          ▼
    Semantic Engine
          │
          ▼
       Planner
          │
          ▼
 Platform Type Mapping
          │
          ▼
    Value Conversion
          │
          ▼
    Driver Binding
```

---

# 367. DB-QTYPE-001

PHP Type no será equivalente a Query Semantic Type.

---

# 368. DB-QTYPE-002

Domain Type no será equivalente a Query Semantic Type.

---

# 369. DB-QTYPE-003

Query Semantic Type no será equivalente a Platform Database Type.

---

# 370. DB-QTYPE-004

Platform Database Type no será equivalente a Driver Binding Type.

---

# 371. DB-QTYPE-005

Los Query Types no contendrán SQL.

---

# 372. DB-QTYPE-006

Los Query Types no contendrán constantes PDO.

---

# 373. DB-QTYPE-007

Los Query Types no contendrán conexiones.

---

# 374. DB-QTYPE-008

Los Query Types no contendrán valores runtime.

---

# 375. DB-QTYPE-009

Los Query Type descriptors serán inmutables después de bootstrap.

---

# 376. DB-QTYPE-010

El Type Registry se congelará antes del runtime normal.

---

# 377. DB-QTYPE-011

No existirá un registry universal para todos los componentes Database.

---

# 378. DB-QTYPE-012

UnknownType representará información no resuelta, no un bypass permanente del tipado.

---

# 379. DB-QTYPE-013

NullType y nullability serán conceptos diferentes.

---

# 380. DB-QTYPE-014

DecimalType y FloatType permanecerán separados.

---

# 381. DB-QTYPE-015

StringType y BinaryType permanecerán separados.

---

# 382. DB-QTYPE-016

JsonType y StringType permanecerán separados.

---

# 383. DB-QTYPE-017

UuidType será semántico e independiente de su representación física.

---

# 384. DB-QTYPE-018

Date, Time, DateTime y DateTimeTz permanecerán diferenciados.

---

# 385. DB-QTYPE-019

CollectionType no implicará native database array.

---

# 386. DB-QTYPE-020

TupleType tendrá arity explícita.

---

# 387. DB-QTYPE-021

Domain types podrán preservar identidades semánticas aunque compartan representación física.

---

# 388. DB-QTYPE-022

UserIdType y OrderIdType no serán automáticamente equivalentes.

---

# 389. DB-QTYPE-023

La inferencia utilizará contexto antes que PHP runtime type.

---

# 390. DB-QTYPE-024

Los explicit semantic types tendrán prioridad.

---

# 391. DB-QTYPE-025

Schema metadata podrá participar en type resolution.

---

# 392. DB-QTYPE-026

ORM metadata podrá participar sin hacer al Query Type System dependiente del ORM.

---

# 393. DB-QTYPE-027

Function signatures serán explícitas.

---

# 394. DB-QTYPE-028

Operator signatures serán explícitas.

---

# 395. DB-QTYPE-029

Los overloads ambiguos producirán error.

---

# 396. DB-QTYPE-030

Las coerciones implícitas serán conservadoras.

---

# 397. DB-QTYPE-031

Las coerciones lossy no serán silenciosas.

---

# 398. DB-QTYPE-032

El servidor no será utilizado como type checker principal.

---

# 399. DB-QTYPE-033

Las implicit casts del servidor no definirán la arquitectura semántica de VoltStack.

---

# 400. DB-QTYPE-034

Los tipos se resolverán antes del Compiler cuando sea posible.

---

# 401. DB-QTYPE-035

El Compiler no será el principal Type Resolver.

---

# 402. DB-QTYPE-036

El Driver no conocerá Domain Types.

---

# 403. DB-QTYPE-037

El Driver recibirá Database Representation y Driver Binding Type.

---

# 404. DB-QTYPE-038

La conversión de valores será independiente del AST.

---

# 405. DB-QTYPE-039

El AST permanecerá inmutable durante Type Analysis.

---

# 406. DB-QTYPE-040

Los tipos resueltos vivirán en artifacts semánticos externos al AST.

---

# 407. DB-QTYPE-041

Type constraints serán explícitos.

---

# 408. DB-QTYPE-042

La resolución de constraints será determinista.

---

# 409. DB-QTYPE-043

El mismo parámetro deberá satisfacer todos sus type constraints.

---

# 410. DB-QTYPE-044

No se utilizará first-type-wins.

---

# 411. DB-QTYPE-045

CommonType resolution será explícita.

---

# 412. DB-QTYPE-046

UNION deberá validar compatibilidad de proyecciones.

---

# 413. DB-QTYPE-047

CASE deberá resolver un tipo común.

---

# 414. DB-QTYPE-048

COALESCE deberá resolver tipo y nullability semánticamente.

---

# 415. DB-QTYPE-049

Las transformaciones del Optimizer preservarán tipos.

---

# 416. DB-QTYPE-050

El Optimizer no introducirá casts arbitrarios para reparar queries inválidas.

---

# 417. DB-QTYPE-051

Los tipos podrán declarar requisitos de capabilities.

---

# 418. DB-QTYPE-052

Type support y operation support serán conceptos distintos.

---

# 419. DB-QTYPE-053

Almacenar JSON no implicará soportar todas las operaciones JSON.

---

# 420. DB-QTYPE-054

El Type System no utilizará vendor-name conditionals en sus capas portables.

---

# 421. DB-QTYPE-055

Las diferencias de Platform se resolverán mediante Platform mappings y capabilities.

---

# 422. DB-QTYPE-056

Los Platform-specific types estarán explícitamente clasificados.

---

# 423. DB-QTYPE-057

Las extensiones registrarán tipos durante bootstrap.

---

# 424. DB-QTYPE-058

No habrá registro arbitrario de tipos durante una request.

---

# 425. DB-QTYPE-059

Las extensiones no podrán redefinir silenciosamente tipos core.

---

# 426. DB-QTYPE-060

Los Query Types no contendrán secrets.

---

# 427. DB-QTYPE-061

Los diagnostics no expondrán runtime values sensibles.

---

# 428. DB-QTYPE-062

Los fingerprints de tipos no incluirán runtime values.

---

# 429. DB-QTYPE-063

Los caches de mappings deberán incluir Platform/Capability fingerprints cuando corresponda.

---

# 430. DB-QTYPE-064

El Type System será compatible con compilación offline.

---

# 431. DB-QTYPE-065

El Type System será compatible con capabilities descubiertas online.

---

# 432. DB-QTYPE-066

Los registries compartidos serán seguros para FrankenPHP.

---

# 433. DB-QTYPE-067

Los contexts de resolución serán execution/analysis scoped.

---

# 434. DB-QTYPE-068

No habrá mutable global current TypeResolutionContext.

---

# 435. DB-QTYPE-069

El mismo registry podrá ser utilizado concurrentemente por múltiples OpenSwoole coroutines.

---

# 436. DB-QTYPE-070

Los converters compartidos deberán ser stateless o declarar explícitamente su lifetime.

---

# 437. DB-QTYPE-071

El Type System no utilizará Service Locator.

---

# 438. DB-QTYPE-072

Los converters no resolverán servicios mediante container global.

---

# 439. DB-QTYPE-073

Type validation no reemplazará business validation.

---

# 440. DB-QTYPE-074

Quantum/Validation y Query Type Validation permanecerán separados.

---

# 441. DB-QTYPE-075

Los errores semánticos de tipo deberán ocurrir antes de ejecución cuando sea posible.

---

# 442. DB-QTYPE-076

Los errores de conversión deberán distinguirse de errores semánticos.

---

# 443. DB-QTYPE-077

Los errores de binding nativo deberán distinguirse de errores de conversión.

---

# 444. DB-QTYPE-078

La metadata de resultados deberá conservar tipos semánticos de proyección.

---

# 445. DB-QTYPE-079

Hydration podrá consumir Type Metadata sin que Type System instancie entidades.

---

# 446. DB-QTYPE-080

Query, Schema, ORM y Hydration deberán reutilizar conceptos de tipo comunes cuando sean realmente equivalentes.

---

# 447. DB-QTYPE-081

La infraestructura común podrá elevarse a `Database\Type`.

---

# 448. DB-QTYPE-082

El análisis específico de consultas permanecerá en `Query\Type`.

---

# 449. DB-QTYPE-083

El Type System no se convertirá en God Service.

---

# 450. DB-QTYPE-084

Resolver, ConstraintSolver, CompatibilityResolver, CoercionResolver y CommonTypeResolver permanecerán separables.

---

# 451. DB-QTYPE-085

La igualdad de tipos será semántica y estructural.

---

# 452. DB-QTYPE-086

La igualdad de tipos no dependerá de identidad de objeto PHP.

---

# 453. DB-QTYPE-087

Los tipos tendrán fingerprints deterministas.

---

# 454. DB-QTYPE-088

La serialización de metadata de tipos será versionada.

---

# 455. DB-QTYPE-089

No se serializarán closures ni recursos runtime dentro de metadata de tipos.

---

# 456. DB-QTYPE-090

La arquitectura permitirá agregar nuevos tipos sin modificar el core.

---

# 457. Anti-pattern — PHP type equals DB type

Incorrecto:

```text
PHP string
→ VARCHAR
```

Correcto:

```text
PHP string
+
Semantic Context
→ Query Type
→ Platform Mapping
```

---

# 458. Anti-pattern — SQL type inside AST

Incorrecto:

```text
ParameterNode
type = VARCHAR(255)
```

Correcto:

```text
ParameterNode
→ semantic ParameterId

Semantic Type Table
→ StringType
```

---

# 459. Anti-pattern — PDO type in Type System

Incorrecto:

```text
IntegerType
→ PDO::PARAM_INT
```

dentro del Query Type core.

Correcto:

```text
IntegerType
→ Platform Mapping
→ Database Representation
→ Driver Binding Adapter
```

---

# 460. Anti-pattern — JSON as string

Incorrecto:

```text
JSON
→ StringType
```

Correcto:

```text
JsonType
→ platform-specific representation
```

---

# 461. Anti-pattern — UUID as string

Incorrecto:

```text
Uuid
→ StringType
```

por el simple hecho de que su representación PHP sea string.

---

# 462. Anti-pattern — enum as backing type

Incorrecto:

```text
Status
→ StringType
```

perdiendo identidad semántica.

Correcto:

```text
StatusType
→ backing String representation
```

---

# 463. Anti-pattern — domain IDs interchangeable

Incorrecto:

```text
UserId
=
OrderId
```

porque ambos son integers.

---

# 464. Anti-pattern — server fixes types

Incorrecto:

```text
Send query
→ let PostgreSQL/MySQL decide
```

Correcto:

```text
Semantic Analysis
→ Type Resolution
→ Capability Validation
→ Compile
→ Execute
```

---

# 465. Anti-pattern — cast everything

Incorrecto:

```text
type mismatch
→ automatically CAST
```

Correcto:

```text
type mismatch
→ compatibility analysis
→ safe coercion if defined
→ otherwise error
```

---

# 466. Anti-pattern — mutable AST typing

Incorrecto:

```php
$node->type = $resolvedType;
```

Correcto:

```text
AST
+
QueryTypeTable
```

---

# 467. Anti-pattern — ORM owns types

Incorrecto:

```text
ORM Type System
→ Query Builder Type System
→ Schema Type System
```

como tres sistemas duplicados.

Correcto:

```text
Database Semantic Type Core
        │
        ├── Query
        ├── ORM
        ├── Schema
        └── Hydration
```

con especializaciones por dominio.

---

# 468. Anti-pattern — giant Type class

Evitar:

```text
Type
├── compileSql()
├── hydrateEntity()
├── bindPdo()
├── createColumn()
├── validateBusinessRule()
└── resolveConnection()
```

---

# 469. Correct separation

```text
QueryType
→ semantic meaning

PlatformTypeMapper
→ physical type mapping

ValueConverter
→ representation conversion

DriverBinder
→ native binding

Hydrator
→ result/entity integration
```

---

# 470. Example — UserId comparison

Query:

```php
DB::table('users')
    ->where('id', $userId);
```

AST:

```text
ComparisonPredicate
├── ColumnReference(users.id)
├── EQUAL
└── Parameter(p1)
```

Schema:

```text
users.id
→ UserIdType
```

Constraint:

```text
p1 compatible with UserIdType
```

Resolved:

```text
p1
→ UserIdType
```

---

# 471. Platform mapping

PostgreSQL:

```text
UserIdType
→ BIGINT
```

MySQL:

```text
UserIdType
→ BIGINT
```

SQLite:

```text
UserIdType
→ INTEGER
```

pero la semántica continúa siendo:

```text
UserIdType
```

---

# 472. Example — UUID

Application:

```text
Uuid object
```

Semantic:

```text
UuidType
```

PostgreSQL:

```text
UUID
```

MySQL configured strategy:

```text
BINARY(16)
```

SQLite:

```text
BLOB
```

Driver:

```text
appropriate native binding
```

---

# 473. Example — JSON

Expression:

```text
metadata["language"]
```

Semantic input:

```text
metadata → JsonType
```

Semantic operation:

```text
JsonPathExtract
```

Result:

```text
StringType
```

si la operation signature lo define.

Dialect:

```text
PostgreSQL
→ operator/function syntax

MySQL
→ JSON function syntax

SQLite
→ JSON function syntax if capability exists
```

AST no cambia de significado.

---

# 474. Example — arithmetic

```text
price
→ Decimal(18,2)

tax
→ Decimal(8,4)
```

Expression:

```text
price * tax
```

Constraint:

```text
both numeric
```

Promotion:

```text
Decimal result
```

Precision/scale result deberá resolverse mediante regla explícita.

---

# 475. Example — invalid arithmetic

```text
created_at + json_document
```

Sin operación registrada:

```text
UnsupportedTypeOperationException
```

antes de SQL.

---

# 476. Example — COALESCE

```text
COALESCE(
    nullable_integer,
    0
)
```

Types:

```text
Nullable<Integer>
Integer
```

Result:

```text
Integer
NON_NULL
```

si las reglas semánticas lo garantizan.

---

# 477. Example — CASE

```text
CASE
    WHEN active THEN 1
    ELSE 0.5
END
```

Branches:

```text
Integer
Decimal
```

Common type:

```text
Decimal
```

---

# 478. Example — UNION mismatch

```text
SELECT created_at
UNION
SELECT json_document
```

si no existe common type válido:

```text
SetOperationTypeMismatchException
```

---

# 479. Example — typed parameter

```text
users.uuid = p1
```

Schema:

```text
UuidType
```

Binding:

```text
p1 = "550e..."
```

Runtime PHP type:

```text
string
```

Resolved semantic type:

```text
UuidType
```

Conversion:

```text
validate UUID
→ platform representation
```

---

# 480. Example — wrong UUID

```text
p1 = "hello"
```

Semantic query continúa siendo válida estructuralmente.

Binding conversion falla:

```text
ParameterConversionException
```

antes de Driver execution.

---

# 481. Example — Domain ID mismatch

```text
users.id
→ UserIdType

orders.id
→ OrderIdType
```

Query:

```text
users.id = orders.id
```

Con política estricta:

```text
TypeMismatchException
```

Esto puede detectar bugs que SQL tradicional aceptaría porque ambos son BIGINT.

---

# 482. Developer escape hatch

Cuando la comparación sea intencional:

```text
explicit cast
```

o:

```text
explicit domain conversion
```

deberá expresarlo.

---

# 483. Benefit

El sistema detecta errores en:

```text
framework semantic layer
```

antes de que se conviertan en bugs silenciosos de producción.

---

# 484. V1 priorities

## V1.1 Core Types

```text
Unknown
Null
Boolean
Integer
BigInteger
Decimal
Float
String
Text
Binary
Date
Time
DateTime
DateTimeTz
Uuid
Json
Enum
Collection
Tuple
Domain
```

---

# 485. V1.2 Core infrastructure

```text
QueryTypeId
QueryTypeDescriptor
ResolvedQueryType
QueryTypeRegistry
TypeResolver
TypeCompatibilityResolver
TypeConstraint
TypeConstraintSolver
QueryTypeTable
```

---

# 486. V1.3 Query integration

```text
Expression typing
Predicate typing
Parameter typing
Column typing
Literal typing
Function signatures
Operator signatures
Projection typing
```

---

# 487. V1.4 Platform integration

```text
PlatformTypeMapper
DatabaseRepresentation
Input conversion
Driver binding metadata
```

---

# 488. V1.5 Safety

```text
Strict compatibility
Safe coercions
No lossy implicit conversions
Diagnostics
Sensitive-value redaction
Persistent runtime safety
```

---

# 489. V2

```text
Advanced generic signatures
Range types
Native arrays
Advanced JSON types
Network types
Domain type algebra
Advanced overload resolution
Compiled conversion plans
Type-aware optimizer diagnostics
```

---

# 490. V3

```text
Composite database types
User-defined database types
Advanced structural types
Geospatial types
Vector types
Runtime specialization
Advanced static query analysis
```

---

# 491. Query Type architecture formula

```text
Query Type
=
Semantic Identity
+
Type Parameters
+
Nullability
+
Semantic Characteristics
+
Constraints
+
Portability
```

---

# 492. Resolution formula

```text
Resolved Query Type
=
Explicit Type
+
Schema Metadata
+
ORM Metadata
+
Expression Constraints
+
Function/Operator Signatures
+
Parameter Constraints
+
Safe Inference
```

---

# 493. Compatibility formula

```text
Type Compatibility
=
Left Type
+
Right Type
+
Operation
+
Coercion Rules
+
Semantic Policy
```

---

# 494. Platform mapping formula

```text
Platform Type
=
Query Semantic Type
+
Resolved Platform
+
Capability Snapshot
+
Mapping Policy
```

---

# 495. Conversion formula

```text
Database Representation
=
Runtime Value
+
Resolved Query Type
+
Platform Type Mapping
+
Conversion Plan
```

---

# 496. Driver formula

```text
Native Binding
=
Database Representation
+
Driver Binding Profile
```

---

# 497. Complete type pipeline

```text
Application / Domain Value
           │
           ▼
       PHP Type
           │
           ▼
     Semantic Evidence
           │
           ▼
     Query Type Resolver
           │
           ▼
   Resolved Query Type
           │
           ▼
 Semantic Compatibility
           │
           ▼
  Capability Validation
           │
           ▼
   Platform Type Mapping
           │
           ▼
    Conversion Plan
           │
           ▼
 Database Representation
           │
           ▼
 Driver Binding Adapter
           │
           ▼
      Native Client
```

---

# 498. Query semantic architecture

Con los documentos:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

se establece:

```text
Expression
    │
    ├── produces Type
    │
    ▼
Predicate
    │
    ├── constrains Types
    │
    ▼
Parameter
    │
    ├── receives Runtime Value
    │
    ▼
Query Type System
    │
    ├── resolves meaning
    ├── validates compatibility
    ├── determines coercion
    └── prepares mapping
```

---

# 499. Central architectural invariant

```text
Expression
+
Predicate
+
Parameter
+
Type
```

permanecen completamente independientes de:

```text
PDO
Physical Connection
Native Statement
EntityManager
UnitOfWork
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 500. Master rule

> VoltStack deberá conocer qué significa un valor antes de decidir cómo almacenarlo, compilarlo, convertirlo o bindearlo.

---

# 501. Final rule

La arquitectura deberá preservar permanentemente:

```text
PHP Representation
        ≠
Domain Meaning
        ≠
Query Semantic Type
        ≠
Platform Database Type
        ≠
Database Representation
        ≠
Driver Binding Type
```

---

# 502. Resultado arquitectónico

El Query Type System permitirá que una misma consulta semántica pueda analizarse como:

```text
Portable Query
        │
        ▼
Semantic Types
        │
        ├──────────────┬──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
      MySQL          MariaDB      PostgreSQL       SQLite
        │              │              │              │
        ▼              ▼              ▼              ▼
Physical Types   Physical Types  Physical Types  Physical Types
        │              │              │              │
        ▼              ▼              ▼              ▼
Driver Binding   Driver Binding  Driver Binding  Driver Binding
```

sin introducir condicionales de vendor en Query Builder, AST, Expressions, Predicates o Parameters.

---

# 503. Próximo documento

El siguiente documento será:

```text
31_DATABASE_QUERY_METADATA_SYSTEM.md
```

y formalizará la metadata asociada a una consulta sin contaminar su estructura semántica.

Deberá distinguir:

```text
Query Structure
≠
Query Metadata
≠
Semantic Metadata
≠
Execution Metadata
≠
Diagnostic Metadata
≠
Extension Metadata
```

y definir conceptos como:

```text
QueryMetadata
QueryOrigin
QueryLabel
QueryIntent
ConnectionIntent
ConsistencyRequirement
TransactionRequirement
ExecutionPolicy
CachePolicy
SecurityMetadata
TelemetryMetadata
DiagnosticMetadata
ExtensionMetadata
QueryProvenance
QueryFingerprintMetadata
```

manteniendo toda metadata mutable o request-specific fuera de los artifacts compartidos.