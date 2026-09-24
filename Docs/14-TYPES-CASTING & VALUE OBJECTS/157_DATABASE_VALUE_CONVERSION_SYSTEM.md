# 157_DATABASE_VALUE_CONVERSION_SYSTEM.md

# VoltStack Quantum Database
## Database Value Conversion System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 157 — Database Value Conversion System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `156_DATABASE_TYPE_REGISTRY_SYSTEM.md`

---

# 1. Propósito

`Database Value Conversion System` define la infraestructura responsable de transformar valores entre las distintas representaciones utilizadas por VoltStack Database.

El sistema será la frontera controlada entre:

```text
PHP / Domain Values
        ↕
ORM Logical Values
        ↕
Database Logical Representation
        ↕
Platform Representation
        ↕
Driver Values
```

y deberá evitar conversiones implícitas, ambiguas o destructivas que puedan producir:

- overflow;
- pérdida de precisión;
- cambios silenciosos de timezone;
- booleanos inconsistentes;
- UUID inválidos;
- JSON mal formado;
- enums desconocidos;
- conversiones `NULL` incorrectas;
- casting inseguro;
- corrupción de Value Objects.

La regla central será:

> **Value Conversion será una frontera bidireccional, explícita y tipada entre representaciones PHP/domain y database/driver; toda conversión conocerá su `TypeReference`, dirección y contexto, preservará semántica cuando se declare lossless y rechazará overflow, pérdida de precisión, ambigüedad o coerción insegura en lugar de alterar silenciosamente el valor.**

---

# 2. Relación con documentos anteriores

El documento:

```text
155_DATABASE_TYPE_SYSTEM.md
```

definió:

```text
TypeId
TypeReference
TypeDefinition
TypeBehavior
TypeCompatibility
```

El documento:

```text
156_DATABASE_TYPE_REGISTRY_SYSTEM.md
```

definió:

```text
TypeRegistry
TypeBehaviorRegistry
TypeReferenceFactory
Registry Generation
```

Ahora:

```text
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
```

define cómo esos tipos transforman valores.

---

# 3. Problema arquitectónico

Una entidad puede contener:

```php
final class Product
{
    private Uuid $id;

    private string $price;

    private bool $active;

    private DateTimeImmutable $createdAt;
}
```

pero la base de datos podría almacenar:

```text
id
→ native UUID
→ CHAR(36)
→ BINARY(16)

price
→ DECIMAL(19,4)

active
→ BOOLEAN
→ INTEGER 0/1

created_at
→ TIMESTAMP
→ DATETIME string
```

Por tanto:

```text
PHP Value
≠
Database Value
```

aunque representen el mismo concepto lógico.

---

# 4. Pipeline general

Escritura:

```text
PHP / Domain Value
        │
        ▼
TypeReference
        │
        ▼
TypeValueConverter
        │
        ▼
Logical DB Representation
        │
        ▼
Platform Conversion
        │
        ▼
Driver Representation
        │
        ▼
Parameter Binding
        │
        ▼
Database
```

Lectura:

```text
Database
    │
    ▼
Driver Raw Value
    │
    ▼
Platform Normalization
    │
    ▼
Logical DB Representation
    │
    ▼
TypeValueConverter
    │
    ▼
PHP / Domain Value
    │
    ▼
Hydration
```

---

# 5. Conversión ≠ Binding

Debe mantenerse:

```text
Value Conversion
≠
Parameter Binding
```

Value Conversion responde:

> ¿Qué valor lógico debe enviarse?

Binding responde:

> ¿Cómo lo transmite el driver?

Ejemplo:

```text
Uuid object
    ↓ conversion
"550e8400-e29b-41d4-a716-446655440000"
    ↓ binding
driver string parameter
```

---

# 6. Conversión ≠ Casting

También:

```text
Value Conversion
≠
Casting
```

Casting será desarrollado en:

```text
158_DATABASE_CASTING_SYSTEM.md
```

Conversión transforma representaciones necesarias para persistencia.

Casting define cómo una capa desea interpretar/exponer un valor.

---

# 7. Conversión ≠ Hydration

Hydration:

```text
converted value
→ entity property
```

Conversion:

```text
raw/database value
→ typed PHP/domain value
```

Por tanto:

```text
Hydrator
≠
TypeValueConverter
```

---

# 8. Conversión ≠ Validation

El sistema puede validar si un valor puede convertirse correctamente.

No reemplaza:

```text
business validation
```

Ejemplo:

```text
decimal accepts -100
```

no significa que:

```text
product.price = -100
```

sea permitido por dominio.

---

# 9. Direcciones de conversión

```php
enum ValueConversionDirection
{
    case PHP_TO_DATABASE;
    case DATABASE_TO_PHP;
    case PHP_TO_LOGICAL;
    case LOGICAL_TO_PHP;
    case LOGICAL_TO_PLATFORM;
    case PLATFORM_TO_LOGICAL;
}
```

En V1, las dos principales serán:

```text
PHP_TO_DATABASE
DATABASE_TO_PHP
```

pero internamente se conservarán fronteras explícitas.

---

# 10. TypeValueConverter

Contrato:

```php
interface TypeValueConverter
{
    public function toDatabase(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;

    public function toPhp(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;
}
```

---

# 11. Converter resolution

El converter deberá resolverse mediante:

```text
TypeReference
    ↓
TypeRegistry
    ↓
TypeBehavior
    ↓
TypeValueConverter
```

No mediante:

```text
switch ($typeName)
```

distribuido.

---

# 12. ValueConversionContext

```php
final readonly class ValueConversionContext
{
    public function __construct(
        public ValueConversionDirection $direction,
        public DatabasePlatform $platform,
        public TypeReference $type,
        public ConversionPolicy $policy,
        public ?ColumnConversionContext $column = null,
    ) {}
}
```

---

# 13. Contexto mínimo

El contexto no deberá contener por defecto:

- `EntityManager`;
- `UnitOfWork`;
- `Repository`;
- HTTP request;
- actor autenticado;
- entidad actual.

Los tipos escalares deberán poder convertirse independientemente del ORM.

---

# 14. ColumnConversionContext

Cuando sea útil:

```php
final readonly class ColumnConversionContext
{
    public function __construct(
        public ?string $columnName,
        public ?TableIdentifier $table,
        public Nullability $nullability,
    ) {}
}
```

Principalmente para diagnostics.

---

# 15. Contexto ≠ estado mutable global

Nunca:

```php
ValueConverter::$currentPlatform;
```

---

# 16. Conversión bidireccional

Para un mapping lossless:

```text
PHP
→ Database
→ PHP
```

deberá conservar semántica.

Formalmente:

```text
decode(encode(x)) ≡ x
```

para valores válidos.

---

# 17. Round-trip equivalence

No siempre exige igualdad binaria.

Ejemplo:

```text
DateTimeImmutable UTC
```

puede serializarse y reconstruirse como un nuevo objeto equivalente.

Por tanto:

```text
=== object identity
```

no es requisito.

La equivalencia será semántica.

---

# 18. Conversion result

En operaciones avanzadas puede ser útil distinguir el resultado de la conversión.

```php
final readonly class ConversionResult
{
    public function __construct(
        public mixed $value,
        public ConversionSafety $safety,
    ) {}
}
```

---

# 19. ConversionSafety

```php
enum ConversionSafety
{
    case LOSSLESS;
    case NORMALIZED;
    case LOSSY;
}
```

---

# 20. LOSSLESS

La información semántica se conserva.

---

# 21. NORMALIZED

El formato cambia, pero no la semántica.

Ejemplo:

```text
UUID uppercase
→ UUID canonical lowercase
```

---

# 22. LOSSY

Parte de la información se pierde.

Ejemplo:

```text
DateTime microseconds
→ DB datetime without fractional seconds
```

---

# 23. Lossy conversion policy

```php
enum LossyConversionPolicy
{
    case FORBID;
    case WARN;
    case ALLOW_EXPLICIT;
}
```

Default recomendado:

```text
FORBID
```

cuando el TypeReference exige preservar esa información.

---

# 24. No silent coercion

Esto deberá fallar por defecto:

```text
"123abc"
→ integer 123
```

---

# 25. Strict conversion

Preferir:

```text
valid integer representation
```

sobre coerciones PHP automáticas.

---

# 26. PHP coercion hazard

Evitar:

```php
(int) 'abc';
```

porque produce:

```text
0
```

sin evidencia de que ese sea el valor correcto.

---

# 27. Conversion policies

```php
final readonly class ConversionPolicy
{
    public function __construct(
        public ConversionStrictness $strictness,
        public LossyConversionPolicy $lossy,
        public NullConversionPolicy $nulls,
    ) {}
}
```

---

# 28. ConversionStrictness

```php
enum ConversionStrictness
{
    case STRICT;
    case NORMALIZING;
    case COMPATIBILITY;
}
```

---

# 29. STRICT

Solo representaciones válidas explícitas.

---

# 30. NORMALIZING

Permite variantes equivalentes.

Ejemplo:

```text
UUID uppercase
→ lowercase
```

---

# 31. COMPATIBILITY

Puede aceptar formatos legacy explícitamente definidos.

No debe convertirse en "aceptar cualquier cosa".

---

# 32. Null handling

Regla:

```text
NULL
```

debe manejarse antes de conversiones específicas cuando corresponda.

---

# 33. Nullable value

Si:

```text
TypeReference.nullability = NULLABLE
```

entonces:

```text
PHP null
↔
DB NULL
```

normalmente debe pasar sin converter específico.

---

# 34. Non-nullable value

Si:

```text
null
```

se intenta convertir para un tipo `NOT_NULL`:

```text
NullConversionException
```

---

# 35. Null converter anti-pattern

Evitar que cada converter implemente manualmente:

```php
if ($value === null) {
    return null;
}
```

si la infraestructura puede centralizarlo.

---

# 36. Null policy

Puede existir:

```php
enum NullConversionPolicy
{
    case RESPECT_TYPE_NULLABILITY;
    case ALLOW_RUNTIME_NULL;
}
```

Default ORM:

```text
RESPECT_TYPE_NULLABILITY
```

---

# 37. Scalar conversion categories

VoltStack necesitará conversión para:

```text
boolean
integer
decimal
floating point
string
binary
identifier
JSON
temporal
enum
value object
custom
```

---

# 38. Boolean PHP → DB

Entrada canónica:

```text
true
false
```

La Platform podrá decidir representación física.

---

# 39. Boolean DB → PHP

Driver puede devolver:

```text
true
false
1
0
"1"
"0"
```

según plataforma/driver.

La normalización deberá estar controlada.

---

# 40. Boolean strictness

Valores:

```text
2
"yes"
"false"
```

no deberán convertirse automáticamente salvo policy explícita.

---

# 41. Boolean converter

```php
final class BooleanValueConverter
    implements TypeValueConverter
{
    // ...
}
```

deberá validar representación compatible con el Platform Context.

---

# 42. Integer PHP → DB

Debe aceptar valores cuya semántica sea integer y que estén dentro del rango lógico.

---

# 43. Integer parsing

Strings numéricos pueden permitirse solo si:

```text
mapping/policy
```

lo autoriza.

---

# 44. Leading zeros

```text
"00123"
```

puede ser integer 123 o identificador textual.

No inferir sin TypeReference.

---

# 45. Integer range

Cada tipo puede definir:

```text
min
max
signedness
```

---

# 46. Overflow

Si:

```text
value > max
```

deberá lanzar:

```text
IntegerOverflowException
```

---

# 47. Big integers

Un DB `BIGINT` puede exceder `PHP_INT_MAX`.

Por ello, `DATABASE_TO_PHP` puede producir:

```text
string
```

o:

```text
BigInteger value object
```

según mapping.

---

# 48. BigInteger strategy

```php
enum BigIntegerPhpStrategy
{
    case NATIVE_INT_WHEN_SAFE;
    case STRING;
    case VALUE_OBJECT;
}
```

---

# 49. No architecture-dependent corruption

El comportamiento no debe cambiar silenciosamente entre:

```text
64-bit PHP
32-bit PHP
```

sin que metadata/policy pueda detectarlo.

---

# 50. Decimal conversion

`DECIMAL` deberá preservar precisión exacta.

---

# 51. Decimal PHP representation

Recomendado:

```text
canonical decimal string
```

o:

```text
Decimal value object
```

---

# 52. Decimal input normalization

Ejemplo:

```text
"00100.5000"
```

puede normalizarse a:

```text
"100.5000"
```

si la scale semántica debe preservarse.

---

# 53. Decimal scale

Para:

```text
decimal(19,4)
```

valor:

```text
"100.5"
```

podrá normalizarse a:

```text
"100.5000"
```

según policy.

---

# 54. Rounding

Si un valor tiene más scale:

```text
"100.12345"
```

para:

```text
scale = 4
```

no debe redondearse silenciosamente.

---

# 55. Decimal rounding policy

```php
enum DecimalRoundingPolicy
{
    case FORBID;
    case HALF_UP;
    case HALF_DOWN;
    case HALF_EVEN;
    case FLOOR;
    case CEIL;
}
```

Default Database Type System:

```text
FORBID
```

salvo declaración explícita.

---

# 56. Decimal precision overflow

Debe comprobarse antes de binding cuando sea posible.

---

# 57. Float conversion

Los valores floating point deberán reconocer:

- `NaN`;
- `INF`;
- `-INF`.

---

# 58. Platform capability

No toda DB/platform representa estos valores igual.

La conversión deberá consultar capabilities.

---

# 59. Non-finite floats

Default recomendado:

```text
reject if platform/type semantics do not explicitly support
```

---

# 60. Float ≠ Decimal

Nunca convertir `DECIMAL` a `float` por comodidad.

---

# 61. String conversion

Entrada normalmente:

```text
string
```

---

# 62. Stringable

Objetos `Stringable` no deberán aceptarse universalmente.

Pueden aceptarse mediante mapping específico.

---

# 63. String length

La conversión puede detectar límites lógicos conocidos.

Pero:

```text
character length
```

y:

```text
byte length
```

no son siempre equivalentes.

---

# 64. String truncation

Prohibido por defecto.

Nunca:

```text
VARCHAR(10)
"abcdefghijkl"
→ "abcdefghij"
```

silenciosamente.

---

# 65. Encoding

El converter lógico no deberá asumir automáticamente:

```text
UTF-8
```

para todo driver/platform, aunque VoltStack pueda adoptar UTF-8 como default framework.

La representación física pertenece a Platform/Schema.

---

# 66. Unicode normalization

NFC/NFD normalization no deberá aplicarse silenciosamente salvo tipo/policy explícita.

---

# 67. Binary conversion

Datos binarios deberán mantenerse separados de texto.

---

# 68. Binary string

PHP puede usar:

```text
string
```

como bytes.

El TypeReference determina que su semántica es binaria.

---

# 69. Binary encoding

Nunca hacer automáticamente:

```text
base64_encode()
```

solo porque un driver espera texto.

Eso requiere una estrategia declarada.

---

# 70. Blob conversion

Valores grandes podrán usar:

```text
stream/resource abstraction
```

---

# 71. LOB streaming

Pipeline:

```text
Stream
   ↓
LOB converter
   ↓
Driver LOB representation
```

sin materializar todo en memoria cuando el driver lo permita.

---

# 72. Stream ownership

Debe definirse quién:

- abre;
- lee;
- cierra;

el stream.

---

# 73. Recommended ownership

El Value Conversion System no deberá cerrar arbitrariamente un stream externo salvo contrato explícito.

---

# 74. UUID conversion

Debe validar estructura UUID.

---

# 75. UUID canonical form

Podrá normalizar:

```text
550E8400-E29B-41D4-A716-446655440000
```

a:

```text
550e8400-e29b-41d4-a716-446655440000
```

sin pérdida semántica.

---

# 76. UUID object

Un Value Object UUID podrá convertirse a:

```text
canonical string
```

o:

```text
16-byte binary
```

según storage strategy.

---

# 77. Binary UUID

Conversión:

```text
Uuid object
→ 16 bytes
```

debe estar encapsulada en estrategia específica.

---

# 78. UUID DB → PHP

La infraestructura deberá saber si el raw value es:

```text
native driver UUID representation
string
binary
```

mediante contexto/platform mapping.

---

# 79. ULID conversion

Mismas reglas:

- validación;
- canonicalización;
- storage strategy;
- no parsing ambiguo.

---

# 80. JSON conversion

El sistema completo se detallará en:

```text
161_DATABASE_JSON_TYPE_SYSTEM.md
```

Pero Value Conversion deberá proporcionar la frontera.

---

# 81. JSON PHP → DB

Entrada puede ser:

```text
array
object
JsonSerializable
explicit JsonValue
```

según mapping.

---

# 82. JSON encoding failures

Deben producir:

```text
JsonValueConversionException
```

No retornar `false` o string vacío.

---

# 83. JSON invalid numbers

Debe gobernarse qué ocurre con:

```text
NaN
INF
```

---

# 84. JSON DB → PHP

Puede producir:

```text
array
stdClass
JsonValue
string
```

dependiendo del mapping.

No asumir siempre array.

---

# 85. JSON duplicate keys

Si la representación o decoder pierde información relevante, la política deberá estar documentada.

---

# 86. Temporal conversion

Se detallará en:

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
```

pero la infraestructura general debe preservar:

- precision;
- timezone semantics;
- offset;
- instant/local distinction.

---

# 87. Date conversion

```text
LocalDate
↔
YYYY-MM-DD
```

---

# 88. Time conversion

Debe considerar:

```text
fractional seconds
```

si el TypeReference las declara.

---

# 89. DateTime conversion

Debe distinguir:

```text
LocalDateTime
```

de:

```text
Instant
```

---

# 90. Timezone hazard

Nunca:

```text
2026-09-07 10:00
```

convertir a UTC asumiendo timezone del servidor sin policy explícita.

---

# 91. Precision truncation

Si PHP contiene microsegundos:

```text
.123456
```

pero DB soporta:

```text
.123
```

eso es conversión lossy.

---

# 92. Temporal loss policy

Debe aplicarse la misma infraestructura:

```text
LOSSLESS
LOSSY
```

y `LossyConversionPolicy`.

---

# 93. Enum integration

El documento:

```text
159_DATABASE_ENUM_MAPPING_SYSTEM.md
```

definirá la semántica completa.

---

# 94. Enum conversion

Deberá transformar:

```text
PHP Enum Case
→ backing database value
```

y:

```text
database backing value
→ PHP Enum Case
```

---

# 95. Unknown enum value

Nunca convertir silenciosamente a:

```text
null
```

si la columna no es null.

Debe generar:

```text
UnknownEnumBackingValueException
```

o una policy explícita de legacy handling.

---

# 96. Value Object integration

El documento:

```text
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md
```

definirá mappings completos.

---

# 97. Single-column value objects

Ejemplo:

```text
EmailAddress
→ string
```

puede utilizar un converter compuesto.

---

# 98. Multi-column value objects

No deberán forzarse dentro de un `TypeValueConverter` escalar único.

Ejemplo:

```text
Money
├── amount
└── currency
```

pertenece al Value Object Mapping System.

---

# 99. Converter composition

Debe poder construirse:

```text
EmailAddress
    ↓ domain converter
string
    ↓ database converter
VARCHAR representation
```

---

# 100. Converter chain

```php
interface ValueConverterChain
{
    public function convert(
        mixed $value,
        ConversionPath $path,
        ValueConversionContext $context,
    ): mixed;
}
```

---

# 101. ConversionPath

```php
final readonly class ConversionPath
{
    /**
     * @param list<ConversionStep> $steps
     */
    public function __construct(
        public array $steps,
    ) {}
}
```

---

# 102. Conversion step

Ejemplo:

```text
Money
→ DecimalValue
→ canonical decimal string
→ driver representation
```

---

# 103. Composition constraint

La cadena deberá ser:

```text
deterministic
finite
validated
```

---

# 104. No recursive converter discovery

No deberá descubrir converters en medio de la conversión mediante reflection.

---

# 105. Converter graph

Durante bootstrap podrán validarse conversion paths.

---

# 106. Circular conversion path

Debe rechazarse:

```text
A → B
B → A
```

si no existe un punto terminal.

---

# 107. Conversion pipeline compiler

Puede existir:

```php
interface ConversionPipelineCompiler
{
    public function compile(
        TypeReference $type,
        DatabasePlatform $platform,
        ValueConversionDirection $direction,
    ): CompiledConversionPipeline;
}
```

---

# 108. CompiledConversionPipeline

```php
final readonly class CompiledConversionPipeline
{
    public function __construct(
        public TypeReference $type,
        public ValueConversionDirection $direction,
        public array $steps,
        public ConversionPipelineFingerprint $fingerprint,
    ) {}
}
```

---

# 109. Hot-path objective

Una vez compilado:

```text
TypeReference + Platform + Direction
→ CompiledConversionPipeline
```

sin reconstruir reglas.

---

# 110. Pipeline cache

Podrá cachearse usando:

```text
TypeReferenceFingerprint
+
PlatformCapabilityFingerprint
+
Direction
+
TypeRegistryGeneration
+
ConversionPolicyGeneration
```

---

# 111. Cache stores behavior, not values

Nunca:

```text
input value → converted output
```

en cache global.

---

# 112. Why

Valores pueden ser:

- sensibles;
- enormes;
- mutables;
- request-specific.

---

# 113. Converter statelessness

Los converters core deberían ser:

```text
stateless
```

o depender únicamente de contextos explícitos.

---

# 114. Pure conversion ideal

Idealmente:

```text
output = f(value, TypeReference, Context)
```

sin efectos secundarios.

---

# 115. No database queries

Converters no deberán:

- consultar DB;
- resolver entities;
- abrir connection;
- disparar lazy loads.

---

# 116. No EntityManager

Nunca:

```php
$converter->setEntityManager(...);
```

para tipos escalares.

---

# 117. No global locale dependence

Conversión decimal no deberá depender de:

```text
current locale
```

---

# 118. Example locale hazard

```text
"1,25"
```

puede significar:

```text
1.25
```

en ciertos locales.

Database conversion no debe adivinarlo.

---

# 119. Canonical numeric representation

Preferir:

```text
"." decimal separator
```

para representación lógica.

---

# 120. Display formatting ≠ DB conversion

Moneda:

```text
"$1,234.56"
```

es presentation formatting.

No database conversion.

---

# 121. Custom converters

Extensiones podrán registrar converters.

---

# 122. Converter registration

Deberá ocurrir durante bootstrap mediante:

```text
TypeBehavior
```

o un registry especializado compilado.

---

# 123. Custom converter constraints

Deberá declarar:

- input PHP types;
- output logical types;
- direction;
- loss profile;
- platform dependencies;
- determinism;
- context requirements.

---

# 124. Arbitrary closure rejection

No almacenar closures arbitrarias en metadata cache.

---

# 125. Service-based converter

Preferir:

```text
ConverterServiceId
```

resuelto por container compilado.

---

# 126. Converter security

No cargar una clase convertidora desde:

- column value;
- user input;
- DB discriminator;
- JSON field.

---

# 127. Converter allowlist

Solo converters registrados durante bootstrap.

---

# 128. Platform conversion layer

Debe distinguirse:

```text
Logical Converter
```

de:

```text
Platform Representation Converter
```

---

# 129. Ejemplo UUID

```text
Uuid object
    ↓ logical converter
canonical UUID bytes/string
    ↓ platform converter
PostgreSQL native UUID representation
```

---

# 130. Ejemplo boolean

```text
bool
    ↓ logical value
true
    ↓ platform representation
1
```

en una plataforma que lo requiera.

---

# 131. Platform converter contract

```php
interface PlatformValueConverter
{
    public function toPlatform(
        mixed $logicalValue,
        TypeReference $type,
        DatabasePlatform $platform,
    ): mixed;

    public function fromPlatform(
        mixed $value,
        TypeReference $type,
        DatabasePlatform $platform,
    ): mixed;
}
```

---

# 132. Layering

```text
PHP Converter
    ↓
Logical DB Value
    ↓
Platform Converter
    ↓
Driver Value
```

No mezclar todas las responsabilidades en cada type class.

---

# 133. Driver raw representation

Los drivers pueden devolver valores diferentes.

Ejemplo:

```text
COUNT()
```

puede llegar como:

```text
int
string
```

según driver.

---

# 134. Driver normalization

Deberá existir una fase clara:

```text
Driver Raw
→ Platform Logical Raw
```

antes de conversiones de dominio cuando sea necesario.

---

# 135. Driver type metadata

Si el driver proporciona metadata confiable, podrá utilizarse.

No asumir que siempre está disponible.

---

# 136. Source type evidence

El conversion context puede incluir:

```php
final readonly class RawValueDescriptor
{
    public function __construct(
        public string $runtimeType,
        public ?string $driverType = null,
        public ?string $physicalType = null,
    ) {}
}
```

para diagnostics.

---

# 137. UNKNOWN source evidence

La falta de metadata no deberá producir certezas falsas.

---

# 138. Conversion validation phases

Pipeline recomendado:

```text
Input Value
    ↓
Null Handling
    ↓
Runtime Type Validation
    ↓
Semantic Parsing
    ↓
Range/Precision Validation
    ↓
Normalization
    ↓
Loss Analysis
    ↓
Platform Conversion
    ↓
Binding-ready Value
```

---

# 139. Parsing ≠ coercion

Parsing deberá requerir formatos válidos.

---

# 140. Canonicalization

Puede ser parte de la conversión.

Ejemplo:

```text
UUID uppercase
→ lowercase canonical representation
```

---

# 141. Normalization transparency

Si la normalización cambia representación pero no semántica:

```text
ConversionSafety::NORMALIZED
```

---

# 142. Conversion diagnostics

Un error deberá responder:

```text
What type?
What direction?
What stage?
Why failed?
Was it overflow?
Was it precision?
Was it invalid format?
Was it unsupported platform representation?
```

---

# 143. Sensitive values

No incluir el valor completo por defecto en exception messages.

---

# 144. Redacted preview

Puede existir:

```text
Value descriptor:
    type=string
    length=250
```

en lugar de:

```text
actual secret string
```

---

# 145. Safe scalar preview

Solo en debugging explícito podrían mostrarse valores no sensibles bajo policy.

---

# 146. Value sensitivity

El conversion context puede recibir:

```text
SensitiveValuePolicy
```

de metadata si existe.

---

# 147. Conversion exception hierarchy

```text
DatabaseValueConversionException
├── NullConversionException
├── InvalidSourceValueTypeException
├── InvalidValueFormatException
├── ValueOverflowException
├── IntegerOverflowException
├── DecimalPrecisionException
├── DecimalScaleException
├── LossyValueConversionException
├── UnsupportedValueConversionException
├── PlatformValueConversionException
├── DriverValueNormalizationException
├── BooleanConversionException
├── StringConversionException
├── BinaryConversionException
├── LobConversionException
├── IdentifierConversionException
├── UuidConversionException
├── UlidConversionException
├── JsonValueConversionException
├── TemporalValueConversionException
├── EnumValueConversionException
├── ValueObjectConversionException
├── ConversionPipelineException
├── ConversionCycleException
├── ConversionContextException
└── ValueConversionInvariantViolationException
```

---

# 148. ConversionStage

```php
enum ConversionStage
{
    case NULL_HANDLING;
    case INPUT_VALIDATION;
    case PARSING;
    case NORMALIZATION;
    case RANGE_VALIDATION;
    case PRECISION_VALIDATION;
    case LOGICAL_CONVERSION;
    case PLATFORM_CONVERSION;
    case DRIVER_NORMALIZATION;
}
```

---

# 149. Exception context

```php
final readonly class ValueConversionFailureContext
{
    public function __construct(
        public TypeReference $type,
        public ValueConversionDirection $direction,
        public ConversionStage $stage,
        public string $sourceRuntimeType,
        public ?string $platform,
    ) {}
}
```

---

# 150. Determinism

Con mismas:

```text
value
TypeReference
PlatformCapabilities
ConversionPolicy
```

el resultado deberá ser determinista.

---

# 151. Environment independence

No deberá depender silenciosamente de:

- locale del proceso;
- timezone global;
- `ini` variable no declarada;
- current request language.

---

# 152. Timezone dependency

Si un tipo necesita timezone:

```text
timezone
```

debe formar parte del `TypeReference` o conversion context explícito.

---

# 153. Precision dependency

Igual para:

```text
precision
scale
```

---

# 154. Platform capabilities

Converters deberán consultar abstractions cuando una decisión dependa del DBMS.

Ejemplo:

```text
supportsFractionalSeconds(6)
```

---

# 155. No vendor switches in core converter

Evitar:

```php
if ($platform === 'mysql') {
}
```

en converters generales.

---

# 156. Platform-specific converter

El adapter específico sí puede contener conocimiento del vendor.

---

# 157. MySQL/MariaDB separation

No asumir representaciones idénticas.

---

# 158. SQLite

Conversion deberá considerar su typing flexible sin degradar el logical type system.

---

# 159. Conversion support status

```php
enum ConversionSupportLevel
{
    case NATIVE;
    case SUPPORTED;
    case EMULATED;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 160. UNKNOWN

Nunca tratar como soportado automáticamente.

---

# 161. Emulation

Si una representación requiere emulación:

```text
type semantics
+
conversion behavior
+
schema convention
```

deberán concordar.

---

# 162. Example bool emulation

Si boolean se almacena en integer:

```text
true → 1
false → 0
```

debe existir una convención única y verificable.

---

# 163. Read normalization

Valores distintos:

```text
1
"1"
```

pueden normalizarse a `true` solo si el physical/platform mapping confirma que pertenecen a una representación boolean.

---

# 164. Avoid type guessing

Nunca:

```text
raw "1"
→ bool
```

sin TypeReference esperado.

---

# 165. Query parameter conversion

Cuando:

```php
User::query()
    ->where('active', true)
```

el semantic analyzer determina:

```text
User.active
→ bool TypeReference
```

---

# 166. Parameter conversion pipeline

```text
true
 ↓
bool converter
 ↓
platform representation
 ↓
binding descriptor
 ↓
driver
```

---

# 167. Parameter list conversion

Para:

```php
->whereIn('id', $ids)
```

cada valor deberá cumplir el mismo TypeReference esperado.

---

# 168. Collection conversion

La infraestructura podrá aplicar:

```text
convertEach(values, TypeReference)
```

con límites de recursos.

---

# 169. Batch conversion failure

Si un elemento falla:

```text
parameter index
```

deberá reportarse sin exponer todos los valores.

---

# 170. Composite IDs

Cada componente tendrá su propio TypeReference.

---

# 171. Composite conversion

```text
(country_id, customer_id)
```

no se convertirá como un único string improvisado.

---

# 172. Result-set conversion

HydrationPlan debería contener conversion instructions precompiladas.

---

# 173. Column conversion descriptor

```php
final readonly class ColumnValueConversion
{
    public function __construct(
        public ColumnAlias $column,
        public TypeReference $type,
        public CompiledConversionPipeline $pipeline,
    ) {}
}
```

---

# 174. Hydration hot path

Ideal:

```text
raw column
→ compiled pipeline
→ converted value
→ accessor assignment
```

---

# 175. No registry lookup per row when avoidable

HydrationPlan debe resolver converters previamente.

---

# 176. Scalar result conversion

También aplica a:

```text
COUNT
SUM
AVG
computed expressions
```

cuando Query Type System pueda determinar el tipo.

---

# 177. Aggregate type semantics

Ejemplo:

```text
COUNT()
```

puede tener logical `bigint`.

No asumir PHP int si puede exceder su rango.

---

# 178. SUM decimal

Debe preservar decimal semantics.

---

# 179. AVG

La semántica puede depender de input type y plataforma.

Query Type System deberá proporcionar TypeReference resultante.

---

# 180. Raw expression

Cuando tipo sea desconocido:

```text
UNKNOWN
```

deberá permanecer explícito.

---

# 181. Explicit raw type

API avanzada podrá permitir:

```php
raw(
    expression: '...',
    type: 'decimal',
)
```

para conversion segura.

---

# 182. Unknown result

No ejecutar conversion agresiva.

Puede devolver raw driver value o requerir tipo explícito según API.

---

# 183. Conversion and Schema defaults

Default values también necesitan conversión.

Ejemplo:

```php
$table->boolean('active')
    ->default(true);
```

---

# 184. Schema default pipeline

```text
PHP default
→ TypeReference validation
→ logical default
→ Platform schema literal/expression
```

---

# 185. Schema default ≠ runtime binding

Defaults del schema pueden requerir compilación literal distinta.

El Value Converter puede validar el valor, pero el Schema Compiler genera la representación SQL.

---

# 186. Migration conversion

Cambiar:

```text
string → int
```

puede requerir data conversion.

Eso no significa reutilizar ciegamente runtime value converter para millones de rows.

---

# 187. Migration data transformation

Pertenece a Migration/Data Transformation tooling.

Puede reutilizar reglas semánticas del Type System.

---

# 188. Bulk conversion

Import/export puede utilizar conversion pipelines en batch.

---

# 189. Vectorized conversion

Una futura optimización puede convertir batches de valores.

---

# 190. Semantic equivalence

Vectorized conversion deberá producir el mismo resultado que convertir individualmente.

---

# 191. Streaming conversion

Import/large datasets deberán poder:

```text
read row
→ convert
→ release
```

sin acumular todo en memoria.

---

# 192. Resource governance

Conversion podrá limitar:

- maximum string length for parsing;
- JSON depth;
- JSON bytes;
- binary size;
- decimal digit count;
- conversion chain depth.

---

# 193. Why

Valores hostiles podrían provocar:

```text
CPU exhaustion
memory exhaustion
recursive parsing
```

---

# 194. ConversionResourcePolicy

```php
final readonly class ConversionResourcePolicy
{
    public function __construct(
        public int $maxStringBytes,
        public int $maxBinaryBytes,
        public int $maxJsonDepth,
        public int $maxJsonBytes,
        public int $maxConversionChainDepth,
    ) {}
}
```

---

# 195. Resource limit failure

Debe ser explícito:

```text
ValueConversionResourceLimitException
```

---

# 196. Conversion recursion

Custom Value Objects pueden generar chains.

Debe limitarse profundidad.

---

# 197. Conversion stack

```text
Money
→ Decimal
→ String
```

deberá conservar un stack de diagnóstico bounded.

---

# 198. Cycle detection

```text
A → B → A
```

debe fallar.

---

# 199. Security

El sistema deberá proteger contra:

- object injection;
- arbitrary class instantiation;
- unsafe unserialize;
- dynamic converter loading;
- malformed JSON bombs;
- oversized payloads;
- pathologically large decimals.

---

# 200. No unserialize

Tipos core nunca deberán usar:

```php
unserialize($databaseValue);
```

como estrategia genérica.

---

# 201. PHP serialized object type

Si una extensión desea soportarlo, deberá ser explícitamente opt-in y sujeto a política de seguridad fuerte.

No será core recomendado.

---

# 202. JSON preferred

Para estructuras interoperables:

```text
JSON
```

es preferible a serialización de objetos PHP.

---

# 203. Value Object reconstruction

Debe utilizar factories/converters registrados.

No:

```php
new $classFromDatabase(...)
```

---

# 204. Mass assignment independence

Value Conversion no autoriza assignment.

Solo transforma valores.

---

# 205. Sensitive field handling

Password hashes, tokens u otros valores sensibles se convierten como tipos correspondientes, pero su redaction se controlará en diagnostics/telemetry.

---

# 206. Encryption separation

Encryption pertenece al Encryption System.

No al Value Conversion System.

---

# 207. Encrypted type future integration

Un custom type puede componer:

```text
Domain Value
→ serialization
→ encryption
→ binary/string database representation
```

pero la criptografía seguirá siendo responsabilidad del Encryption component.

---

# 208. Conversion events

Por defecto no emitir un framework event por cada conversión.

Sería demasiado costoso.

---

# 209. Telemetry

Métricas agregadas:

```text
database.value_conversion.total
database.value_conversion.failure
database.value_conversion.lossy_rejected
database.value_conversion.overflow
database.value_conversion.pipeline_cache_hit
database.value_conversion.platform_conversion_failure
```

---

# 210. Metric dimensions

Seguras:

```text
TypeId
direction
failure kind
platform
```

si el TypeId registry permanece acotado.

---

# 211. No value labels

Nunca:

```text
value="john@example.com"
```

en metrics.

---

# 212. Tracing

No crear un span por conversión normal.

---

# 213. Profiler

El Query Profiler podrá acumular:

```text
conversion time
conversion count
failure count
```

por query si resulta útil.

---

# 214. Slow converter diagnostics

Custom converter muy costoso puede detectarse en profiling.

---

# 215. Conversion budgets

Opcional:

```text
max conversion CPU per query
```

no es necesario en V1, pero architecture deberá permitir medición.

---

# 216. Persistent runtime

Compartible:

```text
CompiledConversionPipeline
Converter definitions
Stateless converter services
Conversion policies
```

---

# 217. Scoped state

Solo:

```text
current failure context
temporary conversion stack
resource counters
```

cuando sea necesario.

---

# 218. No cross-request conversion stack

Especialmente en FrankenPHP.

---

# 219. FrankenPHP

Modelo:

```text
Worker
├── immutable type registry
├── immutable conversion pipelines
├── stateless converters
│
├── Request A conversion context
└── Request B conversion context
```

---

# 220. RoadRunner

Misma regla.

---

# 221. OpenSwoole

Contextos temporales deberán ser coroutine-safe.

---

# 222. Converter thread/coroutine safety

Un converter compartido no deberá guardar:

```text
lastValue
lastError
currentPlatform
```

como estado mutable.

---

# 223. Development reload

Nuevos converters/type definitions generan nueva registry/conversion generation.

---

# 224. Pipeline generation

```php
final readonly class ConversionPipelineGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

Puede derivarse de:

```text
TypeRegistryGeneration
+
PlatformCapabilities
+
ConversionPolicyGeneration
```

---

# 225. Pipeline fingerprint

```text
H(
 TypeReference
 + direction
 + converter bindings
 + platform capabilities
 + policies
)
```

---

# 226. Deterministic compilation

Mismas entradas → mismo pipeline logical fingerprint.

---

# 227. Cache invalidation

Cambio en converter binding deberá invalidar pipelines.

---

# 228. Runtime policy changes

Por defecto, políticas core no deberían mutar durante request.

---

# 229. Conversion policy scope

Application configuration se resuelve durante bootstrap cuando sea posible.

---

# 230. Per-operation explicit policy

Una API avanzada puede permitir override explícito local.

Ejemplo:

```php
$converter->convert(
    value: $value,
    type: $type,
    policy: $strictPolicy,
);
```

sin modificar configuración global.

---

# 231. Compatibility mode

Útil para migrar sistemas legacy.

Ejemplo:

```text
"1"
→ bool true
```

solo bajo policy explícita.

---

# 232. Compatibility diagnostics

Debe poder indicar:

```text
legacy coercion applied
```

para facilitar migración hacia strict mode.

---

# 233. No compatibility magic

No ocultar qué coerciones están habilitadas.

---

# 234. Conversion contract example

```php
interface ValueConversionService
{
    public function toDatabase(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;

    public function toPhp(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;
}
```

---

# 235. Default service

```php
final class DefaultValueConversionService
    implements ValueConversionService
{
    public function __construct(
        private TypeBehaviorRegistry $behaviors,
        private ConversionPipelineResolver $pipelines,
    ) {}
}
```

---

# 236. ConversionPipelineResolver

```php
interface ConversionPipelineResolver
{
    public function resolve(
        TypeReference $type,
        DatabasePlatform $platform,
        ValueConversionDirection $direction,
        ConversionPolicy $policy,
    ): CompiledConversionPipeline;
}
```

---

# 237. Pipeline execution

```php
interface ConversionPipelineExecutor
{
    public function execute(
        CompiledConversionPipeline $pipeline,
        mixed $value,
        ValueConversionContext $context,
    ): mixed;
}
```

---

# 238. Separation compiler/executor

```text
PipelineCompiler
≠
PipelineExecutor
```

mismo patrón del Query Engine.

---

# 239. Why

Permite:

- cache;
- diagnostics;
- testing;
- persistent-runtime reuse;
- deterministic hot path.

---

# 240. Conversion steps

Tipos:

```php
enum ConversionStepKind
{
    case VALIDATE;
    case PARSE;
    case NORMALIZE;
    case RANGE_CHECK;
    case PRECISION_CHECK;
    case DOMAIN_TO_LOGICAL;
    case LOGICAL_TO_PLATFORM;
    case PLATFORM_TO_LOGICAL;
    case LOGICAL_TO_DOMAIN;
}
```

---

# 241. Compiled step

```php
final readonly class ConversionStep
{
    public function __construct(
        public ConversionStepKind $kind,
        public ConversionOperationReference $operation,
    ) {}
}
```

---

# 242. Operation references

No almacenar closures arbitrarias.

Preferir:

```text
registered service / stateless operation
```

---

# 243. Conversion explain

API conceptual:

```php
Database::types()
    ->conversion()
    ->explain(
        type: 'uuid',
        direction: ValueConversionDirection::PHP_TO_DATABASE,
    );
```

---

# 244. Explain output

```text
TYPE CONVERSION PLAN

Type:
    uuid

Direction:
    PHP_TO_DATABASE

PHP Input:
    Uuid|string

Steps:
    1. Validate UUID representation
    2. Canonicalize UUID
    3. Convert to platform representation
    4. Produce binding-ready value

Platform:
    PostgreSQL

Loss Profile:
    LOSSLESS
```

---

# 245. Decimal explain

```text
Type:
    decimal(19,4)

PHP representation:
    canonical decimal string

Steps:
    validate syntax
    normalize sign
    validate precision
    validate scale
    pad scale if configured
    produce DB decimal string

Float conversion:
    forbidden
```

---

# 246. Testing strategy

Debe existir test suite por converter.

---

# 247. Core test categories

```text
valid conversion
invalid runtime type
invalid format
boundary values
nullability
normalization
loss profile
platform support
round trip
resource limits
```

---

# 248. Round-trip tests

Para cada valor válido:

```text
toPhp(toDatabase(x)) ≡ x
```

cuando mapping es lossless.

---

# 249. Property-based testing

Especialmente para:

- integers;
- decimals;
- UUID;
- ULID;
- temporal values.

---

# 250. Integer boundaries

Probar:

```text
MIN
MIN+1
0
MAX-1
MAX
MIN-1
MAX+1
```

---

# 251. Decimal tests

Probar:

```text
precision boundary
scale boundary
negative values
zero
trailing zeros
leading zeros
excess scale
excess precision
```

---

# 252. Boolean tests

Probar raw driver representations por plataforma.

---

# 253. UUID tests

Probar:

- canonical;
- uppercase;
- invalid length;
- invalid characters;
- binary representation;
- object representation.

---

# 254. JSON tests

Probar:

- nested structures;
- Unicode;
- invalid JSON;
- depth limit;
- large payload;
- numeric edge cases.

---

# 255. Temporal tests

Probar:

- leap days;
- DST-sensitive values;
- offsets;
- fractional seconds;
- precision loss;
- platform range.

---

# 256. Custom converter conformance

Toda extensión debería pasar un contrato común.

---

# 257. ConverterConformanceSuite

Conceptualmente:

```php
abstract class ConverterConformanceSuite
{
    abstract protected function converter(): TypeValueConverter;

    abstract protected function type(): TypeReference;
}
```

---

# 258. Cross-platform tests

Tipos portables deberán probarse con:

- MySQL;
- MariaDB;
- PostgreSQL;
- SQLite.

---

# 259. Driver tests

Verificar que valores binding-ready realmente sobrevivan:

```text
bind
→ write
→ read
→ normalize
→ convert
```

---

# 260. Persistent-runtime tests

Request A no puede dejar conversion state que altere Request B.

---

# 261. Concurrency tests

Converters shared deben comportarse igual bajo acceso concurrente.

---

# 262. Fuzz testing

Útil para:

- numeric parsers;
- UUID parser;
- JSON converter;
- custom converter chains.

---

# 263. Performance testing

Medir:

```text
conversions/sec
pipeline resolution
pipeline cache hit
decimal parser
UUID binary conversion
JSON encode/decode
temporal conversion
```

---

# 264. Hot path optimization

Prioridades:

1. compiled pipelines;
2. prevalidated TypeReferences;
3. stateless converter reuse;
4. no reflection;
5. no service lookup por value cuando se pueda resolver antes.

---

# 265. Avoid polymorphic mega-converter

No crear:

```php
convert(mixed $value, string $type)
{
    switch ($type) {
       ...
    }
}
```

con cientos de cases.

---

# 266. Specialized converters

Preferir:

```text
BooleanConverter
IntegerConverter
DecimalConverter
UuidConverter
JsonConverter
TemporalConverter
```

resueltos por behavior.

---

# 267. Directory structure

```text
src/Quantum/Database/Type/
│
├── Conversion/
│   │
│   ├── Contract/
│   │   ├── ValueConversionService.php
│   │   ├── TypeValueConverter.php
│   │   ├── PlatformValueConverter.php
│   │   ├── ConversionPipelineResolver.php
│   │   ├── ConversionPipelineCompiler.php
│   │   └── ConversionPipelineExecutor.php
│   │
│   ├── Context/
│   │   ├── ValueConversionContext.php
│   │   ├── ColumnConversionContext.php
│   │   ├── RawValueDescriptor.php
│   │   └── ValueConversionFailureContext.php
│   │
│   ├── Direction/
│   │   └── ValueConversionDirection.php
│   │
│   ├── Policy/
│   │   ├── ConversionPolicy.php
│   │   ├── ConversionStrictness.php
│   │   ├── LossyConversionPolicy.php
│   │   ├── NullConversionPolicy.php
│   │   ├── DecimalRoundingPolicy.php
│   │   └── ConversionResourcePolicy.php
│   │
│   ├── Pipeline/
│   │   ├── CompiledConversionPipeline.php
│   │   ├── ConversionPipelineFingerprint.php
│   │   ├── ConversionPipelineGeneration.php
│   │   ├── ConversionPath.php
│   │   ├── ConversionStep.php
│   │   └── ConversionStepKind.php
│   │
│   ├── Converter/
│   │   ├── BooleanValueConverter.php
│   │   ├── IntegerValueConverter.php
│   │   ├── DecimalValueConverter.php
│   │   ├── FloatingPointValueConverter.php
│   │   ├── StringValueConverter.php
│   │   ├── BinaryValueConverter.php
│   │   ├── BlobValueConverter.php
│   │   ├── UuidValueConverter.php
│   │   ├── UlidValueConverter.php
│   │   ├── JsonValueConverter.php
│   │   ├── TemporalValueConverter.php
│   │   ├── EnumValueConverter.php
│   │   └── ValueObjectConverter.php
│   │
│   ├── Numeric/
│   │   ├── IntegerRangeValidator.php
│   │   ├── DecimalParser.php
│   │   ├── DecimalNormalizer.php
│   │   └── DecimalPrecisionValidator.php
│   │
│   ├── Identifier/
│   │   ├── UuidNormalizer.php
│   │   └── UlidNormalizer.php
│   │
│   ├── Platform/
│   │   ├── DefaultPlatformValueConverter.php
│   │   ├── MySQL/
│   │   ├── MariaDB/
│   │   ├── PostgreSQL/
│   │   └── SQLite/
│   │
│   ├── Result/
│   │   ├── ConversionResult.php
│   │   └── ConversionSafety.php
│   │
│   ├── Cache/
│   │   └── ConversionPipelineCache.php
│   │
│   ├── Diagnostics/
│   │   ├── ConversionExplainer.php
│   │   └── ConversionDiagnosticReport.php
│   │
│   └── Exception/
│       ├── DatabaseValueConversionException.php
│       ├── NullConversionException.php
│       ├── InvalidSourceValueTypeException.php
│       ├── InvalidValueFormatException.php
│       ├── ValueOverflowException.php
│       ├── IntegerOverflowException.php
│       ├── DecimalPrecisionException.php
│       ├── DecimalScaleException.php
│       ├── LossyValueConversionException.php
│       ├── UnsupportedValueConversionException.php
│       ├── PlatformValueConversionException.php
│       ├── DriverValueNormalizationException.php
│       ├── BooleanConversionException.php
│       ├── StringConversionException.php
│       ├── BinaryConversionException.php
│       ├── LobConversionException.php
│       ├── IdentifierConversionException.php
│       ├── UuidConversionException.php
│       ├── UlidConversionException.php
│       ├── JsonValueConversionException.php
│       ├── TemporalValueConversionException.php
│       ├── EnumValueConversionException.php
│       ├── ValueObjectConversionException.php
│       ├── ConversionPipelineException.php
│       ├── ConversionCycleException.php
│       ├── ValueConversionResourceLimitException.php
│       └── ValueConversionInvariantViolationException.php
```

---

# 268. Dependency rules

Permitido:

```text
Value Conversion
    ↓
Type Registry

Value Conversion
    ↓
Type Behavior Registry

Value Conversion
    ↓
Platform Capabilities

Value Conversion
    ↓
TypeReference
```

---

# 269. Prohibido

```text
Value Conversion
    ↓
EntityManager
```

```text
Value Conversion
    ↓
UnitOfWork
```

```text
Value Conversion
    ↓
Repository
```

```text
Value Conversion
    ↓
HTTP Request
```

```text
Value Conversion
    ↓
Current authenticated user
```

---

# 270. Interaction with Parameter Binding

```text
PHP Value
    ↓
Value Conversion
    ↓
Binding-Ready Value
    ↓
ParameterBindingSystem
    ↓
Driver
```

---

# 271. Interaction with Hydration

```text
Driver Raw Value
    ↓
Value Conversion
    ↓
PHP/Domain Value
    ↓
Hydrator
    ↓
Entity Field
```

---

# 272. Interaction with Query Type System

```text
Expression
    ↓
Expected TypeReference
    ↓
Parameter value
    ↓
Value Conversion
```

---

# 273. Interaction with Schema

```text
Default PHP value
    ↓
Value Conversion validation
    ↓
Logical default
    ↓
Schema Compiler
```

---

# 274. Interaction with Casting

```text
Database Value Conversion
        │
        ▼
Canonical PHP/Domain Value
        │
        ▼
Casting System
```

cuando una API desee una representación adicional.

---

# 275. Architectural invariants

## DB-VALUE-CONV-001

Value Conversion será independiente del ORM lifecycle.

## DB-VALUE-CONV-002

Value Conversion no ejecutará queries.

## DB-VALUE-CONV-003

Value Conversion no abrirá conexiones.

## DB-VALUE-CONV-004

Value Conversion no hará flush.

## DB-VALUE-CONV-005

Value Conversion no hará persist.

## DB-VALUE-CONV-006

Value Conversion no hará commit.

## DB-VALUE-CONV-007

Value Conversion será distinta de Parameter Binding.

## DB-VALUE-CONV-008

Value Conversion será distinta de Casting.

## DB-VALUE-CONV-009

Value Conversion será distinta de Hydration.

## DB-VALUE-CONV-010

Value Conversion será distinta de domain validation.

## DB-VALUE-CONV-011

Toda conversión conocerá un TypeReference cuando la semántica sea conocida.

## DB-VALUE-CONV-012

Toda conversión conocerá su dirección.

## DB-VALUE-CONV-013

Conversiones platform-sensitive tendrán contexto de Platform.

## DB-VALUE-CONV-014

Conversiones lossless preservarán semántica round-trip.

## DB-VALUE-CONV-015

Lossy conversion será explícita.

## DB-VALUE-CONV-016

Silent lossy conversion estará prohibida por defecto.

## DB-VALUE-CONV-017

PHP coercion insegura no será la base del sistema.

## DB-VALUE-CONV-018

`(int)`, `(float)` y `(bool)` no sustituirán validación semántica.

## DB-VALUE-CONV-019

NULL se gestionará centralmente cuando sea posible.

## DB-VALUE-CONV-020

NULL en tipo NOT_NULL fallará antes de binding.

## DB-VALUE-CONV-021

Converters no asumirán que `null` es siempre válido.

## DB-VALUE-CONV-022

Boolean conversion será type-aware.

## DB-VALUE-CONV-023

Valores arbitrarios no serán tratados como bool automáticamente.

## DB-VALUE-CONV-024

Integer range será validado.

## DB-VALUE-CONV-025

Integer overflow no truncará valores.

## DB-VALUE-CONV-026

Big integers no dependerán silenciosamente de PHP_INT_MAX.

## DB-VALUE-CONV-027

Decimal conversion preservará exactitud.

## DB-VALUE-CONV-028

Decimal no será convertido automáticamente a float.

## DB-VALUE-CONV-029

Excess decimal scale no se redondeará silenciosamente.

## DB-VALUE-CONV-030

Decimal rounding requerirá policy explícita.

## DB-VALUE-CONV-031

Decimal precision será validada.

## DB-VALUE-CONV-032

Non-finite floating values serán platform-aware.

## DB-VALUE-CONV-033

String conversion no truncará silenciosamente.

## DB-VALUE-CONV-034

Text encoding no será adivinado mediante locale.

## DB-VALUE-CONV-035

Unicode normalization no será implícita por defecto.

## DB-VALUE-CONV-036

Binary values serán semánticamente distintos de strings textuales.

## DB-VALUE-CONV-037

Binary conversion no aplicará base64 automáticamente.

## DB-VALUE-CONV-038

LOB conversion podrá soportar streams.

## DB-VALUE-CONV-039

Stream ownership será explícito.

## DB-VALUE-CONV-040

UUID conversion validará formato.

## DB-VALUE-CONV-041

UUID normalization podrá canonicalizar representación.

## DB-VALUE-CONV-042

UUID binary storage no alterará identidad lógica.

## DB-VALUE-CONV-043

ULID conversion será validada.

## DB-VALUE-CONV-044

JSON conversion fallará ante encode/decode inválido.

## DB-VALUE-CONV-045

JSON errors no producirán strings vacíos silenciosos.

## DB-VALUE-CONV-046

Temporal conversion preservará semántica temporal declarada.

## DB-VALUE-CONV-047

Timezone no será adivinada.

## DB-VALUE-CONV-048

Temporal precision loss será detectada.

## DB-VALUE-CONV-049

Enum conversion no convertirá unknown backing values a null silenciosamente.

## DB-VALUE-CONV-050

Value Object reconstruction utilizará converters registrados.

## DB-VALUE-CONV-051

Database values no seleccionarán clases arbitrarias.

## DB-VALUE-CONV-052

Multi-column Value Objects no serán forzados a scalar converter.

## DB-VALUE-CONV-053

Converter chains serán finitas.

## DB-VALUE-CONV-054

Converter cycles serán rechazados.

## DB-VALUE-CONV-055

Converter chains podrán compilarse.

## DB-VALUE-CONV-056

Compiled conversion pipelines serán immutable.

## DB-VALUE-CONV-057

Compiled pipelines podrán cachearse.

## DB-VALUE-CONV-058

Pipeline cache no almacenará user values.

## DB-VALUE-CONV-059

Pipeline cache incluirá TypeRegistry generation.

## DB-VALUE-CONV-060

Platform-dependent pipeline cache incluirá platform capabilities.

## DB-VALUE-CONV-061

Policy-dependent pipeline cache incluirá policy generation.

## DB-VALUE-CONV-062

Converters core serán stateless cuando sea posible.

## DB-VALUE-CONV-063

Converters no capturarán request state.

## DB-VALUE-CONV-064

Converters no dependerán de EntityManager.

## DB-VALUE-CONV-065

Converters no dependerán de UnitOfWork.

## DB-VALUE-CONV-066

Converters no dependerán de Repository.

## DB-VALUE-CONV-067

Converters no utilizarán global locale implícito.

## DB-VALUE-CONV-068

Converters temporales no usarán timezone global implícita.

## DB-VALUE-CONV-069

Custom converters se registrarán en bootstrap.

## DB-VALUE-CONV-070

Custom converters no se descubrirán por reflection en hot path.

## DB-VALUE-CONV-071

Custom converters no se cargarán desde DB input.

## DB-VALUE-CONV-072

Custom converter metadata no almacenará arbitrary closures.

## DB-VALUE-CONV-073

Logical conversion y Platform conversion serán capas separables.

## DB-VALUE-CONV-074

Platform conversion no cambiará logical TypeId.

## DB-VALUE-CONV-075

Driver normalization será una frontera explícita.

## DB-VALUE-CONV-076

Driver raw runtime type no determinará solo el TypeReference.

## DB-VALUE-CONV-077

Expected TypeReference tendrá prioridad sobre guessing.

## DB-VALUE-CONV-078

UNKNOWN type no será convertido agresivamente.

## DB-VALUE-CONV-079

Raw expression podrá requerir explicit type para typed conversion.

## DB-VALUE-CONV-080

Schema defaults podrán usar conversion validation.

## DB-VALUE-CONV-081

Schema literal compilation seguirá separada.

## DB-VALUE-CONV-082

Migration transformations no serán equivalentes a runtime conversion automática.

## DB-VALUE-CONV-083

Bulk conversion preservará la misma semántica que scalar conversion.

## DB-VALUE-CONV-084

Streaming conversion estará permitida.

## DB-VALUE-CONV-085

Resource governance limitará entradas patológicas.

## DB-VALUE-CONV-086

JSON depth estará bounded.

## DB-VALUE-CONV-087

Conversion chain depth estará bounded.

## DB-VALUE-CONV-088

Oversized values podrán fallar antes de consumo excesivo.

## DB-VALUE-CONV-089

Core conversion no utilizará unserialize() genérico.

## DB-VALUE-CONV-090

Arbitrary object deserialization no será core.

## DB-VALUE-CONV-091

Encryption estará separada del Value Conversion core.

## DB-VALUE-CONV-092

Conversion no será authorization.

## DB-VALUE-CONV-093

Conversion no será mass assignment protection.

## DB-VALUE-CONV-094

Sensitive values no se imprimirán en exceptions por defecto.

## DB-VALUE-CONV-095

Metrics no contendrán raw values.

## DB-VALUE-CONV-096

Cada conversion normal no creará un trace span.

## DB-VALUE-CONV-097

Conversion telemetry será agregada.

## DB-VALUE-CONV-098

Value Conversion funcionará sin Telemetry.

## DB-VALUE-CONV-099

Compiled pipelines podrán compartirse en FrankenPHP.

## DB-VALUE-CONV-100

Mutable conversion context será operation/request scoped.

## DB-VALUE-CONV-101

RoadRunner tendrá el mismo aislamiento.

## DB-VALUE-CONV-102

OpenSwoole requerirá context safety.

## DB-VALUE-CONV-103

Shared converters no almacenarán `lastValue`.

## DB-VALUE-CONV-104

Shared converters no almacenarán `lastError`.

## DB-VALUE-CONV-105

Shared converters no almacenarán current Platform mutable.

## DB-VALUE-CONV-106

Development reload generará nuevas conversion generations.

## DB-VALUE-CONV-107

Mid-operation pipeline mutation estará prohibida.

## DB-VALUE-CONV-108

Pipeline compilation será determinista.

## DB-VALUE-CONV-109

Equivalent inputs producirán equivalent pipeline fingerprints.

## DB-VALUE-CONV-110

Parameter list conversion validará cada elemento.

## DB-VALUE-CONV-111

Batch parameter error identificará posición sin revelar toda la colección.

## DB-VALUE-CONV-112

Composite IDs convertirán cada componente por su TypeReference.

## DB-VALUE-CONV-113

HydrationPlan podrá incorporar compiled conversion pipelines.

## DB-VALUE-CONV-114

Hydration no hará repeated converter discovery por row.

## DB-VALUE-CONV-115

Scalar query results serán typed cuando Query Type System conozca el tipo.

## DB-VALUE-CONV-116

COUNT no se asumirá siempre safe PHP int.

## DB-VALUE-CONV-117

SUM decimal preservará decimal semantics.

## DB-VALUE-CONV-118

Aggregate types serán definidos por Query Type System.

## DB-VALUE-CONV-119

Type Conversion no inferirá semántica ausente de una raw expression.

## DB-VALUE-CONV-120

Runtime compatibility coercions serán explícitas.

## DB-VALUE-CONV-121

Compatibility coercions podrán emitir diagnostics.

## DB-VALUE-CONV-122

Strict mode será el baseline recomendado.

## DB-VALUE-CONV-123

Round-trip tests serán requeridos para tipos lossless.

## DB-VALUE-CONV-124

Cross-platform converters tendrán conformance tests.

## DB-VALUE-CONV-125

Driver binding round-trips serán probados.

## DB-VALUE-CONV-126

Boundary values serán parte obligatoria del test suite.

## DB-VALUE-CONV-127

Property-based tests serán recomendados para tipos numéricos/identificadores.

## DB-VALUE-CONV-128

Fuzz testing será permitido para parsers.

## DB-VALUE-CONV-129

Conversion failure tendrá stage explícito.

## DB-VALUE-CONV-130

Conversion failure tendrá TypeReference explícito.

## DB-VALUE-CONV-131

Conversion failure tendrá direction explícita.

## DB-VALUE-CONV-132

Conversion failure tendrá platform context cuando aplique.

## DB-VALUE-CONV-133

Conversion errors no fabricarán valores de fallback.

## DB-VALUE-CONV-134

Invalid value no se convertirá a zero automáticamente.

## DB-VALUE-CONV-135

Invalid bool no se convertirá a false automáticamente.

## DB-VALUE-CONV-136

Invalid enum no se convertirá a first case automáticamente.

## DB-VALUE-CONV-137

Invalid JSON no se convertirá a empty array automáticamente.

## DB-VALUE-CONV-138

Invalid date no se convertirá a current date automáticamente.

## DB-VALUE-CONV-139

Invalid UUID no generará uno nuevo automáticamente.

## DB-VALUE-CONV-140

Correctness tendrá prioridad sobre convenience coercion.

## DB-VALUE-CONV-141

Conversion behavior será explainable.

## DB-VALUE-CONV-142

Explain no ejecutará conversiones destructivas.

## DB-VALUE-CONV-143

Explain no necesitará user values.

## DB-VALUE-CONV-144

ValueConversionService dependerá de contracts, no de concrete ORM services.

## DB-VALUE-CONV-145

Pipeline Compiler y Pipeline Executor permanecerán separados.

## DB-VALUE-CONV-146

Conversion pipeline no será SQL Compiler pipeline.

## DB-VALUE-CONV-147

Platform Value Converter no será Driver.

## DB-VALUE-CONV-148

Driver binding seguirá siendo responsabilidad separada.

## DB-VALUE-CONV-149

Logical value semantics permanecerán estables entre plataformas.

## DB-VALUE-CONV-150

VoltStack nunca modificará silenciosamente un valor cuando no pueda demostrar que la conversión preserva la semántica requerida.

---

# 276. Anti-patterns

## 276.1 PHP casts como sistema de conversión

```php
(int) $value;
(float) $value;
(bool) $value;
```

como estrategia general.

**Rechazado.**

---

## 276.2 Decimal → float

**Rechazado por defecto.**

---

## 276.3 Truncar strings

**Rechazado.**

---

## 276.4 Redondear decimal sin policy

**Rechazado.**

---

## 276.5 Asumir timezone del servidor

**Rechazado.**

---

## 276.6 Invalid JSON → []

**Rechazado.**

---

## 276.7 Invalid enum → null

**Rechazado.**

---

## 276.8 Invalid integer → 0

**Rechazado.**

---

## 276.9 Type guessing desde raw driver value

**Rechazado.**

---

## 276.10 Queries dentro de converter

**Rechazado.**

---

## 276.11 EntityManager dentro del converter

**Rechazado.**

---

## 276.12 Arbitrary class desde DB value

**Rechazado.**

---

## 276.13 `unserialize()` genérico

**Rechazado.**

---

## 276.14 Converter global mutable

**Rechazado.**

---

## 276.15 Cache global de valores convertidos

**Rechazado.**

---

# 277. Ejemplo boolean

Mapping:

```text
bool
```

PHP:

```php
true
```

Pipeline:

```text
true
 ↓
validate bool
 ↓
logical boolean true
 ↓
Platform representation
 ↓
driver-ready value
```

Read:

```text
driver value 1
 ↓
Platform confirms boolean representation
 ↓
normalize logical true
 ↓
PHP true
```

---

# 278. Ejemplo decimal

Mapping:

```text
decimal(19,4)
```

PHP value:

```text
"1250.5000"
```

Pipeline:

```text
validate syntax
 ↓
validate precision <= 19
 ↓
validate scale <= 4
 ↓
canonical decimal
 ↓
database representation
```

Valor:

```text
"1250.50001"
```

con `scale=4`:

```text
DecimalScaleException
```

salvo rounding policy explícita.

---

# 279. Ejemplo UUID

PHP:

```php
$uuid = Uuid::fromString(
    '550e8400-e29b-41d4-a716-446655440000'
);
```

TypeReference:

```text
uuid(storage=platform_native)
```

PostgreSQL:

```text
Uuid Object
 ↓
canonical UUID
 ↓
native UUID representation
```

MySQL con binary policy:

```text
Uuid Object
 ↓
canonical UUID
 ↓
16-byte binary
```

Mismo tipo lógico.

---

# 280. Ejemplo temporal

```php
$createdAt = new DateTimeImmutable(
    '2026-09-07T18:30:00.123456+00:00'
);
```

TypeReference:

```text
instant(precision=6)
```

Platform con precision 6:

```text
LOSSLESS
```

Platform con precision 3:

```text
LOSSY
```

Default:

```text
LossyValueConversionException
```

---

# 281. Ejemplo query parameter

```php
Product::query()
    ->where('price', '>=', '100.0000')
    ->get();
```

Pipeline:

```text
Query Type System
    ↓
Product.price = decimal(19,4)
    ↓
Expected parameter type
    ↓
ValueConversionService
    ↓
"100.0000"
    ↓
validated decimal
    ↓
Parameter Binding
```

---

# 282. Ejemplo hydration

Database result:

```text
price = "125.5000"
```

HydrationPlan:

```text
column price
→ decimal(19,4)
→ Decimal conversion pipeline
```

resultado PHP:

```text
"125.5000"
```

o futuro `DecimalValue`, según mapping.

---

# 283. Master formula

Para escritura:

```text
DBValue
=
PlatformConvert(
    LogicalConvert(
        PHPValue,
        TypeReference
    ),
    Platform
)
```

sujeto a:

```text
ValidInput
∧
WithinRange
∧
WithinPrecision
∧
LossPolicySatisfied
∧
PlatformSupported
```

---

# 284. Read formula

```text
PHPValue
=
DomainConvert(
    LogicalNormalize(
        PlatformNormalize(
            DriverValue
        )
    )
)
```

---

# 285. Round-trip formula

Para mapping lossless:

```text
Decode(
    Encode(x)
)
≡ x
```

---

# 286. Safe conversion formula

```text
SafeConversion
=
KnownSourceSemantics
∧
KnownTargetSemantics
∧
ValidRepresentation
∧
NoForbiddenInformationLoss
∧
WithinResourceLimits
```

---

# 287. Master architecture

```text
                    PHP / Domain Value
                           │
                           ▼
                    TypeReference
                           │
                           ▼
                Conversion Pipeline
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      Validation       Normalization      Loss Check
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                  Logical DB Value
                           │
                           ▼
                Platform Conversion
                           │
                           ▼
                  Binding-Ready Value
                           │
                           ▼
                     Driver / DB
```

Read reverses the direction.

---

# 288. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Explicit Type-Aware Conversion
```

en lugar de coerción PHP.

Adoptará:

```text
Compiled Conversion Pipelines
```

para hot paths.

Adoptará:

```text
Logical Conversion
+
Platform Conversion
```

como capas separables.

Adoptará:

```text
Loss Detection
```

en lugar de truncamiento silencioso.

Adoptará:

```text
Strict Default Conversion
```

con compatibility mode explícito.

Adoptará:

```text
Round-Trip Guarantees
```

para tipos lossless.

Adoptará:

```text
Stateless Shared Converters
```

compatibles con runtimes persistentes.

---

# 289. Regla maestra

> **Un valor que atraviese VoltStack Database nunca dependerá de coerciones accidentales de PHP, del formato incidental de un driver o de supuestos implícitos sobre el motor de base de datos. Su transformación estará determinada por un `TypeReference`, una dirección, una política y un contexto de plataforma explícitos.**

Esto mantiene:

```text
Domain Semantics
        =
Stable
```

aunque:

```text
Driver Representation
        =
Platform-dependent
```

y garantiza que:

```text
Convenience
<
Correctness
```

en toda conversión.

---

# 290. Siguiente documento

```text
158_DATABASE_CASTING_SYSTEM.md
```

El siguiente documento deberá definir la capa encargada de proyectar valores persistentes hacia representaciones adicionales de aplicación sin confundir casting con persistencia o conversión física.

Regla central propuesta:

> **Casting en VoltStack será una transformación declarativa de representación aplicada sobre valores cuyos tipos persistentes ya son conocidos; nunca sustituirá al Type System ni decidirá por sí mismo cómo se almacena físicamente un valor en la base de datos.**

Deberá cubrir:

- Cast definitions;
- cast registry;
- inbound casts;
- outbound casts;
- bidirectional casts;
- immutable casts;
- scalar casts;
- collection casts;
- enum casts;
- date casts;
- JSON casts;
- Value Object casts;
- encrypted cast integration;
- custom casts;
- cast arguments;
- cast pipelines;
- cast composition;
- ORM mapping integration;
- Model API;
- Data Mapper entities;
- dirty checking;
- snapshot semantics;
- hydration interaction;
- serialization interaction;
- null handling;
- casting failures;
- persistent runtime safety;
- caching;
- telemetry;
- diagnostics;
- testing;
- architectural invariants.