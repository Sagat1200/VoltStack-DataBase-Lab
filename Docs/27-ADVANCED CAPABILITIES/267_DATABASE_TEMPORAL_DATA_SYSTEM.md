# 267_DATABASE_TEMPORAL_DATA_SYSTEM.md

# VoltStack Quantum Database
## Temporal Data System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 267 — Temporal Data System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura temporal de VoltStack Database.

El objetivo es proporcionar primitivas consistentes para representar, consultar, comparar, persistir y transformar información cuyo significado depende del tiempo.

El sistema deberá soportar correctamente conceptos como:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
OffsetDateTime
Duration
Period
TemporalInterval
TemporalRange
ValidTime
TransactionTime
SystemTime
BusinessTime
AsOf
TemporalSnapshot
TemporalPredicate
TemporalPrecision
```

sin reducir todos ellos a:

```php
DateTime
```

o:

```text
TIMESTAMP
```

---

# 2. Regla arquitectónica central

> **El sistema temporal de VoltStack deberá representar explícitamente qué significa el tiempo dentro del dominio y de la persistencia; almacenar un timestamp no convierte automáticamente una entidad, tabla o query en un modelo temporal.**

Por tanto:

```text
Temporal Data
≠
DateTime Casting
≠
Entity History
≠
Audit Log
≠
Soft Delete
≠
Transaction Timestamp
```

y:

```text
Instant
≠
LocalDateTime
≠
ZonedDateTime
```

---

# 3. Problema fundamental

Una columna como:

```text
2026-09-19 14:30:00
```

no responde por sí sola:

```text
¿es UTC?
¿es hora local?
¿qué zona horaria representa?
¿qué precisión tiene?
¿representa un evento real?
¿representa vigencia de negocio?
¿representa momento de persistencia?
¿representa creación?
¿representa modificación?
```

Por eso VoltStack no tratará todos los valores temporales como equivalentes.

---

# 4. Temporal semantics first

El principio será:

```text
Meaning
    ↓
Temporal Type
    ↓
Canonical Representation
    ↓
Platform Mapping
    ↓
Database Physical Type
```

Nunca:

```text
Database TIMESTAMP
    ↓
guess semantics
```

---

# 5. Relación con el Type System

Este sistema extiende:

```text
155_DATABASE_TYPE_SYSTEM
157_DATABASE_VALUE_CONVERSION_SYSTEM
158_DATABASE_CASTING_SYSTEM
162_DATABASE_DATE_TIME_TYPE_SYSTEM
```

`162_DATABASE_DATE_TIME_TYPE_SYSTEM.md` define tipos básicos de fecha/hora.

El presente documento añade:

```text
temporal semantics
intervals
ranges
validity
temporal predicates
temporal query context
as-of semantics
temporal planning
temporal platform capabilities
```

---

# 6. Separación de responsabilidades

```text
Date/Time Type System
        │
        ▼
Canonical Temporal Values
        │
        ▼
Temporal Data System
        │
        ├── semantics
        ├── ranges
        ├── intervals
        ├── predicates
        ├── contexts
        └── queries
        │
        ▼
Query Engine
        │
        ▼
Compiler
        │
        ▼
Platform
```

---

# 7. Temporal Data System no genera SQL

Regla:

```text
Temporal System
→ Query AST
→ Compiler
→ SQL
```

Nunca:

```text
Temporal System
→ SQL string
```

---

# 8. Conceptos temporales fundamentales

VoltStack distinguirá al menos:

```text
Instant
LocalDate
LocalTime
LocalDateTime
OffsetDateTime
ZonedDateTime
Duration
Period
TemporalInterval
TemporalRange
TemporalPoint
TemporalPrecision
```

---

# 9. Instant

Representa un punto inequívoco en la línea temporal global.

Ejemplo conceptual:

```text
2026-09-19T20:30:00Z
```

Dos instants pueden compararse universalmente.

---

# 10. Instant invariant

```text
Instant A == Instant B
```

si ambos representan exactamente el mismo punto temporal, independientemente de cómo fueron presentados originalmente.

---

# 11. LocalDate

Representa:

```text
YYYY-MM-DD
```

sin:

```text
hora
offset
zona horaria
```

Ejemplo:

```text
2026-09-19
```

---

# 12. LocalTime

Representa una hora del día:

```text
14:30:00
```

sin fecha ni zona.

---

# 13. LocalDateTime

Representa:

```text
date + local clock time
```

pero no identifica por sí solo un instante global.

Ejemplo:

```text
2026-09-19 14:30
```

puede ocurrir en:

```text
Mexico City
New York
Madrid
Tokyo
```

en instantes diferentes.

---

# 14. Critical invariant

```text
LocalDateTime
≠
Instant
```

---

# 15. OffsetDateTime

Representa:

```text
LocalDateTime
+
UTC offset
```

Ejemplo:

```text
2026-09-19T14:30:00-06:00
```

---

# 16. ZonedDateTime

Representa:

```text
LocalDateTime
+
TimeZoneId
+
resolved offset
```

Ejemplo:

```text
2026-09-19T14:30:00
America/Mexico_City
```

---

# 17. Offset ≠ Time Zone

```text
UTC-06:00
```

no es equivalente a:

```text
America/Mexico_City
```

porque una zona horaria contiene reglas históricas y potencialmente futuras.

---

# 18. TimeZoneId

VoltStack utilizará identificadores canónicos de zona.

Conceptualmente:

```php
final readonly class TimeZoneId
{
    public function __construct(
        public string $value
    ) {}
}
```

---

# 19. No timezone abbreviations by default

Valores ambiguos como:

```text
CST
EST
IST
```

no deberán utilizarse como identificadores canónicos.

---

# 20. Duration

Representa cantidad exacta de tiempo.

Ejemplo:

```text
3600 seconds
```

---

# 21. Period

Representa unidades calendáricas.

Ejemplo:

```text
1 month
3 days
2 years
```

---

# 22. Duration ≠ Period

```text
1 month
≠
30 days
```

universalmente.

---

# 23. Calendar arithmetic

Ejemplo:

```text
2026-01-31 + 1 month
```

requiere política calendárica.

No deberá convertirse automáticamente en:

```text
+ 30 days
```

---

# 24. TemporalPoint

Abstracción para un punto temporal compatible con determinada semántica.

No todos los tipos temporales serán intercambiables.

---

# 25. TemporalPrecision

VoltStack representará precisión explícitamente.

```php
enum TemporalPrecision
{
    case DATE;
    case SECOND;
    case MILLISECOND;
    case MICROSECOND;
    case NANOSECOND;
}
```

La lista física efectiva dependerá de capabilities.

---

# 26. Precision requested ≠ precision stored

Ejemplo:

```text
Application: microseconds
Database: milliseconds
```

deberá detectarse como:

```text
precision loss
```

---

# 27. Precision loss

No deberá ocurrir silenciosamente cuando pueda afectar semántica.

---

# 28. TemporalNormalization

Valores equivalentes deberán normalizarse antes de:

```text
comparison
binding
cache key generation
cursor generation
change tracking
```

cuando corresponda.

---

# 29. Canonical instant representation

Internamente, los instants deberán poseer representación canónica independiente del vendor.

---

# 30. Database physical type

Puede ser:

```text
TIMESTAMP
TIMESTAMP WITH TIME ZONE
DATETIME
INTEGER epoch
TEXT ISO-8601
```

según plataforma y mapping.

Pero:

```text
Physical Type
≠
Temporal Semantics
```

---

# 31. Platform mapping

```text
Temporal Logical Type
        ↓
Platform Capability
        ↓
Physical Mapping
```

---

# 32. Capability-driven temporal mapping

Ejemplos de capabilities:

```text
supportsTimeZoneAwareTimestamp
supportsMicrosecondPrecision
supportsNativeDate
supportsNativeTime
supportsTemporalRange
supportsSystemVersionedTables
supportsIntervalType
supportsTemporalOperators
```

---

# 33. Version ≠ capability

No se deberán codificar reglas del tipo:

```php
if ($database === 'postgres') {
    // temporal behavior
}
```

en el Temporal Engine.

Se consultará:

```text
PlatformCapabilities
```

---

# 34. MySQL y MariaDB

Continuarán siendo plataformas separadas.

Aunque compartan sintaxis, sus capabilities no deberán asumirse idénticas.

---

# 35. SQLite

SQLite puede requerir emulación de ciertas semánticas temporales.

La emulación deberá declararse.

---

# 36. Temporal support status

```php
enum TemporalCapabilityStatus
{
    case NATIVE;
    case EMULATED;
    case LIMITED;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 37. UNKNOWN ≠ supported

---

# 38. TemporalInterval

Representa un intervalo entre dos límites temporales.

Conceptualmente:

```php
final readonly class TemporalInterval
{
    public function __construct(
        public ?TemporalPoint $start,
        public ?TemporalPoint $end,
        public BoundaryType $startBoundary,
        public BoundaryType $endBoundary,
    ) {}
}
```

---

# 39. Boundary types

```text
OPEN
CLOSED
UNBOUNDED
```

---

# 40. Mathematical notation

Ejemplo:

```text
[start, end)
```

significa:

```text
start inclusive
end exclusive
```

---

# 41. Recommended interval convention

Para muchos casos temporales se preferirá:

```text
[start, end)
```

porque facilita intervalos adyacentes sin solapamiento.

---

# 42. Example

```text
A = [10:00, 11:00)
B = [11:00, 12:00)
```

Entonces:

```text
A ∩ B = ∅
```

---

# 43. Open-ended interval

Ejemplo:

```text
[start, +∞)
```

representa vigencia actual sin fecha final conocida.

---

# 44. Infinity ≠ NULL

Conceptualmente:

```text
unbounded
```

no debe confundirse automáticamente con:

```text
unknown
```

---

# 45. Database NULL semantics

Si una implementación usa:

```text
valid_to = NULL
```

para representar infinito, el mapping deberá conservar la semántica.

---

# 46. Unknown end ≠ unbounded end

El dominio puede necesitar distinguir:

```text
no end
```

de:

```text
end unknown
```

VoltStack no los fusionará obligatoriamente.

---

# 47. TemporalRange

Representa un conjunto continuo de valores temporales.

Puede mapearse a tipos nativos donde existan.

---

# 48. Native range ≠ required architecture

La API temporal no dependerá de que el motor posea un tipo RANGE nativo.

---

# 49. Temporal predicates

El Query Engine deberá poder expresar:

```text
before
after
at
between
contains
containedBy
overlaps
startsBefore
startsAfter
endsBefore
endsAfter
isCurrent
asOf
```

---

# 50. Predicates produce AST

Ejemplo conceptual:

```php
$query->whereTemporal(
    'validity',
    Temporal::overlaps($range)
);
```

produce:

```text
TemporalPredicateNode
```

no SQL.

---

# 51. Temporal AST

Jerarquía conceptual:

```text
ExpressionNode
    │
    └── TemporalExpressionNode
            │
            ├── TemporalPointNode
            ├── TemporalRangeNode
            ├── TemporalIntervalNode
            └── TemporalOperationNode

PredicateNode
    │
    └── TemporalPredicateNode
            ├── BeforePredicate
            ├── AfterPredicate
            ├── OverlapsPredicate
            ├── ContainsPredicate
            └── AsOfPredicate
```

---

# 52. Compiler responsibility

El Compiler decide si:

```text
overlaps
```

se representa como:

```text
native range operator
```

o:

```text
start/end comparisons
```

---

# 53. Semantic equivalence required

Una emulación sólo será válida si conserva la semántica definida.

---

# 54. Temporal query

Una Temporal Query es una query cuyo resultado depende explícitamente de una dimensión temporal.

---

# 55. Temporal query ≠ query with date filter

Esto:

```php
->where('created_at', '>', $date)
```

es una query convencional sobre una columna temporal.

No necesariamente es una:

```text
Temporal Domain Query
```

---

# 56. Temporal context

Podrá existir:

```php
final readonly class TemporalQueryContext
{
    public function __construct(
        public ?Instant $asOf,
        public TemporalConsistencyMode $consistency,
        public TemporalZonePolicy $zonePolicy,
    ) {}
}
```

---

# 57. Current query

Sin contexto histórico:

```text
Current Temporal View
```

---

# 58. As-Of query

Permite preguntar:

```text
¿qué era válido en T?
```

Ejemplo:

```php
$account = $repository
    ->asOf($instant)
    ->find($id);
```

---

# 59. AsOf ≠ database snapshot automatically

Muy importante:

```text
AsOf(T)
≠
Database MVCC Snapshot(T)
```

---

# 60. AsOf semantics

Puede significar:

```text
domain valid time
transaction time
system version time
history version time
```

según metadata.

---

# 61. Temporal dimension

```php
enum TemporalDimension
{
    case VALID_TIME;
    case TRANSACTION_TIME;
    case SYSTEM_TIME;
}
```

---

# 62. Valid Time

Representa:

> ¿Cuándo era verdadero este hecho en el dominio?

Ejemplo:

```text
Employee salary:
valid from January 1
valid until March 31
```

---

# 63. Transaction Time

Representa:

> ¿Cuándo conoció/registró la base de datos este hecho?

---

# 64. Valid Time ≠ Transaction Time

Supongamos que el 10 de febrero se registra:

```text
Salary effective since January 1
```

Entonces:

```text
Valid Time       = January 1
Transaction Time = February 10
```

---

# 65. Bitemporal data

Un modelo puede contener:

```text
Valid Time
+
Transaction Time
```

Esto produce:

```text
Bitemporal Model
```

---

# 66. Bitemporal support

La arquitectura deberá permitirlo, aunque no todos los modelos temporales necesiten bitemporalidad.

---

# 67. System Time

Puede representar el tiempo administrado por capacidades nativas del DBMS.

---

# 68. System Time ≠ application transaction time universally

La semántica depende de plataforma.

---

# 69. Temporal metadata

Una entidad podrá declarar:

```text
TemporalEntityMetadata
```

---

# 70. Example

```php
#[Temporal(
    dimension: TemporalDimension::VALID_TIME,
    from: 'validFrom',
    to: 'validTo'
)]
final class Price
{
}
```

---

# 71. Metadata compiled

La metadata temporal deberá integrarse con:

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM
```

---

# 72. Temporal metadata immutable

Una vez compilada:

```text
TemporalMetadata
```

será inmutable durante su generation.

---

# 73. Temporal entity ≠ versioned entity

Una entidad temporal puede representar vigencia sin mantener historial completo.

---

# 74. Versioned entity ≠ temporal entity universally

Versioning será definido en el documento 268.

---

# 75. Audit ≠ temporal history

Un audit log responde:

```text
who did what?
```

Un temporal model responde:

```text
what state was valid/known at time T?
```

---

# 76. Temporal validity columns

Convención conceptual:

```text
valid_from
valid_to
```

pero los nombres físicos serán configurables por mapping.

---

# 77. Temporal transaction columns

Conceptualmente:

```text
recorded_from
recorded_to
```

---

# 78. No magic column names

VoltStack no deberá asumir que toda columna:

```text
created_at
updated_at
```

define temporal semantics.

---

# 79. created_at

Representa normalmente:

```text
creation timestamp
```

No:

```text
valid_from
```

automáticamente.

---

# 80. updated_at

Representa normalmente:

```text
last modification timestamp
```

No historial.

---

# 81. Temporal consistency

Consultas temporales pueden requerir políticas distintas.

```php
enum TemporalConsistencyMode
{
    case BEST_EFFORT;
    case CURRENT_DATABASE_STATE;
    case SNAPSHOT;
    case TRANSACTIONAL;
    case CUSTOM;
}
```

---

# 82. Historical consistency

Una query histórica no significa automáticamente que los datos históricos sean inmutables.

---

# 83. Retroactive correction

Un sistema bitemporal puede registrar:

```text
we learned today
that a fact was valid last month
```

sin sobrescribir conocimiento histórico previo.

---

# 84. Retroactive ≠ destructive rewrite

Preferentemente se modelará mediante nuevas versiones temporales.

---

# 85. Future-effective data

El sistema deberá soportar hechos:

```text
valid in the future
```

Ejemplo:

```text
new price effective next month
```

---

# 86. Current predicate

Formalmente, para intervalo:

```text
[from, to)
```

un valor es vigente en `T` cuando:

```text
from <= T
AND
(to > T OR to = +∞)
```

---

# 87. isCurrent

`isCurrent()` requiere definir qué reloj/contexto proporciona `T`.

---

# 88. Current ≠ PHP now() scattered

Nunca deberá depender de llamadas arbitrarias:

```php
new DateTimeImmutable();
```

dispersas por ORM/query logic.

---

# 89. Clock abstraction

```php
interface Clock
{
    public function now(): Instant;
}
```

---

# 90. SystemClock

Producción:

```text
SystemClock
```

---

# 91. FrozenClock

Testing:

```text
FrozenClock
```

---

# 92. AdvancingClock

Tests temporales podrán usar:

```text
AdvancingClock
```

---

# 93. Clock injection

Clock será una dependencia contextual, no estado global mutable.

---

# 94. Database clock

Algunas operaciones pueden requerir:

```text
database current time
```

---

# 95. Application clock ≠ database clock

Podrán diferir.

---

# 96. Clock source policy

```php
enum TemporalClockSource
{
    case APPLICATION;
    case DATABASE;
    case TRANSACTION;
    case EXPLICIT;
}
```

---

# 97. No silent clock mixing

Una operación no deberá mezclar clocks sin política explícita.

---

# 98. Clock skew

La arquitectura deberá asumir que:

```text
Application Clock
Database Clock
Distributed Node Clock
```

pueden diferir.

---

# 99. Clock skew ≠ ordering guarantee

Dos timestamps generados por hosts distintos no garantizan causalidad.

---

# 100. Temporal ordering

```text
timestamp(A) < timestamp(B)
```

no implica necesariamente:

```text
A caused B
```

---

# 101. Temporal identity

Un objeto histórico puede identificarse mediante:

```text
LogicalEntityId
+
TemporalVersion
```

o:

```text
LogicalEntityId
+
ValidityInterval
```

según modelo.

---

# 102. IdentityMap interaction

La IdentityMap tradicional usa:

```text
EntityType
+
EntityId
+
PersistenceContext
```

Una consulta temporal introduce una dimensión adicional.

---

# 103. Critical identity invariant

La entidad actual y una representación histórica del mismo ID no deberán colisionar incorrectamente dentro de IdentityMap.

---

# 104. TemporalIdentityContext

Conceptualmente:

```text
EntityIdentity
+
TemporalViewIdentity
```

---

# 105. Example

```text
User #42 @ CURRENT
User #42 @ 2025-01-01
```

no necesariamente representan el mismo estado.

---

# 106. Historical entity mutability

Por default, entidades hidratadas desde vistas históricas deberían considerarse:

```text
READ_ONLY
```

o:

```text
DETACHED
```

según política.

---

# 107. Historical state persistence

No deberá permitirse:

```php
$historicalUser->save();
```

como si fuera el estado actual sin una operación explícita de restauración/revisión.

---

# 108. Temporal hydration

Hydration deberá conocer:

```text
TemporalView
```

cuando el resultado represente estado histórico.

---

# 109. Hydration ≠ history resolution

El Temporal Query Planner determina qué rows/versiones representan el estado temporal.

El Hydrator sólo materializa el resultado.

---

# 110. UnitOfWork interaction

El UoW deberá distinguir:

```text
CURRENT_MANAGED
HISTORICAL_READ_ONLY
```

cuando corresponda.

---

# 111. Change tracking

No deberá registrar automáticamente modificaciones sobre historical snapshots como cambios persistibles.

---

# 112. Repository API

Ejemplo conceptual:

```php
$user = $repository->find($id);
```

Estado actual.

```php
$user = $repository
    ->asOf($instant)
    ->find($id);
```

Estado temporal.

---

# 113. Model API

Podrá ofrecer:

```php
User::query()
    ->asOf($instant)
    ->find(42);
```

sin crear otro ORM.

---

# 114. Same ORM engine

```text
Model API
Repository API
Entity Query API
```

convergerán en:

```text
Temporal Query Context
+
Canonical Query Engine
```

---

# 115. Temporal query propagation

```text
Repository
    ↓
Entity Query
    ↓
Temporal Context
    ↓
Semantic Query
    ↓
Temporal Planner
    ↓
Query AST
    ↓
Compiler
```

---

# 116. Temporal Query Planner

Responsabilidades:

```text
resolve temporal metadata
resolve requested dimension
validate temporal eligibility
normalize temporal points
construct semantic temporal predicates
choose native/emulated strategy
validate platform capabilities
preserve tenant/shard context
```

No ejecutará queries.

---

# 117. Planner output

```php
final readonly class TemporalQueryPlan
{
    public function __construct(
        public TemporalDimension $dimension,
        public TemporalView $view,
        public TemporalExecutionStrategy $strategy,
        public QueryModel $query,
    ) {}
}
```

---

# 118. TemporalView

```php
interface TemporalView
{
}
```

Implementaciones:

```text
CurrentTemporalView
AsOfTemporalView
BetweenTemporalView
HistoryTemporalView
```

---

# 119. CurrentTemporalView

Representa:

```text
current effective state
```

---

# 120. AsOfTemporalView

Representa:

```text
state valid/known at T
```

---

# 121. BetweenTemporalView

Representa estados o eventos relevantes dentro de:

```text
[T1, T2)
```

---

# 122. HistoryTemporalView

Representa una secuencia temporal completa cuando el sistema de versionado lo soporte.

---

# 123. TemporalExecutionStrategy

```text
PREDICATE
NATIVE_SYSTEM_TIME
NATIVE_RANGE
HISTORY_TABLE
VERSION_TABLE
CUSTOM
```

---

# 124. Strategy ≠ public semantics

La API debe conservar significado aunque cambie la representación física.

---

# 125. Native system-versioning

Si el DBMS lo soporta, VoltStack podrá utilizarlo.

Pero la arquitectura no dependerá obligatoriamente de ello.

---

# 126. Emulated temporal data

Podrá implementarse mediante:

```text
valid_from
valid_to
```

y predicates normales.

---

# 127. Emulation transparency

Diagnostics deberá indicar:

```text
Temporal Strategy: EMULATED_INTERVAL
```

cuando corresponda.

---

# 128. Temporal relationships

Una relación también puede tener vigencia.

Ejemplo:

```text
Employee
  workedFor
Company
```

durante:

```text
[2024-01-01, 2026-03-01)
```

---

# 129. Relationship temporal semantics

Se distinguirán:

```text
entity validity
relationship validity
```

---

# 130. Entity current ≠ relationship current

Una entidad puede seguir existiendo aunque una relación haya expirado.

---

# 131. Temporal relationship metadata

Conceptualmente:

```php
#[TemporalRelationship(
    from: 'assignedFrom',
    to: 'assignedTo'
)]
```

---

# 132. Many-to-many temporal relationships

Una association entity será normalmente preferible cuando la relación tenga:

```text
validity
attributes
history
business meaning
```

---

# 133. Temporal eager loading

Una query:

```text
Employee @ T
```

que eager-loads:

```text
department
```

deberá preservar el mismo contexto temporal cuando la relación sea temporalmente dependiente.

---

# 134. Temporal context propagation

```text
Root @ T
    ↓
Relationship @ T
```

por default cuando metadata lo requiera.

---

# 135. Mixed temporal context

Debe ser explícito si se desea:

```text
historical root
+
current relation
```

---

# 136. Temporal N+1

Las consultas temporales siguen sujetas a:

```text
N+1
```

y el detector deberá conservar awareness del contexto temporal.

---

# 137. Temporal pagination

Pagination sobre datos temporales deberá tener ordering estable.

---

# 138. Temporal cursor

Si el cursor incluye valores temporales:

```text
created_at
valid_from
transaction_time
```

deberán usar canonical temporal encoding.

---

# 139. Cursor temporal precision

La precisión del cursor deberá preservar la precisión efectiva de comparación en DB.

---

# 140. Precision mismatch risk

Si DB almacena microsegundos pero cursor sólo serializa segundos:

```text
duplicate/skip risk
```

---

# 141. Chunk temporal processing

Chunk processing sobre temporal datasets deberá fijar claramente:

```text
LIVE
SNAPSHOT
AS_OF
UPPER_BOUND
```

según el objetivo.

---

# 142. Lazy temporal collection

Una lazy traversal no garantiza snapshot temporal por sí sola.

---

# 143. Same definition ≠ same temporal result

```text
Query.asOf(now)
```

ejecutada dos veces puede producir distinto `T` si `now` no fue capturado.

---

# 144. Capture now once

Operaciones que necesiten consistencia deberán resolver:

```text
now
```

una sola vez en el boundary adecuado.

---

# 145. Example

Incorrecto:

```text
row1 → now()
row2 → now()
row3 → now()
```

Preferido:

```text
T = clock.now()

row1 → T
row2 → T
row3 → T
```

---

# 146. Temporal transaction context

Una transaction podrá capturar:

```text
TransactionTemporalAnchor
```

cuando la política lo requiera.

---

# 147. Transaction start ≠ statement time

Deben poder distinguirse.

---

# 148. Transaction time semantics

Una plataforma puede ofrecer diferentes funciones para:

```text
transaction start
statement start
wall clock
```

El Compiler/Platform adapter deberá mapearlas según capability.

---

# 149. Temporal writes

Una actualización temporal puede significar:

```text
replace current state
close current interval
open new interval
correct historical interval
schedule future interval
```

---

# 150. Temporal mutation intent

Se representará explícitamente.

```php
enum TemporalMutationIntent
{
    case CURRENT_UPDATE;
    case EFFECTIVE_CHANGE;
    case RETROACTIVE_CORRECTION;
    case FUTURE_CHANGE;
    case CLOSE_INTERVAL;
}
```

---

# 151. Mutation intent ≠ SQL UPDATE

---

# 152. Temporal persistence

El Persistence Engine podrá transformar una mutation temporal en múltiples operaciones semánticas.

Ejemplo:

```text
close old interval
+
insert new interval
```

---

# 153. Temporal persistence ≠ generic UPDATE

---

# 154. Atomicity

Cuando múltiples operaciones formen una transición temporal lógica, deberán ejecutarse dentro de una transaction cuando sea posible y requerido.

---

# 155. Temporal overlap protection

Algunos modelos deberán impedir:

```text
overlapping validity intervals
```

para el mismo logical entity.

---

# 156. Constraint example

Para:

```text
Price(product_id)
```

podría exigirse:

```text
no two active price intervals overlap
```

---

# 157. Temporal constraint

Conceptualmente:

```text
TemporalNonOverlapConstraint
```

---

# 158. Native vs application enforcement

Puede implementarse mediante:

```text
native exclusion/range constraint
transactional validation
locking strategy
custom mechanism
```

según capabilities.

---

# 159. Check-then-write race

Esto:

```text
SELECT no overlap
UPDATE
```

sin concurrency protection no garantiza ausencia de race.

---

# 160. Temporal concurrency

El planner deberá coordinarse con:

```text
Transaction System
Optimistic Locking
Pessimistic Locking
Platform Constraints
```

---

# 161. Temporal uniqueness

Una propiedad puede ser única:

```text
currently
```

pero no históricamente.

---

# 162. Example

```text
username = "alice"
```

puede pertenecer a una sola entidad vigente, aunque haya pertenecido históricamente a otra.

---

# 163. Temporal unique constraint

Es distinto de:

```text
UNIQUE(username)
```

convencional.

---

# 164. Schema integration

Schema Model deberá poder representar metadata temporal cuando tenga efectos físicos.

---

# 165. Temporal schema definitions

Conceptualmente:

```php
$table->temporalValidity(
    from: 'valid_from',
    to: 'valid_to'
);
```

---

# 166. Schema Builder ≠ Temporal Engine

Schema Builder sólo describe estructura.

---

# 167. Migrations

Cambiar un modelo convencional a temporal puede requerir:

```text
new columns
history table
indexes
constraints
backfill
application compatibility window
```

---

# 168. Temporal migration safety

Una migration temporal puede ser:

```text
expensive
locking
data-transforming
irreversible
```

y deberá pasar por Migration Safety.

---

# 169. Backfill

Convertir registros existentes a temporal puede usar:

```text
valid_from = migration-defined anchor
valid_to   = infinity
```

pero la semántica debe definirse explícitamente.

---

# 170. Do not fabricate historical truth

Si sólo se conoce el estado actual:

```text
Current state known
```

no deberá inventarse:

```text
historical validity
```

---

# 171. Unknown historical start

Puede requerir un estado:

```text
UNKNOWN_START
```

o una política de baseline explícita.

---

# 172. Temporal indexing

Queries frecuentes pueden requerir índices sobre:

```text
entity_id
valid_from
valid_to
transaction_from
transaction_to
```

---

# 173. Index strategy

Dependerá de:

```text
query patterns
platform
cardinality
temporal dimension
range capabilities
```

---

# 174. Temporal Engine ≠ Index Advisor

Puede emitir hints/evidence, pero el sistema de performance/schema decide índices.

---

# 175. Query optimizer

El Optimizer podrá aplicar reglas temporales.

Ejemplos:

```text
normalize overlapping predicates
remove impossible intervals
push temporal predicates
simplify unbounded ranges
```

sólo si preservan semántica.

---

# 176. Empty interval

Para:

```text
[start, end)
```

si:

```text
start >= end
```

el intervalo puede ser inválido o vacío según el tipo/policy.

---

# 177. Invalid temporal interval

Debe detectarse antes de SQL cuando sea posible.

---

# 178. Interval normalization

```text
[10,10)
```

es vacío.

---

# 179. Overlap formula

Para intervalos half-open:

```text
A = [a1, a2)
B = [b1, b2)
```

existe overlap cuando:

```text
a1 < b2
AND
b1 < a2
```

considerando límites infinitos.

---

# 180. Contains formula

`A` contiene punto `t` cuando:

```text
a1 <= t
AND
t < a2
```

para `[a1,a2)`.

---

# 181. Temporal arithmetic

Debe ser type-aware.

Permitido:

```text
Instant + Duration
LocalDate + Period
```

No necesariamente:

```text
LocalDate + Duration
```

sin reglas explícitas.

---

# 182. DST

Cambios de horario son una razón crítica para separar:

```text
Duration
Period
LocalDateTime
ZonedDateTime
```

---

# 183. Example DST

```text
ZonedDateTime + 1 day
```

no siempre equivale a:

```text
+ 24 hours
```

---

# 184. Ambiguous local times

Al cambiar reloj, una hora local puede ocurrir dos veces.

---

# 185. Nonexistent local times

También puede existir una hora local que no ocurrió.

---

# 186. Zone resolution policy

```php
enum ZoneResolutionPolicy
{
    case REJECT_AMBIGUOUS;
    case EARLIER_OFFSET;
    case LATER_OFFSET;
    case REJECT_NONEXISTENT;
    case SHIFT_FORWARD;
}
```

---

# 187. No silent DST guess

VoltStack no deberá escoger arbitrariamente un offset si la conversión es ambigua.

---

# 188. Timezone database

La conversión zoned deberá depender de una base de reglas de timezone reconocida por el runtime.

---

# 189. Timezone rule changes

Reglas de zona pueden cambiar con el tiempo.

---

# 190. Stored zone semantics

Si el dominio necesita conservar intención original, puede ser necesario almacenar:

```text
local datetime
+
timezone id
```

además o en lugar del instant.

---

# 191. Instant-only storage

Guardar sólo UTC es excelente para eventos globales, pero no representa todos los casos de negocio.

---

# 192. Example

Evento:

```text
"meeting every day at 09:00 local time"
```

no debe modelarse simplemente como un único UTC instant.

---

# 193. Business Time

Representa reglas temporales de negocio.

Ejemplos:

```text
business day
working hours
holiday calendar
settlement day
billing period
```

---

# 194. Business calendar

Será una extensión opcional.

No deberá incrustarse en el core temporal básico.

---

# 195. Business calendar interface

```php
interface BusinessCalendar
{
    public function isBusinessDay(LocalDate $date): bool;

    public function nextBusinessDay(LocalDate $date): LocalDate;
}
```

---

# 196. Calendar identity

Un calendario deberá tener identidad explícita.

Ejemplo:

```text
MX_BANKING
US_NYSE
COMPANY_SUPPORT
```

---

# 197. Business day ≠ weekday

Festivos y reglas especiales importan.

---

# 198. Temporal partitioning

Grandes datasets temporales pueden particionarse por:

```text
day
month
year
time range
```

---

# 199. Temporal System ≠ Partition Manager

Sólo proporcionará semántica temporal al sistema de partition routing cuando corresponda.

---

# 200. Partition pruning

El Query Planner podrá transmitir temporal ranges para facilitar pruning.

---

# 201. Retention

La existencia de datos temporales no implica retención infinita.

---

# 202. Retention integration

Se conectará posteriormente con:

```text
270_DATABASE_DATA_RETENTION_SYSTEM
271_DATABASE_DATA_ARCHIVAL_SYSTEM
```

---

# 203. Temporal retention ≠ soft delete

---

# 204. Archival

Un historical interval puede moverse a almacenamiento de archivo sin cambiar necesariamente su significado temporal.

---

# 205. Full-text and JSON

Campos temporales dentro de JSON no deberán recibir automáticamente las mismas garantías que columnas temporalmente tipadas.

---

# 206. JSON temporal value

Puede mapearse explícitamente mediante:

```text
JSON Type
+
Temporal Value Converter
```

---

# 207. Cache integration

Temporal queries pueden usar Result Cache si la consistency policy lo permite.

---

# 208. Current-time query cache

Una query:

```text
WHERE valid_from <= now
AND valid_to > now
```

es sensible al paso del tiempo incluso sin escrituras.

---

# 209. Critical cache rule

```text
No database mutation
```

no significa:

```text
temporal cached result still valid
```

---

# 210. Time-dependent cache expiration

El Cache System podrá calcular:

```text
next semantic invalidation time
```

cuando sea conocido.

---

# 211. Example

Si una oferta termina:

```text
15:00
```

un resultado cacheado a:

```text
14:50
```

no deberá considerarse válido después de las 15:00 sólo porque no ocurrió ningún UPDATE.

---

# 212. Temporal cache dependency

Puede incluir:

```text
time boundary dependency
```

además de data dependencies.

---

# 213. Entity Cache

Una historical entity cache deberá incluir:

```text
TemporalViewIdentity
```

en su key cuando corresponda.

---

# 214. Result Cache

Key:

```text
QueryFingerprint
+
Parameters
+
TemporalContext
+
Tenant
+
Shard
+
MetadataGeneration
```

cuando sean relevantes.

---

# 215. Compiled Query Cache

Puede reutilizar estructura compilada independientemente del valor temporal si la query está parametrizada.

---

# 216. Security

Temporal queries pueden exponer información histórica que un usuario no está autorizado a ver actualmente.

---

# 217. Current authorization ≠ historical authorization

Debe definirse qué policy aplica.

---

# 218. Historical access policy

Puede ser:

```text
CURRENT_POLICY
POLICY_AS_OF_TIME
AUDIT_POLICY
ADMIN_ONLY
CUSTOM
```

---

# 219. Temporal query cannot bypass authorization

`asOf()` no deberá convertirse en mecanismo para leer datos históricos restringidos.

---

# 220. Sensitive historical data

Datos borrados/anulados pueden seguir existiendo históricamente.

Su acceso seguirá sujeto a:

```text
Data Access Security
Sensitive Data Protection
Retention
Legal Erasure Policy
```

---

# 221. Tenant context

Toda temporal query tenant-aware conservará:

```text
TenantQueryContext
```

---

# 222. Temporal context ≠ tenant context

Son dimensiones independientes.

---

# 223. Query identity

Puede depender de:

```text
Tenant
Shard
TemporalView
AuthorizationScope
Query
```

---

# 224. Cross-tenant temporal query

Seguirá prohibida por default.

---

# 225. Sharding

Temporal data puede estar distribuida entre shards.

---

# 226. Temporal routing

Si shard routing depende de fecha:

```text
TemporalRange
```

puede participar en:

```text
PartitionRouting
```

---

# 227. Temporal range ≠ shard automatically

El Partition Router decide.

---

# 228. Cross-shard temporal query

Puede requerir:

```text
fan-out
merge
global temporal ordering
```

---

# 229. Global ordering

Si se requiere:

```text
ORDER BY event_time
```

entre shards, deberán existir tie-breakers globales.

---

# 230. Same timestamp ≠ same event

---

# 231. Distributed clocks

No se asumirá sincronización perfecta entre shards.

---

# 232. Replica considerations

Una temporal `asOf()` histórica puede seguir siendo sensible a replica lag si la historia fue registrada recientemente.

---

# 233. Historical ≠ replica-safe automatically

---

# 234. Read routing

Respetará:

```text
Transaction Context
Sticky Connection
Read-Your-Writes
Replica Lag Policy
Temporal Consistency
```

---

# 235. Temporal query timeout

Usará el Query Timeout System normal.

---

# 236. Cancellation

Temporal queries grandes podrán cancelarse normalmente.

---

# 237. Telemetry

Métricas posibles:

```text
database_temporal_queries_total
database_temporal_query_duration
database_temporal_asof_queries_total
database_temporal_emulation_total
database_temporal_precision_loss_total
database_temporal_overlap_conflicts_total
```

---

# 238. Cardinality

No incluir:

```text
exact timestamp
entity id
tenant id
```

como labels por default.

---

# 239. Tracing

Span attributes bounded:

```text
temporal.dimension
temporal.strategy
temporal.native
temporal.precision
temporal.view_kind
```

---

# 240. No sensitive temporal values by default

No registrar automáticamente:

```text
birth dates
contract dates
medical dates
financial event times
```

como telemetry attributes.

---

# 241. Diagnostics

Ejemplo:

```text
Temporal Query
────────────────────────────────────

Entity:
  Price

Dimension:
  VALID_TIME

View:
  AS_OF

Temporal Type:
  Instant

Requested Precision:
  MICROSECOND

Platform Precision:
  MICROSECOND

Strategy:
  INTERVAL_PREDICATE

Native Temporal Table:
  no

Interval:
  [valid_from, valid_to)

Clock:
  EXPLICIT

Tenant Context:
  present

Shard Routing:
  single shard

Historical Entity Mode:
  READ_ONLY

Cache:
  time-sensitive

Status:
  VALID
```

---

# 242. Explain

```text
Temporal Plan
────────────────────────────────────

1. Entity temporal metadata resolved.
2. VALID_TIME dimension selected.
3. AS_OF temporal view requested.
4. Input converted to canonical Instant.
5. Precision compatibility verified.
6. Platform has no required native temporal-table feature.
7. Interval predicate strategy selected.
8. Temporal predicate added to semantic query.
9. Tenant context preserved.
10. Shard routing resolved.
11. Historical hydration marked READ_ONLY.
12. Result cache marked time-aware.
```

---

# 243. Errors

Base:

```php
class TemporalDatabaseException extends DatabaseException
{
}
```

---

# 244. Error hierarchy

```text
TemporalDatabaseException
├── InvalidTemporalValueException
├── InvalidTemporalIntervalException
├── TemporalPrecisionLossException
├── TemporalTypeMismatchException
├── TemporalCapabilityException
├── UnsupportedTemporalOperationException
├── TemporalZoneException
├── AmbiguousLocalTimeException
├── NonexistentLocalTimeException
├── TemporalOverlapException
├── TemporalConstraintException
├── TemporalConsistencyException
├── TemporalIdentityException
├── HistoricalEntityMutationException
├── TemporalQueryPlanningException
└── TemporalSecurityException
```

---

# 245. Error context

Debe incluir cuando sea seguro:

```text
temporal type
dimension
precision
strategy
platform capability
operation
```

sin exponer información sensible.

---

# 246. Persistent runtime

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

no deberán existir globals mutables como:

```php
Temporal::$currentAsOf
Temporal::$timezone
Temporal::$clock
```

---

# 247. Scope-local temporal state

```text
TemporalQueryContext
Clock
TenantContext
TransactionTemporalAnchor
```

serán scope-local/contextual.

---

# 248. Shared immutable objects

Podrán compartirse:

```text
TemporalMetadata
TemporalTypeDefinition
PlatformTemporalCapabilities
CompiledTemporalRules
```

si son inmutables y generation-aware.

---

# 249. Request reset

Al finalizar scope:

```text
TemporalQueryContext → released
Temporal anchors      → released
Temporary clocks      → released
Historical identity   → cleared
```

---

# 250. Coroutine safety

OpenSwoole no deberá compartir temporal context mutable entre coroutines.

---

# 251. Serialization

Valores temporales canónicos sí podrán serializarse mediante formatos definidos.

Pero:

```text
TemporalQueryContext
```

no deberá serializar conexiones, clocks activos ni runtime state.

---

# 252. External temporal representation

Para APIs, el Database System no impondrá un formato HTTP.

---

# 253. Database ≠ HTTP serializer

La capa HTTP podrá decidir:

```text
ISO-8601
RFC-compatible formats
localized presentation
```

---

# 254. Localization

La persistencia temporal no deberá almacenar fechas formateadas para presentación como:

```text
19/09/2026
```

salvo que sean literalmente strings del dominio.

---

# 255. Storage ≠ presentation

---

# 256. Testing

Se requerirán:

```text
Temporal Type Tests
Precision Tests
Timezone Tests
DST Tests
Interval Tests
Range Tests
Predicate Tests
As-Of Tests
Valid-Time Tests
Transaction-Time Tests
Bitemporal Tests
Temporal Identity Tests
Hydration Tests
UoW Tests
Relationship Tests
Pagination Tests
Cursor Tests
Chunk Tests
Cache Tests
Replica Tests
Shard Tests
Tenant Tests
Security Tests
Persistent Runtime Tests
Platform Conformance Tests
```

---

# 257. Clock tests

Todo comportamiento dependiente de tiempo deberá poder probarse sin esperar tiempo real.

---

# 258. FrozenClock example

```php
$clock = FrozenClock::at(
    Instant::parse('2026-09-19T20:00:00Z')
);
```

---

# 259. Boundary tests

Probar:

```text
exact start
just after start
just before end
exact end
unbounded start
unbounded end
empty interval
```

---

# 260. DST tests

Probar:

```text
ambiguous time
nonexistent time
offset change
historical timezone rules
```

---

# 261. Precision tests

Probar conversiones:

```text
seconds
milliseconds
microseconds
```

y detectar truncation.

---

# 262. Platform conformance

Cada plataforma deberá probar:

```text
date mapping
time mapping
instant mapping
precision
range behavior
timezone behavior
interval operations
native temporal features
```

---

# 263. Native/emulated equivalence

Cuando exista emulación:

```text
NativeSemantics(input)
=
EmulatedSemantics(input)
```

para casos soportados.

---

# 264. Temporal property tests

Podrán comprobar invariantes matemáticos como:

```text
A overlaps B
⇔
B overlaps A
```

cuando aplique.

---

# 265. Range property

Para intervalos válidos:

```text
contains(A,t)
```

deberá respetar exactamente las boundary semantics.

---

# 266. Proposed directory

```text
src/Quantum/Database/Temporal/
│
├── Contract/
│   ├── Clock.php
│   ├── TemporalValue.php
│   ├── TemporalPoint.php
│   ├── TemporalQueryPlanner.php
│   └── TemporalCapabilityResolver.php
│
├── Value/
│   ├── Instant.php
│   ├── LocalDate.php
│   ├── LocalTime.php
│   ├── LocalDateTime.php
│   ├── OffsetDateTime.php
│   ├── ZonedDateTime.php
│   ├── Duration.php
│   ├── Period.php
│   └── TimeZoneId.php
│
├── Range/
│   ├── TemporalInterval.php
│   ├── TemporalRange.php
│   ├── TemporalBoundary.php
│   ├── BoundaryType.php
│   └── TemporalRangeNormalizer.php
│
├── Clock/
│   ├── SystemClock.php
│   ├── DatabaseClock.php
│   ├── FrozenClock.php
│   └── AdvancingClock.php
│
├── Metadata/
│   ├── TemporalEntityMetadata.php
│   ├── TemporalRelationshipMetadata.php
│   ├── TemporalDimension.php
│   └── TemporalMetadataCompiler.php
│
├── Query/
│   ├── TemporalQueryContext.php
│   ├── TemporalQueryPlan.php
│   ├── TemporalView.php
│   ├── CurrentTemporalView.php
│   ├── AsOfTemporalView.php
│   ├── BetweenTemporalView.php
│   └── HistoryTemporalView.php
│
├── AST/
│   ├── TemporalExpressionNode.php
│   ├── TemporalPointNode.php
│   ├── TemporalRangeNode.php
│   ├── TemporalPredicateNode.php
│   ├── TemporalOverlapsPredicate.php
│   ├── TemporalContainsPredicate.php
│   └── TemporalAsOfPredicate.php
│
├── Planning/
│   ├── DefaultTemporalQueryPlanner.php
│   ├── TemporalStrategyResolver.php
│   ├── TemporalPrecisionValidator.php
│   └── TemporalCapabilityAnalyzer.php
│
├── Persistence/
│   ├── TemporalMutationIntent.php
│   ├── TemporalPersistencePlanner.php
│   ├── TemporalOverlapGuard.php
│   └── TemporalConstraintValidator.php
│
├── ORM/
│   ├── TemporalIdentityContext.php
│   ├── HistoricalEntityState.php
│   └── TemporalHydrationContext.php
│
├── Platform/
│   ├── TemporalPlatformCapabilities.php
│   ├── TemporalPlatformMapping.php
│   └── TemporalCapabilityStatus.php
│
├── Cache/
│   ├── TemporalCacheDependency.php
│   └── TemporalCacheBoundaryResolver.php
│
├── Diagnostics/
│   └── TemporalDiagnostics.php
│
└── Exception/
    ├── TemporalDatabaseException.php
    ├── InvalidTemporalValueException.php
    ├── InvalidTemporalIntervalException.php
    ├── TemporalPrecisionLossException.php
    ├── TemporalCapabilityException.php
    ├── AmbiguousLocalTimeException.php
    ├── NonexistentLocalTimeException.php
    ├── TemporalOverlapException.php
    ├── TemporalConsistencyException.php
    └── HistoricalEntityMutationException.php
```

---

# 267. Dependency rules

Permitido:

```text
ORM
 ↓
Temporal System
 ↓
Query Engine
 ↓
Compiler
 ↓
Execution
 ↓
Connection
 ↓
Driver
```

También:

```text
Temporal System
→ Type System
→ Platform Capabilities
```

---

# 268. Forbidden dependencies

No:

```text
Temporal System
→ PDO
```

No:

```text
Temporal System
→ HTTP Request
```

No:

```text
Temporal System
→ global Tenant state
```

No:

```text
Temporal System
→ SQL strings
```

---

# 269. Invariantes arquitectónicos

## DB-TEMP-001

Temporal Data será distinto de DateTime Casting.

## DB-TEMP-002

Temporal Data será distinto de Entity History.

## DB-TEMP-003

Temporal Data será distinto de Audit Log.

## DB-TEMP-004

Temporal Data será distinto de Soft Delete.

## DB-TEMP-005

Instant será distinto de LocalDateTime.

## DB-TEMP-006

Offset será distinto de TimeZoneId.

## DB-TEMP-007

Duration será distinto de Period.

## DB-TEMP-008

Physical database type no definirá por sí solo temporal semantics.

## DB-TEMP-009

Temporal semantics determinarán logical type.

## DB-TEMP-010

Logical type determinará platform mapping.

## DB-TEMP-011

Temporal System no generará SQL.

## DB-TEMP-012

Compiler será responsable de SQL temporal.

## DB-TEMP-013

Platform capabilities determinarán native/emulated support.

## DB-TEMP-014

Version no equivaldrá a capability.

## DB-TEMP-015

MySQL y MariaDB permanecerán plataformas independientes.

## DB-TEMP-016

UNKNOWN capability no equivaldrá a supported.

## DB-TEMP-017

Temporal precision será explícita.

## DB-TEMP-018

Requested precision será distinta de effective precision.

## DB-TEMP-019

Precision loss significativa no será silenciosa.

## DB-TEMP-020

Temporal values tendrán canonical representation.

## DB-TEMP-021

Temporal intervals tendrán boundary semantics explícitas.

## DB-TEMP-022

Unbounded no equivaldrá automáticamente a unknown.

## DB-TEMP-023

Database NULL no definirá universalmente infinity.

## DB-TEMP-024

Half-open intervals serán la convención recomendada cuando corresponda.

## DB-TEMP-025

Temporal predicates serán AST nodes.

## DB-TEMP-026

Temporal predicate no concatenará SQL.

## DB-TEMP-027

Native range support no será requisito del API.

## DB-TEMP-028

Emulation deberá preservar semantic equivalence.

## DB-TEMP-029

Query con date filter no será automáticamente Temporal Domain Query.

## DB-TEMP-030

Temporal Query Context será explícito.

## DB-TEMP-031

AsOf no equivaldrá a MVCC snapshot.

## DB-TEMP-032

Valid Time será distinto de Transaction Time.

## DB-TEMP-033

System Time será distinto de application transaction time cuando la plataforma así lo defina.

## DB-TEMP-034

Bitemporal model podrá combinar Valid Time y Transaction Time.

## DB-TEMP-035

Temporal metadata será compilable.

## DB-TEMP-036

Temporal metadata será immutable por generation.

## DB-TEMP-037

Temporal entity no equivaldrá a versioned entity.

## DB-TEMP-038

Versioned entity no será necesariamente temporal.

## DB-TEMP-039

Audit log no equivaldrá a temporal history.

## DB-TEMP-040

created_at no equivaldrá a valid_from.

## DB-TEMP-041

updated_at no equivaldrá a history.

## DB-TEMP-042

No se inferirá temporal behavior por nombres mágicos de columnas.

## DB-TEMP-043

Historical consistency será explícita.

## DB-TEMP-044

Retroactive correction no requerirá destructive overwrite.

## DB-TEMP-045

Future-effective data será soportable.

## DB-TEMP-046

isCurrent utilizará clock/context explícito.

## DB-TEMP-047

Temporal behavior no dependerá de now() disperso.

## DB-TEMP-048

Clock será abstraído.

## DB-TEMP-049

Clock será injectable.

## DB-TEMP-050

Testing podrá congelar clock.

## DB-TEMP-051

Application clock será distinto de database clock.

## DB-TEMP-052

Clocks no se mezclarán silenciosamente.

## DB-TEMP-053

Timestamp ordering no implicará causal ordering.

## DB-TEMP-054

Historical identity no colisionará incorrectamente con current identity.

## DB-TEMP-055

TemporalView participará en identity cuando sea necesario.

## DB-TEMP-056

Historical entities serán read-only/detached por default.

## DB-TEMP-057

Historical snapshot no será persistido como current entity accidentalmente.

## DB-TEMP-058

Hydrator no resolverá history semantics.

## DB-TEMP-059

Temporal Planner resolverá temporal semantics.

## DB-TEMP-060

UoW distinguirá historical read-only state cuando corresponda.

## DB-TEMP-061

Model API y Repository API compartirán Temporal Engine.

## DB-TEMP-062

No existirá un segundo ORM temporal.

## DB-TEMP-063

TemporalQueryPlanner no ejecutará queries.

## DB-TEMP-064

TemporalView será explícita.

## DB-TEMP-065

Execution strategy no cambiará public semantics.

## DB-TEMP-066

Native temporal tables serán opcionales.

## DB-TEMP-067

Emulated temporal model será first-class.

## DB-TEMP-068

Temporal relationship validity será distinta de entity validity.

## DB-TEMP-069

Historical relationship loading preservará temporal context por default cuando metadata lo requiera.

## DB-TEMP-070

Mixed temporal context será explícito.

## DB-TEMP-071

Temporal queries seguirán sujetas a N+1 detection.

## DB-TEMP-072

Temporal cursor values usarán canonical encoding.

## DB-TEMP-073

Cursor precision preservará DB comparison precision.

## DB-TEMP-074

Chunk temporal processing declarará horizon/consistency.

## DB-TEMP-075

Lazy temporal iteration no implicará snapshot.

## DB-TEMP-076

Same query definition no implicará same temporal result.

## DB-TEMP-077

Consistency-sensitive now será capturado una vez.

## DB-TEMP-078

Transaction start time será distinto de statement time.

## DB-TEMP-079

Temporal mutation intent será explícito.

## DB-TEMP-080

Temporal mutation intent no equivaldrá a SQL UPDATE.

## DB-TEMP-081

Temporal persistence podrá producir múltiples operations.

## DB-TEMP-082

Multi-operation temporal transition usará transaction cuando sea requerido y soportado.

## DB-TEMP-083

Temporal overlap constraints serán representables.

## DB-TEMP-084

Check-then-write no se considerará race-safe automáticamente.

## DB-TEMP-085

Temporal concurrency se integrará con transaction/locking systems.

## DB-TEMP-086

Temporal uniqueness será distinta de conventional uniqueness.

## DB-TEMP-087

Schema Model podrá representar physical temporal metadata.

## DB-TEMP-088

Schema Builder no será Temporal Engine.

## DB-TEMP-089

Temporal migrations usarán canonical Migration Engine.

## DB-TEMP-090

Historical truth no será fabricada durante backfill.

## DB-TEMP-091

Unknown historical start no se convertirá silenciosamente en known start.

## DB-TEMP-092

Temporal indexing será query-pattern aware.

## DB-TEMP-093

Temporal System no será Index Advisor.

## DB-TEMP-094

Optimizer sólo aplicará temporal rewrites semánticamente equivalentes.

## DB-TEMP-095

Invalid intervals serán detectados temprano.

## DB-TEMP-096

Interval overlap respetará boundary semantics.

## DB-TEMP-097

Temporal arithmetic será type-aware.

## DB-TEMP-098

Calendar month no equivaldrá universalmente a fixed duration.

## DB-TEMP-099

DST ambiguity será explícita.

## DB-TEMP-100

Nonexistent local times serán detectables.

## DB-TEMP-101

No habrá silent timezone resolution para casos ambiguos.

## DB-TEMP-102

TimeZoneId será preferido sobre abbreviation.

## DB-TEMP-103

UTC instant storage no resolverá todos los dominios temporales.

## DB-TEMP-104

Business Time será extensión separada.

## DB-TEMP-105

Business day no equivaldrá a weekday.

## DB-TEMP-106

Temporal partitioning será integración, no responsabilidad central.

## DB-TEMP-107

Temporal data no implicará infinite retention.

## DB-TEMP-108

Retention será responsabilidad de su sistema especializado.

## DB-TEMP-109

Temporal JSON values requerirán mapping explícito.

## DB-TEMP-110

Temporal cache entries podrán expirar por paso del tiempo.

## DB-TEMP-111

No-write no implicará temporal-cache validity.

## DB-TEMP-112

Temporal cache dependencies podrán incluir future time boundaries.

## DB-TEMP-113

Historical entity cache incluirá temporal identity cuando corresponda.

## DB-TEMP-114

Result cache incluirá temporal context cuando sea relevante.

## DB-TEMP-115

Compiled query cache podrá reutilizar queries temporalmente parametrizadas.

## DB-TEMP-116

Temporal query no bypassará authorization.

## DB-TEMP-117

Current authorization será distinto de historical authorization conceptualmente.

## DB-TEMP-118

Historical access policy será explícita.

## DB-TEMP-119

Sensitive historical data seguirá protegido.

## DB-TEMP-120

Temporal context será distinto de tenant context.

## DB-TEMP-121

Cross-tenant temporal query estará prohibida por default.

## DB-TEMP-122

Temporal ranges podrán participar en partition routing.

## DB-TEMP-123

Temporal range no determinará shard directamente.

## DB-TEMP-124

Cross-shard temporal ordering requerirá global ordering semantics.

## DB-TEMP-125

Equal timestamps no implicarán equal event identity.

## DB-TEMP-126

Distributed clocks no se asumirán perfectamente sincronizados.

## DB-TEMP-127

Historical query no será automáticamente replica-safe.

## DB-TEMP-128

Read routing respetará temporal consistency.

## DB-TEMP-129

Temporal queries usarán normal timeout/cancellation infrastructure.

## DB-TEMP-130

Temporal telemetry será bounded.

## DB-TEMP-131

Temporal values sensibles no serán telemetry labels por default.

## DB-TEMP-132

Temporal errors serán typed.

## DB-TEMP-133

Temporal context no será static global state.

## DB-TEMP-134

FrankenPHP será temporal-state safe.

## DB-TEMP-135

RoadRunner será temporal-state safe.

## DB-TEMP-136

OpenSwoole será temporal-state safe.

## DB-TEMP-137

Coroutine temporal state estará aislado.

## DB-TEMP-138

Immutable temporal metadata podrá compartirse.

## DB-TEMP-139

Request reset eliminará temporal scope state.

## DB-TEMP-140

Database layer no impondrá HTTP temporal serialization.

## DB-TEMP-141

Storage representation será distinta de presentation format.

## DB-TEMP-142

Temporal behavior será deterministically testable.

## DB-TEMP-143

DST edge cases tendrán tests.

## DB-TEMP-144

Precision loss tendrá tests.

## DB-TEMP-145

Native/emulated temporal semantics tendrán conformance tests.

## DB-TEMP-146

Platform temporal capabilities tendrán conformance tests.

## DB-TEMP-147

Temporal intervals tendrán property tests.

## DB-TEMP-148

Temporal identity tendrá tests.

## DB-TEMP-149

Historical UoW protection tendrá tests.

## DB-TEMP-150

Temporal relationship propagation tendrá tests.

## DB-TEMP-151

Temporal cache invalidation tendrá tests.

## DB-TEMP-152

Temporal tenant isolation tendrá tests.

## DB-TEMP-153

Temporal sharding tendrá tests.

## DB-TEMP-154

Temporal System dependerá del Query Engine, no del Driver directamente.

## DB-TEMP-155

Driver no conocerá TemporalQueryPlanner.

## DB-TEMP-156

Compiler no interpretará domain history rules.

## DB-TEMP-157

Executor no decidirá temporal business semantics.

## DB-TEMP-158

Connection no conocerá TemporalEntityMetadata.

## DB-TEMP-159

Temporal state será operation/request scoped.

## DB-TEMP-160

El significado temporal siempre tendrá precedencia sobre la conveniencia del tipo físico de base de datos.

---

# 270. Modelo formal de Valid Time

Sea una versión:

```text
v
```

con intervalo:

```text
I(v) = [from(v), to(v))
```

La versión es válida en `t` si:

```text
Valid(v,t)
⇔
from(v) ≤ t
∧
t < to(v)
```

considerando `+∞` cuando corresponda.

---

# 271. Estado actual

Sea:

```text
Tnow = TemporalClock.resolve(context)
```

Entonces:

```text
Current(v)
⇔
Valid(v,Tnow)
```

---

# 272. As-Of

Para una entidad lógica `e`:

```text
AsOf(e,t)
=
{v ∈ Versions(e) | Valid(v,t)}
```

Si el modelo exige una sola versión vigente:

```text
|AsOf(e,t)| ≤ 1
```

---

# 273. Temporal non-overlap

Para dos versiones distintas:

```text
v1
v2
```

del mismo logical entity:

```text
I(v1) ∩ I(v2) = ∅
```

cuando el modelo declare exclusividad temporal.

---

# 274. Bitemporal model

Para cada versión:

```text
ValidInterval(v)
TransactionInterval(v)
```

Una query bitemporal puede resolver:

```text
state valid at Tv
as known at Tt
```

Formalmente:

```text
Bitemporal(v,Tv,Tt)
⇔
Tv ∈ ValidInterval(v)
∧
Tt ∈ TransactionInterval(v)
```

---

# 275. Arquitectura completa

```text
Application / Repository / Model API
                 │
                 ▼
        Temporal Query Context
                 │
                 ▼
        Temporal Metadata Resolver
                 │
                 ▼
         Temporal Query Planner
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
   Current      AsOf      Between
      │          │          │
      └──────────┼──────────┘
                 ▼
         Temporal Semantics
                 │
                 ▼
             Query AST
                 │
                 ▼
         Semantic Analysis
                 │
                 ▼
            Optimizer
                 │
                 ▼
             Planner
                 │
                 ▼
             Compiler
                 │
                 ▼
          Execution Engine
                 │
                 ▼
             Database
                 │
                 ▼
              Result
                 │
                 ▼
        Temporal Hydration
                 │
                 ▼
      Current / Historical Entity
                 │
          ┌──────┴───────┐
          ▼              ▼
       MANAGED       READ_ONLY
```

---

# 276. Arquitectura bitemporal

```text
                    Logical Entity
                          │
                          ▼
                  Temporal Versions
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
        VALID TIME              TRANSACTION TIME
   "when was it true?"        "when did we know it?"
             │                         │
             └────────────┬────────────┘
                          ▼
                    Temporal View
                          │
                    (Tv, Tt)
                          │
                          ▼
                  Historical State
```

---

# 277. Ejemplo práctico

Supongamos precios:

```text
Product #100
```

con:

```text
$100 valid [Jan 1, Apr 1)
$120 valid [Apr 1, Jul 1)
$130 valid [Jul 1, +∞)
```

Consulta:

```php
$product = Product::query()
    ->asOf(Instant::parse('2026-05-15T12:00:00Z'))
    ->find(100);
```

deberá resolver:

```text
Price = $120
```

si esa es la dimensión temporal configurada.

---

# 278. Retroactive correction example

El 10 de junio descubrimos que:

```text
Price from Apr 1 should have been $115
```

En modelo bitemporal no es necesario borrar el hecho de que antes creíamos que era `$120`.

Podemos representar:

```text
Valid Time:
Apr 1 → Jul 1

Transaction knowledge:
old record known until Jun 10
new correction known from Jun 10
```

Esto permite preguntar:

```text
What was actually valid on May 15?
```

y también:

```text
What did the system believe on May 20?
```

Son preguntas diferentes.

---

# 279. Anti-patrones

## 279.1 Todo es DateTime

```php
DateTime $value;
```

para cualquier concepto temporal.

Problema:

```text
semantic ambiguity
```

---

## 279.2 Todo se convierte a UTC

UTC es apropiado para instants, pero no sustituye:

```text
LocalDate
LocalTime
BusinessCalendar
FutureLocalSchedule
TimeZoneId
```

---

## 279.3 created_at como historial

```text
created_at + updated_at
```

no proporciona historial de estados.

---

## 279.4 NULL siempre significa infinity

No universalmente.

---

## 279.5 now() por todas partes

Produce:

```text
non-determinism
test difficulty
inconsistent operation anchors
```

---

## 279.6 Historical entities managed as current

Puede sobrescribir estado actual accidentalmente.

---

## 279.7 SQL temporal dentro del ORM

Incorrecto:

```text
ORM → FOR SYSTEM_TIME AS OF ...
```

como string manual.

Correcto:

```text
ORM
→ Temporal Query
→ AST
→ Compiler
```

---

## 279.8 Cache sólo invalidado por writes

Incorrecto para resultados dependientes del tiempo.

---

## 279.9 Ignorar precisión

Puede romper:

```text
ordering
cursor pagination
optimistic checks
history boundaries
```

---

## 279.10 Inferir causalidad por timestamps

No es seguro en sistemas distribuidos.

---

# 280. Decisión arquitectónica

VoltStack adoptará un modelo temporal:

```text
semantic-first
typed
capability-driven
query-integrated
ORM-aware
tenant-aware
distribution-aware
cache-aware
persistent-runtime-safe
```

en lugar de reducir temporalidad a helpers de `DateTime`.

---

# 281. Resultado

Con este sistema VoltStack podrá representar de forma coherente:

```text
"ocurrió en T"
"es válido durante I"
"era válido en T"
"lo conocíamos en T"
"será válido desde T"
"esta relación existía en T"
"este resultado deja de ser válido en T"
```

manteniendo separadas las responsabilidades de:

```text
Type System
Temporal Semantics
History
Audit
Persistence
Query Engine
Compiler
Transactions
Cache
Retention
```

---

# 282. Regla definitiva

> **En VoltStack, el tiempo será una dimensión semántica explícita del modelo de datos y no un efecto accidental de almacenar fechas. Los tipos temporales describirán qué clase de tiempo representa un valor; los intervalos describirán su vigencia; el Temporal Query Context describirá desde qué perspectiva temporal se consulta; el Query Engine expresará esa intención mediante AST; y el Compiler decidirá cómo representarla sobre las capacidades reales de cada plataforma.**

Por tanto:

```text
Timestamp
≠
Temporal Model
```

```text
Instant
≠
LocalDateTime
```

```text
Offset
≠
TimeZone
```

```text
Duration
≠
Period
```

```text
Valid Time
≠
Transaction Time
```

```text
AsOf
≠
Database Snapshot
```

```text
History
≠
Audit
```

```text
Current
≠
now() scattered throughout code
```

y:

```text
Temporal Semantics
>
Physical Database Representation
```

---

# 283. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
□ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
□ 269_DATABASE_SOFT_DELETE_SYSTEM.md
□ 270_DATABASE_DATA_RETENTION_SYSTEM.md
□ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
□ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
□ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 284. Siguiente documento

```text
268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
```

El siguiente documento deberá construir sobre este modelo temporal para definir:

```text
Entity History
Version Identity
Version Sequence
Historical Snapshot
Change Version
Revision Metadata
Version Creation
Version Reconstruction
Version Query
Version Diff
Version Restore
Version Compaction
Version Storage
History Tables
Current vs Historical State
Optimistic Version ≠ Historical Version
Temporal Version ≠ Audit Event
```

y establecer como regla central:

> **El historial de VoltStack representará estados/versiones persistentes de una entidad a través del tiempo sin confundirlos con auditoría, locking optimista, snapshots del ORM ni logs de eventos; una versión histórica será un estado consultable con identidad y semántica explícitas, no simplemente una copia accidental de una fila anterior.**