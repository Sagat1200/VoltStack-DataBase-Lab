# 162_DATABASE_DATE_TIME_TYPE_SYSTEM.md

# VoltStack Quantum Database
## Database Date & Time Type System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 162 — Database Date & Time Type System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `161_DATABASE_JSON_TYPE_SYSTEM.md`  
**Siguiente documento:** `163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md`

---

# 1. Propósito

`Database Date & Time Type System` define el modelo temporal canónico mediante el cual VoltStack representará, convertirá, comparará, consultará, persistirá e hidratará fechas, horas, timestamps, offsets y zonas horarias.

El objetivo principal será impedir que conceptos temporalmente diferentes terminen reducidos indiscriminadamente a:

```php
DateTime
```

o a:

```text
YYYY-MM-DD HH:MM:SS
```

sin saber qué representan realmente.

La regla central será:

> **VoltStack no tratará fecha, hora, instante y zona horaria como variantes intercambiables de `DateTime`: cada concepto temporal tendrá semántica lógica explícita, y cualquier conversión entre tiempo local, offset, zona e instante deberá ser intencional, determinista y visible para el Type System.**

---

# 2. Problema fundamental

El valor:

```text
2026-09-08 12:30:00
```

no responde por sí mismo a:

```text
¿Es una hora local?
¿En qué zona?
¿Es UTC?
¿Representa un instante global?
¿Tiene offset?
¿La zona debe conservarse?
¿Tiene microsegundos?
```

Sin metadata adicional:

```text
LocalDateTime
```

y:

```text
Instant
```

podrían terminar almacenándose exactamente igual aunque tengan semánticas distintas.

VoltStack deberá impedirlo.

---

# 3. Separaciones fundamentales

```text
Date
≠
Time
≠
LocalDateTime
≠
Instant
≠
OffsetDateTime
≠
ZonedDateTime
≠
Duration
```

También:

```text
Timezone
≠
UTC Offset
```

y:

```text
LocalDateTime
≠
Timestamp
```

---

# 4. Arquitectura temporal

```text
Application Temporal Value
           │
           ▼
    Temporal Type System
           │
    ┌──────┴───────┐
    │              │
Semantic Type   Temporal Policy
    │              │
    └──────┬───────┘
           ▼
   Canonical Temporal Value
           │
           ▼
      Value Conversion
           │
           ▼
    Platform Resolution
           │
           ▼
 Physical Temporal Type
           │
           ▼
          Driver
           │
           ▼
        Database
```

---

# 5. Tipos temporales canónicos

VoltStack deberá proporcionar al menos:

```text
DATE
TIME
LOCAL_DATETIME
INSTANT
OFFSET_DATETIME
ZONED_DATETIME
DURATION
```

con extensiones posteriores cuando sean necesarias.

---

# 6. Date

`Date` representa exclusivamente:

```text
year
month
day
```

Ejemplo:

```text
2026-09-08
```

No contiene:

- hora;
- timezone;
- offset;
- instante.

---

# 7. Date invariant

```text
Date
≠
midnight DateTime
```

La fecha:

```text
2026-09-08
```

no deberá convertirse conceptualmente en:

```text
2026-09-08T00:00:00 UTC
```

sin operación explícita.

---

# 8. Time

`Time` representa una hora local del día:

```text
12:30:45.123456
```

sin fecha y sin zona por defecto.

---

# 9. Time ≠ elapsed duration

```text
23:30:00
```

es hora del día.

No significa:

```text
23 horas y 30 minutos
```

---

# 10. LocalDateTime

Representa:

```text
Date + Time
```

sin determinar un instante global.

Ejemplo:

```text
2026-09-08T12:30:00
```

---

# 11. LocalDateTime ambiguity

El mismo `LocalDateTime` puede corresponder a distintos instantes dependiendo de la zona.

Por tanto:

```text
LocalDateTime
→ Instant
```

requiere explícitamente:

```text
Timezone
+
DST resolution policy
```

---

# 12. Instant

`Instant` representa un punto inequívoco sobre la línea temporal global.

Conceptualmente:

```text
UTC timeline position
```

Ejemplo:

```text
2026-09-08T18:30:00Z
```

---

# 13. Instant invariant

Dos valores:

```text
2026-09-08T18:30:00Z
```

y:

```text
2026-09-08T12:30:00-06:00
```

representan el mismo `Instant`.

---

# 14. OffsetDateTime

Representa:

```text
LocalDateTime
+
UTC Offset
```

Ejemplo:

```text
2026-09-08T12:30:00-06:00
```

---

# 15. Offset ≠ timezone

```text
-06:00
```

no identifica necesariamente:

```text
America/Monterrey
```

Muchas zonas pueden compartir temporalmente el mismo offset.

---

# 16. ZonedDateTime

Representa conceptualmente:

```text
LocalDateTime
+
Timezone Identifier
+
Resolved Offset
```

Ejemplo:

```text
2026-09-08T12:30:00
America/Monterrey
UTC-06:00
```

---

# 17. Timezone identity

VoltStack deberá utilizar identificadores IANA cuando la semántica requiera una zona:

```text
America/Monterrey
America/New_York
Europe/Madrid
Asia/Tokyo
```

---

# 18. Timezone ≠ abbreviation

No persistir como identidad primaria:

```text
CST
EST
IST
```

porque pueden ser ambiguas.

---

# 19. Duration

`Duration` representa cantidad de tiempo transcurrido.

Ejemplo:

```text
3600 seconds
```

o:

```text
PT1H
```

---

# 20. Duration ≠ DateTime

Una duración:

```text
24 hours
```

no necesariamente equivale a:

```text
1 local calendar day
```

durante cambios DST.

---

# 21. Period / Calendar interval

VoltStack podrá en el futuro distinguir:

```text
Duration
```

de:

```text
CalendarPeriod
```

como:

```text
1 month
1 year
1 calendar day
```

porque no tienen duración fija universal.

---

# 22. PHP representation

VoltStack podrá interoperar con:

```php
DateTimeImmutable
DateTimeInterface
DateInterval
DateTimeZone
```

pero internamente no deberá asumir que `DateTimeImmutable` expresa por sí mismo toda la semántica requerida.

---

# 23. Immutable preference

La representación recomendada será immutable.

```text
DateTimeImmutable
```

será preferido frente a:

```text
DateTime
```

---

# 24. Razón

Los temporales mutables complican:

- snapshots;
- dirty tracking;
- thread/coroutine safety;
- cache;
- reproducibilidad.

---

# 25. Temporal Value Objects

VoltStack podrá definir objetos internos:

```text
LocalDate
LocalTime
LocalDateTime
Instant
OffsetDateTime
ZonedDateTime
Duration
```

o integrar librerías externas mediante adapters.

---

# 26. Framework independence

El Type System no deberá depender obligatoriamente de una librería temporal third-party.

---

# 27. TemporalTypeId

Tipos lógicos:

```text
date
time
local_datetime
instant
offset_datetime
zoned_datetime
duration
```

---

# 28. Temporal descriptor

```php
final readonly class TemporalTypeDescriptor
{
    public function __construct(
        public TemporalKind $kind,
        public TemporalPrecision $precision,
        public TemporalTimezonePolicy $timezone,
        public TemporalAmbiguityPolicy $ambiguity,
    ) {}
}
```

---

# 29. TemporalKind

```php
enum TemporalKind
{
    case DATE;
    case TIME;
    case LOCAL_DATETIME;
    case INSTANT;
    case OFFSET_DATETIME;
    case ZONED_DATETIME;
    case DURATION;
}
```

---

# 30. Precision

VoltStack deberá representar precisión fraccional explícitamente.

Ejemplo:

```text
seconds precision = 0
milliseconds = 3
microseconds = 6
```

---

# 31. TemporalPrecision

```php
final readonly class TemporalPrecision
{
    public function __construct(
        public int $fractionalDigits,
    ) {}
}
```

---

# 32. Precision invariant

```text
Requested precision
≠
Platform precision
```

La compatibilidad deberá analizarse.

---

# 33. Precision loss

Valor:

```text
12:30:45.123456
```

almacenado en:

```text
precision 3
```

produce:

```text
12:30:45.123
```

y constituye pérdida de información.

---

# 34. No silent truncation

Por defecto:

```text
precision 6
→ platform precision 3
```

deberá ser:

```text
LOSSY
```

y gobernarse por policy.

---

# 35. TemporalPrecisionPolicy

```php
enum TemporalPrecisionLossPolicy
{
    case FORBID;
    case TRUNCATE_EXPLICIT;
    case ROUND_EXPLICIT;
}
```

---

# 36. Timestamp ambiguity

El término:

```text
TIMESTAMP
```

tiene semánticas distintas entre plataformas.

Por ello:

```text
SQL TIMESTAMP
≠
VoltStack Instant
```

automáticamente.

---

# 37. Logical type first

VoltStack decidirá primero:

```text
Instant
LocalDateTime
OffsetDateTime
```

y después pedirá a Platform determinar la representación física.

---

# 38. Platform resolution

```text
TemporalTypeDescriptor
        ↓
TemporalPlatformResolver
        ↓
PlatformCapabilities
        ↓
PhysicalTemporalType
```

---

# 39. Date physical mapping

Frecuentemente:

```text
DATE
```

pero seguirá siendo resolución Platform.

---

# 40. Time physical mapping

Puede mapearse a:

```text
TIME(p)
```

o representación equivalente.

---

# 41. LocalDateTime mapping

Puede mapearse a:

```text
DATETIME
TIMESTAMP WITHOUT TIME ZONE
TEXT
numeric representation
```

según plataforma.

---

# 42. Instant mapping

Puede utilizar:

```text
UTC normalized DATETIME
timestamp type
timestamp with timezone semantics
integer epoch
```

según policy y plataforma.

---

# 43. OffsetDateTime mapping

Puede requerir:

```text
single native type
```

o:

```text
datetime
+
offset column
```

cuando el motor no conserve offset.

---

# 44. ZonedDateTime mapping

Para conservar zona real puede requerir:

```text
instant
+
timezone identifier
```

por ejemplo:

```text
occurred_at
occurred_timezone
```

---

# 45. ZonedDateTime ≠ TIMESTAMP WITH TIME ZONE

Incluso cuando una DB dispone de un tipo con zona horaria, deberá verificarse si conserva:

```text
America/Monterrey
```

o únicamente normaliza el instante/offset.

---

# 46. Zone-preserving strategy

Para requisitos de zona explícita, VoltStack podrá usar:

```text
Instant
+
IANA Timezone ID
```

como estrategia portable.

---

# 47. TemporalStorageStrategy

```php
enum TemporalStorageStrategy
{
    case PLATFORM_NATIVE;
    case UTC_NORMALIZED;
    case LOCAL;
    case OFFSET_PRESERVING;
    case ZONE_PRESERVING;
    case UNIX_EPOCH;
    case TEXTUAL_ISO8601;
}
```

---

# 48. Recommended instant strategy

Para instantes:

```text
UTC_NORMALIZED
```

será la recomendación general cuando la aplicación no necesite conservar la zona original.

---

# 49. UTC normalization

Entrada:

```text
2026-09-08T12:30:00-06:00
```

Instant:

```text
2026-09-08T18:30:00Z
```

La DB puede almacenar el valor UTC normalizado.

---

# 50. Losing zone information

Sin embargo, UTC normalization pierde:

```text
original timezone identity
```

si solo se almacena el instante.

Esto deberá ser una decisión explícita.

---

# 51. User-facing schedules

Casos como:

```text
"todos los días a las 09:00 en America/Monterrey"
```

no deberán modelarse como un `Instant` fijo.

---

# 52. Scheduling semantics

Un schedule requiere típicamente:

```text
LocalTime
+
Timezone
+
Recurrence
```

y pertenece a un sistema de scheduling, aunque reutilice temporal types.

---

# 53. Timezone database

La resolución de zonas deberá utilizar una fuente confiable compatible con IANA tz database.

---

# 54. Tz database version

La versión de timezone data puede afectar:

```text
historical offsets
future legal changes
DST rules
```

---

# 55. Timezone database versioning

Cuando sea material para reproducibilidad, el contexto podrá registrar:

```text
TimezoneDatabaseVersion
```

---

# 56. No timezone rules in DB Type

El tipo temporal no codificará manualmente reglas DST.

Delegará a un timezone resolver.

---

# 57. DST ambiguity

Algunas horas locales pueden ocurrir dos veces.

Ejemplo conceptual:

```text
01:30
```

durante retroceso DST.

---

# 58. AmbiguousLocalTime

```text
LocalDateTime + Timezone
```

puede producir:

```text
two candidate Instants
```

---

# 59. Ambiguity policy

```php
enum TemporalAmbiguityPolicy
{
    case REJECT;
    case EARLIER_OFFSET;
    case LATER_OFFSET;
    case REQUIRE_EXPLICIT_OFFSET;
}
```

Default recomendado:

```text
REJECT
```

para conversiones sensibles.

---

# 60. Nonexistent local time

Durante adelanto DST puede existir una hora como:

```text
02:30
```

que nunca ocurrió en una zona determinada.

---

# 61. Nonexistent policy

```php
enum NonexistentLocalTimePolicy
{
    case REJECT;
    case SHIFT_FORWARD;
    case SHIFT_BACKWARD;
}
```

Default:

```text
REJECT
```

---

# 62. No silent DST correction

VoltStack no deberá modificar:

```text
02:30
```

a:

```text
03:30
```

sin policy explícita.

---

# 63. Offset validation

Cuando se proporciona:

```text
LocalDateTime
+
Timezone
+
Offset
```

VoltStack deberá poder verificar que el offset sea válido para esa zona en ese instante.

---

# 64. Temporal consistency

Ejemplo inconsistente:

```text
America/Monterrey
+
offset +09:00
```

deberá rechazarse si se declara que ambos deben corresponder.

---

# 65. Leap seconds

La mayoría de stacks y bases SQL no manejan leap seconds uniformemente.

VoltStack deberá declarar una policy.

---

# 66. Recommended leap-second policy

V1:

```text
REJECT_UNREPRESENTABLE_LEAP_SECOND
```

en vez de inventar normalización silenciosa.

---

# 67. Unix epoch

Un `Instant` puede representarse como:

```text
seconds since epoch
milliseconds since epoch
microseconds since epoch
```

---

# 68. Epoch unit

Nunca inferir:

```text
1690000000000
```

como seconds o milliseconds automáticamente.

---

# 69. EpochUnit

```php
enum EpochUnit
{
    case SECOND;
    case MILLISECOND;
    case MICROSECOND;
    case NANOSECOND;
}
```

---

# 70. Epoch strategy portability

Integer epochs pueden ser portables, pero sacrifican legibilidad y algunas capacidades DB nativas.

---

# 71. Textual ISO representation

SQLite o custom mappings pueden usar:

```text
ISO-8601 string
```

---

# 72. Canonical textual format

Cuando se utilice texto, deberá definirse exactamente:

```text
date format
separator
fraction precision
offset policy
timezone policy
```

---

# 73. Locale independence

Database temporal representation nunca dependerá de:

```text
08/09/2026
09/08/2026
```

según locale.

---

# 74. Canonical date format

Usar semántica equivalente a:

```text
YYYY-MM-DD
```

---

# 75. Canonical time format

```text
HH:mm:ss[.fraction]
```

---

# 76. Canonical instant format

Cuando textual:

```text
ISO-8601/RFC3339-compatible UTC representation
```

según policy definida.

---

# 77. Value Conversion integration

Read:

```text
Driver Raw Temporal
       ↓
Platform normalization
       ↓
Canonical Temporal Value
       ↓
PHP/domain temporal value
```

Write:

```text
PHP/domain temporal
       ↓
Canonicalization
       ↓
Platform representation
       ↓
Binding
```

---

# 78. Casting integration

Casting puede convertir:

```text
Canonical Instant
→ DateTimeImmutable
```

o:

```text
Canonical Date
→ LocalDate Value Object
```

---

# 79. Formatting is not casting by default

Transformar:

```text
Instant
→ "8 Sep 2026"
```

es presentación/serialization, no DB cast persistente.

---

# 80. Hydration

```text
Result Row
    ↓
Temporal Value Conversion
    ↓
Canonical Temporal Value
    ↓
Temporal Cast
    ↓
Entity Property
```

---

# 81. Hydration no timezone guessing

El Hydrator no deberá consultar:

```php
date_default_timezone_get()
```

para decidir la semántica de un campo.

---

# 82. PHP global timezone

Será irrelevante para mappings explícitos.

---

# 83. DateTimeImmutable adaptation

Si una propiedad es:

```php
private DateTimeImmutable $createdAt;
```

metadata deberá especificar qué semántica tiene:

```text
Instant?
LocalDateTime?
OffsetDateTime?
```

---

# 84. PHP property type alone insufficient

```text
DateTimeImmutable
```

no determina automáticamente el Database Temporal Type.

---

# 85. Example mapping

```php
#[Column(type: 'instant', precision: 6)]
private DateTimeImmutable $createdAt;
```

---

# 86. Local example

```php
#[Column(type: 'local_datetime')]
private DateTimeImmutable $meetingLocalTime;
```

Aunque ambas propiedades usen la misma clase PHP, poseen semánticas distintas.

---

# 87. Parameter binding

Query:

```php
Event::query()
    ->where('occurredAt', '>=', $instant);
```

Pipeline:

```text
Event.occurredAt
→ Instant TypeReference
→ temporal conversion
→ normalized DB representation
→ binding
```

---

# 88. Query Engine

El Query Type System deberá conocer:

```text
Date
Time
LocalDateTime
Instant
OffsetDateTime
Duration
```

como tipos semánticamente distintos.

---

# 89. Invalid comparison

Ejemplo:

```text
Date = Instant
```

no deberá ser considerado compatible automáticamente.

---

# 90. Explicit conversion

Podría escribirse mediante operaciones:

```text
date(instant, timezone)
```

cuando la semántica esté declarada.

---

# 91. Temporal comparison

Para `Instant`:

```text
earlier/later
```

representa orden global.

---

# 92. LocalDateTime comparison

Compara valores locales.

No necesariamente el orden global de eventos.

---

# 93. Example

```text
09:00 Tokyo
09:00 Monterrey
```

son `LocalTime` iguales pero corresponden a instantes distintos.

---

# 94. OffsetDateTime equality

VoltStack deberá distinguir:

```text
same instant
```

de:

```text
same local representation and offset
```

---

# 95. Temporal equality modes

```php
enum TemporalEqualityMode
{
    case VALUE;
    case INSTANT;
    case REPRESENTATION;
}
```

El tipo/mapping determinará cuál aplica.

---

# 96. Instant equality

```text
2026-09-08T12:00-06:00
```

y:

```text
2026-09-08T18:00Z
```

son iguales como `Instant`.

---

# 97. ZonedDateTime equality

Puede necesitar distinguir:

```text
same instant
```

de:

```text
same zone semantics
```

---

# 98. Dirty tracking

Debe usar la igualdad canónica correspondiente al temporal type.

---

# 99. No raw string dirty comparison

Evitar:

```text
2026-09-08 18:00:00+00
```

vs:

```text
2026-09-08 12:00:00-06
```

como diferentes si el campo representa `Instant`.

---

# 100. Temporal snapshot

El UoW podrá conservar una forma canónica como:

```text
epoch seconds + fractional component
```

para `Instant`.

---

# 101. Local temporal snapshot

Para `LocalDateTime` conservar:

```text
date fields
time fields
precision
```

sin zona artificial.

---

# 102. Zoned snapshot

Para `ZonedDateTime` podrá conservar:

```text
instant
+
timezone ID
```

cuando ambos sean semánticamente relevantes.

---

# 103. Mutable DateTime problem

Si una propiedad contiene:

```php
DateTime
```

el objeto puede modificarse internamente:

```php
$entity->createdAt->modify('+1 day');
```

sin setter.

---

# 104. Recommended default

Favorecer:

```php
DateTimeImmutable
```

para fields persistentes.

---

# 105. Mutable temporal snapshot

Si se admite `DateTime`, el UoW deberá mantener una copia canónica independiente.

---

# 106. Schema mapping

API conceptual:

```php
$table->date('birth_date');

$table->time(
    'opens_at',
    precision: 0,
);

$table->localDateTime(
    'scheduled_at',
    precision: 6,
);

$table->instant(
    'created_at',
    precision: 6,
);
```

---

# 107. Logical Schema Type

Schema almacenará:

```text
TemporalTypeDescriptor
```

no simplemente:

```text
TIMESTAMP(6)
```

---

# 108. Schema Compiler

```text
Temporal Logical Type
        ↓
Platform Resolver
        ↓
Physical Declaration
        ↓
SQL Compiler
```

---

# 109. Schema introspection

Reverse:

```text
TIMESTAMP
DATETIME
TEXT
INTEGER
        ↓
Platform Temporal Introspector
        ↓
Logical Candidate
+
Confidence
```

---

# 110. Ambiguous introspection

Un:

```text
DATETIME
```

no prueba si la aplicación pretendía:

```text
LocalDateTime
```

o:

```text
UTC Instant
```

---

# 111. UNKNOWN semantics

La introspección deberá preservar:

```text
physical type known
logical intent unknown
```

cuando no pueda reconstruirse.

---

# 112. Schema Diff

Deberá distinguir:

```text
LocalDateTime
→ Instant
```

aunque ambos tengan la misma declaración física.

---

# 113. Why

Es un cambio semántico que puede requerir:

```text
timezone interpretation
+
data conversion
```

---

# 114. Migration example

Cambiar:

```text
local appointment time
```

a:

```text
UTC instant
```

requiere conocer la zona usada por los datos existentes.

Sin ella:

```text
migration = ambiguous
```

---

# 115. Migration Safety

No deberá permitir una conversión destructiva cuando timezone histórica sea desconocida.

---

# 116. Defaults

Distinguir:

```text
Application default
```

de:

```text
Database default
```

---

# 117. CURRENT_TIMESTAMP

Debe modelarse como expresión temporal DB-side.

No como valor PHP calculado durante schema compilation.

---

# 118. DatabaseTemporalExpression

```php
enum DatabaseTemporalExpression
{
    case CURRENT_DATE;
    case CURRENT_TIME;
    case CURRENT_TIMESTAMP;
}
```

o AST equivalente.

---

# 119. CURRENT_TIMESTAMP semantics

Cada plataforma puede tener reglas diferentes sobre:

- timezone;
- transaction start;
- statement start;
- precision.

VoltStack deberá declarar capacidades y semántica.

---

# 120. Application now

```php
$clock->now();
```

es diferente de:

```sql
CURRENT_TIMESTAMP
```

---

# 121. Clock abstraction

Para valores application-generated, VoltStack debería depender de:

```text
Clock
```

en vez de invocar:

```php
new DateTimeImmutable('now');
```

por toda la infraestructura.

---

# 122. Clock belongs outside Type System

El Type System define valores.

No administra el reloj.

---

# 123. Created/updated timestamps

Automatizaciones tipo:

```text
created_at
updated_at
```

pertenecen al lifecycle/persistence metadata.

No a `DateTimeType` directamente.

---

# 124. Optimistic locking timestamp

Utilizar timestamps como versioning requiere cuidado por:

- precision collisions;
- clock resolution;
- concurrent writes.

---

# 125. Recommended optimistic locking

Preferir:

```text
integer/version token
```

cuando se requiere garantía fuerte.

Timestamp locking podrá existir como estrategia explícita.

---

# 126. Precision collision

Dos updates dentro del mismo:

```text
second
```

pueden tener mismo timestamp si precision=0.

---

# 127. Query arithmetic

Temporal AST podrá soportar operaciones semánticas:

```text
ADD_DURATION
SUBTRACT_DURATION
DIFFERENCE
EXTRACT_DATE_PART
TRUNCATE_TEMPORAL
```

---

# 128. Query arithmetic ≠ PHP arithmetic

Las operaciones deberán compilarse según Platform.

---

# 129. Date arithmetic

```text
Date + 1 calendar day
```

debe distinguirse de:

```text
Instant + 24 hours
```

---

# 130. DST example

Durante transición DST:

```text
local tomorrow same time
```

puede diferir de:

```text
instant + 86400 seconds
```

---

# 131. Duration arithmetic

Duration opera sobre tiempo transcurrido.

---

# 132. Calendar arithmetic

Months/years requieren reglas de calendario.

Ejemplo:

```text
January 31 + 1 month
```

necesita policy.

---

# 133. CalendarPeriod future extension

V1 deberá evitar ocultar estas reglas dentro de `Duration`.

---

# 134. Temporal truncation

Operaciones como:

```text
truncate to day
```

sobre un `Instant` requieren timezone para obtener "día local".

---

# 135. Explicit timezone in query

Debe poder expresarse:

```text
dateOf(instant, timezone)
```

---

# 136. Indexing

Campos temporales podrán indexarse normalmente mediante Schema.

---

# 137. Functional temporal index

Expresiones como:

```text
DATE(created_at)
```

pueden requerir expression indexes o generated columns.

Dependerán de Platform capabilities.

---

# 138. Temporal partitions

El futuro sistema de partitioning podrá utilizar temporal metadata.

---

# 139. Query planner

Podrá evitar funciones sobre columna cuando sea posible para preservar índices.

Ejemplo:

En vez de:

```text
DATE(created_at) = 2026-09-08
```

podría reescribir semánticamente a:

```text
created_at >= day_start_instant
AND
created_at < next_day_start_instant
```

si se conoce la timezone.

---

# 140. Optimizer correctness

La reescritura solo será válida si:

- timezone es conocida;
- boundaries son correctos;
- DST se considera.

---

# 141. Date ranges

Una consulta de "día local" puede abarcar:

```text
23
24
25
```

horas reales alrededor de DST.

No asumir siempre 24 horas.

---

# 142. Platform time zone session

Algunos DBMS tienen timezone de sesión/conexión.

Eso representa riesgo en connection pooling.

---

# 143. Session timezone state

Si una plataforma usa timezone de conexión, deberá administrarse mediante:

```text
Connection State
```

de los documentos 15–16.

---

# 144. Request leakage

En workers persistentes:

```text
Request A sets timezone
Request B inherits timezone
```

debe ser imposible.

---

# 145. Preferred strategy

Evitar depender de timezone de sesión para semántica fundamental cuando pueda utilizarse una representación explícita/UTC.

---

# 146. Connection reset

Cualquier timezone session setting deberá restaurarse/verificarse antes de reutilizar conexión.

---

# 147. Database server timezone

No asumir:

```text
server timezone
```

como timezone de aplicación.

---

# 148. PHP timezone

Tampoco asumir:

```text
PHP default timezone
```

como DB timezone.

---

# 149. Application timezone

Puede existir configuración de aplicación, pero deberá ser un input explícito cuando se necesite convertir `LocalDateTime` a `Instant`.

---

# 150. Tenant timezone

Una futura aplicación multitenant puede tener timezone por tenant.

Eso no deberá almacenarse dentro de los type definitions globales.

---

# 151. Tenant timezone integration

Debe proporcionarse mediante un:

```text
TemporalConversionContext
```

scoped cuando una operación realmente dependa de ella.

---

# 152. Type identity remains global

```text
LocalDateTime
```

sigue siendo el mismo tipo lógico independientemente del tenant.

---

# 153. TemporalConversionContext

```php
final readonly class TemporalConversionContext
{
    public function __construct(
        public ?TimezoneId $timezone,
        public TemporalAmbiguityPolicy $ambiguity,
        public NonexistentLocalTimePolicy $nonexistent,
        public TemporalPrecisionLossPolicy $precision,
    ) {}
}
```

---

# 154. No hidden tenant lookup

El converter no deberá buscar automáticamente:

```text
current tenant timezone
```

desde estado global.

---

# 155. Security

Las zonas horarias provenientes de input deberán validarse contra un registry conocido.

---

# 156. No arbitrary filesystem paths

Nunca tratar una timezone input como ruta hacia archivos tzdata.

---

# 157. Timezone ID size

Aplicar límites razonables para evitar entradas patológicas.

---

# 158. Temporal parsing

Parsing externo no será permisivo por defecto.

---

# 159. Ambiguous text

Entrada:

```text
01/02/03
```

deberá rechazarse como formato DB temporal general.

---

# 160. Explicit parser

Formats legacy deberán utilizar parsers configurados explícitamente.

---

# 161. Database decoding strictness

Si una columna temporal devuelve un formato inesperado:

```text
→ TemporalDecodingException
```

no fallback silencioso.

---

# 162. Invalid dates

```text
2026-02-30
```

deberá fallar.

---

# 163. Zero dates

Algunas bases/configuraciones legacy pueden contener valores especiales equivalentes a:

```text
0000-00-00
```

VoltStack no deberá convertirlos automáticamente a `null`.

---

# 164. Legacy zero-date policy

Podrá existir:

```php
enum LegacyZeroDatePolicy
{
    case REJECT;
    case MAP_TO_NULL_EXPLICIT;
    case PRESERVE_LEGACY;
}
```

Default:

```text
REJECT
```

---

# 165. Infinity dates

Algunas plataformas ofrecen valores temporales especiales.

Deberán modelarse mediante capability/extension explícita, no confundirse con fechas normales.

---

# 166. Platform-specific temporal ranges

Cada plataforma tiene límites distintos.

El Type System deberá analizar:

```text
logical allowed range
∩
platform physical range
```

---

# 167. Out-of-range handling

No truncar ni envolver.

Producir:

```text
TemporalRangeException
```

---

# 168. Portability

Un mapping temporal es portable si todas las plataformas target pueden preservar las semánticas requeridas.

---

# 169. Portability example

```text
Date
```

suele ser altamente portable.

---

# 170. ZonedDateTime

Puede requerir representación multi-column para portabilidad completa.

---

# 171. TemporalPortabilityReport

Puede indicar:

```text
SUPPORTED
SUPPORTED_WITH_DIFFERENT_REPRESENTATION
REQUIRES_MULTICOLUMN_MAPPING
LOSSY
UNSUPPORTED
UNKNOWN
```

---

# 172. MySQL

Deberá distinguir capabilities y semánticas de:

```text
DATE
TIME
DATETIME
TIMESTAMP
```

según versión/configuración.

---

# 173. MariaDB

Será evaluado independientemente.

```text
MariaDB
≠
MySQL alias
```

---

# 174. PostgreSQL

Podrá utilizar capacidades como:

```text
DATE
TIME
TIMESTAMP WITHOUT TIME ZONE
TIMESTAMP WITH TIME ZONE
INTERVAL
```

pero la lógica VoltStack seguirá siendo independiente de esos nombres físicos.

---

# 175. PostgreSQL timestamptz

No deberá interpretarse simplemente como:

```text
ZonedDateTime preserving timezone identity
```

sin verificar su semántica física.

---

# 176. SQLite

Podrá utilizar:

```text
TEXT
INTEGER
REAL
```

según strategy.

La semántica lógica deberá conservarse mediante mapping/conversion.

---

# 177. SQLite portability

La ausencia de un tipo temporal rígido nativo no deberá forzar al framework a perder sus tipos lógicos.

---

# 178. Schema compatibility

Un físico:

```text
TEXT
```

puede representar temporal correctamente bajo una convención, pero introspection quizá no pueda demostrar la intención.

---

# 179. Metadata persistence

La aplicación deberá conservar su mapping en metadata compilada.

---

# 180. Serialization boundary

Una entidad temporal puede serializarse para API de distinta manera.

Ejemplo:

```text
Instant
→ ISO UTC API string
```

Eso pertenece al Serializer.

---

# 181. Database representation independence

Cambiar formato API no deberá cambiar automáticamente el DB mapping.

---

# 182. Locale formatting

```text
8 de septiembre de 2026
```

pertenece a presentación.

No al Database Type System.

---

# 183. Temporal Value Objects

Value Objects como:

```text
DateRange
BusinessHours
BillingPeriod
```

podrán construirse sobre los temporal types.

---

# 184. DateRange

Ejemplo:

```text
start: Date
end: Date
```

Value Object Mapping controla composición.

Temporal Type System controla cada valor.

---

# 185. Query parameters from strings

Si:

```php
->where('createdAt', '>', '2026-09-08')
```

y `createdAt` es `Instant`, VoltStack no deberá adivinar automáticamente qué instante representa esa fecha.

---

# 186. Explicit conversion

Requerir:

```text
Date + Timezone + boundary policy
```

o un `Instant` real.

---

# 187. Developer DX

Se podrán ofrecer helpers claros:

```php
Instant::parse(...)
LocalDate::parse(...)
LocalDateTime::parse(...)
TimezoneId::of(...)
```

o adapters equivalentes.

---

# 188. Avoid ambiguous helpers

No favorecer una API genérica:

```php
Date::parseAnything($string)
```

dentro del Database core.

---

# 189. Clock precision

Application clock puede producir microsegundos/nanosegundos superiores a la DB.

Debe analizarse antes de persistence.

---

# 190. Generated timestamp reconciliation

Si el DB genera `created_at`, la Entity podrá requerir:

```text
RETURNING
reload
generated value reconciliation
```

según Platform capabilities.

---

# 191. ORM EntityState

Un generated temporal value no deberá inventarse localmente si DB es authority.

---

# 192. Application authority

Si Application Clock es authority, el mismo valor podrá enviarse al DB y mantener snapshot.

---

# 193. Authority metadata

Podrá existir:

```php
enum TemporalValueAuthority
{
    case APPLICATION;
    case DATABASE;
}
```

---

# 194. Mixed authority risk

No usar simultáneamente:

```text
PHP now()
```

y:

```text
DB CURRENT_TIMESTAMP
```

esperando igualdad exacta.

---

# 195. Transaction timestamps

DB puede evaluar `CURRENT_TIMESTAMP` a nivel:

- transaction;
- statement;
- function invocation;

según plataforma.

Esto deberá ser capability metadata.

---

# 196. Retry semantics

En transaction retry, un application-generated timestamp podría:

```text
stay same
```

o:

```text
be regenerated
```

según operation semantics.

---

# 197. Temporal value generation policy

Deberá determinarse fuera del Type System.

---

# 198. Caching

Se podrán cachear:

```text
Temporal Type Descriptors
Platform resolutions
Formatters/parsers stateless
timezone registry metadata
```

---

# 199. No current-time cache

Nunca cachear:

```text
now
```

como parte de Type metadata.

---

# 200. Timezone resolution cache

Puede cachear:

```text
TimezoneId → immutable timezone rules object
```

si la runtime implementation es segura.

---

# 201. Cache invalidation

Timezone database update puede requerir invalidar resoluciones afectadas.

---

# 202. Persistent runtime

Compartible:

```text
Temporal type definitions
Temporal platform mappings
Timezone metadata
Compiled conversion plans
```

cuando immutable.

---

# 203. Scoped

```text
TemporalConversionContext
parsing buffers
diagnostics
operation-specific timezone
```

---

# 204. FrankenPHP

```text
Worker
├── immutable temporal metadata
├── immutable timezone registry
│
├── Request A temporal context
└── Request B temporal context
```

---

# 205. RoadRunner

Mismo principio.

---

# 206. OpenSwoole

Contextos temporales variables deberán ser coroutine-safe.

---

# 207. No static current timezone

Nunca:

```php
TemporalContext::$currentTimezone
```

como mutable global request state.

---

# 208. Telemetry

Métricas potenciales:

```text
database.temporal.convert.total
database.temporal.convert.failure
database.temporal.precision_loss
database.temporal.ambiguous_local_time
database.temporal.nonexistent_local_time
database.temporal.timezone_resolution_failure
database.temporal.range_failure
database.temporal.legacy_zero_date
database.temporal.platform_emulation
```

---

# 209. Metric cardinality

Puede incluir:

```text
temporal_kind
direction
platform
failure kind
```

Evitar timezone IDs arbitrarias como labels de alta cardinalidad salvo allowlist controlada.

---

# 210. No raw temporal user data

Telemetry no deberá registrar valores sensibles completos por defecto.

---

# 211. Diagnostics

API conceptual:

```php
Database::types()
    ->temporal()
    ->explain(Event::class, 'occurredAt');
```

---

# 212. Diagnostic example

```text
TEMPORAL MAPPING

Entity:
    Event

Property:
    occurredAt

Logical Type:
    INSTANT

PHP Type:
    DateTimeImmutable

Precision:
    6

Storage Strategy:
    UTC_NORMALIZED

Platform:
    PostgreSQL

Physical Type:
    platform resolved timestamp type

Timezone Identity Preserved:
    no

Offset Preserved:
    no

Instant Preserved:
    yes

Precision:
    lossless

Ambiguous Local Time Policy:
    not applicable
```

---

# 213. Zoned diagnostic

```text
TEMPORAL MAPPING

Property:
    scheduledFor

Logical Type:
    ZONED_DATETIME

Physical Mapping:
    scheduled_at
        → UTC instant

    scheduled_timezone
        → IANA timezone ID

Zone Identity:
    preserved

DST Resolution:
    explicit

Portability:
    FULL_WITH_MULTICOLUMN_MAPPING
```

---

# 214. Error hierarchy

```text
DatabaseTemporalException
├── TemporalTypeException
├── TemporalParsingException
├── TemporalDecodingException
├── TemporalEncodingException
├── TemporalRangeException
├── TemporalPrecisionException
├── TemporalPrecisionLossException
├── TemporalTimezoneException
├── UnknownTimezoneException
├── TemporalOffsetException
├── AmbiguousLocalTimeException
├── NonexistentLocalTimeException
├── TemporalLeapSecondException
├── TemporalComparisonException
├── TemporalArithmeticException
├── TemporalPlatformMappingException
├── TemporalSchemaCompatibilityException
├── TemporalMigrationException
├── TemporalLegacyValueException
├── TemporalRuntimeStateException
└── TemporalInvariantViolationException
```

---

# 215. Testing architecture

Debe existir:

```text
TemporalTypeConformanceSuite
```

---

# 216. Date tests

Probar:

- leap years;
- month lengths;
- min/max ranges;
- invalid dates.

---

# 217. Time tests

Probar:

- `00:00:00`;
- `23:59:59`;
- fractional precision;
- invalid hours/minutes/seconds.

---

# 218. LocalDateTime tests

Probar construcción y round-trip sin introducir timezone.

---

# 219. Instant tests

Probar equivalencia entre offsets:

```text
12:00-06:00
=
18:00Z
```

como instant.

---

# 220. DST tests

Probar zonas que tengan:

```text
ambiguous local times
nonexistent local times
```

---

# 221. Timezone tests

Incluir:

- valid IANA ID;
- invalid ID;
- offset transitions;
- historical rules.

---

# 222. Precision tests

Para:

```text
0
3
6
```

fractional digits y límites del platform.

---

# 223. Loss tests

Intentar:

```text
precision 6
→ precision 3
```

con cada policy.

---

# 224. Null tests

Separar:

```text
SQL NULL
```

de valores temporales reales.

---

# 225. PHP mutable tests

Cuando se soporte `DateTime`, verificar dirty detection tras mutación interna.

---

# 226. Immutable tests

Verificar no-op replacement por valor equivalente.

---

# 227. Query tests

Probar:

- equality;
- range;
- ordering;
- date extraction;
- timezone-aware boundaries.

---

# 228. Day range DST test

Verificar que "día local" no se traduzca siempre a 86,400 segundos.

---

# 229. Schema tests

Por plataforma:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

incluyendo:

- create;
- introspection;
- diff;
- precision;
- defaults;
- migration.

---

# 230. Round-trip conformance

Para mappings lossless:

```text
TemporalValue
→ DB
→ TemporalValue
```

debe preservar la semántica correspondiente.

---

# 231. Instant round-trip

Debe preservar:

```text
instant
```

aunque cambie textual representation.

---

# 232. Zoned round-trip

Si mapping promete zone-preserving:

```text
timezone ID
```

también deberá conservarse.

---

# 233. Portability tests

Mismo logical mapping deberá validarse sobre todas las plataformas target.

---

# 234. Persistent runtime tests

Request A timezone context no deberá contaminar Request B.

---

# 235. Connection reuse tests

Si session timezone se utiliza:

```text
set
→ execute
→ reset
→ reuse
```

deberá probarse.

---

# 236. Performance benchmarks

Medir:

```text
date parse/sec
instant conversion/sec
timezone resolution/sec
DST resolution/sec
temporal hydration/sec
parameter conversion/sec
snapshot comparison/sec
```

---

# 237. Hot-path optimization

Evitar:

- recrear timezone definitions;
- reflection;
- parsing de type metadata;
- repeated platform capability checks;
- string reformatting innecesario.

---

# 238. Directory structure

```text
src/Quantum/Database/Type/Temporal/
│
├── Contract/
│   ├── TemporalValue.php
│   ├── TemporalConverter.php
│   ├── TemporalPlatformResolver.php
│   ├── TimezoneResolver.php
│   ├── TemporalComparator.php
│   └── TemporalPortabilityAnalyzer.php
│
├── Type/
│   ├── TemporalKind.php
│   ├── TemporalTypeDescriptor.php
│   ├── TemporalPrecision.php
│   ├── TemporalStorageStrategy.php
│   └── TemporalValueAuthority.php
│
├── Value/
│   ├── LocalDate.php
│   ├── LocalTime.php
│   ├── LocalDateTime.php
│   ├── Instant.php
│   ├── OffsetDateTime.php
│   ├── ZonedDateTime.php
│   └── Duration.php
│
├── Timezone/
│   ├── TimezoneId.php
│   ├── TimezoneOffset.php
│   ├── TimezoneRegistry.php
│   ├── TimezoneDatabaseVersion.php
│   └── IanaTimezoneResolver.php
│
├── Policy/
│   ├── TemporalAmbiguityPolicy.php
│   ├── NonexistentLocalTimePolicy.php
│   ├── TemporalPrecisionLossPolicy.php
│   ├── LegacyZeroDatePolicy.php
│   └── LeapSecondPolicy.php
│
├── Context/
│   └── TemporalConversionContext.php
│
├── Conversion/
│   ├── DateConverter.php
│   ├── TimeConverter.php
│   ├── LocalDateTimeConverter.php
│   ├── InstantConverter.php
│   ├── OffsetDateTimeConverter.php
│   ├── ZonedDateTimeConverter.php
│   └── DurationConverter.php
│
├── Platform/
│   ├── TemporalPhysicalTypeResolver.php
│   ├── TemporalPlatformCapabilities.php
│   ├── MySQL/
│   ├── MariaDB/
│   ├── PostgreSQL/
│   └── SQLite/
│
├── Query/
│   ├── TemporalExpression.php
│   ├── TemporalArithmeticOperation.php
│   └── TemporalBoundaryResolver.php
│
├── Schema/
│   ├── TemporalSchemaProjector.php
│   └── TemporalSchemaCompatibilityAnalyzer.php
│
├── Comparison/
│   ├── TemporalEqualityMode.php
│   └── CanonicalTemporalComparator.php
│
├── Diagnostics/
│   ├── TemporalMappingExplainer.php
│   └── TemporalPortabilityReport.php
│
└── Exception/
    ├── DatabaseTemporalException.php
    ├── TemporalParsingException.php
    ├── TemporalDecodingException.php
    ├── TemporalRangeException.php
    ├── TemporalPrecisionException.php
    ├── TemporalPrecisionLossException.php
    ├── TemporalTimezoneException.php
    ├── UnknownTimezoneException.php
    ├── TemporalOffsetException.php
    ├── AmbiguousLocalTimeException.php
    ├── NonexistentLocalTimeException.php
    ├── TemporalLeapSecondException.php
    ├── TemporalPlatformMappingException.php
    ├── TemporalSchemaCompatibilityException.php
    └── TemporalInvariantViolationException.php
```

---

# 239. Dependency model

Permitido:

```text
Temporal Type System
        ↓
Database Type System

Temporal Type System
        ↓
Value Conversion

Temporal Type System
        ↓
Platform Capabilities

Temporal Type System
        ↓
Timezone Resolver contracts
```

---

# 240. ORM integration

```text
ORM
 ↓
Temporal Type System
```

No:

```text
Temporal Type System
 ↓
EntityManager
```

---

# 241. Query integration

```text
Query Semantic Engine
        ↓
Temporal Semantics
        ↓
Temporal Query AST
        ↓
Planner
        ↓
Compiler
```

---

# 242. Schema integration

```text
Schema
 ↓
Temporal Logical Type
 ↓
Platform Resolver
 ↓
Physical Type
```

---

# 243. Architectural invariants

## DB-TEMP-001
Date será distinto de Time.

## DB-TEMP-002
Date será distinto de LocalDateTime.

## DB-TEMP-003
LocalDateTime será distinto de Instant.

## DB-TEMP-004
OffsetDateTime será distinto de ZonedDateTime.

## DB-TEMP-005
Timezone será distinta de UTC offset.

## DB-TEMP-006
Duration será distinta de Time.

## DB-TEMP-007
Duration será distinta de CalendarPeriod.

## DB-TEMP-008
SQL TIMESTAMP no definirá por sí mismo el logical temporal type.

## DB-TEMP-009
PHP DateTimeImmutable no definirá por sí mismo semántica persistente.

## DB-TEMP-010
Logical temporal type precederá a physical type resolution.

## DB-TEMP-011
Timezone conversion será explícita.

## DB-TEMP-012
PHP default timezone no decidirá DB semantics.

## DB-TEMP-013
DB server timezone no decidirá application semantics.

## DB-TEMP-014
Session timezone será estado explícitamente administrado.

## DB-TEMP-015
Connection pooling no filtrará timezone state.

## DB-TEMP-016
Instant podrá normalizarse a UTC.

## DB-TEMP-017
UTC normalization no implicará zone preservation.

## DB-TEMP-018
Zone-preserving mapping deberá conservar timezone identity.

## DB-TEMP-019
Offset preservation no equivaldrá a zone preservation.

## DB-TEMP-020
IANA IDs serán preferidos para zonas.

## DB-TEMP-021
Timezone abbreviations no serán identidad canónica.

## DB-TEMP-022
DST ambiguity será detectada.

## DB-TEMP-023
Ambiguous local times no se resolverán silenciosamente.

## DB-TEMP-024
Nonexistent local times no se ajustarán silenciosamente.

## DB-TEMP-025
Ambiguity policy será explícita.

## DB-TEMP-026
Nonexistent-time policy será explícita.

## DB-TEMP-027
Offset/zone inconsistencies serán rechazables.

## DB-TEMP-028
Leap-second behavior será explícito.

## DB-TEMP-029
Unrepresentable leap seconds no serán inventados.

## DB-TEMP-030
Temporal precision será metadata explícita.

## DB-TEMP-031
Precision loss no será silenciosa.

## DB-TEMP-032
Temporal range será validado.

## DB-TEMP-033
Platform range limits serán respetados.

## DB-TEMP-034
Epoch units serán explícitas.

## DB-TEMP-035
Epoch unit no será inferida por magnitud.

## DB-TEMP-036
Textual representation será locale-independent.

## DB-TEMP-037
Ambiguous date formats serán rechazados por default.

## DB-TEMP-038
Value Conversion será distinta de Casting.

## DB-TEMP-039
Temporal formatting será distinto de DB conversion.

## DB-TEMP-040
Hydrator no adivinará timezone.

## DB-TEMP-041
Temporal converters no dependerán de EntityManager.

## DB-TEMP-042
Temporal converters no ejecutarán queries.

## DB-TEMP-043
Temporal Type System no abrirá conexiones.

## DB-TEMP-044
Query comparisons serán type-aware.

## DB-TEMP-045
Date y Instant no serán comparables implícitamente.

## DB-TEMP-046
Instant equality será timeline-based.

## DB-TEMP-047
LocalDateTime equality será local-value based.

## DB-TEMP-048
Zoned equality policy será explícita cuando importe zone identity.

## DB-TEMP-049
Dirty tracking utilizará semantic temporal equality.

## DB-TEMP-050
Raw temporal strings no serán dirty identity por default.

## DB-TEMP-051
Immutable temporal values serán preferidos.

## DB-TEMP-052
Mutable DateTime requerirá snapshot independiente.

## DB-TEMP-053
Schema almacenará logical temporal descriptors.

## DB-TEMP-054
Schema Compiler resolverá physical temporal types.

## DB-TEMP-055
Introspection no inventará logical intent.

## DB-TEMP-056
DATETIME físico no implicará LocalDateTime automáticamente.

## DB-TEMP-057
Temporal semantic type changes serán migration-sensitive.

## DB-TEMP-058
LocalDateTime→Instant requerirá timezone evidence.

## DB-TEMP-059
Unknown historical timezone hará la migración uncertain.

## DB-TEMP-060
Database temporal expressions serán distintas de PHP values.

## DB-TEMP-061
CURRENT_TIMESTAMP no será calculado durante schema compilation.

## DB-TEMP-062
Application clock y DB clock serán authorities distintas.

## DB-TEMP-063
Created/updated timestamp lifecycle no pertenecerá al temporal type core.

## DB-TEMP-064
Timestamp optimistic locking reconocerá precision limitations.

## DB-TEMP-065
Temporal arithmetic será semantic AST.

## DB-TEMP-066
Date + calendar day será distinto de Instant + 24h.

## DB-TEMP-067
Local-day query no asumirá 86,400 segundos.

## DB-TEMP-068
Temporal optimizer rewrites requerirán timezone certainty.

## DB-TEMP-069
Session timezone reset será obligatorio cuando se modifique.

## DB-TEMP-070
Tenant timezone no será global type metadata.

## DB-TEMP-071
Tenant temporal context será scoped.

## DB-TEMP-072
Timezone IDs de input serán validadas.

## DB-TEMP-073
Legacy zero dates no serán null silenciosamente.

## DB-TEMP-074
Legacy zero-date policy será explícita.

## DB-TEMP-075
Platform special temporal values serán capabilities/extensions.

## DB-TEMP-076
Portability no significará identical physical type.

## DB-TEMP-077
PostgreSQL timestamptz no será asumido ZonedDateTime preserving zone ID.

## DB-TEMP-078
MySQL y MariaDB tendrán temporal resolvers independientes.

## DB-TEMP-079
SQLite storage representation no cambiará logical semantics.

## DB-TEMP-080
Serialization permanecerá separada.

## DB-TEMP-081
Locale formatting permanecerá separada.

## DB-TEMP-082
Temporal Value Objects reutilizarán el Temporal Type System.

## DB-TEMP-083
Strings ambiguos no se convertirán automáticamente a Instant.

## DB-TEMP-084
Query parameters conservarán expected Temporal TypeReference.

## DB-TEMP-085
Raw SQL temporal parameters requerirán type context cuando sea ambiguo.

## DB-TEMP-086
Clock no será parte del Type System.

## DB-TEMP-087
Current-time values no serán cacheados como metadata.

## DB-TEMP-088
Timezone metadata immutable podrá compartirse.

## DB-TEMP-089
Operation timezone mutable será scoped.

## DB-TEMP-090
FrankenPHP requests no compartirán mutable temporal context.

## DB-TEMP-091
RoadRunner requests no compartirán mutable temporal context.

## DB-TEMP-092
OpenSwoole temporal context será coroutine-safe.

## DB-TEMP-093
No habrá static mutable current timezone.

## DB-TEMP-094
Telemetry no expondrá raw temporal data innecesariamente.

## DB-TEMP-095
Hot path no hará repeated timezone discovery.

## DB-TEMP-096
Hot path no hará repeated platform capability discovery.

## DB-TEMP-097
Round-trip tests serán obligatorios para mappings lossless.

## DB-TEMP-098
DST tests serán parte del conformance suite.

## DB-TEMP-099
Precision boundaries serán parte del conformance suite.

## DB-TEMP-100
Cross-platform tests serán obligatorios.

## DB-TEMP-101
DateTime mutable dirty tracking tendrá tests.

## DB-TEMP-102
Connection timezone reset tendrá tests.

## DB-TEMP-103
Timezone database changes podrán invalidar caches dependientes.

## DB-TEMP-104
Temporal Type definitions serán immutable.

## DB-TEMP-105
Compiled temporal plans serán cacheables.

## DB-TEMP-106
Temporal mapping será deterministic.

## DB-TEMP-107
Calendar-period arithmetic no se fingirá como fixed duration.

## DB-TEMP-108
Generated temporal DB values serán reconciliados según DB authority.

## DB-TEMP-109
Application authority values conservarán su exact canonical snapshot.

## DB-TEMP-110
Mixed DB/application clock authority no asumirá equality exacta.

## DB-TEMP-111
Timezone-aware indexing será capability-driven.

## DB-TEMP-112
Functional temporal indexes pertenecerán al Schema System.

## DB-TEMP-113
Temporal Type System no será Scheduler.

## DB-TEMP-114
Temporal Type System no será Calendar UI.

## DB-TEMP-115
Temporal Type System no será Localization System.

## DB-TEMP-116
Temporal Type System no será Transaction Manager.

## DB-TEMP-117
Temporal Type System no generará SQL.

## DB-TEMP-118
Temporal Type System no realizará authorization.

## DB-TEMP-119
Correctness temporal tendrá prioridad sobre convenience parsing.

## DB-TEMP-120
VoltStack nunca convertirá entre tiempo local e instante sin suficiente información temporal explícita.

---

# 244. Anti-patterns

## 244.1 Todo es DateTime

```php
private DateTime $date;
private DateTime $time;
private DateTime $createdAt;
```

sin metadata semántica.

**Rechazado.**

---

## 244.2 Guardar todo con timezone del servidor

**Rechazado.**

---

## 244.3 Convertir siempre a UTC

Para cualquier temporal sin considerar si representa un horario local.

**Rechazado.**

UTC es apropiado para `Instant`, no universalmente para `LocalDateTime`.

---

## 244.4 Guardar timezone como CST

**Rechazado como identidad canónica.**

---

## 244.5 Asumir que offset es timezone

```text
-06:00 = America/Monterrey
```

**Rechazado.**

---

## 244.6 Truncar microsegundos silenciosamente

**Rechazado.**

---

## 244.7 Usar PHP default timezone en Hydrator

**Rechazado.**

---

## 244.8 `strtotime()` como parser universal

**Rechazado para Database core.**

---

## 244.9 Date = midnight Instant

**Rechazado.**

---

## 244.10 Calendar day = 24 hours

**Rechazado.**

---

## 244.11 DateTime mutable sin snapshot

**Rechazado.**

---

# 245. Ejemplo — Instant

Entidad:

```php
final class AuditLog
{
    #[Column(
        type: 'instant',
        precision: 6,
    )]
    private DateTimeImmutable $createdAt;
}
```

Application:

```text
2026-09-08T12:30:00.123456-06:00
```

Canonical Instant:

```text
2026-09-08T18:30:00.123456Z
```

DB persistence:

```text
Platform-specific UTC representation
```

---

# 246. Ejemplo — Local meeting

```php
final class Meeting
{
    #[Column(
        type: 'local_datetime',
        precision: 0,
    )]
    private DateTimeImmutable $scheduledLocalTime;
}
```

Valor:

```text
2026-10-15T09:00:00
```

No se convierte a UTC por sí mismo.

---

# 247. Ejemplo — Zoned meeting

Value Object conceptual:

```text
ZonedDateTime
├── local: 2026-10-15 09:00
├── timezone: America/Monterrey
└── instant: resolved
```

Mapping portable:

```text
scheduled_at_utc
scheduled_timezone
```

---

# 248. Ejemplo — Same Instant

```text
A =
2026-09-08T12:30:00-06:00

B =
2026-09-08T18:30:00Z
```

Para tipo:

```text
INSTANT
```

resultado:

```text
Equivalent(A, B) = true
```

Para representation-sensitive `OffsetDateTime`:

```text
RepresentationEqual(A, B) = false
```

aunque compartan instante.

---

# 249. Ejemplo — DST ambiguity

Input conceptual:

```text
LocalDateTime:
    2026-11-01 01:30

Timezone:
    America/New_York
```

Si existen dos offsets válidos:

```text
TemporalAmbiguityPolicy::REJECT
```

produce:

```text
AmbiguousLocalTimeException
```

en vez de seleccionar uno arbitrariamente.

---

# 250. Ejemplo — Precision

Application:

```text
2026-09-08T18:30:00.123456Z
```

Type:

```text
instant(precision=6)
```

Platform:

```text
precision=3
```

Assessment:

```text
LOSSY
```

Default:

```text
TemporalPrecisionLossException
```

---

# 251. Ejemplo — Query por día local

Solicitud semántica:

```text
Events occurring on
2026-09-08
in America/Monterrey
```

Resolver:

```text
Local day start
    ↓ timezone rules
Instant start

Next local day start
    ↓ timezone rules
Instant end
```

Query semantic:

```text
occurred_at >= startInstant
AND
occurred_at < endInstant
```

No:

```text
occurred_at between midnight and midnight + 86400 seconds
```

---

# 252. Master temporal model

```text
                        TEMPORAL DOMAIN
                               │
      ┌────────┬────────┬──────┼───────┬─────────────┐
      │        │        │      │       │             │
     Date     Time   LocalDT  Instant OffsetDT     ZonedDT
                               │
                               ▼
                      Canonical Semantics
                               │
                               ▼
                        TypeReference
                               │
                               ▼
                       Value Conversion
                               │
                               ▼
                    Platform Type Resolver
                               │
                               ▼
                  Physical DB Representation
```

---

# 253. Master Instant formula

Para local datetime `L` en zona `Z`:

```text
Instant
=
Resolve(
    L,
    Z,
    AmbiguityPolicy,
    NonexistentTimePolicy
)
```

Nunca:

```text
Instant = LocalDateTime
```

directamente.

---

# 254. Master equality formula

```text
Date:
    compare calendar fields

LocalDateTime:
    compare local calendar/time fields

Instant:
    compare global timeline position

OffsetDateTime:
    equality policy may preserve offset representation

ZonedDateTime:
    equality policy may include timezone identity
```

---

# 255. Master persistence formula

```text
PhysicalTemporalValue
=
PlatformConvert(
    Canonicalize(
        ApplicationTemporalValue,
        TemporalTypeDescriptor,
        TemporalConversionContext
    )
)
```

sujeto a:

```text
ValidTemporalValue
∧
TimezoneSemanticsSatisfied
∧
PrecisionSatisfied
∧
RangeSupported
∧
PlatformRepresentable
```

---

# 256. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Explicit Temporal Semantics
```

en lugar de un único `DateTimeType`.

Adoptará:

```text
Date
Time
LocalDateTime
Instant
OffsetDateTime
ZonedDateTime
Duration
```

como conceptos separados.

Adoptará:

```text
UTC normalization
```

como estrategia recomendada para instantes cuando no sea necesaria la zona original.

Adoptará:

```text
Instant + IANA Timezone
```

cuando deba preservarse zone identity.

Adoptará:

```text
Explicit DST Resolution
```

en vez de ajustes automáticos.

Adoptará:

```text
Explicit Precision
```

y detección de pérdida.

Adoptará:

```text
Immutable Temporal Values
```

como recomendación ORM.

Adoptará:

```text
Platform Capability Resolution
```

para la representación física.

---

# 257. Regla maestra final

> **VoltStack distinguirá entre "qué hora muestra el reloj", "en qué zona ocurre", "qué offset tenía esa zona" y "qué instante representa realmente". Ninguna de esas respuestas será inferida accidentalmente desde un string SQL, la timezone global de PHP o la configuración de sesión del servidor de base de datos.**

Esto permitirá que:

```text
2026-09-08 12:30
```

pueda ser correctamente modelado como:

```text
LocalDateTime
```

sin convertirlo en un instante inexistente, mientras:

```text
2026-09-08T18:30:00Z
```

se mantendrá como:

```text
Instant
```

independientemente del lugar desde donde la aplicación lo observe.

La arquitectura preservará:

```text
Temporal Domain Semantics
≠
PHP Runtime Representation
≠
SQL Temporal Type
≠
Database Session Timezone
≠
Presentation Format
```

---

# 258. Siguiente documento

```text
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 14 — Types, Casting & Value Objects** definiendo cómo terceros, paquetes oficiales y aplicaciones podrán ampliar el Type System sin modificar el core.

Deberá cubrir:

- custom logical types;
- custom TypeIds;
- namespacing;
- type registration;
- custom descriptors;
- custom TypeBehavior;
- custom Value Conversion;
- custom Casting;
- custom Query type semantics;
- custom Schema mappings;
- platform-specific mappings;
- custom physical types;
- custom parameter binding;
- custom hydration;
- custom query operators;
- custom compatibility rules;
- type dependencies;
- extension packages;
- conflict resolution;
- override policies;
- capabilities;
- registry generations;
- metadata cache invalidation;
- compiled extension plans;
- security boundaries;
- plugin isolation;
- persistent runtime safety;
- diagnostics;
- telemetry;
- testing;
- conformance suites;
- backward compatibility;
- versioning;
- architectural invariants.

Regla central propuesta:

> **Un Custom Type de VoltStack ampliará el modelo lógico mediante contratos registrados y verificables, no mediante excepciones dispersas, `if` por vendor, reflection en runtime o acceso directo del tipo al Driver; toda extensión deberá integrarse a las mismas fronteras de Type Registry, Conversion, Platform, Schema, Query y Hydration utilizadas por los tipos core.**