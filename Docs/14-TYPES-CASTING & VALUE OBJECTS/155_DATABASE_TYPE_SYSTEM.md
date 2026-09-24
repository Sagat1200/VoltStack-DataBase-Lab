# 155_DATABASE_TYPE_SYSTEM.md

# VoltStack Quantum Database
## Database Type System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 155 — Database Type System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Type System` define el modelo canónico mediante el cual VoltStack representa, valida, compara y transporta tipos de datos entre:

- PHP;
- ORM;
- Query Engine;
- Schema System;
- SQL Compiler;
- Parameter Binding;
- Result/Hydration;
- plataformas de base de datos;
- drivers.

El sistema deberá impedir que conceptos distintos como:

```text
PHP int
INTEGER lógico
BIGINT PostgreSQL
BIGINT UNSIGNED MySQL
PDO::PARAM_INT
```

sean tratados como si fueran el mismo nivel de abstracción.

La regla fundamental será:

> **VoltStack representará los tipos mediante un modelo lógico canónico independiente del dialecto; cada Platform determinará su representación física y los sistemas de conversión controlarán explícitamente el tránsito entre valores PHP, valores ORM y valores de base de datos.**

---

# 2. Problema arquitectónico

Un valor atraviesa varias representaciones.

Ejemplo:

```php
$user->id = 42;
```

puede conceptualmente recorrer:

```text
PHP
int
 ↓
ORM
IntegerType
 ↓
Query AST
IntegerType
 ↓
Parameter Binding
INTEGER binding
 ↓
Platform
BIGINT
 ↓
Driver
native integer/string representation
 ↓
Database
BIGINT
```

En lectura ocurre el camino inverso:

```text
Database
 ↓
Driver
 ↓
Raw Result Value
 ↓
Database Type Conversion
 ↓
ORM Value
 ↓
Hydration
 ↓
PHP Value
```

VoltStack deberá modelar explícitamente esas fronteras.

---

# 3. Separación fundamental

```text
PHP Type
≠
ORM Type
≠
Database Logical Type
≠
Platform Physical Type
≠
SQL Type Declaration
≠
Driver Binding Type
```

Esta separación será una de las invariantes centrales del subsistema Database.

---

# 4. Objetivos

El sistema deberá proporcionar:

1. tipos lógicos canónicos;
2. identificadores estables de tipos;
3. descriptores PHP;
4. descriptores de base de datos;
5. representación física por Platform;
6. compatibilidad entre tipos;
7. normalización;
8. nullability;
9. precisión y escala;
10. longitud;
11. signed/unsigned cuando aplique;
12. charset/collation cuando corresponda;
13. conversiones explícitas;
14. detección de conversiones con pérdida;
15. binding de parámetros;
16. conversión de resultados;
17. integración con Schema;
18. integración con Query Type System;
19. integración con ORM;
20. integración con Hydration;
21. integración con SQL Compiler;
22. extensibilidad;
23. cache seguro;
24. comportamiento determinista;
25. compatibilidad entre plataformas.

---

# 5. No objetivos

`Database Type System` no será:

```text
Serializer
Validator
Hydrator
Caster
Schema Compiler
SQL Compiler
Driver
Entity Mapper
```

aunque dichos sistemas consumirán información del Type System.

---

# 6. Posición arquitectónica

```text
                     Application
                         │
                         ▼
                        ORM
                         │
                ┌────────┴────────┐
                ▼                 ▼
          Entity Mapping      Hydration
                │                 │
                └────────┬────────┘
                         ▼
                    Type System
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Query Engine       Schema          Conversion
        │                │                │
        ▼                ▼                ▼
   SQL Compiler    Schema Compiler    Binding
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                      Platform
                         │
                         ▼
                       Driver
                         │
                         ▼
                     Database
```

---

# 7. Capas de tipos

VoltStack distinguirá al menos seis niveles.

```text
1. PHP Type
2. Domain/ORM Type
3. Database Logical Type
4. Platform Physical Type
5. SQL Declaration
6. Driver Binding Type
```

---

# 8. PHP Type

Representa el tipo visible en PHP.

Ejemplos:

```text
int
float
string
bool
array
DateTimeImmutable
Uuid
Money
UserStatus
```

No implica automáticamente cómo se almacena.

---

# 9. Domain Type

Puede representar conceptos de dominio.

Ejemplo:

```php
final readonly class Money
{
    public function __construct(
        public string $currency,
        public string $amount,
    ) {}
}
```

`Money` no es un tipo SQL.

---

# 10. ORM Type

Define cómo una propiedad persistente es entendida por el ORM.

Ejemplo:

```text
MoneyType
UuidType
EnumType
DateTimeImmutableType
```

Puede usar uno o varios tipos lógicos de base de datos.

---

# 11. Database Logical Type

Es la representación canónica independiente del vendor.

Ejemplos:

```text
BOOLEAN
INTEGER
BIG_INTEGER
DECIMAL
STRING
TEXT
BINARY
UUID
JSON
DATE
TIME
DATETIME
TIMESTAMP
```

---

# 12. Platform Physical Type

Es la representación que una plataforma específica puede utilizar.

Ejemplo:

```text
Logical UUID
```

podría resolverse conceptualmente como:

```text
PostgreSQL → UUID
MySQL      → CHAR(36) / BINARY(16)
MariaDB    → platform-specific representation
SQLite     → TEXT/BLOB affinity
```

La decisión concreta pertenece a `Database Platform`.

---

# 13. SQL Type Declaration

Ejemplos:

```sql
VARCHAR(255)
DECIMAL(19,4)
TIMESTAMP(6)
CHAR(36)
```

Es output del Schema Compiler/Platform.

No es el tipo canónico.

---

# 14. Driver Binding Type

Representa cómo el driver recibe un valor.

Ejemplo conceptual:

```text
INTEGER
STRING
BOOLEAN
BINARY
LOB
```

No debe confundirse con el tipo SQL físico.

---

# 15. TypeId

Cada tipo lógico tendrá un identificador estable.

```php
final readonly class TypeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
bool
int
bigint
decimal
float
string
text
binary
uuid
json
date
time
datetime
timestamp
```

---

# 16. TypeId ≠ PHP class

No utilizar:

```text
VoltStack\Quantum\Database\Type\IntegerType
```

como identidad persistente del tipo.

La clase puede cambiar.

El `TypeId` será estable.

---

# 17. Type descriptor

La descripción de un tipo será immutable.

```php
interface DatabaseType
{
    public function id(): TypeId;

    public function family(): TypeFamily;

    public function phpType(): PhpTypeDescriptor;

    public function characteristics(): TypeCharacteristics;
}
```

---

# 18. TypeFamily

```php
enum TypeFamily
{
    case BOOLEAN;

    case INTEGER;

    case DECIMAL;

    case FLOATING_POINT;

    case STRING;

    case BINARY;

    case TEMPORAL;

    case IDENTIFIER;

    case STRUCTURED;

    case ENUMERATION;

    case DOMAIN;

    case CUSTOM;
}
```

---

# 19. Familia ≠ tipo

Por ejemplo:

```text
INTEGER
BIG_INTEGER
SMALL_INTEGER
```

pueden pertenecer a:

```text
TypeFamily::INTEGER
```

pero continúan siendo tipos distintos.

---

# 20. Tipos canónicos iniciales

VoltStack debería proporcionar inicialmente:

```text
BooleanType

TinyIntegerType
SmallIntegerType
IntegerType
BigIntegerType

DecimalType
FloatType
DoubleType

StringType
TextType

BinaryType
BlobType

UuidType
UlidType

JsonType

DateType
TimeType
DateTimeType
DateTimeImmutableType
TimestampType

EnumType

ValueObjectType
```

La lista podrá crecer mediante extensiones.

---

# 21. Integer model

Los enteros requieren representar:

```text
width/range
signedness
```

pero sin asumir que todos los motores soportan físicamente las mismas variantes.

---

# 22. Integer characteristics

```php
final readonly class IntegerCharacteristics
{
    public function __construct(
        public IntegerRange $range,
        public Signedness $signedness,
    ) {}
}
```

---

# 23. Signedness

```php
enum Signedness
{
    case SIGNED;
    case UNSIGNED;
    case PLATFORM_DEFAULT;
}
```

---

# 24. UNSIGNED

`UNSIGNED` no debe convertirse en una suposición universal.

Una Platform puede reportar:

```text
SUPPORTED
REQUIRES_EMULATION
UNSUPPORTED
```

---

# 25. Integer overflow

Conversión:

```text
database bigint
→
PHP int
```

puede ser insegura dependiendo de:

- valor;
- arquitectura PHP;
- representación del driver.

VoltStack nunca deberá truncar silenciosamente.

---

# 26. IntegerConversionPolicy

```php
enum IntegerOverflowPolicy
{
    case THROW;
    case STRING;
    case BIG_INTEGER_OBJECT;
}
```

La política exacta podrá depender del mapping.

---

# 27. Decimal

Dinero y valores de precisión exacta no deberán pasar automáticamente por `float`.

```text
DECIMAL
≠
FLOAT
```

---

# 28. Decimal descriptor

```php
final readonly class DecimalCharacteristics
{
    public function __construct(
        public int $precision,
        public int $scale,
    ) {}
}
```

---

# 29. Precision

Cantidad máxima de dígitos significativos.

---

# 30. Scale

Cantidad de dígitos después del punto decimal.

---

# 31. Decimal invariant

Para:

```text
DECIMAL(p, s)
```

debe cumplirse:

```text
p > 0
s >= 0
s <= p
```

---

# 32. Decimal PHP representation

Por defecto se deberá favorecer una representación exacta.

Ejemplo:

```text
string
```

o un futuro:

```text
DecimalValue
```

antes que convertir automáticamente a `float`.

---

# 33. Floating point

Tipos:

```text
FLOAT
DOUBLE
```

representan aproximaciones binarias.

VoltStack deberá preservar esa diferencia semántica frente a `DECIMAL`.

---

# 34. String

Un tipo string podrá tener características como:

```text
length
fixed/variable
unicode expectations
```

---

# 35. String logical model

```php
final readonly class StringCharacteristics
{
    public function __construct(
        public ?int $length,
        public StringLengthSemantics $lengthSemantics,
    ) {}
}
```

---

# 36. VARCHAR ≠ StringType

`VARCHAR(255)` es una posible representación física.

El tipo lógico puede ser:

```text
StringType(length: 255)
```

---

# 37. Text

Para datos textuales grandes:

```text
TextType
```

separado de strings de longitud limitada.

---

# 38. Charset

Charset pertenece principalmente a la representación/schema/platform.

No deberá contaminar la identidad básica de todos los tipos lógicos.

---

# 39. Collation

Mismo principio.

```text
String logical semantics
≠
Database collation
```

---

# 40. Binary

VoltStack distinguirá:

```text
BinaryType
BlobType
```

cuando la semántica o capacidad de plataforma lo requiera.

---

# 41. Binary PHP value

Puede representarse como:

```text
string bytes
```

pero debe mantenerse explícita su semántica binaria.

---

# 42. Stream values

LOBs grandes podrán utilizar streams.

```text
Blob
→
resource/stream abstraction
```

sin obligar a cargar todo el contenido en memoria.

---

# 43. Boolean

El tipo lógico:

```text
BOOLEAN
```

debe existir incluso si una plataforma utiliza internamente:

```text
TINYINT
INTEGER
```

---

# 44. Boolean invariant

La representación física no cambia la semántica lógica.

```text
BOOLEAN
≠
INTEGER
```

aunque una plataforma use `0/1`.

---

# 45. UUID

VoltStack tendrá un tipo lógico UUID.

```text
UuidType
```

independiente de si físicamente se almacena como:

```text
native UUID
CHAR(36)
BINARY(16)
```

---

# 46. UUID representation policy

```php
enum UuidStorageStrategy
{
    case PLATFORM_NATIVE;
    case STRING;
    case BINARY;
}
```

---

# 47. UUID PHP representation

Podrá ser:

```text
string
Uuid value object
```

según mapping.

---

# 48. ULID

Mismo principio.

```text
ULID logical value
```

no debe estar ligado a `CHAR(26)` como identidad conceptual.

---

# 49. JSON

`JsonType` será un tipo estructurado.

Puede mapearse a:

```text
PostgreSQL JSON/JSONB
MySQL JSON
MariaDB representation
SQLite TEXT + capabilities
```

según Platform.

La semántica completa se detallará en:

```text
161_DATABASE_JSON_TYPE_SYSTEM.md
```

---

# 50. Temporal types

VoltStack deberá distinguir:

```text
DATE
TIME
DATETIME
TIMESTAMP
```

y sus semánticas.

---

# 51. Temporal ambiguity

Un valor:

```text
2026-09-07 14:00:00
```

no expresa por sí solo:

```text
timezone
offset
instant
local datetime
```

---

# 52. Temporal semantics

El Type System deberá permitir distinguir:

```text
LocalDate
LocalTime
LocalDateTime
Instant
OffsetDateTime
```

aunque la API PHP final pueda utilizar objetos conocidos.

---

# 53. DateTimeImmutable

VoltStack favorecerá valores temporales immutable en mappings modernos.

---

# 54. Timestamp ≠ DateTime

No deben convertirse automáticamente en sinónimos conceptuales.

---

# 55. Timezone

Timezone deberá ser una decisión explícita.

Nunca:

```text
database datetime
→
guess server timezone
```

silenciosamente.

---

# 56. Enum

Los enums PHP podrán mapearse mediante:

```text
EnumType
```

pero:

```text
PHP Enum
≠
Database Native Enum
```

---

# 57. Enum storage

Puede utilizar:

```text
string
integer
native DB enum
```

dependiendo del mapping.

El sistema completo se definirá en:

```text
159_DATABASE_ENUM_MAPPING_SYSTEM.md
```

---

# 58. Value Objects

Tipos de dominio como:

```text
Money
EmailAddress
Percentage
CountryCode
Coordinates
```

podrán integrarse mediante mappings.

---

# 59. Value Object ≠ scalar cast

Un Value Object puede necesitar:

```text
multiple columns
```

por ejemplo:

```text
Money
├── amount
└── currency
```

---

# 60. Value Object Type

Por tanto, el Type System deberá permitir:

```text
SingleColumnType
MultiColumnValueObjectMapping
```

sin forzar ambos conceptos a una sola abstracción.

---

# 61. Nullability

Regla:

```text
Type
≠
Nullability
```

---

# 62. Ejemplo

```text
StringType
nullable=false
```

y:

```text
StringType
nullable=true
```

usan el mismo tipo lógico.

---

# 63. NullableType anti-pattern

Evitar una proliferación:

```text
NullableStringType
NullableIntegerType
NullableUuidType
```

---

# 64. TypeReference

Un uso concreto del tipo puede representarse mediante:

```php
final readonly class TypeReference
{
    public function __construct(
        public TypeId $type,
        public TypeArguments $arguments,
        public Nullability $nullability,
    ) {}
}
```

---

# 65. Type arguments

Permiten representar:

```text
string(length=255)
decimal(precision=19, scale=4)
uuid(storage=binary)
datetime(precision=6)
```

sin crear una clase diferente por cada combinación.

---

# 66. Type definition ≠ Type reference

```text
DatabaseType
```

define el comportamiento lógico.

```text
TypeReference
```

representa un uso concreto.

---

# 67. Canonical type expression

Conceptualmente:

```text
decimal(
    precision=19,
    scale=4,
    nullable=false
)
```

---

# 68. Immutability

`DatabaseType`, `TypeReference` y descriptors serán immutable.

Esto permite:

- compartirlos entre requests;
- cachearlos;
- fingerprinting determinista;
- uso seguro en workers persistentes.

---

# 69. Type normalization

Diferentes entradas equivalentes deberán producir la misma representación canónica.

Ejemplo:

```text
INTEGER
int
integer
```

podrán normalizarse a:

```text
TypeId("int")
```

cuando provengan de APIs de configuración.

---

# 70. Runtime canonical form

Después del bootstrap, internamente deberá usarse únicamente la forma canónica.

---

# 71. Alias ≠ TypeId

Aliases pueden existir para DX/migración.

Pero:

```text
alias
≠
canonical TypeId
```

---

# 72. TypeRegistry

El registro detallado se define en:

```text
156_DATABASE_TYPE_REGISTRY_SYSTEM.md
```

Conceptualmente:

```text
TypeId
    ↓
DatabaseType
```

---

# 73. Registry bootstrap

Tipos deberán registrarse durante bootstrap.

No durante query execution.

---

# 74. Frozen registry

Después de bootstrap:

```text
TypeRegistry
→ FROZEN
```

por defecto.

---

# 75. Runtime type discovery

No realizar:

```text
scan classes
reflection discovery
autoload arbitrary type
```

durante el hot path.

---

# 76. Type compatibility

VoltStack necesitará distinguir varios conceptos.

```text
Equality
Compatibility
Convertibility
Assignability
Representability
```

No son sinónimos.

---

# 77. Type equality

```text
A == B
```

cuando representan la misma referencia lógica canónica.

---

# 78. Compatibility

Dos tipos pueden no ser iguales pero ser compatibles.

Ejemplo:

```text
SmallInteger
→
Integer
```

---

# 79. Convertibility

Puede existir conversión:

```text
String
→
UUID
```

sin que ambos tipos sean compatibles de forma implícita.

---

# 80. Assignability

Pregunta:

> ¿Puede un valor del tipo A utilizarse donde se espera B sin violar semántica?

---

# 81. Platform representability

Pregunta diferente:

> ¿Puede la Platform almacenar correctamente este TypeReference?

---

# 82. Compatibility result

Evitar booleanos simples cuando se requiere más información.

```php
enum TypeCompatibility
{
    case EXACT;
    case SAFE;
    case LOSSY;
    case REQUIRES_EXPLICIT_CONVERSION;
    case INCOMPATIBLE;
    case UNKNOWN;
}
```

---

# 83. UNKNOWN

Regla:

```text
UNKNOWN
≠
COMPATIBLE
```

---

# 84. Safe widening

Ejemplo conceptual:

```text
SmallInteger
→
Integer
```

puede ser SAFE.

---

# 85. Narrowing

```text
BigInteger
→
SmallInteger
```

puede ser:

```text
LOSSY
```

o incompatible dependiendo del valor.

---

# 86. Value-dependent conversion

Algunas conversiones solo pueden decidirse con el valor real.

---

# 87. Static compatibility

El Type System puede decir:

```text
potentially narrowing
```

---

# 88. Runtime conversion

El Conversion System verifica:

```text
actual value fits target range
```

---

# 89. Lossless conversion

Una conversión es lossless si conserva toda la información semántica relevante.

---

# 90. Lossy conversion

Ejemplos:

```text
DECIMAL → FLOAT
DATETIME(6) → DATETIME(0)
BIGINT → INT32
Unicode text → incompatible charset
```

---

# 91. Silent lossy conversion

Prohibida por defecto.

---

# 92. Explicit policy

Cuando una conversión lossy sea permitida deberá ser explícita.

---

# 93. Conversion direction

VoltStack distinguirá:

```text
PHP_TO_DATABASE
DATABASE_TO_PHP
TYPE_TO_TYPE
```

---

# 94. Value conversion

Se detallará en:

```text
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
```

---

# 95. Casting

Se detallará en:

```text
158_DATABASE_CASTING_SYSTEM.md
```

Conversión y casting estarán relacionados pero no serán idénticos.

---

# 96. Conversion ≠ Casting

Conceptualmente:

```text
Conversion
```

es infraestructura de tránsito entre representaciones.

```text
Casting
```

es una política declarada para exponer/interpretar valores.

---

# 97. Parameter binding

Cuando una query contiene:

```php
->where('active', true)
```

el Query Engine deberá conocer el tipo semántico.

---

# 98. Typed parameter

```php
final readonly class TypedParameter
{
    public function __construct(
        public mixed $value,
        public TypeReference $type,
    ) {}
}
```

---

# 99. Parameter pipeline

```text
PHP Value
    ↓
TypedParameter
    ↓
Value Conversion
    ↓
Platform Binding Resolution
    ↓
Driver Binding
```

---

# 100. Query Builder

El Query Builder no deberá resolver directamente:

```text
PDO::PARAM_*
```

---

# 101. SQL Compiler

El SQL Compiler tampoco deberá interpretar arbitrariamente valores PHP.

---

# 102. Binding resolver

Conceptualmente:

```php
interface ParameterBindingResolver
{
    public function resolve(
        TypeReference $type,
        DatabasePlatform $platform,
    ): DriverBindingDescriptor;
}
```

---

# 103. Driver abstraction

El descriptor deberá permanecer independiente de PDO cuando sea posible.

---

# 104. PDO adapter

Un driver PDO podrá convertir:

```text
DriverBindingDescriptor
→
PDO::PARAM_*
```

cuando corresponda.

---

# 105. Result conversion

Lectura:

```text
Driver Raw Value
    ↓
Physical Type Context
    ↓
Logical Type
    ↓
Value Converter
    ↓
PHP Value
```

---

# 106. Raw DB values

Los drivers pueden devolver:

```text
int
string
float
resource
null
```

de formas distintas.

El ORM no deberá asumir uniformidad absoluta entre drivers.

---

# 107. Driver normalization

El Type System + Conversion System deberán proporcionar la frontera de normalización.

---

# 108. Hydration

Hydration consume valores ya convertidos al nivel requerido.

Idealmente:

```text
Raw Result
↓
Type Conversion
↓
Hydration Assignment
```

---

# 109. Hydrator ≠ Type Converter

Invariante:

```text
Hydrator
≠
TypeConverter
```

---

# 110. Schema integration

Schema utiliza tipos lógicos.

Ejemplo:

```php
$table->string('name', 255);
```

deberá producir conceptualmente:

```text
ColumnDefinition
    type = TypeReference(
        TypeId("string"),
        length=255
    )
```

---

# 111. Schema Compiler

Posteriormente:

```text
TypeReference
+
Platform
↓
PhysicalTypeDescriptor
↓
SQL declaration
```

---

# 112. Schema model independence

El Schema Model no deberá almacenar únicamente:

```text
VARCHAR(255)
```

como raw SQL.

Debe conservar semántica canónica cuando sea conocida.

---

# 113. Introspection

La introspección realiza el proceso inverso.

```text
Physical Database Type
    ↓
Platform Type Resolver
    ↓
Canonical TypeReference
```

---

# 114. Introspection uncertainty

No toda representación física puede reconstruirse exactamente.

Por tanto:

```text
Physical Type
→
Logical Type
```

puede producir:

```text
EXACT
INFERRED
AMBIGUOUS
UNKNOWN
```

---

# 115. TypeResolutionConfidence

```php
enum TypeResolutionConfidence
{
    case EXACT;
    case INFERRED;
    case AMBIGUOUS;
    case UNKNOWN;
}
```

---

# 116. Schema Diff

Debe considerar:

```text
Logical compatibility
+
Physical compatibility
+
Platform capabilities
```

antes de decidir que dos columnas son diferentes.

---

# 117. ORM mapping

Ejemplo:

```php
#[Column(type: 'uuid')]
private Uuid $id;
```

El mapping resolverá:

```text
Property
↓
PHP Type Descriptor
↓
ORM Mapping
↓
TypeReference
```

---

# 118. PHP reflection

Reflection puede ayudar durante metadata compilation.

No deberá repetirse durante hydration de cada row.

---

# 119. Compiled type metadata

ORM Metadata conservará:

```text
PropertyMetadata
├── PHP type
├── Database TypeReference
├── Converter
└── Accessor
```

precompilados.

---

# 120. Query Type System integration

El Query Engine ya posee:

```text
DATABASE_QUERY_TYPE_SYSTEM
```

El Database Type System será la fuente canónica de tipos persistibles.

---

# 121. Query semantic types

Query Engine puede además necesitar tipos conceptuales como:

```text
ENTITY
RELATIONSHIP
TUPLE
NULL
UNKNOWN
```

que no necesariamente son tipos almacenables.

---

# 122. Separation

```text
QuerySemanticType
≠
DatabaseType
```

---

# 123. Bridge

Debe existir un bridge explícito:

```text
Persistent Query Expression
    ↓
Database TypeReference
```

cuando aplique.

---

# 124. Literal inference

Ejemplo:

```php
->where('age', '>', 18)
```

puede inferir:

```text
18 → integer
```

pero el contexto de la columna puede refinar la decisión.

---

# 125. Contextual typing

```text
ColumnType
+
LiteralType
↓
ExpectedParameterType
```

---

# 126. Parameter inference

Debe evitar:

```text
PHP value alone
→ always final DB type
```

porque:

```php
"42"
```

puede representar:

- string;
- numeric ID;
- decimal;
- enum backing value.

---

# 127. Explicit typing wins

Cuando el usuario proporciona un tipo explícito válido:

```text
Explicit TypeReference
```

debe prevalecer sobre inferencia.

---

# 128. Validation

El sistema validará TypeReferences antes del hot path cuando sea posible.

---

# 129. Invalid decimal

```text
decimal(precision=2, scale=4)
```

debe fallar en metadata/schema compilation.

---

# 130. Invalid string length

```text
string(length=-1)
```

debe fallar.

---

# 131. Invalid arguments

Un tipo no deberá aceptar argumentos desconocidos silenciosamente.

---

# 132. Type argument schema

Cada tipo podrá declarar:

```php
interface TypeArgumentSchema
{
    public function normalize(array $arguments): TypeArguments;

    public function validate(TypeArguments $arguments): void;
}
```

---

# 133. Canonical arguments

Orden de argumentos no afectará identidad.

```text
decimal(scale=4, precision=19)
```

y:

```text
decimal(precision=19, scale=4)
```

deberán normalizarse igual.

---

# 134. Type fingerprint

```text
TypeFingerprint
=
Hash(
    TypeId
    +
    CanonicalArguments
    +
    RelevantSemantics
)
```

---

# 135. Nullability fingerprint

Cuando se fingerprinta un `TypeReference`, nullability sí puede formar parte.

---

# 136. DatabaseType fingerprint

Cuando se fingerprinta la definición global del tipo, no.

---

# 137. Platform type resolution

Contrato conceptual:

```php
interface PlatformTypeResolver
{
    public function resolve(
        TypeReference $logicalType,
        DatabasePlatform $platform,
    ): PhysicalTypeResolution;
}
```

---

# 138. PhysicalTypeResolution

```php
final readonly class PhysicalTypeResolution
{
    public function __construct(
        public PhysicalTypeDescriptor $type,
        public TypeSupportLevel $support,
        public array $limitations = [],
    ) {}
}
```

---

# 139. TypeSupportLevel

```php
enum TypeSupportLevel
{
    case NATIVE;
    case COMPATIBLE;
    case EMULATED;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 140. NATIVE

La plataforma posee representación nativa adecuada.

---

# 141. COMPATIBLE

No es idéntica, pero conserva las semánticas requeridas.

---

# 142. EMULATED

VoltStack necesita convenciones adicionales.

---

# 143. UNSUPPORTED

La semántica requerida no puede preservarse correctamente.

---

# 144. UNKNOWN

No existe evidencia suficiente.

---

# 145. Platform capabilities

Resolución deberá consultar capabilities, no:

```php
if ($platform->name() === 'mysql') {
}
```

por todas partes.

---

# 146. Vendor specialization

Los adapters concretos sí pueden contener conocimiento específico.

```text
MySQLTypeResolver
PostgreSQLTypeResolver
MariaDBTypeResolver
SQLiteTypeResolver
```

---

# 147. MySQL

Debe considerar características propias sin asumir que MariaDB es idéntico.

---

# 148. MariaDB

Será plataforma de primera clase.

```text
MariaDB
≠
MySQL alias
```

---

# 149. PostgreSQL

Podrá utilizar capacidades nativas como:

```text
UUID
JSONB
ARRAY
```

mediante tipos/extensiones cuando corresponda.

---

# 150. SQLite

Requiere especial atención a:

```text
type affinity
```

---

# 151. SQLite affinity ≠ logical type

El Type System conservará la semántica VoltStack aunque SQLite tenga un modelo físico más flexible.

---

# 152. Type affinity

Puede existir un descriptor:

```php
enum TypeAffinity
{
    case INTEGER;
    case REAL;
    case TEXT;
    case BLOB;
    case NUMERIC;
    case NONE;
}
```

principalmente para plataformas que lo necesiten.

---

# 153. Physical descriptor

```php
final readonly class PhysicalTypeDescriptor
{
    public function __construct(
        public string $canonicalName,
        public PhysicalTypeArguments $arguments,
        public TypeAffinity $affinity,
    ) {}
}
```

No debe convertirse en raw SQL demasiado pronto.

---

# 154. SQL declaration

Solo después:

```text
PhysicalTypeDescriptor
↓
Platform SQL Type Declaration Compiler
↓
VARCHAR(255)
```

---

# 155. Portable types

Un tipo es portable cuando sus semánticas requeridas pueden representarse en todas las plataformas objetivo.

---

# 156. Portable ≠ identical SQL

Ejemplo:

```text
UUID
```

puede ser portable aunque tenga declaraciones SQL diferentes.

---

# 157. Portability analysis

```php
interface TypePortabilityAnalyzer
{
    public function analyze(
        TypeReference $type,
        PlatformSet $targets,
    ): TypePortabilityReport;
}
```

---

# 158. Portability report

Puede indicar:

```text
portable
portable_with_limitations
requires_emulation
non_portable
unknown
```

---

# 159. Application portability

Esto permitirá advertir durante schema/migration planning antes del despliegue.

---

# 160. Custom types

Extensiones podrán registrar tipos.

Ejemplo:

```text
geography
money
inet
vector
```

---

# 161. Custom type rules

Un tipo custom deberá declarar explícitamente:

- TypeId;
- PHP representation;
- logical semantics;
- arguments;
- converters;
- binding requirements;
- platform support;
- schema behavior;
- compatibility rules.

---

# 162. No hidden custom type behavior

Un custom type no deberá depender de:

```text
global state
current request
current entity
EntityManager
```

para definir su identidad.

---

# 163. Contextual converters

Si una conversión requiere contexto, deberá declararlo explícitamente.

---

# 164. Type extension

El sistema detallado estará en:

```text
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

---

# 165. TypeDefinition

Modelo conceptual:

```php
final readonly class TypeDefinition
{
    public function __construct(
        public TypeId $id,
        public TypeFamily $family,
        public PhpTypeDescriptor $php,
        public TypeArgumentSchema $arguments,
        public TypeBehavior $behavior,
    ) {}
}
```

---

# 166. Behavior separation

Puede ser preferible mantener:

```text
TypeDefinition
```

como metadata pura y:

```text
TypeBehavior
```

como servicios stateless.

Esto evita almacenar servicios dentro de metadata serializable.

---

# 167. Recommended architecture

```text
TypeDefinition
    immutable metadata

TypeBehavior
    stateless operations

TypeRegistry
    definitions

TypeBehaviorRegistry
    behavior providers
```

---

# 168. Metadata purity

No colocar en `TypeDefinition`:

- Connection;
- EntityManager;
- current tenant;
- request;
- mutable caches;
- entity instances.

---

# 169. TypeBehavior

Puede exponer conceptualmente:

```php
interface TypeBehavior
{
    public function compatibility(): TypeCompatibilityRule;

    public function converter(): TypeValueConverter;

    public function binding(): TypeBindingStrategy;
}
```

---

# 170. Stateless preference

Los behaviors deberían ser stateless cuando sea posible.

---

# 171. ValueConversionContext

Cuando se necesite contexto:

```php
final readonly class ValueConversionContext
{
    public function __construct(
        public ConversionDirection $direction,
        public DatabasePlatform $platform,
        public TypeReference $type,
    ) {}
}
```

---

# 172. No EntityManager in conversion context

El Type System no deberá usar EntityManager para conversiones escalares.

---

# 173. Tenant context

Un tipo lógico no deberá cambiar porque cambie el tenant.

---

# 174. Tenant-specific physical schema

Si existe, pertenece a schema/platform resolution, no a identidad lógica del tipo.

---

# 175. Sharding

Mismo principio.

---

# 176. Security

El Type System deberá impedir conversiones arbitrarias basadas en nombres de clase provenientes de DB.

---

# 177. No arbitrary class instantiation

Nunca:

```php
$class = $row['type'];
return new $class($value);
```

---

# 178. Registered converters

Value Objects y custom types usarán converters previamente registrados.

---

# 179. Serialization safety

Metadata de tipos debe ser segura para:

- compiled container;
- metadata cache;
- preload;
- persistent workers.

---

# 180. Secrets

Type metadata nunca deberá contener:

- passwords;
- credentials;
- raw sensitive values.

---

# 181. Error taxonomy

Excepciones base:

```text
DatabaseTypeException
```

---

# 182. UnknownTypeException

Cuando un `TypeId` no exista.

---

# 183. InvalidTypeArgumentException

Cuando los argumentos sean inválidos.

---

# 184. TypeConversionException

Cuando un valor no pueda convertirse.

---

# 185. LossyTypeConversionException

Cuando se intenta una conversión no permitida con pérdida.

---

# 186. TypeOverflowException

Cuando un valor exceda el rango.

---

# 187. UnsupportedPlatformTypeException

Cuando una plataforma no pueda representar el tipo.

---

# 188. AmbiguousTypeResolutionException

Cuando la introspección física no permita resolver un tipo lógico con suficiente certeza y la operación requiera certeza.

---

# 189. TypeBindingException

Cuando un tipo no pueda enlazarse correctamente al driver.

---

# 190. InvalidTypeMappingException

Cuando ORM/PHP/database mapping sea incoherente.

---

# 191. Error context

Errores deberán incluir información estructurada:

```text
TypeId
TypeReference
Operation
Platform
Source representation
Target representation
```

sin exponer valores sensibles innecesariamente.

---

# 192. Diagnostics

Ejemplo:

```text
Database type conversion failed.

Logical type:
    decimal(19,4)

Direction:
    PHP_TO_DATABASE

Platform:
    PostgreSQL

Reason:
    Value exceeds declared precision.

Expected:
    maximum precision 19

Actual:
    precision 22
```

---

# 193. Explain API

Conceptualmente:

```php
Database::types()->explain('uuid');
```

podría mostrar:

```text
Logical Type:
    uuid

PHP Representation:
    Uuid|string

PostgreSQL:
    native UUID

MySQL:
    CHAR(36) / BINARY(16), depending on policy

SQLite:
    TEXT/BLOB representation

Binding:
    platform dependent
```

---

# 194. Type inspection

CLI futura:

```text
volt database:type uuid
```

---

# 195. Type portability CLI

Conceptualmente:

```text
volt database:type:portability uuid
```

---

# 196. Cache architecture

Podrán cachearse:

```text
normalized TypeReferences
compatibility decisions
platform resolutions
binding descriptors
```

cuando todas las entradas sean immutable.

---

# 197. Cache key

Ejemplo:

```text
TypeReferenceFingerprint
×
PlatformCapabilityFingerprint
×
TypeRegistryGeneration
```

---

# 198. Registry generation

Cuando cambie la configuración de tipos:

```text
TypeRegistryGeneration
```

deberá invalidar caches dependientes.

---

# 199. Platform generation

Cambios de capabilities/version también podrán invalidar resoluciones.

---

# 200. No value cache

No cachear valores convertidos globalmente como parte del Type System.

---

# 201. Persistent runtime

Compartible entre requests:

```text
Type definitions
Type registry
Argument schemas
Stateless behaviors
Platform mappings
Compiled normalization rules
```

---

# 202. Request scoped

Solo cuando sea estrictamente necesario:

```text
conversion operation context
temporary diagnostics
```

---

# 203. No cross-request mutable type state

Invariante obligatoria para FrankenPHP/RoadRunner/OpenSwoole.

---

# 204. Performance

Resolución de tipos estará en hot paths.

Debe ser extremadamente barata.

---

# 205. Hot path target

Después de bootstrap:

```text
TypeId
→ indexed registry lookup
```

idealmente O(1).

---

# 206. No reflection hot path

Reflection solo en metadata compilation cuando sea necesaria.

---

# 207. No repeated argument parsing

`TypeReference` deberá estar normalizado previamente.

---

# 208. No repeated platform negotiation

Platform resolutions podrán cachearse.

---

# 209. Type interning

VoltStack podrá internar `TypeReference` comunes.

Ejemplo:

```text
int:not-null
string(255):nullable
uuid:not-null
```

---

# 210. Interning requirement

Solo objetos immutable.

---

# 211. Interning ≠ mutable singleton

No confundir.

---

# 212. Memory management

Caches deberán ser bounded cuando puedan recibir TypeReferences dinámicos.

---

# 213. User-defined arbitrary lengths

Un usuario podría producir:

```text
string(1)
string(2)
...
string(1000000)
```

No permitir crecimiento global ilimitado del intern pool.

---

# 214. Bounded cache

Aplicar políticas:

```text
max entries
LRU
generation invalidation
```

cuando corresponda.

---

# 215. Telemetry

Métricas posibles:

```text
database.type.resolution
database.type.resolution.cache_hit
database.type.conversion
database.type.conversion.failure
database.type.lossy_conversion
database.type.binding.failure
database.type.platform_unsupported
database.type.introspection_ambiguous
```

---

# 216. Metric cardinality

`TypeId` puede ser label si el registry está acotado.

Evitar valores concretos.

---

# 217. Tracing

Las conversiones normales no necesitan un span por valor.

Eso sería demasiado costoso.

---

# 218. Error events

Errores importantes sí pueden adjuntarse al span de query/schema/migration correspondiente.

---

# 219. Testing

Cada tipo core deberá probar:

```text
normalization
argument validation
PHP → DB
DB → PHP
binding
platform resolution
compatibility
overflow
null
invalid input
round trip
```

---

# 220. Round-trip invariant

Cuando el tipo sea lossless:

```text
PHP value
→ DB representation
→ PHP value
```

deberá preservar semántica.

Formalmente:

```text
decode(encode(x)) ≡ x
```

para todo `x` válido bajo el mapping.

---

# 221. Cross-platform tests

Los tipos portables deberán probarse en:

- MySQL;
- MariaDB;
- PostgreSQL;
- SQLite.

---

# 222. Driver conformance

Drivers deberán cumplir las expectativas de binding/result representation declaradas.

---

# 223. Property-based tests

Especialmente útiles para:

- integers;
- decimals;
- UUID;
- temporal;
- strings;
- binary.

---

# 224. Boundary tests

Ejemplo integer:

```text
MIN
MIN - 1
MAX
MAX + 1
```

---

# 225. Decimal boundaries

Probar:

```text
maximum precision
maximum scale
rounding boundaries
negative values
zero
```

---

# 226. Temporal boundaries

Probar:

- leap years;
- DST transitions cuando aplique;
- microseconds;
- timezone offsets;
- min/max platform ranges.

---

# 227. Fuzzing

Custom converters pueden beneficiarse de fuzz testing.

---

# 228. Directory structure

```text
src/Quantum/Database/Type/
│
├── Contract/
│   ├── DatabaseType.php
│   ├── TypeArgumentSchema.php
│   ├── TypeBehavior.php
│   ├── TypeCompatibilityRule.php
│   ├── TypeValueConverter.php
│   ├── TypeBindingStrategy.php
│   ├── PlatformTypeResolver.php
│   └── TypePortabilityAnalyzer.php
│
├── Definition/
│   ├── TypeDefinition.php
│   ├── TypeId.php
│   ├── TypeFamily.php
│   ├── TypeReference.php
│   ├── TypeArguments.php
│   ├── TypeCharacteristics.php
│   ├── PhpTypeDescriptor.php
│   └── DatabaseTypeDescriptor.php
│
├── Builtin/
│   ├── Boolean/
│   ├── Integer/
│   ├── Decimal/
│   ├── FloatingPoint/
│   ├── String/
│   ├── Binary/
│   ├── Identifier/
│   ├── Json/
│   ├── Temporal/
│   └── Enum/
│
├── Compatibility/
│   ├── TypeCompatibility.php
│   ├── TypeCompatibilityAnalyzer.php
│   ├── TypeAssignabilityAnalyzer.php
│   └── TypeConversionSafetyAnalyzer.php
│
├── Normalization/
│   ├── TypeNormalizer.php
│   ├── TypeArgumentNormalizer.php
│   └── TypeAliasResolver.php
│
├── Platform/
│   ├── PhysicalTypeDescriptor.php
│   ├── PhysicalTypeArguments.php
│   ├── PhysicalTypeResolution.php
│   ├── TypeSupportLevel.php
│   ├── TypeAffinity.php
│   ├── MySQL/
│   ├── MariaDB/
│   ├── PostgreSQL/
│   └── SQLite/
│
├── Binding/
│   ├── DriverBindingDescriptor.php
│   └── ParameterBindingResolver.php
│
├── Portability/
│   ├── TypePortabilityReport.php
│   └── DefaultTypePortabilityAnalyzer.php
│
├── Cache/
│   ├── TypeResolutionCache.php
│   ├── TypeCompatibilityCache.php
│   └── TypeReferenceInternPool.php
│
├── Diagnostics/
│   ├── TypeExplainer.php
│   └── TypeDiagnosticReport.php
│
└── Exception/
    ├── DatabaseTypeException.php
    ├── UnknownTypeException.php
    ├── InvalidTypeArgumentException.php
    ├── TypeConversionException.php
    ├── LossyTypeConversionException.php
    ├── TypeOverflowException.php
    ├── UnsupportedPlatformTypeException.php
    ├── AmbiguousTypeResolutionException.php
    ├── TypeBindingException.php
    └── InvalidTypeMappingException.php
```

---

# 229. Dependency rules

Permitido:

```text
ORM
 ↓
Type System

Query Engine
 ↓
Type System

Schema
 ↓
Type System

Hydration
 ↓
Type System

Compiler
 ↓
Type System

Binding
 ↓
Type System

Platform
 ↓
Type contracts
```

---

# 230. Dependency inversion

El Type System no deberá depender de:

```text
EntityManager
Repository
UnitOfWork
HTTP
Controller
Jobs
Authentication
Authorization
```

---

# 231. Driver boundary

El core Type System tampoco deberá depender directamente de PDO.

---

# 232. Type pipeline

```text
                PHP / Domain Value
                        │
                        ▼
                 PHP Type Descriptor
                        │
                        ▼
                  ORM Type Mapping
                        │
                        ▼
                  TypeReference
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
       Query Type     Schema      Conversion
           │            │            │
           ▼            ▼            ▼
       Parameter     Physical      DB Value
        Binding       Type            │
           │            │             │
           └────────────┼─────────────┘
                        ▼
                     Platform
                        │
                        ▼
                      Driver
```

---

# 233. Write pipeline

```text
PHP Value
    ↓
ORM Property Metadata
    ↓
TypeReference
    ↓
Value Converter
    ↓
Database Representation
    ↓
Binding Resolver
    ↓
Driver
    ↓
Database
```

---

# 234. Read pipeline

```text
Database
    ↓
Driver
    ↓
Raw Value
    ↓
TypeReference
    ↓
Value Converter
    ↓
PHP/Domain Value
    ↓
Hydrator
    ↓
Entity
```

---

# 235. Schema pipeline

```text
Schema Builder
    ↓
TypeReference
    ↓
Platform Type Resolver
    ↓
PhysicalTypeDescriptor
    ↓
Schema Compiler
    ↓
SQL Type Declaration
```

---

# 236. Introspection pipeline

```text
Database Metadata
    ↓
Physical Type
    ↓
Platform Introspection Resolver
    ↓
TypeResolution
    ↓
Canonical TypeReference
    ↓
Schema Model
```

---

# 237. Architectural invariants

## DB-TYPE-001

PHP Type no será equivalente a Database Type.

## DB-TYPE-002

ORM Type no será equivalente a SQL Type.

## DB-TYPE-003

Logical Type no será equivalente a Physical Type.

## DB-TYPE-004

Physical Type no será equivalente a SQL declaration.

## DB-TYPE-005

SQL declaration no será equivalente a driver binding.

## DB-TYPE-006

TypeId será estable.

## DB-TYPE-007

TypeId no dependerá de FQCN.

## DB-TYPE-008

DatabaseType será immutable.

## DB-TYPE-009

TypeReference será immutable.

## DB-TYPE-010

TypeDefinition será immutable.

## DB-TYPE-011

Nullability será independiente del TypeId.

## DB-TYPE-012

Type arguments serán canonicalizados.

## DB-TYPE-013

Argument order no alterará identidad.

## DB-TYPE-014

Argumentos inválidos fallarán explícitamente.

## DB-TYPE-015

Argumentos desconocidos no serán ignorados silenciosamente.

## DB-TYPE-016

DECIMAL será semánticamente distinto de FLOAT.

## DB-TYPE-017

Conversión DECIMAL→FLOAT no será implícitamente lossless.

## DB-TYPE-018

Integer overflow nunca truncará silenciosamente.

## DB-TYPE-019

BOOLEAN será semánticamente distinto de INTEGER.

## DB-TYPE-020

UUID será independiente de su representación física.

## DB-TYPE-021

ULID será independiente de su representación física.

## DB-TYPE-022

JSON será independiente de JSON/JSONB/TEXT físico.

## DB-TYPE-023

DateTime no será automáticamente equivalente a Timestamp.

## DB-TYPE-024

Timezone no será inferida silenciosamente cuando sea semánticamente necesaria.

## DB-TYPE-025

PHP Enum no será equivalente a DB native enum.

## DB-TYPE-026

Value Object no será reducido obligatoriamente a scalar.

## DB-TYPE-027

Multi-column Value Objects serán posibles.

## DB-TYPE-028

Type equality será distinta de compatibility.

## DB-TYPE-029

Compatibility será distinta de convertibility.

## DB-TYPE-030

Convertibility será distinta de assignability.

## DB-TYPE-031

Platform representability será una pregunta independiente.

## DB-TYPE-032

UNKNOWN no significará compatible.

## DB-TYPE-033

Lossy conversion requerirá política explícita.

## DB-TYPE-034

Silent lossy conversion estará prohibida por defecto.

## DB-TYPE-035

Value-dependent conversion será validada con el valor real.

## DB-TYPE-036

Hydrator no será TypeConverter.

## DB-TYPE-037

Query Builder no resolverá PDO binding.

## DB-TYPE-038

SQL Compiler no interpretará arbitrariamente PHP values.

## DB-TYPE-039

Driver binding se resolverá mediante abstracción.

## DB-TYPE-040

Core Type System no dependerá de PDO.

## DB-TYPE-041

Schema utilizará TypeReference.

## DB-TYPE-042

Schema Model no dependerá exclusivamente de raw SQL type strings.

## DB-TYPE-043

Schema Compiler resolverá tipos mediante Platform.

## DB-TYPE-044

Introspection realizará physical→logical resolution.

## DB-TYPE-045

Introspection uncertainty será explícita.

## DB-TYPE-046

AMBIGUOUS no será convertido en EXACT.

## DB-TYPE-047

UNKNOWN no será convertido en EXACT.

## DB-TYPE-048

ORM metadata compilará type information.

## DB-TYPE-049

Reflection no será requerida por row hydration.

## DB-TYPE-050

Query Semantic Type será distinto de DatabaseType.

## DB-TYPE-051

El bridge Query→DatabaseType será explícito.

## DB-TYPE-052

Literal inference será contextual.

## DB-TYPE-053

PHP runtime type no determinará siempre el DB type final.

## DB-TYPE-054

Explicit TypeReference válido tendrá prioridad sobre inferencia.

## DB-TYPE-055

TypeRegistry se congelará después de bootstrap por defecto.

## DB-TYPE-056

No habrá runtime arbitrary class discovery para tipos.

## DB-TYPE-057

Type aliases serán distintos de canonical TypeId.

## DB-TYPE-058

Hot path usará canonical TypeIds.

## DB-TYPE-059

Type lookup será O(1) promedio.

## DB-TYPE-060

No habrá reflection en type lookup hot path.

## DB-TYPE-061

Platform capability checks reemplazarán vendor conditionals dispersos.

## DB-TYPE-062

MySQL y MariaDB tendrán resolvers separados.

## DB-TYPE-063

SQLite affinity no reemplazará logical type semantics.

## DB-TYPE-064

Portable no significará identical SQL.

## DB-TYPE-065

Platform emulation será explícita.

## DB-TYPE-066

UNSUPPORTED no será emulado silenciosamente.

## DB-TYPE-067

UNKNOWN platform support no será tratado como NATIVE.

## DB-TYPE-068

Custom types tendrán TypeId estable.

## DB-TYPE-069

Custom type identity no dependerá de request state.

## DB-TYPE-070

Custom types no dependerán de EntityManager para identidad.

## DB-TYPE-071

Registered converters reemplazarán arbitrary class instantiation.

## DB-TYPE-072

DB values no determinarán clases PHP arbitrarias.

## DB-TYPE-073

Type metadata no contendrá entity instances.

## DB-TYPE-074

Type metadata no contendrá Connection.

## DB-TYPE-075

Type metadata no contendrá current tenant mutable state.

## DB-TYPE-076

Type metadata no contendrá credentials.

## DB-TYPE-077

Type metadata será segura para persistent workers.

## DB-TYPE-078

Type registry immutable podrá compartirse entre requests.

## DB-TYPE-079

Mutable conversion state no se compartirá entre requests.

## DB-TYPE-080

FrankenPHP no reutilizará conversion state mutable entre requests.

## DB-TYPE-081

RoadRunner mantendrá el mismo aislamiento.

## DB-TYPE-082

OpenSwoole mantendrá aislamiento concurrente.

## DB-TYPE-083

Type resolution podrá cachearse.

## DB-TYPE-084

Cache key incluirá registry generation.

## DB-TYPE-085

Platform-dependent cache incluirá platform capability fingerprint.

## DB-TYPE-086

Converted user values no serán globalmente cacheados.

## DB-TYPE-087

Interning solo se aplicará a objetos immutable.

## DB-TYPE-088

Intern pool será bounded cuando acepte combinaciones dinámicas.

## DB-TYPE-089

Un usuario no podrá provocar crecimiento global ilimitado de TypeReferences.

## DB-TYPE-090

Telemetry no emitirá raw values como labels.

## DB-TYPE-091

Normal conversion no creará un trace span por valor.

## DB-TYPE-092

Conversion failures podrán adjuntarse a telemetry existente.

## DB-TYPE-093

Core types tendrán round-trip tests.

## DB-TYPE-094

Lossless mappings cumplirán decode(encode(x)) ≡ x.

## DB-TYPE-095

Portable types tendrán cross-platform tests.

## DB-TYPE-096

Drivers tendrán type-binding conformance tests.

## DB-TYPE-097

Boundary values serán probados.

## DB-TYPE-098

Decimal precision/scale serán validados.

## DB-TYPE-099

Temporal precision será explícita.

## DB-TYPE-100

Physical representation no alterará silenciosamente logical semantics.

## DB-TYPE-101

Schema Diff considerará type compatibility.

## DB-TYPE-102

Schema Diff no comparará únicamente SQL strings.

## DB-TYPE-103

Migration planning podrá consultar portability.

## DB-TYPE-104

Migration safety podrá conocer conversiones lossy.

## DB-TYPE-105

Parameter values estarán asociados a TypeReference cuando sea posible.

## DB-TYPE-106

Parameter binding será platform-aware.

## DB-TYPE-107

Result conversion será type-aware.

## DB-TYPE-108

Driver raw result types no se asumirán uniformes.

## DB-TYPE-109

Driver normalization tendrá frontera explícita.

## DB-TYPE-110

Type conversion errors tendrán contexto estructurado.

## DB-TYPE-111

Error diagnostics evitarán valores sensibles.

## DB-TYPE-112

TypeDefinition metadata y TypeBehavior podrán separarse.

## DB-TYPE-113

TypeBehavior será stateless cuando sea posible.

## DB-TYPE-114

Contextual behavior recibirá contexto explícito.

## DB-TYPE-115

Type conversion context no contendrá EntityManager por defecto.

## DB-TYPE-116

Tenant no cambiará identidad lógica del tipo.

## DB-TYPE-117

Shard no cambiará identidad lógica del tipo.

## DB-TYPE-118

Platform-specific representation pertenecerá al Platform resolver.

## DB-TYPE-119

SQL declaration pertenecerá al compiler/platform layer.

## DB-TYPE-120

Type System no ejecutará queries.

## DB-TYPE-121

Type System no abrirá conexiones.

## DB-TYPE-122

Type System no persistirá entidades.

## DB-TYPE-123

Type System no hidratará entidades.

## DB-TYPE-124

Type System no administrará UnitOfWork.

## DB-TYPE-125

Type System no será serializer.

## DB-TYPE-126

Type System no será validator de dominio general.

## DB-TYPE-127

Type System podrá validar restricciones propias del tipo.

## DB-TYPE-128

Domain validation será responsabilidad separada.

## DB-TYPE-129

Type portability será explicable.

## DB-TYPE-130

Platform limitations serán explícitas.

## DB-TYPE-131

Emulation limitations serán explícitas.

## DB-TYPE-132

No se prometerá portabilidad cuando exista UNKNOWN.

## DB-TYPE-133

Type fingerprints serán deterministas.

## DB-TYPE-134

Canonical TypeReference será estable dentro de una registry generation.

## DB-TYPE-135

Registry changes invalidarán caches dependientes.

## DB-TYPE-136

Type aliases legacy podrán normalizarse.

## DB-TYPE-137

Canonical writes deberán usar canonical TypeId cuando el TypeId se persista en metadata.

## DB-TYPE-138

Type registry conflicts fallarán en bootstrap.

## DB-TYPE-139

Duplicate TypeId será error.

## DB-TYPE-140

Invalid core type override será error salvo extension policy explícita.

## DB-TYPE-141

Core type semantics no cambiarán silenciosamente mediante plugin.

## DB-TYPE-142

Plugin type registration será determinista.

## DB-TYPE-143

Plugin loading order no deberá alterar resolución final sin conflicto explícito.

## DB-TYPE-144

Type resolution podrá explicarse mediante diagnostics.

## DB-TYPE-145

Explain no ejecutará queries.

## DB-TYPE-146

Explain no necesitará entity instances.

## DB-TYPE-147

Type system behavior será independiente del HTTP layer.

## DB-TYPE-148

Type system behavior será independiente del application server.

## DB-TYPE-149

Correctness tendrá prioridad sobre conversiones convenientes.

## DB-TYPE-150

VoltStack nunca fabricará certeza de tipo cuando la evidencia sea ambigua.

---

# 238. Anti-patterns

## 238.1 Usar SQL strings como tipos

```php
$type = 'VARCHAR(255)';
```

como representación canónica.

**Rechazado.**

---

## 238.2 TypeId basado en clase

```php
$typeId = IntegerType::class;
```

**Rechazado.**

---

## 238.3 Decimal convertido automáticamente a float

**Rechazado.**

---

## 238.4 BigInt truncado a int

**Rechazado.**

---

## 238.5 UUID siempre CHAR(36)

**Rechazado.**

---

## 238.6 Boolean siempre TINYINT

**Rechazado.**

---

## 238.7 MySQL === MariaDB

**Rechazado.**

---

## 238.8 SQLite affinity como semántica lógica

**Rechazado.**

---

## 238.9 PDO constants dentro del Query Builder

**Rechazado.**

---

## 238.10 Reflection por cada row

**Rechazado.**

---

## 238.11 Global mutable TypeRegistry durante runtime

**Rechazado.**

---

## 238.12 Converter que obtiene EntityManager global

**Rechazado.**

---

## 238.13 Arbitrary class instantiation desde datos DB

**Rechazado.**

---

## 238.14 Silent lossy conversion

**Rechazado.**

---

## 238.15 UNKNOWN tratado como compatible

**Rechazado.**

---

# 239. Ejemplo de API

Schema:

```php
Schema::create('products', function (Table $table) {
    $table->uuid('id');

    $table->string('name', length: 200);

    $table->decimal(
        'price',
        precision: 19,
        scale: 4,
    );

    $table->boolean('active');

    $table->json('metadata');

    $table->datetime(
        'created_at',
        precision: 6,
    );
});
```

Conceptualmente se convierte a:

```text
id
→ uuid

name
→ string(length=200)

price
→ decimal(precision=19, scale=4)

active
→ bool

metadata
→ json

created_at
→ datetime(precision=6)
```

---

# 240. ORM example

```php
final class Product
{
    #[Column(type: 'uuid')]
    private Uuid $id;

    #[Column(type: 'string', length: 200)]
    private string $name;

    #[Column(type: 'decimal', precision: 19, scale: 4)]
    private string $price;

    #[Column(type: 'bool')]
    private bool $active;

    #[Column(type: 'json')]
    private array $metadata;

    #[Column(type: 'datetime_immutable')]
    private DateTimeImmutable $createdAt;
}
```

La metadata compilada no conservará simplemente los strings del atributo.

Generará `TypeReference` canónicos.

---

# 241. Compiled metadata

```text
Product.price
│
├── PHP Type
│   └── string
│
├── Database Type
│   └── decimal
│
├── Arguments
│   ├── precision = 19
│   └── scale = 4
│
├── Nullability
│   └── NOT_NULL
│
└── Converter
    └── DecimalConverter
```

---

# 242. Query example

```php
Product::query()
    ->where('price', '>=', '100.0000')
    ->get();
```

Semantic analysis:

```text
Product.price
→ decimal(19,4)

literal
→ string candidate

contextual expected type
→ decimal(19,4)

conversion
→ validated decimal representation

binding
→ platform-compatible binding
```

---

# 243. Portability example

Logical schema:

```text
id: uuid
metadata: json
price: decimal(19,4)
```

Puede resolverse como:

```text
PostgreSQL
├── UUID
├── JSONB/JSON according to policy
└── NUMERIC(19,4)

MySQL
├── CHAR/BINARY according to UUID policy
├── JSON
└── DECIMAL(19,4)

MariaDB
├── platform capability based UUID representation
├── platform capability based JSON representation
└── DECIMAL(19,4)

SQLite
├── TEXT/BLOB
├── TEXT
└── NUMERIC-compatible representation
```

sin modificar el modelo lógico de la aplicación.

---

# 244. Relación con otros documentos

```text
030_DATABASE_QUERY_TYPE_SYSTEM
        │
        ▼
155_DATABASE_TYPE_SYSTEM
        │
        ├── 156 Type Registry
        ├── 157 Value Conversion
        ├── 158 Casting
        ├── 159 Enum Mapping
        ├── 160 Value Object Mapping
        ├── 161 JSON Type
        ├── 162 Date/Time Type
        └── 163 Custom Type Extension
```

Además:

```text
155 Type System
│
├── Schema
├── Schema Diff
├── Migration
├── ORM Mapping
├── Query Semantic Analysis
├── Parameter Binding
├── Result Conversion
├── Hydration
└── Platform Capabilities
```

---

# 245. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Canonical Logical Type Model
```

en lugar de:

```text
Vendor SQL Type Model
```

como núcleo.

Adoptará:

```text
TypeId + TypeReference
```

como representación estable.

Adoptará:

```text
Platform Type Resolution
```

para decidir representación física.

Adoptará:

```text
Explicit Value Conversion
```

para cruzar fronteras PHP/database.

Adoptará:

```text
Explicit Compatibility Classification
```

en lugar de booleanos ambiguos.

Adoptará:

```text
Immutable Compiled Type Metadata
```

para rendimiento y persistent runtimes.

---

# 246. Regla maestra

> **VoltStack no confundirá la forma en que PHP representa un valor, la forma en que el ORM entiende ese valor, la semántica lógica con la que la base de datos debe almacenarlo, la representación física elegida por una plataforma ni el mecanismo con el que el driver lo transmite. Cada frontera será explícita, tipada y verificable.**

Formalmente:

```text
Domain/PHP Value
        │
        ▼
    ORM Mapping
        │
        ▼
Canonical TypeReference
        │
        ├──────────────► Query Semantics
        │
        ├──────────────► Schema Semantics
        │
        ├──────────────► Conversion
        │
        └──────────────► Binding
        │
        ▼
Platform Type Resolution
        │
        ▼
Physical Representation
        │
        ▼
Driver
        │
        ▼
Database
```

La consecuencia es:

```text
Logical Semantics
    remain stable

while

Physical Representation
    may vary by platform
```

Esta separación permitirá que el resto del Database System sea portable sin sacrificar las capacidades específicas de PostgreSQL, MySQL, MariaDB o SQLite.

---

# 247. Siguiente documento

```text
156_DATABASE_TYPE_REGISTRY_SYSTEM.md
```

El siguiente documento deberá definir la infraestructura encargada de registrar, resolver, congelar, versionar y extender los tipos disponibles.

Regla central propuesta:

> **`TypeRegistry` será el catálogo canónico, immutable después del bootstrap y determinista de todos los tipos conocidos por una generación de VoltStack Database; ningún componente del hot path deberá descubrir tipos mediante reflection, class scanning o nombres de clase provenientes de datos externos.**

Deberá cubrir:

- `TypeRegistry`;
- `TypeRegistration`;
- `TypeId`;
- aliases;
- core types;
- custom types;
- plugin registrations;
- duplicate detection;
- override policy;
- registry builder;
- registry compiler;
- frozen registry;
- registry generations;
- registry fingerprint;
- alias normalization;
- canonical lookup;
- behavior registry;
- platform extensions;
- dependency validation;
- deterministic ordering;
- cache invalidation;
- persistent runtime sharing;
- diagnostics;
- testing;
- extension safety.