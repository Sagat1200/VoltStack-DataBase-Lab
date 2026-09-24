# 159_DATABASE_ENUM_MAPPING_SYSTEM.md

# VoltStack Quantum Database
## Database Enum Mapping System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 159 — Database Enum Mapping System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `158_DATABASE_CASTING_SYSTEM.md`  
**Siguiente documento:** `160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md`

---

# 1. Propósito

`Database Enum Mapping System` define cómo VoltStack representa, registra, convierte, consulta, hidrata, persiste y evoluciona dominios enumerados dentro del subsistema Database.

Debe soportar como mínimo:

- PHP backed enums;
- PHP pure enums;
- enums persistidos como `VARCHAR`;
- enums persistidos como enteros;
- tipos `ENUM` nativos cuando la plataforma los soporte;
- dominios equivalentes de PostgreSQL;
- aliases persistentes;
- valores legacy;
- evolución de casos;
- compatibilidad durante despliegues;
- schema y migrations;
- Query Engine;
- ORM;
- Hydration;
- Casting;
- UnitOfWork;
- Change Tracking;
- prepared statements;
- cache;
- telemetry;
- persistent runtimes.

La regla central será:

> **Un enum persistente en VoltStack representa un dominio lógico cerrado cuya identidad pertenece al modelo de aplicación; su backing value o representación física pertenece al mapping y nunca deberá confundirse con la identidad conceptual del caso.**

Por tanto:

```text
Enum Case Identity
        ≠
PHP Backing Value
        ≠
Database Physical Representation
```

aunque en determinados mappings puedan coincidir.

---

# 2. Problema arquitectónico

Un enum aparentemente sencillo:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';
}
```

introduce varias capas:

```text
Domain
    OrderStatus::Paid

PHP Representation
    enum object

Logical Persistent Value
    "paid"

Database Representation
    VARCHAR / ENUM / DOMAIN / INTEGER / ...
```

Si esas capas se mezclan, aparecen problemas al:

- renombrar casos PHP;
- cambiar backing values;
- migrar datos;
- cambiar de PostgreSQL a MySQL;
- ejecutar despliegues rolling;
- leer datos legacy;
- retirar valores antiguos.

VoltStack deberá separar esas identidades explícitamente.

---

# 3. Posición arquitectónica

```text
Entity Property
      │
      ▼
 Enum Mapping
      │
      ▼
Enum Logical Value
      │
      ▼
Casting / Type System
      │
      ▼
Canonical Persistent Value
      │
      ▼
Value Conversion
      │
      ▼
Query / Binding
      │
      ▼
Database
```

En lectura:

```text
Database
   ↓
Driver
   ↓
Value Conversion
   ↓
Canonical Persistent Value
   ↓
Enum Mapping
   ↓
PHP Enum Case
   ↓
Entity
```

---

# 4. Enum Mapping ≠ Enum Definition

VoltStack no define el enum de negocio.

El enum pertenece a la aplicación:

```php
enum PaymentStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
}
```

Database únicamente define:

```text
cómo se persiste
cómo se recupera
cómo evoluciona
```

---

# 5. Enum Mapping ≠ Casting

Casting ejecuta la transformación.

Enum Mapping describe su semántica.

```text
EnumMappingMetadata
        ↓
EnumCast
        ↓
Runtime conversion
```

---

# 6. Enum Mapping ≠ Schema ENUM

Muy importante:

```text
Application Enum
≠
Database ENUM
```

Un enum PHP puede almacenarse como:

```text
VARCHAR
INTEGER
SMALLINT
CHECK CONSTRAINT
MySQL ENUM
PostgreSQL ENUM TYPE
```

---

# 7. Enum Mapping ≠ Validation

Si una API permite únicamente:

```text
pending
paid
```

es una regla de validación/input.

Database Enum Mapping garantiza que:

```text
persistent value
↔
known enum semantic value
```

sea consistente.

---

# 8. Modelo conceptual

```text
EnumTypeMetadata
├── EnumTypeId
├── PHP Enum Class
├── Enum Cases
├── Persistence Strategy
├── Case Mappings
├── Unknown Value Policy
├── Compatibility Policy
└── Evolution Metadata
```

---

# 9. EnumTypeId

Cada enum persistente tendrá identidad lógica estable.

```php
final readonly class EnumTypeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
commerce.order_status
billing.payment_status
users.account_state
```

---

# 10. No usar FQCN como identidad persistente

Nunca asumir:

```text
App\Domain\OrderStatus
```

como identidad estable del tipo persistente.

El namespace PHP puede cambiar.

---

# 11. PHP class reference

La metadata sí podrá contener:

```text
PHPEnumClassReference
```

pero:

```text
EnumTypeId ≠ PHPClassName
```

---

# 12. Enum case identity

Cada caso deberá tener una identidad conceptual.

Ejemplo:

```text
EnumType:
    commerce.order_status

Case:
    PAID
```

Podemos modelarlo como:

```php
final readonly class EnumCaseId
{
    public function __construct(
        public EnumTypeId $enum,
        public string $case,
    ) {}
}
```

---

# 13. Case name ≠ backing value

PHP:

```php
case Paid = 'paid';
```

produce:

```text
Case Name:
    Paid

Backing Value:
    "paid"
```

No son la misma cosa.

---

# 14. Backing value ≠ DB value

También puede existir:

```text
OrderStatus::Paid
    ↓
PHP backing:
"paid"
    ↓
DB mapping:
2
```

VoltStack no deberá prohibirlo arquitectónicamente.

---

# 15. EnumMappingMetadata

```php
final readonly class EnumMappingMetadata
{
    /**
     * @param list<EnumCaseMapping> $cases
     */
    public function __construct(
        public EnumTypeId $id,
        public PHPEnumReference $phpEnum,
        public EnumPersistenceStrategy $strategy,
        public array $cases,
        public EnumUnknownValuePolicy $unknownPolicy,
        public EnumCompatibilityPolicy $compatibility,
    ) {}
}
```

---

# 16. EnumCaseMapping

```php
final readonly class EnumCaseMapping
{
    /**
     * @param list<EnumPersistentAlias> $legacyAliases
     */
    public function __construct(
        public EnumCaseId $case,
        public mixed $canonicalPersistentValue,
        public array $legacyAliases = [],
    ) {}
}
```

---

# 17. Canonical persistent value

Para cada caso existirá un valor canónico de escritura.

Ejemplo:

```text
OrderStatus::Paid
        ↓
canonical persistent value
        ↓
"paid"
```

---

# 18. Canonical-write principle

VoltStack deberá favorecer:

```text
multiple accepted read values
+
one canonical write value
```

durante migraciones.

Formalmente:

```text
ReadSet(Paid) = {"paid", "PAYED", "completed_payment"}

WriteValue(Paid) = "paid"
```

---

# 19. EnumPersistenceStrategy

```php
enum EnumPersistenceStrategy
{
    case BACKING_VALUE;
    case CASE_NAME;
    case EXPLICIT_STRING;
    case EXPLICIT_INTEGER;
    case NATIVE_DATABASE_ENUM;
    case CHECK_CONSTRAINED_VALUE;
    case CUSTOM;
}
```

---

# 20. BACKING_VALUE

Para:

```php
enum Status: string
{
    case Active = 'active';
}
```

se persiste:

```text
"active"
```

---

# 21. CASE_NAME

Puede persistir:

```text
Active
```

independientemente del backing value.

Debe ser explícito.

---

# 22. EXPLICIT_STRING

Ejemplo:

```text
Pending   → "P"
Paid      → "D"
Cancelled → "C"
```

---

# 23. EXPLICIT_INTEGER

Ejemplo:

```text
Pending   → 10
Paid      → 20
Cancelled → 30
```

---

# 24. Nunca ordinal implícito

Prohibido:

```text
first enum case  → 0
second enum case → 1
third enum case  → 2
```

como persistencia automática.

---

# 25. Razón

Reordenar:

```php
enum Status
{
    case Pending;
    case Paid;
}
```

podría corromper semánticamente datos existentes.

---

# 26. Pure enums

PHP pure enum:

```php
enum Priority
{
    case Low;
    case Medium;
    case High;
}
```

no tiene backing value.

VoltStack podrá mapearlo mediante:

```text
CASE_NAME
```

o mapping explícito.

---

# 27. Recomendación para pure enums

Preferir mapping explícito si el enum tiene expectativa de larga duración.

Ejemplo:

```text
Low    → "low"
Medium → "medium"
High   → "high"
```

---

# 28. String-backed enum

Será el caso recomendado para muchos dominios.

```php
enum UserStatus: string
{
    case Active = 'active';
    case Suspended = 'suspended';
}
```

Ventajas:

- legibilidad;
- debugging;
- estabilidad;
- migraciones comprensibles.

---

# 29. Integer-backed enum

Soportado:

```php
enum Priority: int
{
    case Low = 10;
    case Medium = 20;
    case High = 30;
}
```

---

# 30. No asumir continuidad

Valores:

```text
10
20
30
```

son válidos.

No asumir:

```text
0
1
2
```

---

# 31. Integer enum evolution

Nunca reutilizar un valor retirado para otro significado.

Ejemplo:

```text
20 = APPROVED
```

no deberá convertirse posteriormente en:

```text
20 = REJECTED
```

---

# 32. Enum Registry

VoltStack utilizará:

```text
EnumMappingRegistry
```

---

# 33. Registry contract

```php
interface EnumMappingRegistry
{
    public function get(EnumTypeId $id): EnumMappingMetadata;

    public function forClass(string $enumClass): EnumMappingMetadata;

    public function has(EnumTypeId $id): bool;
}
```

---

# 34. Registry lifecycle

```text
REGISTER
   ↓
NORMALIZE
   ↓
VALIDATE
   ↓
COMPILE
   ↓
FREEZE
```

---

# 35. Immutable registry

En producción:

```text
EnumMappingRegistry
```

será immutable.

---

# 36. Enum mapping discovery

Puede provenir de:

- PHP attributes;
- Model API;
- configuration;
- package registration;
- explicit metadata builders.

---

# 37. Attribute mapping

Ejemplo conceptual:

```php
#[DatabaseEnum(
    id: 'commerce.order_status',
    strategy: EnumPersistenceStrategy::BACKING_VALUE,
)]
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
}
```

---

# 38. Field mapping

```php
#[Column(type: 'string')]
#[EnumMapping(OrderStatus::class)]
private OrderStatus $status;
```

---

# 39. Model API

```php
protected function casts(): array
{
    return [
        'status' => OrderStatus::class,
    ];
}
```

---

# 40. Normalización

Las dos APIs convergen en:

```text
EnumMappingMetadata
+
FieldEnumMapping
```

---

# 41. FieldEnumMapping

```php
final readonly class FieldEnumMapping
{
    public function __construct(
        public PropertyPath $property,
        public EnumTypeId $enumType,
        public TypeReference $persistentType,
        public bool $nullable,
    ) {}
}
```

---

# 42. Type Registry integration

`EnumMappingSystem` se apoyará en:

```text
156_DATABASE_TYPE_REGISTRY_SYSTEM
```

para resolver el tipo persistente subyacente.

---

# 43. Value Conversion integration

Ejemplo:

```text
DB VARCHAR
   ↓
Value Conversion
   ↓
canonical string
   ↓
Enum Mapping
   ↓
OrderStatus::Paid
```

---

# 44. Casting integration

El `EnumCast` será un adapter runtime sobre metadata compilada.

```text
EnumMappingMetadata
       ↓
CompiledEnumCast
```

---

# 45. Enum mapper

```php
interface EnumValueMapper
{
    public function toEnum(
        EnumMappingMetadata $mapping,
        mixed $persistentValue,
    ): UnitEnum;

    public function toPersistent(
        EnumMappingMetadata $mapping,
        UnitEnum $case,
    ): mixed;
}
```

---

# 46. Mapping tables

La compilación deberá generar lookup tables eficientes:

```text
persistent value → enum case
enum case        → persistent value
```

---

# 47. No reflection en hot path

No:

```text
UnitEnum::cases()
```

por cada fila hidratada.

---

# 48. CompiledEnumMapping

```php
final readonly class CompiledEnumMapping
{
    public function __construct(
        public EnumMappingMetadata $metadata,
        public array $readMap,
        public array $writeMap,
        public EnumMappingFingerprint $fingerprint,
    ) {}
}
```

---

# 49. Read map

Ejemplo:

```text
"pending" → Pending
"paid"    → Paid
"PAYED"   → Paid
```

---

# 50. Write map

```text
Pending → "pending"
Paid    → "paid"
```

---

# 51. Legacy aliases

Permiten compatibilidad temporal.

```text
PAYED
   ↓
Paid
```

pero escritura:

```text
Paid
   ↓
paid
```

---

# 52. Legacy alias ≠ canonical value

Una lectura legacy no deberá provocar que nuevas escrituras continúen produciendo el valor antiguo.

---

# 53. Unknown values

Uno de los problemas más importantes.

DB contiene:

```text
"archived"
```

pero el enum actual no tiene ese caso.

---

# 54. Default policy

Por defecto:

```text
UNKNOWN VALUE
→ exception
```

---

# 55. Nunca null silencioso

Prohibido:

```text
"archived"
→ null
```

porque:

```text
unknown
≠
NULL
```

---

# 56. EnumUnknownValuePolicy

```php
enum EnumUnknownValuePolicy
{
    case FAIL;
    case MAP_TO_FALLBACK;
    case PRESERVE_UNKNOWN;
}
```

`FAIL` será el default.

---

# 57. MAP_TO_FALLBACK

Ejemplo:

```php
enum Status: string
{
    case Unknown = 'unknown';
    case Active = 'active';
}
```

Puede configurarse explícitamente:

```text
unrecognized DB value
→ Status::Unknown
```

---

# 58. Riesgo del fallback

Puede ocultar corrupción o despliegues incompatibles.

Por eso deberá ser opt-in.

---

# 59. PRESERVE_UNKNOWN

Puede existir para integración legacy.

Conceptualmente:

```text
KnownEnumValue
|
UnknownEnumValue
```

---

# 60. UnknownEnumValue

```php
final readonly class UnknownEnumValue
{
    public function __construct(
        public EnumTypeId $enum,
        public mixed $persistentValue,
    ) {}
}
```

---

# 61. Typed property restriction

Una propiedad declarada:

```php
private OrderStatus $status;
```

no puede recibir `UnknownEnumValue`.

Por tanto `PRESERVE_UNKNOWN` requerirá una representación compatible.

---

# 62. Recommended domain model

Para entidades estrictas:

```text
FAIL
```

es la política recomendada.

---

# 63. NULL

Si la columna es nullable:

```text
DB NULL
→ PHP null
```

---

# 64. NULL ≠ enum case

No convertir automáticamente:

```text
NULL
→ Status::Unknown
```

---

# 65. Missing field

Como en el Casting System:

```text
MISSING
≠
NULL
```

Un campo no hidratado no deberá mapearse a ningún enum.

---

# 66. Enum hydration

Pipeline:

```text
Result Row
   ↓
Value Conversion
   ↓
Canonical Persistent Value
   ↓
Compiled Enum Mapping
   ↓
PHP Enum Case
   ↓
Entity Hydrator
```

---

# 67. Enum object identity

PHP enum cases son singleton-like dentro del proceso.

Pero eso no significa que pertenezcan al `IdentityMap`.

---

# 68. IdentityMap separation

```text
Enum Case
≠
Entity
```

Nunca:

```text
IdentityMap[OrderStatus::Paid]
```

---

# 69. UnitOfWork integration

Snapshots deberán utilizar una representación estable.

Recomendado:

```text
canonical persistent enum value
```

---

# 70. Example snapshot

Entity:

```text
OrderStatus::Paid
```

Snapshot:

```text
"paid"
```

---

# 71. Dirty tracking

```text
canonical(current enum)
≠
canonical(snapshot)
```

determina cambio.

---

# 72. Same enum case

```text
Paid → Paid
```

no produce ChangeSet.

---

# 73. Alias normalization

Si DB tenía:

```text
"PAYED"
```

y se hidrata:

```text
Paid
```

la snapshot necesita una decisión.

---

# 74. Observed vs canonical value

VoltStack podrá conservar conceptualmente:

```text
ObservedPersistentValue = "PAYED"
CanonicalPersistentValue = "paid"
```

---

# 75. No accidental UPDATE

La simple hidratación de un alias legacy no deberá necesariamente provocar:

```sql
UPDATE ...
```

durante un flush sin cambios.

---

# 76. Canonicalization policy

Deberá existir una política explícita:

```php
enum EnumCanonicalizationPolicy
{
    case ON_EXPLICIT_CHANGE;
    case ON_FLUSH;
    case MIGRATION_ONLY;
}
```

---

# 77. Recommended default

```text
MIGRATION_ONLY
```

o `ON_EXPLICIT_CHANGE`, evitando write amplification inesperado.

---

# 78. Migration owns bulk canonicalization

Normalizar millones de valores legacy pertenece normalmente al Migration System.

---

# 79. Query Builder

El usuario podrá escribir:

```php
Order::query()
    ->where('status', OrderStatus::Paid);
```

---

# 80. Query semantics

El Query Builder no deberá convertir directamente el enum a SQL.

Pipeline:

```text
OrderStatus::Paid
       ↓
Query Parameter
       ↓
Semantic Resolution
       ↓
Enum Mapping
       ↓
Canonical Persistent Value
       ↓
Type System
       ↓
Compiler / Binding
```

---

# 81. Query AST

El AST puede preservar un typed parameter:

```text
EnumParameter(
    enumType = commerce.order_status,
    case = Paid
)
```

o normalizarlo posteriormente a un typed canonical value.

---

# 82. SQL Compiler

No deberá conocer PHP enums.

Idealmente recibe:

```text
TypedParameter(
    type = string,
    value = "paid"
)
```

---

# 83. Driver

Nunca deberá recibir:

```text
OrderStatus::Paid
```

como conocimiento semántico del ORM.

---

# 84. Enum predicates

Soportar:

```php
->where('status', Status::Active)
```

```php
->whereIn('status', [
    Status::Active,
    Status::Pending,
])
```

---

# 85. Mixed enum types

Esto deberá fallar:

```php
[
    OrderStatus::Paid,
    PaymentStatus::Paid,
]
```

si el campo espera únicamente `OrderStatus`.

---

# 86. Invalid enum parameter

Debe producir:

```text
EnumQueryTypeMismatchException
```

antes de ejecución cuando sea detectable.

---

# 87. Enum equality

Semánticamente:

```text
EnumTypeId + EnumCaseId
```

determina identidad lógica.

---

# 88. Same backing value across enums

```php
OrderStatus::Pending->value   === 'pending';
PaymentStatus::Pending->value === 'pending';
```

no significa:

```text
OrderStatus::Pending
==
PaymentStatus::Pending
```

---

# 89. Enum ordering

No asumir que declaration order tiene significado SQL.

---

# 90. Explicit semantic ordering

Si el dominio requiere:

```text
LOW < MEDIUM < HIGH
```

deberá declararse como metadata/policy separada.

---

# 91. ORDER BY

Ordenar por columna persistente puede producir:

```text
high
low
medium
```

alfabéticamente.

Eso no equivale a orden de negocio.

---

# 92. EnumOrderDefinition

Opcional:

```php
final readonly class EnumOrderDefinition
{
    public function __construct(
        public EnumTypeId $enum,
        public array $orderedCases,
    ) {}
}
```

---

# 93. Query planner

Podrá compilar orden semántico mediante:

```text
CASE expression
native enum order
explicit rank mapping
```

según capabilities.

---

# 94. Portability warning

Native DB enum ordering puede variar conceptualmente entre plataformas.

No depender de él para lógica portable.

---

# 95. Schema integration

Enum Mapping podrá producir recomendaciones para Schema System.

Pero:

```text
Enum Mapping
≠
Schema
```

---

# 96. Portable default

La opción más portable será:

```text
VARCHAR/INTEGER
+
CHECK constraint when appropriate
```

---

# 97. MySQL native ENUM

MySQL permite:

```sql
ENUM('pending', 'paid', 'cancelled')
```

VoltStack podrá soportarlo mediante capability.

---

# 98. MariaDB

Deberá tratarse como plataforma independiente.

No asumir comportamiento idéntico a MySQL.

---

# 99. PostgreSQL

Puede utilizar tipos:

```sql
CREATE TYPE order_status AS ENUM (...)
```

---

# 100. SQLite

Normalmente utilizará:

```text
TEXT
+
CHECK constraint
```

cuando se requiera enforcement.

---

# 101. Platform strategy

```text
Logical Enum Mapping
        ↓
Schema Capability Analysis
        ↓
Physical Representation
```

---

# 102. Native enum is optional

El uso de native enum deberá ser explícito o seleccionado por una policy claramente documentada.

---

# 103. Recommended portable strategy

Para VoltStack V1:

```text
Application Enum
→ string/int column
→ optional CHECK
```

como default portable.

---

# 104. Why not native by default

Native enums pueden complicar:

- portability;
- migrations;
- adding/removing values;
- rolling deploys;
- database switching;
- schema diff.

---

# 105. Native enum benefits

Aun así ofrecen:

- DB-level validation;
- semantic schema;
- potentially compact representation;
- tooling visibility.

Por eso deberán soportarse.

---

# 106. Schema enum descriptor

```php
final readonly class EnumSchemaDescriptor
{
    public function __construct(
        public EnumTypeId $enum,
        public EnumSchemaStrategy $strategy,
        public array $values,
    ) {}
}
```

---

# 107. Mapping ≠ physical constraint truth

Que el ORM conozca:

```text
pending
paid
cancelled
```

no significa que la DB tenga un CHECK real.

---

# 108. Schema introspection

Deberá poder detectar, cuando sea posible:

```text
native enum
check-constrained enum-like column
plain scalar column
```

---

# 109. Introspection uncertainty

Si no puede probar que un CHECK representa exactamente el enum:

```text
UNKNOWN / PARTIAL
```

no:

```text
ENUM CONFIRMED
```

---

# 110. Schema diff

Debe distinguir:

```text
add enum value
remove enum value
rename enum value
change representation
```

---

# 111. Add case

Ejemplo:

```text
Pending
Paid
Cancelled
```

se convierte en:

```text
Pending
Paid
Cancelled
Refunded
```

---

# 112. Application-first deployment hazard

Si nueva aplicación escribe:

```text
refunded
```

antes de que vieja aplicación pueda leerlo:

```text
old app
→ UnknownEnumValueException
```

---

# 113. Database-first hazard

Si DB constraint todavía no permite:

```text
refunded
```

la nueva aplicación tampoco podrá escribirlo.

---

# 114. Enum evolution is distributed compatibility

Por tanto:

> **Agregar un caso enum en producción no es únicamente un cambio de código; puede ser un cambio de protocolo entre versiones de aplicación y esquema.**

---

# 115. Compatibility window

VoltStack deberá modelar despliegues donde:

```text
App V1
+
App V2
+
Schema transition
```

coexisten temporalmente.

---

# 116. Add-value migration

Estrategia típica:

```text
1. Expand DB constraint/type
2. Deploy readers that understand new value
3. Enable writers
4. Retire old compatibility
```

---

# 117. Expand-contract

Enum evolution deberá integrarse con el modelo:

```text
EXPAND
→ MIGRATE
→ CONTRACT
```

del Zero Downtime Migration System.

---

# 118. Remove case

Mucho más peligroso.

Supongamos:

```text
Status::Legacy
```

se elimina.

Antes deberá demostrarse:

```text
no persisted rows use legacy
+
no active application writes legacy
+
no old worker expects legacy
```

---

# 119. Safe removal

```text
stop writes
→ migrate data
→ verify
→ deploy compatible readers
→ contract schema
→ remove code compatibility
```

---

# 120. Rename case

Supongamos:

```php
case Payed = 'payed';
```

se corrige a:

```php
case Paid = 'paid';
```

Hay dos cambios posibles:

```text
PHP case rename
```

y:

```text
persistent value rename
```

No son obligatoriamente el mismo cambio.

---

# 121. Safe PHP-only rename

Puede mantenerse:

```text
persistent value = "payed"
```

temporalmente aunque el caso PHP se llame `Paid`.

---

# 122. Persistent rename

Después:

```text
read:
    payed → Paid
    paid  → Paid

write:
    Paid → paid
```

---

# 123. Dual-read canonical-write

Patrón recomendado:

```text
OLD VALUE ─┐
           ├──→ NEW ENUM CASE
NEW VALUE ─┘

NEW ENUM CASE
     ↓
NEW VALUE
```

---

# 124. Backfill

Después:

```text
UPDATE old values
→ canonical new values
```

mediante migration controlada.

---

# 125. Contract

Finalmente:

```text
remove old alias
remove old schema allowance
```

---

# 126. EnumCompatibilityPolicy

```php
final readonly class EnumCompatibilityPolicy
{
    public function __construct(
        public bool $allowLegacyAliases,
        public bool $canonicalWrite,
        public bool $warnOnLegacyRead,
        public bool $allowUnknown,
    ) {}
}
```

---

# 127. Legacy read telemetry

Cada lectura legacy puede incrementar:

```text
database.enum.legacy_value_read
```

---

# 128. Migration readiness

Cuando:

```text
legacy reads ≈ 0
```

durante la ventana definida, diagnostics puede indicar que el contract es candidato a ejecución.

No debe hacerlo automáticamente.

---

# 129. Default values

Schema default:

```text
DEFAULT 'pending'
```

y application default:

```php
private Status $status = Status::Pending;
```

son conceptos distintos.

---

# 130. Default consistency analysis

Diagnostics podrá detectar:

```text
application default = Pending
database default = Cancelled
```

como posible inconsistencia.

---

# 131. ORM insert behavior

Si ORM envía explícitamente:

```text
pending
```

el DB default no participa.

Si omite la columna, puede participar.

---

# 132. Generated/default state

El Persistence Engine deberá conocer si:

```text
value supplied
```

o:

```text
column omitted for DB default
```

sin delegar esa decisión al Enum Mapper.

---

# 133. Collections of enums

Ejemplo:

```text
roles = ["admin", "editor"]
```

no debe confundirse con un único enum.

---

# 134. Enum set

Puede modelarse mediante:

```text
JSON/array type
+
element enum mapping
```

---

# 135. Native SET

No deberá utilizarse como abstracción portable base.

---

# 136. EnumCollectionMapping

```php
final readonly class EnumCollectionMapping
{
    public function __construct(
        public EnumTypeId $elementType,
        public EnumCollectionStorage $storage,
    ) {}
}
```

---

# 137. Collection semantics

Debe declarar:

```text
LIST
SET
ORDERED_SET
```

cuando sea relevante.

---

# 138. Duplicate enums

Para SET:

```text
[Paid, Paid]
```

deberá normalizarse/rechazarse según policy.

---

# 139. Querying enum collections

Será responsabilidad de:

```text
JSON Query System
Array-capable platform extensions
```

no del Enum Mapping System por sí solo.

---

# 140. Enum relationships

Un enum no es una entidad.

No usar:

```text
many-to-one enum
```

---

# 141. Lookup table alternative

Cuando el dominio necesita:

- metadata dinámica;
- labels editables;
- relationships;
- lifecycle;
- administration;
- tenant-specific values;

deberá considerarse una entidad/lookup table.

---

# 142. Enum vs lookup entity

Use enum cuando el conjunto sea:

```text
small
closed
code-governed
semantically stable
```

Use entidad cuando sea:

```text
dynamic
data-governed
relationship-bearing
user-configurable
```

---

# 143. Tenant-specific enum values

Si cada tenant puede crear estados:

```text
Tenant A:
    gold

Tenant B:
    vip
```

eso normalmente no es un enum PHP.

---

# 144. Authorization

Que un enum contenga:

```text
Admin
```

no concede permisos.

```text
Enum Mapping
≠
Authorization
```

---

# 145. Validation

El Validation System podrá derivar:

```text
allowed enum cases
```

desde metadata, pero seguirá siendo una integración separada.

---

# 146. API input

Input:

```json
{
  "status": "paid"
}
```

puede convertirse a:

```text
OrderStatus::Paid
```

en Request/Validation/Data Mapping.

No debe depender del DB Hydrator.

---

# 147. API aliases

Un API puede exponer:

```text
completed
```

mientras DB persiste:

```text
paid
```

Eso es válido.

---

# 148. API value ≠ DB value

Regla:

```text
External Representation
≠
Application Enum Identity
≠
Persistent Representation
```

---

# 149. Serialization

El serializer podrá decidir:

```text
case name
backing value
API alias
structured representation
```

independientemente del DB mapping.

---

# 150. Query cache

El Query Cache deberá utilizar valores enum ya normalizados/canonicalizados.

---

# 151. Cache key stability

No construir keys usando:

```text
spl_object_id(enum)
```

---

# 152. Recommended cache identity

Usar:

```text
EnumTypeId
+
EnumCaseId
```

o canonical persistent value dentro del query representation correspondiente.

---

# 153. Metadata cache

`CompiledEnumMapping` sí puede compartirse globalmente dentro del worker.

---

# 154. Persistent runtime

Compartible:

```text
EnumMappingRegistry
CompiledEnumMapping
Enum metadata
lookup maps
```

---

# 155. Scoped state

```text
temporary conversion diagnostics
request counters
migration compatibility context
```

cuando corresponda.

---

# 156. No mutable enum mapping per request

El significado de:

```text
Paid
```

no deberá cambiar entre requests dentro del mismo compiled application generation.

---

# 157. FrankenPHP

```text
Worker
├── Frozen Enum Registry
├── Compiled Enum Maps
│
├── Request A
└── Request B
```

---

# 158. RoadRunner

Mismo principio.

---

# 159. OpenSwoole

Registry immutable será naturalmente compartible.

---

# 160. Hot reload

Cambio en:

- enum class;
- cases;
- backing values;
- aliases;
- mapping;
- persistence strategy;

deberá cambiar:

```text
EnumMappingGeneration
```

---

# 161. EnumMappingGeneration

```php
final readonly class EnumMappingGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 162. Fingerprint

```text
EnumTypeId
+
Case Set
+
Case Mappings
+
Persistence Strategy
+
Compatibility Policy
+
Type Registry Generation
+
Platform Capability Generation
```

---

# 163. Prepared statements

Un enum parameter deberá convertirse antes del binding final.

```text
OrderStatus::Paid
      ↓
"paid"
      ↓
String Type
      ↓
PDO binding
```

---

# 164. No enum object binding

El Driver nunca debe necesitar entender `UnitEnum`.

---

# 165. Bulk insert

Enums deberán normalizarse usando el mismo mapping.

No crear un fast path que omita validación semántica.

---

# 166. Bulk update

Igual:

```text
Bulk DML
```

debe respetar enum mapping.

---

# 167. Raw queries

Si el usuario utiliza:

```text
Raw SQL
```

y pasa:

```text
OrderStatus::Paid
```

VoltStack no deberá adivinar el mapping sin type context explícito.

---

# 168. Typed raw parameter

Podrá existir:

```php
EnumParameter::of(OrderStatus::Paid)
```

para conservar contexto.

---

# 169. Security

Enum Mapping deberá impedir:

- arbitrary class loading;
- unknown enum class injection;
- unsafe unserialization;
- silent unknown values;
- schema type name injection;
- unbounded dynamic mappings.

---

# 170. DB value never chooses class

Nunca:

```text
DB:
"App\Domain\AdminStatus"

→ autoload class
```

---

# 171. Registry-first resolution

Las clases enum válidas deben provenir de metadata compilada.

---

# 172. Native enum identifiers

Los nombres físicos:

```text
order_status
```

deberán pasar por el Schema Identifier System.

No concatenarse directamente desde input.

---

# 173. Diagnostics

API conceptual:

```php
Database::types()
    ->enums()
    ->explain(OrderStatus::class);
```

---

# 174. Example diagnostic

```text
ENUM MAPPING

Enum Type:
    commerce.order_status

PHP Enum:
    App\Domain\OrderStatus

Strategy:
    BACKING_VALUE

Persistent Type:
    string

Cases:
    Pending   → "pending"
    Paid      → "paid"
    Cancelled → "cancelled"

Legacy Aliases:
    "payed" → Paid

Unknown Policy:
    FAIL

Schema:
    VARCHAR(32)

Constraint:
    CHECK compatible

Portability:
    FULL

Canonical Write:
    enabled
```

---

# 175. Explain field

```text
ENTITY FIELD

Order.status

PHP Type:
    OrderStatus

Enum Type:
    commerce.order_status

Nullable:
    false

Hydration:
    string → OrderStatus

Persistence:
    OrderStatus → string
```

---

# 176. Telemetry

Métricas:

```text
database.enum.mapping.total
database.enum.mapping.failure
database.enum.unknown_value
database.enum.legacy_value_read
database.enum.canonical_write
database.enum.query_parameter
database.enum.mapping_cache_hit
database.enum.mapping_cache_miss
```

---

# 177. Avoid high cardinality

No incluir raw enum value como metric dimension cuando pueda crecer sin control.

---

# 178. Known case dimension

Para enums pequeños puede permitirse opcionalmente:

```text
enum_type
case_id
```

con cardinalidad controlada.

---

# 179. Sensitive enums

Algunos enums pueden revelar información sensible.

Ejemplo:

```text
medical_status
risk_classification
```

Telemetry deberá permitir redaction.

---

# 180. Error hierarchy

```text
DatabaseEnumMappingException
├── EnumMappingNotFoundException
├── EnumRegistrationException
├── InvalidEnumMappingException
├── EnumTypeMismatchException
├── EnumCaseMappingException
├── DuplicateEnumValueException
├── UnknownEnumValueException
├── EnumBackingValueException
├── EnumQueryTypeMismatchException
├── EnumSchemaCompatibilityException
├── EnumMigrationSafetyException
├── EnumLegacyAliasException
├── EnumCanonicalizationException
├── EnumCollectionMappingException
├── EnumRuntimeStateException
└── EnumMappingInvariantViolationException
```

---

# 181. Unknown value diagnostic

Ejemplo:

```text
UnknownEnumValueException

Entity:
    Order

Property:
    status

Enum:
    commerce.order_status

Persistent Type:
    string

Observed Value:
    [redacted/hash]

Known Cases:
    Pending
    Paid
    Cancelled

Policy:
    FAIL
```

---

# 182. Migration safety diagnostics

Ejemplo:

```text
ENUM MIGRATION SAFETY

Operation:
    REMOVE CASE

Enum:
    commerce.order_status

Case:
    Legacy

Persisted Rows:
    UNKNOWN

Old Application Writers:
    POSSIBLE

Old Application Readers:
    POSSIBLE

Assessment:
    UNSAFE
```

---

# 183. Testing architecture

Debe existir una:

```text
EnumMappingConformanceSuite
```

---

# 184. Core test matrix

Probar:

- string-backed enum;
- int-backed enum;
- pure enum;
- nullable enum;
- explicit mapping;
- aliases;
- unknown values;
- native enum;
- CHECK strategy;
- query parameters;
- hydration;
- persistence;
- dirty tracking;
- bulk operations;
- migrations;
- persistent workers.

---

# 185. Round-trip

Para cada caso válido:

```text
enum
→ persistent
→ enum
```

debe preservar identidad conceptual.

Formalmente:

```text
decode(encode(case)) = case
```

---

# 186. Persistent round-trip

También:

```text
canonical value
→ enum
→ canonical value
```

debe producir el canonical write value.

---

# 187. Alias round-trip

Para alias legacy:

```text
legacy value
→ enum
→ canonical value
```

Ejemplo:

```text
"payed"
→ Paid
→ "paid"
```

---

# 188. Unknown test

Cada valor no reconocido debe seguir exactamente la configured policy.

---

# 189. NULL test

```text
NULL
→ null
```

solo cuando nullable.

---

# 190. Dirty tracking test

```text
hydrate Paid
→ no change
→ flush
```

debe producir:

```text
NO UPDATE
```

---

# 191. Alias dirty test

```text
hydrate legacy alias
→ no explicit modification
→ flush
```

deberá seguir la canonicalization policy configurada.

---

# 192. Query tests

```php
where('status', Status::Paid)
```

debe producir un typed parameter correcto sin SQL literal injection.

---

# 193. `whereIn` test

Todos los elementos deberán pertenecer al enum esperado.

---

# 194. Cross-enum test

Debe rechazarse:

```text
OrderStatus
+
PaymentStatus
```

en un mismo field parameter.

---

# 195. Schema tests

Por plataforma:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 196. Native enum tests

Probar:

- create;
- introspection;
- add value;
- schema diff;
- migration planning;
- unsupported changes;
- rollback feasibility.

---

# 197. Portability tests

Un mapping portable deberá conservar semántica al cambiar:

```text
MySQL → PostgreSQL
PostgreSQL → SQLite
```

aunque la representación física cambie.

---

# 198. Persistent runtime test

Dos requests deberán compartir metadata immutable pero no diagnostics/state mutable.

---

# 199. Performance

Benchmark:

```text
enum hydration/sec
enum persistence conversion/sec
lookup latency
query parameter conversion
compiled mapping cache hit rate
```

---

# 200. Performance strategy

Hot path:

```text
persistent scalar
↓
hash/map lookup
↓
enum case
```

sin:

- reflection;
- scanning `cases()`;
- parsing metadata;
- container lookup;
- schema inspection.

---

# 201. Directory structure

```text
src/Quantum/Database/Type/Enum/
│
├── Contract/
│   ├── EnumMappingRegistry.php
│   ├── EnumValueMapper.php
│   ├── EnumMappingCompiler.php
│   └── EnumSchemaStrategyResolver.php
│
├── Metadata/
│   ├── EnumMappingMetadata.php
│   ├── FieldEnumMapping.php
│   ├── EnumCaseMapping.php
│   ├── EnumTypeId.php
│   ├── EnumCaseId.php
│   ├── PHPEnumReference.php
│   └── EnumPersistentAlias.php
│
├── Strategy/
│   ├── EnumPersistenceStrategy.php
│   ├── EnumSchemaStrategy.php
│   └── EnumCollectionStorage.php
│
├── Policy/
│   ├── EnumUnknownValuePolicy.php
│   ├── EnumCompatibilityPolicy.php
│   └── EnumCanonicalizationPolicy.php
│
├── Registry/
│   ├── DefaultEnumMappingRegistry.php
│   ├── EnumMappingRegistryBuilder.php
│   └── EnumMappingGeneration.php
│
├── Compiler/
│   ├── DefaultEnumMappingCompiler.php
│   ├── CompiledEnumMapping.php
│   └── EnumMappingFingerprint.php
│
├── Runtime/
│   ├── DefaultEnumValueMapper.php
│   ├── KnownEnumValue.php
│   └── UnknownEnumValue.php
│
├── Casting/
│   └── EnumCast.php
│
├── Schema/
│   ├── EnumSchemaDescriptor.php
│   ├── EnumSchemaProjector.php
│   └── EnumSchemaCompatibilityAnalyzer.php
│
├── Collection/
│   └── EnumCollectionMapping.php
│
├── Evolution/
│   ├── EnumEvolutionAnalyzer.php
│   ├── EnumCompatibilityWindow.php
│   └── EnumMigrationAdvisor.php
│
├── Diagnostics/
│   ├── EnumMappingExplainer.php
│   └── EnumMappingDiagnosticReport.php
│
└── Exception/
    ├── DatabaseEnumMappingException.php
    ├── EnumMappingNotFoundException.php
    ├── EnumRegistrationException.php
    ├── InvalidEnumMappingException.php
    ├── EnumTypeMismatchException.php
    ├── EnumCaseMappingException.php
    ├── DuplicateEnumValueException.php
    ├── UnknownEnumValueException.php
    ├── EnumBackingValueException.php
    ├── EnumQueryTypeMismatchException.php
    ├── EnumSchemaCompatibilityException.php
    ├── EnumMigrationSafetyException.php
    ├── EnumLegacyAliasException.php
    ├── EnumCanonicalizationException.php
    ├── EnumCollectionMappingException.php
    └── EnumMappingInvariantViolationException.php
```

---

# 202. Dependency model

Permitido:

```text
Enum Mapping
     ↓
Type System

Enum Mapping
     ↓
Type Registry

Enum Mapping
     ↓
Casting contracts

Enum Mapping
     ↓
Value Conversion contracts

Enum Mapping
     ↓
Schema metadata contracts
```

---

# 203. ORM dependency

```text
ORM
 ↓
Enum Mapping
```

No:

```text
Enum Mapping
 ↓
EntityManager
```

---

# 204. Compiler boundary

```text
Enum Mapping
→ typed canonical parameter
→ Query Engine
→ SQL Compiler
```

El SQL Compiler no necesita conocer el enum PHP.

---

# 205. Driver boundary

```text
Driver
```

solo recibe valores ya convertidos según el tipo correspondiente.

---

# 206. Architectural invariants

## DB-ENUM-001
Un enum representa un dominio lógico cerrado.

## DB-ENUM-002
EnumTypeId será distinto del FQCN PHP.

## DB-ENUM-003
Enum case identity será distinta del backing value.

## DB-ENUM-004
Backing value será distinto conceptualmente de DB representation.

## DB-ENUM-005
Application Enum no será equivalente a native DB ENUM.

## DB-ENUM-006
Enum Mapping no sustituirá al Type System.

## DB-ENUM-007
Enum Mapping no sustituirá a Casting.

## DB-ENUM-008
Enum Mapping no sustituirá a Value Conversion.

## DB-ENUM-009
Enum Mapping no sustituirá a Validation.

## DB-ENUM-010
Enum Mapping no ejecutará SQL.

## DB-ENUM-011
Enum Mapping no abrirá conexiones.

## DB-ENUM-012
Enum Mapping no hará schema introspection en hot path.

## DB-ENUM-013
Enum Registry será immutable durante runtime productivo.

## DB-ENUM-014
Enum mapping se compilará durante bootstrap.

## DB-ENUM-015
No habrá reflection por fila hidratada.

## DB-ENUM-016
Backed enums serán soportados.

## DB-ENUM-017
Pure enums serán soportados mediante mapping explícito o case-name strategy.

## DB-ENUM-018
Ordinal implícito estará prohibido.

## DB-ENUM-019
Integer mappings no asumirán continuidad.

## DB-ENUM-020
Valores retirados no deberán reutilizarse con otra semántica.

## DB-ENUM-021
Cada caso tendrá un canonical write value.

## DB-ENUM-022
Legacy aliases podrán existir únicamente de forma explícita.

## DB-ENUM-023
Legacy alias no será canonical write value salvo configuración explícita.

## DB-ENUM-024
Unknown value no será convertido silenciosamente a null.

## DB-ENUM-025
FAIL será la política unknown predeterminada.

## DB-ENUM-026
Fallback deberá ser explícito.

## DB-ENUM-027
Preserve-unknown requerirá un PHP type compatible.

## DB-ENUM-028
NULL será distinto de Unknown.

## DB-ENUM-029
MISSING será distinto de NULL.

## DB-ENUM-030
Enum hydration respetará LoadedFieldMask.

## DB-ENUM-031
Enum cases no entrarán al IdentityMap.

## DB-ENUM-032
Snapshots usarán semántica persistente estable.

## DB-ENUM-033
Legacy reads no provocarán necesariamente writes implícitos.

## DB-ENUM-034
Canonicalization policy será explícita.

## DB-ENUM-035
Bulk legacy normalization pertenecerá normalmente a Migration.

## DB-ENUM-036
Query Builder podrá aceptar enum cases tipados.

## DB-ENUM-037
Query Builder no generará SQL enum directamente.

## DB-ENUM-038
SQL Compiler no dependerá de PHP enum classes.

## DB-ENUM-039
Driver no dependerá de UnitEnum.

## DB-ENUM-040
Cross-enum query parameters serán rechazados.

## DB-ENUM-041
Mismo backing value en dos enum types no implica misma identidad.

## DB-ENUM-042
Declaration order no será semántica de orden implícita.

## DB-ENUM-043
Business ordering será explícito.

## DB-ENUM-044
Native DB ordering no será base portable.

## DB-ENUM-045
Enum Mapping podrá proyectar recomendaciones de schema.

## DB-ENUM-046
Enum Mapping no afirmará que una constraint existe sin evidencia de Schema.

## DB-ENUM-047
Portable scalar + CHECK será estrategia base recomendada.

## DB-ENUM-048
Native DB enums serán opcionales.

## DB-ENUM-049
MySQL y MariaDB tendrán análisis independiente.

## DB-ENUM-050
PostgreSQL native enum será capability específica.

## DB-ENUM-051
SQLite utilizará representación portable cuando corresponda.

## DB-ENUM-052
Schema diff distinguirá add/remove/rename/change representation.

## DB-ENUM-053
Enum evolution será tratada como compatibility problem.

## DB-ENUM-054
Agregar un caso puede requerir expand-contract.

## DB-ENUM-055
Eliminar un caso requerirá evidencia de seguridad.

## DB-ENUM-056
PHP rename y persistent rename serán cambios distintos.

## DB-ENUM-057
Dual-read canonical-write será estrategia soportada.

## DB-ENUM-058
Backfill será una operación de Migration, no de Hydration.

## DB-ENUM-059
Compatibility windows serán explícitas.

## DB-ENUM-060
Application default y DB default serán distintos conceptos.

## DB-ENUM-061
ORM no asumirá que DB default coincide con application default.

## DB-ENUM-062
Enum collection no será relación ORM.

## DB-ENUM-063
Enum set semantics serán explícitas.

## DB-ENUM-064
Dynamic tenant values no se modelarán automáticamente como PHP enum.

## DB-ENUM-065
Enums no concederán autorización.

## DB-ENUM-066
API enum representation será independiente del DB mapping.

## DB-ENUM-067
Serialization será independiente del DB mapping.

## DB-ENUM-068
Query cache utilizará identidad enum estable.

## DB-ENUM-069
Enum metadata podrá compartirse entre requests.

## DB-ENUM-070
Mutable runtime state será scoped.

## DB-ENUM-071
Enum meaning no cambiará dinámicamente por request.

## DB-ENUM-072
Mapping changes producirán nueva generation.

## DB-ENUM-073
Prepared statement binding recibirá canonical persistent values.

## DB-ENUM-074
Bulk persistence respetará enum mapping.

## DB-ENUM-075
Raw SQL requerirá type context para enum parameters.

## DB-ENUM-076
DB values nunca seleccionarán PHP classes.

## DB-ENUM-077
Enum classes deberán provenir del registry compilado.

## DB-ENUM-078
Native enum identifiers pasarán por identifier validation.

## DB-ENUM-079
Diagnostics no revelarán valores sensibles innecesariamente.

## DB-ENUM-080
Telemetry tendrá cardinalidad controlada.

## DB-ENUM-081
Round-trip preservará enum case identity.

## DB-ENUM-082
Alias round-trip terminará en canonical write value.

## DB-ENUM-083
No-op hydrate/flush no producirá false dirty.

## DB-ENUM-084
Unknown values tendrán tests obligatorios.

## DB-ENUM-085
Cross-platform conformance será obligatorio.

## DB-ENUM-086
Persistent runtime isolation será probado.

## DB-ENUM-087
Hot path utilizará compiled lookup maps.

## DB-ENUM-088
Enum Mapping no realizará container lookup por fila.

## DB-ENUM-089
Enum Mapping no hará lazy loading.

## DB-ENUM-090
Enum Mapping no accederá a EntityManager.

## DB-ENUM-091
Enum Mapping no accederá a UnitOfWork mutable directamente.

## DB-ENUM-092
Enum Mapping no hará flush.

## DB-ENUM-093
Enum Mapping no hará commit.

## DB-ENUM-094
Enum Mapping no será un segundo persistence engine.

## DB-ENUM-095
Schema representation será una decisión de mapping/capability.

## DB-ENUM-096
Unknown schema evidence no será asumida compatible.

## DB-ENUM-097
Removed enum values no se considerarán seguros sin data evidence.

## DB-ENUM-098
Rolling deployment compatibility será considerada en migrations.

## DB-ENUM-099
Old readers serán considerados antes de habilitar nuevos writers.

## DB-ENUM-100
Old writers serán considerados antes de retirar valores legacy.

## DB-ENUM-101
Enum aliases tendrán lifecycle explícito.

## DB-ENUM-102
Deprecated aliases podrán generar telemetry.

## DB-ENUM-103
Enum mapping será deterministic.

## DB-ENUM-104
Enum metadata será immutable.

## DB-ENUM-105
Compiled enum mappings serán cacheables.

## DB-ENUM-106
EnumMappingFingerprint incluirá mapping generation.

## DB-ENUM-107
Application enum case no será SQL literal.

## DB-ENUM-108
Enum conversion no decidirá transaction state.

## DB-ENUM-109
Enum conversion no decidirá connection routing.

## DB-ENUM-110
Enum conversion no decidirá authorization.

## DB-ENUM-111
Enum conversion no decidirá HTTP serialization.

## DB-ENUM-112
Enum conversion no fabricará valores desconocidos.

## DB-ENUM-113
Case removal deberá ser contract operation explícita.

## DB-ENUM-114
Case addition deberá considerar DB constraint compatibility.

## DB-ENUM-115
Canonical writes deberán ser deterministas.

## DB-ENUM-116
Case mappings duplicados serán rechazados.

## DB-ENUM-117
Ambiguous legacy aliases serán rechazados.

## DB-ENUM-118
Un persistent value no mapeará a dos cases activos.

## DB-ENUM-119
Cada active case tendrá como máximo un canonical write value.

## DB-ENUM-120
VoltStack preservará la separación entre dominio enum y representación física.

---

# 207. Anti-patterns

## 207.1 Persistir ordinal automáticamente

```text
Pending = 0
Paid = 1
Cancelled = 2
```

basándose en declaration order.

**Rechazado.**

---

## 207.2 Convertir unknown a null

```text
"legacy"
→ null
```

**Rechazado.**

---

## 207.3 Usar FQCN como DB value

```text
App\Domain\OrderStatus\Paid
```

**Rechazado por default.**

---

## 207.4 SQL desde EnumCast

```php
$this->connection->query(...);
```

**Rechazado.**

---

## 207.5 Native ENUM obligatorio

```text
PHP enum
→ DB ENUM always
```

**Rechazado.**

---

## 207.6 Reutilizar integer code

```text
1 = ACTIVE
```

posteriormente:

```text
1 = DELETED
```

**Rechazado.**

---

## 207.7 Rename destructivo inmediato

```text
"payed"
→ removed
```

sin compatibility window.

**Rechazado para zero-downtime environments.**

---

## 207.8 Enum para datos configurables

```text
CustomerCategory
```

si administradores pueden crear categorías dinámicamente.

**Normalmente rechazado como enum.**

---

# 208. Ejemplo completo

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';
}
```

Metadata conceptual:

```php
EnumMappingMetadata(
    id: new EnumTypeId('commerce.order_status'),
    phpEnum: OrderStatus::class,
    strategy: EnumPersistenceStrategy::BACKING_VALUE,
    cases: [
        Pending   => 'pending',
        Paid      => 'paid',
        Cancelled => 'cancelled',
    ],
);
```

---

# 209. Hydration example

DB:

```text
status = "paid"
```

Pipeline:

```text
"paid"
   ↓
String Value Conversion
   ↓
canonical "paid"
   ↓
CompiledEnumMapping
   ↓
OrderStatus::Paid
```

---

# 210. Persistence example

Entity:

```php
$order->status = OrderStatus::Cancelled;
```

Pipeline:

```text
OrderStatus::Cancelled
       ↓
Enum Mapper
       ↓
"cancelled"
       ↓
String Value Conversion
       ↓
Prepared Statement
```

---

# 211. Query example

```php
Order::query()
    ->where('status', OrderStatus::Paid)
    ->get();
```

Internamente:

```text
OrderStatus::Paid
       ↓
EnumTypeId
       ↓
Canonical Persistent Value
       ↓
TypedParameter(string, "paid")
       ↓
Query AST
       ↓
Planner
       ↓
Compiler
       ↓
Prepared Statement
```

---

# 212. Migration example

Versión inicial:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Payed = 'payed';
}
```

Nueva versión:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
}
```

Transición:

```text
READ:
"payed" → Paid
"paid"  → Paid

WRITE:
Paid → "paid"
```

Después:

```text
backfill "payed" → "paid"
```

Finalmente:

```text
remove legacy alias "payed"
```

---

# 213. Enum migration state machine

```text
STABLE_OLD
    │
    ▼
EXPANDED
    │
    ▼
DUAL_READ
    │
    ▼
CANONICAL_WRITE
    │
    ▼
BACKFILLED
    │
    ▼
VERIFIED
    │
    ▼
CONTRACTED
    │
    ▼
STABLE_NEW
```

---

# 214. Master mapping formula

Para un caso `E`:

```text
PersistentValue
=
EnumWriteMap(E)
```

Lectura:

```text
EnumCase
=
EnumReadMap(PersistentValue)
```

---

# 215. Alias formula

```text
ReadMap(alias)
=
ReadMap(canonical)
=
Same EnumCase
```

pero:

```text
WriteMap(EnumCase)
=
canonical only
```

---

# 216. Unknown formula

```text
PersistentValue ∉ KnownReadDomain
```

produce:

```text
UnknownValuePolicy(PersistentValue)
```

y nunca implica automáticamente:

```text
NULL
```

---

# 217. Portability formula

```text
LogicalEnumSemantics
=
constant
```

mientras:

```text
PhysicalDatabaseRepresentation
=
PlatformStrategy(
    Capabilities,
    MappingPolicy
)
```

---

# 218. Architecture master model

```text
                    APPLICATION DOMAIN
                           │
                           ▼
                    PHP ENUM CASE
                           │
                           ▼
                ENUM MAPPING SYSTEM
                 ┌─────────┴─────────┐
                 │                   │
          Case Identity       Persistence Mapping
                 │                   │
                 └─────────┬─────────┘
                           ▼
              CANONICAL PERSISTENT VALUE
                           │
                           ▼
                    TYPE SYSTEM
                           │
                           ▼
                  VALUE CONVERSION
                           │
                           ▼
                   QUERY / BINDING
                           │
                           ▼
                       DATABASE
                           │
                  ┌────────┴────────┐
                  │                 │
             VARCHAR/INT       Native ENUM
                  │                 │
                  └────────┬────────┘
                           ▼
                   SCHEMA SYSTEM
                           │
                           ▼
                  MIGRATION SYSTEM
```

---

# 219. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Stable Logical Enum Identity
```

mediante `EnumTypeId`.

Adoptará:

```text
Compiled Bidirectional Mapping
```

para runtime.

Adoptará:

```text
Canonical Write Values
```

con aliases de lectura opcionales.

Adoptará:

```text
FAIL on Unknown
```

como default seguro.

Adoptará:

```text
Portable Scalar Representation
+
Optional CHECK
```

como estrategia portable recomendada.

Adoptará:

```text
Native Database Enums
```

como capability opcional.

Adoptará:

```text
Expand → Migrate → Contract
```

para evolución segura.

Y mantendrá:

```text
Enum Domain
≠
PHP Representation
≠
Persistent Representation
≠
Database Physical Type
```

---

# 220. Regla maestra final

> **VoltStack deberá tratar los enums como dominios lógicos versionados y no como simples strings o integers. El framework podrá optimizar y adaptar su representación física a cada plataforma, pero la identidad del enum, la semántica de sus casos y su evolución deberán permanecer independientes del almacenamiento.**

Esto permite que:

```text
OrderStatus::Paid
```

continúe significando exactamente lo mismo aunque la representación evolucione:

```text
"payed"
   ↓
"paid"
   ↓
2
   ↓
PostgreSQL native enum
```

si una migración explícita decide realizar esas transiciones.

La base de datos cambia de representación.

El dominio no cambia accidentalmente de significado.

---

# 221. Siguiente documento

```text
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md
```

El siguiente documento deberá definir cómo VoltStack persiste objetos de valor que representan conceptos de dominio más ricos que un scalar o enum, incluyendo:

- immutable Value Objects;
- single-column Value Objects;
- multi-column Value Objects;
- embedded/composite mappings;
- nested Value Objects;
- value object identity vs entity identity;
- structural equality;
- canonical representation;
- constructors y named constructors;
- hydration;
- persistence;
- dirty tracking;
- snapshots;
- nullability;
- partial hydration;
- column prefixes;
- overrides;
- nested paths;
- flattening;
- Query Engine;
- querying embedded properties;
- indexing;
- schema projection;
- Value Objects JSON;
- Money;
- EmailAddress;
- Address;
- DateRange;
- GeoPoint;
- encryption;
- custom mapping;
- collections;
- persistent runtime;
- telemetry;
- diagnostics;
- testing;
- performance;
- architectural invariants.

Regla central propuesta:

> **Un Value Object persistente en VoltStack será tratado como un valor de dominio sin identidad ORM independiente: podrá ocupar una o varias representaciones persistentes, pero nunca será promovido implícitamente a Entity, IdentityMap entry o Relationship únicamente por estar compuesto por múltiples campos.**