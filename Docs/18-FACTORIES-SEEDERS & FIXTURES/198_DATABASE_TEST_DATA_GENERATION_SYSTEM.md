# 198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md

# VoltStack Quantum Database
## Database Test Data Generation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 198 — Database Test Data Generation System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `197_DATABASE_FIXTURE_SYSTEM.md`  
**Siguiente documento:** `199_DATABASE_PAGINATION_SYSTEM.md`

---

# 1. Propósito

`Database Test Data Generation System` define la arquitectura mediante la cual VoltStack podrá producir **datos sintéticos, reproducibles, configurables, tipados y conscientes del dominio** para pruebas unitarias, de integración, persistencia, rendimiento, concurrencia y escenarios extremos.

El sistema deberá ser capaz de generar desde un único valor:

```php
$email = $generator->email();
```

hasta datasets complejos:

```php
$dataset = $generator
    ->for(User::class)
    ->count(10_000)
    ->profile('realistic')
    ->seed(42001)
    ->generate();
```

o datos deliberadamente extremos:

```php
$dataset = $generator
    ->for(Order::class)
    ->profile('boundary')
    ->count(1_000)
    ->generate();
```

La regla central será:

> **Test Data Generator produce valores y datasets sintéticos bajo reglas explícitas de tipo, distribución, reproducibilidad, unicidad, privacidad y recursos; Factory construye objetos, Seeder orquesta población y Fixture construye escenarios conocidos.**

Formalmente:

```text
TestDataGenerator
=
Synthetic Value Generation
+
Distribution Model
+
Constraint Awareness
+
Deterministic Randomness
+
Dataset Planning
+
Resource Governance
```

Nunca:

```text
TestDataGenerator
=
Factory
```

ni:

```text
TestDataGenerator
=
Seeder
```

ni:

```text
TestDataGenerator
=
Fixture
```

ni:

```text
TestDataGenerator
=
Fuzzer
```

---

# 2. Cierre del Bloque 18

Con este documento queda completado:

```text
Block 18 — Factories, Seeders & Fixtures

193 DATABASE_FACTORY_SYSTEM
194 DATABASE_MODEL_FACTORY_SYSTEM
195 DATABASE_ENTITY_FACTORY_SYSTEM
196 DATABASE_SEEDER_SYSTEM
197 DATABASE_FIXTURE_SYSTEM
198 DATABASE_TEST_DATA_GENERATION_SYSTEM
```

La arquitectura conjunta será:

```text
                     Factory Engine
                          193
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        Model Factory             Entity Factory
            194                       195
              │                         │
              └────────────┬────────────┘
                           │
              ┌────────────┴─────────────┐
              ▼                          ▼
           Seeder                     Fixture
             196                        197
              │                          │
              └────────────┬─────────────┘
                           │
                           ▼
                  Test Data Generator
                           198
```

Sin embargo, ésta no es una jerarquía rígida.

Más correctamente:

```text
                 Test Data Generation
                        │
              Value/Data Providers
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
    Factory          Fixture         Seeder
       │                │                │
       └────────────────┼────────────────┘
                        ▼
                  Database APIs
```

---

# 3. Distinciones fundamentales

```text
Test Data Generator
≠
Factory
≠
Seeder
≠
Fixture
≠
Fuzzer
≠
Property-Based Test Runner
≠
Benchmark Runner
≠
Database Dump
≠
Production Data Clone
```

---

# 4. Test Data Generator vs Factory

Generator:

```php
$name = $generator->personName();
$email = $generator->email();
$age = $generator->integer(18, 80);
```

Factory:

```php
$user = UserEntityFactory::new()
    ->make();
```

Por tanto:

```text
Generator
=
Values / Synthetic Data

Factory
=
Object Construction
```

La Factory podrá consumir Generators.

---

# 5. Test Data Generator vs Seeder

Seeder responde:

```text
¿Qué dataset debo poblar?
```

Generator responde:

```text
¿Cómo genero los valores sintéticos requeridos?
```

Ejemplo:

```php
final class DevelopmentUserSeeder extends Seeder
{
    public function run(SeederContext $context): void
    {
        User::factory()
            ->count(10_000)
            ->create();
    }
}
```

La cadena puede ser:

```text
Seeder
 ↓
Factory
 ↓
Test Data Generator
```

---

# 6. Test Data Generator vs Fixture

Fixture:

```text
OrderWithExpiredPayment
```

debe ser un escenario semántico conocido.

Puede utilizar:

```text
Test Data Generator
```

para campos irrelevantes al escenario.

Ejemplo:

```text
customer.name = generated
customer.email = generated
payment.expiredAt = fixed scenario value
```

---

# 7. Test Data Generator vs Fuzzer

Un Fuzzer intenta explorar entradas inesperadas y encontrar fallos.

El Test Data Generator produce datos bajo perfiles conocidos.

```text
Generator
=
Controlled Synthetic Generation

Fuzzer
=
Failure-Oriented Input Exploration
```

Podrán integrarse en el futuro, pero no serán el mismo sistema.

---

# 8. Test Data Generator vs Property-Based Testing

Property-based testing ejecuta propiedades contra muchos inputs.

El Generator puede proporcionar esos inputs.

```text
Property Test Runner
        ↓
Test Data Generator
        ↓
Generated Cases
```

Pero:

```text
Generator
≠
Property Test Runner
```

---

# 9. Objetivos

El sistema deberá soportar:

1. generación determinista;
2. generación pseudoaleatoria;
3. tipos primitivos;
4. enums;
5. Value Objects;
6. JSON;
7. fechas;
8. identifiers;
9. strings;
10. datos localizados;
11. distributions;
12. uniqueness;
13. boundary values;
14. edge cases;
15. invalid values explícitos;
16. relationship-aware data;
17. metadata-aware generation;
18. large datasets;
19. lazy generation;
20. privacy-safe synthetic data;
21. reproducible CI;
22. parallel testing;
23. resource budgets;
24. telemetry;
25. extensibilidad.

---

# 10. Arquitectura general

```text
Generation Request
        │
        ▼
TestDataGenerator
        │
        ▼
GenerationContext
        │
        ├── Seed
        ├── RandomSource
        ├── Clock
        ├── Locale
        ├── Profile
        ├── Metadata
        ├── Constraints
        └── ResourceBudget
        │
        ▼
Generation Planner
        │
        ▼
GenerationPlan
        │
        ├── Value Providers
        ├── Distributions
        ├── Constraints
        ├── Dependencies
        └── Uniqueness Scopes
        │
        ▼
Generation Engine
        │
        ▼
Generated Values / Dataset
```

---

# 11. API principal

Conceptualmente:

```php
$generator = TestData::generator();
```

Ejemplos:

```php
$generator->integer(1, 100);

$generator->decimal(
    min: 0,
    max: 1000,
    scale: 2,
);

$generator->string(
    minLength: 5,
    maxLength: 50,
);

$generator->boolean();

$generator->uuid();

$generator->date();

$generator->email();
```

---

# 12. API tipada

La API deberá evitar depender únicamente de:

```php
$generator->fake('whatever');
```

Preferir contratos tipados:

```php
$generator->integer(...);
$generator->email(...);
$generator->instant(...);
```

---

# 13. ValueProvider

Unidad fundamental:

```php
interface ValueProvider
{
    public function generate(
        GenerationContext $context
    ): mixed;
}
```

---

# 14. Providers base

```text
ConstantProvider
RandomProvider
SequenceProvider
DerivedProvider
ReferenceProvider
UniqueProvider
ChoiceProvider
WeightedChoiceProvider
BoundaryProvider
NullProvider
ClockProvider
LocaleProvider
MetadataAwareProvider
```

---

# 15. ConstantProvider

```php
ConstantProvider::of('active');
```

siempre:

```text
active
```

---

# 16. SequenceProvider

Ejemplo:

```php
SequenceProvider::from([
    'basic',
    'premium',
    'enterprise',
]);
```

produce:

```text
basic
premium
enterprise
basic
...
```

según política.

---

# 17. DerivedProvider

Permite:

```text
email
=
normalize(name)
+
sequence
+
@example.test
```

Ejemplo conceptual:

```php
Derived::from(
    ['firstName', 'lastName'],
    fn ($first, $last, $context) =>
        strtolower("$first.$last@example.test")
);
```

---

# 18. Dependency graph

Los providers derivados forman dependencias:

```text
firstName ──┐
            ├──► fullName ──► username
lastName ───┘
```

---

# 19. Ciclos

```text
A depends B
B depends C
C depends A
```

deberán detectarse.

---

# 20. GenerationContext

Propuesta:

```php
final readonly class GenerationContext
{
    public function __construct(
        public GenerationSeed $seed,
        public RandomSource $random,
        public Clock $clock,
        public GenerationLocale $locale,
        public GenerationProfile $profile,
        public GenerationIndex $index,
        public GenerationScope $scope,
        public ResourceBudget $budget,
    ) {}
}
```

Podrá contener adicionalmente metadata/contexto de persistencia cuando sea necesario.

---

# 21. Estado scoped

Nunca:

```php
static $faker;
static $seed;
static $uniqueValues;
```

como estado global mutable.

---

# 22. Determinismo

Con:

```text
same algorithm
same seed
same provider configuration
same locale dataset/version
same generation index
```

el resultado deberá ser reproducible cuando la policy lo exija.

---

# 23. Seed

Ejemplo:

```php
$generator
    ->seed(48219)
    ->email();
```

---

# 24. No global `mt_srand()`

VoltStack no deberá alterar el random state global de PHP.

Se utilizará una abstracción:

```text
RandomSource
```

---

# 25. RandomSource

```php
interface RandomSource
{
    public function nextInt(
        int $min,
        int $max
    ): int;

    public function nextFloat(): float;

    public function bytes(int $length): string;
}
```

---

# 26. Pseudo-random vs secure random

Se distinguirá:

```text
DeterministicRandomSource
SecureRandomSource
```

---

# 27. Testing

Normalmente:

```text
DeterministicRandomSource
```

---

# 28. Security

Para credenciales temporales reales:

```text
SecureRandomSource
```

puede ser necesario.

Pero esto deberá ser explícito.

---

# 29. Secure randomness ≠ reproducible randomness

Nunca deberán intercambiarse silenciosamente.

---

# 30. Seed derivation

Para ejecución paralela:

```text
RootSeed
  │
  ├── Factory A
  │      ├── instance 0
  │      └── instance 1
  │
  └── Factory B
         ├── instance 0
         └── instance 1
```

cada scope podrá derivar su seed.

---

# 31. Seed derivation formal

Conceptualmente:

```text
ChildSeed =
H(
    RootSeed,
    TestId,
    WorkerId,
    FactoryId,
    GenerationIndex
)
```

---

# 32. No usar IDs de proceso como única fuente

`PID` puede cambiar entre ejecuciones.

Si participa, debe hacerlo solo cuando la policy no exige reproducción cross-process exacta.

---

# 33. GenerationIndex

Cada instancia tendrá:

```text
0
1
2
3
...
```

dentro de un scope definido.

---

# 34. Parallel determinism

La generación no debería cambiar simplemente porque:

```text
CI workers = 4
```

pasa a:

```text
CI workers = 8
```

cuando se solicite una política de determinismo independiente de scheduling.

---

# 35. Scheduling-independent generation

Idealmente el valor de un caso se deriva de su identidad lógica:

```text
CaseId
```

no del orden real de ejecución de threads/workers.

---

# 36. Locale

Ejemplo:

```php
$generator->locale('es_MX');
```

podrá afectar:

```text
names
addresses
phone formats
postal formats
localized text
```

---

# 37. Locale ≠ timezone

Separados:

```text
Locale
Timezone
```

---

# 38. Locale datasets

Los datasets deberán estar versionados.

Porque:

```text
same seed
+
different provider dataset
=
different result
```

---

# 39. Reproducibility descriptor

Un test failure podrá registrar:

```text
seed: 581992
locale: es_MX
provider-dataset: 3
generator-version: 2
profile: boundary
```

---

# 40. Failure reproduction

Ejemplo:

```bash
php voltstack test \
    --replay-seed=581992
```

La integración concreta pertenecerá al Testing System.

---

# 41. Profiles

VoltStack deberá soportar perfiles.

```php
enum GenerationProfile
{
    case MINIMAL;
    case REALISTIC;
    case BOUNDARY;
    case EDGE_CASE;
    case INVALID;
    case PERFORMANCE;
    case CUSTOM;
}
```

---

# 42. MINIMAL

Genera el mínimo necesario para satisfacer restricciones.

Útil para tests rápidos.

---

# 43. REALISTIC

Genera datos plausibles.

Ejemplo:

```text
name:
    "Ana Martínez"

email:
    "ana.martinez42@example.test"
```

---

# 44. BOUNDARY

Prioriza límites:

```text
min
max
min + 1
max - 1
zero
empty
maximum length
```

---

# 45. EDGE_CASE

Incluye:

```text
Unicode
emoji
combining characters
long strings
zero
negative numbers
DST transitions
leap years
empty arrays
deep JSON
```

según tipo.

---

# 46. INVALID

Genera deliberadamente datos que violan una restricción concreta.

Debe ser explícito.

---

# 47. Regla importante

Los defaults deberán generar:

```text
valid domain data
```

No datos inválidos accidentalmente.

---

# 48. Invalid generation

Ejemplo:

```php
$generator
    ->invalid()
    ->violating('email.format')
    ->generate();
```

---

# 49. Violación controlada

Idealmente:

```text
violate exactly one intended constraint
```

cuando sea posible.

---

# 50. Razón

Si queremos probar:

```text
invalid email
```

no queremos además:

```text
missing required name
invalid role
negative age
```

salvo que el escenario lo solicite.

---

# 51. Constraint Model

Propuesta:

```text
GenerationConstraint
├── Required
├── Nullable
├── Min
├── Max
├── Length
├── Pattern
├── Choice
├── Unique
├── Precision
├── Scale
├── TemporalRange
└── Custom
```

---

# 52. Constraint sources

Podrán venir de:

```text
Factory definition
ORM metadata
Type metadata
Validation integration
Schema metadata
explicit generator configuration
```

---

# 53. Precedencia

No deberán fusionarse ciegamente.

Propuesta:

```text
Explicit Generation Rule
        ↓
Factory/Fixture Rule
        ↓
Domain/Validation Metadata
        ↓
ORM Metadata
        ↓
Schema Metadata
        ↓
Generic Type Defaults
```

---

# 54. Schema constraint ≠ domain constraint

Ejemplo:

```text
VARCHAR(255)
```

no significa que el dominio permita cualquier string de 255 caracteres.

---

# 55. ORM nullable ≠ business optional

Igualmente:

```text
nullable database column
```

no implica:

```text
domain field optional
```

---

# 56. Metadata conflict

Si diferentes fuentes contradicen:

```text
min = 10
max = 5
```

deberá fallar durante planning.

---

# 57. Constraint provenance

El sistema deberá poder explicar:

```text
email.maxLength = 150

source:
    Validation metadata
```

---

# 58. Type awareness

Debe integrarse con:

```text
155_DATABASE_TYPE_SYSTEM
156_DATABASE_TYPE_REGISTRY_SYSTEM
157_DATABASE_VALUE_CONVERSION_SYSTEM
158_DATABASE_CASTING_SYSTEM
```

---

# 59. Integer generation

```php
$generator->integer(
    min: -100,
    max: 100
);
```

---

# 60. Decimal generation

Debe respetar:

```text
precision
scale
range
```

sin depender de imprecisiones float cuando la semántica requiera decimal exacto.

---

# 61. String generation

Dimensiones:

```text
length
encoding
character set
pattern
locale
normalization
```

---

# 62. Unicode

El sistema deberá soportar casos como:

```text
á
ñ
漢字
العربية
😀
é
```

para edge-case testing cuando corresponda.

---

# 63. Binary

Podrá generar:

```text
bytes
blobs
checksums
```

con límites explícitos.

---

# 64. Boolean

Podrá utilizar:

```text
uniform
weighted
fixed
sequence
```

---

# 65. Enum

Integración con:

```text
159_DATABASE_ENUM_MAPPING_SYSTEM
```

Ejemplo:

```php
$generator->enum(OrderStatus::class);
```

---

# 66. Enum weighting

```php
$generator->weightedEnum(
    OrderStatus::class,
    [
        OrderStatus::PENDING => 0.50,
        OrderStatus::PAID => 0.40,
        OrderStatus::CANCELLED => 0.10,
    ]
);
```

---

# 67. Value Objects

Integración con:

```text
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM
```

Ejemplo:

```php
$generator->valueObject(Money::class);
```

mediante provider registrado.

---

# 68. JSON

Integración con:

```text
161_DATABASE_JSON_TYPE_SYSTEM
```

Podrá generar:

```text
objects
arrays
nested structures
null
missing keys
boundary depth
boundary size
```

---

# 69. JSON null ≠ missing

Debe conservarse:

```text
JSON null
≠
missing property
```

---

# 70. Date/time

Integración con:

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM
```

deberá distinguir:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
Duration
```

---

# 71. Temporal edge cases

Perfiles edge podrán generar:

```text
leap day
end of month
year boundary
DST gap
DST overlap
Unix epoch
far future
far past
```

dentro de capacidades configuradas.

---

# 72. Clock-relative values

Ejemplo:

```php
$generator->instant()
    ->between(
        $context->clock()->now()->minusDays(30),
        $context->clock()->now()
    );
```

---

# 73. Identifiers

Podrá generar:

```text
UUID
ULID
integer IDs
string IDs
composite ID components
custom identifiers
```

---

# 74. Database-generated IDs

No deberá intentar predecir:

```text
AUTO_INCREMENT
SEQUENCE
IDENTITY
```

salvo estrategia explícita.

---

# 75. Identifier Generator ≠ Database Identifier Generator

Un ID sintético puede generarse en aplicación.

Un ID DB-generated pertenece a persistence execution.

---

# 76. Uniqueness

Una necesidad frecuente:

```php
$generator
    ->unique()
    ->email();
```

---

# 77. UniqueRegistry

Cada scope tendrá:

```text
UniqueRegistry
```

---

# 78. Scope

```text
VALUE_PROVIDER
FACTORY
FIXTURE
SEEDER
TEST
DATASET
CUSTOM
```

---

# 79. No global uniqueness

Nunca:

```text
all generated emails across process lifetime
```

como default.

---

# 80. Unique ≠ database guaranteed unique

Incluso si el generator evita duplicados localmente:

```text
GeneratorUnique
≠
DatabaseUniqueConstraintGuarantee
```

Otro proceso puede insertar el mismo valor.

---

# 81. DB sigue siendo autoridad

La constraint DB sigue siendo la protección final de persistencia.

---

# 82. Exhaustion

Dominio:

```text
{A, B, C}
```

y solicitud:

```text
4 unique values
```

debe producir:

```text
GenerationUniquenessExhaustedException
```

no loop infinito.

---

# 83. Attempt budget

Providers de uniqueness tendrán:

```text
maxAttempts
```

---

# 84. Deterministic uniqueness

El algoritmo deberá mantener reproducibilidad bajo seed conocida cuando sea posible.

---

# 85. ChoiceProvider

```php
$generator->choice([
    'basic',
    'premium',
    'enterprise',
]);
```

---

# 86. WeightedChoice

```text
basic       70%
premium     25%
enterprise   5%
```

---

# 87. Distribution system

Además de uniforme:

```text
UNIFORM
NORMAL
WEIGHTED
ZIPF
SEQUENCE
CUSTOM
```

podrán soportarse mediante extensiones.

---

# 88. Distribution ≠ random source

`RandomSource` genera entropía/pseudoentropía.

`Distribution` transforma esa secuencia según un modelo estadístico.

---

# 89. Normal distribution

Útil para:

```text
ages
prices
response times
order sizes
```

cuando el dataset lo requiera.

---

# 90. Clamping

Una distribución normal con límites deberá definir qué ocurre fuera del rango:

```text
CLAMP
RESAMPLE
REJECT
TRANSFORM
```

---

# 91. No sesgo accidental

El algoritmo deberá documentar el efecto estadístico de clipping/resampling cuando importe.

---

# 92. Dataset Profile

Podrá definirse:

```php
final class EcommerceDatasetProfile
{
    // distribution rules
}
```

---

# 93. Ejemplo

```text
Users:
    100,000

Orders/User:
    Zipf distribution

Order Status:
    Paid       70%
    Pending    15%
    Cancelled  10%
    Refunded    5%
```

---

# 94. Dataset profile ≠ Fixture

Fixture define escenario conocido.

Dataset Profile define características de una población sintética.

---

# 95. Dataset Definition

Propuesta:

```php
$dataset = DatasetDefinition::make()
    ->entity(User::class, 100_000)
    ->entity(Product::class, 10_000)
    ->entity(Order::class, 500_000);
```

---

# 96. Relationship-aware generation

El sistema podrá usar metadata de:

```text
142–154 Relationship System
```

para generar grafos coherentes.

---

# 97. Ejemplo

```text
User
 ├── Orders
 │    └── OrderLines
 │         └── Product
 └── Addresses
```

---

# 98. Relationship Generation Plan

```text
RelationshipMetadata
        ↓
Cardinality Rules
        ↓
Generation Distribution
        ↓
RelationshipGenerationPlan
```

---

# 99. No inferir reglas de negocio solo desde FK

Una FK:

```text
orders.user_id
```

no nos dice cuántos Orders debe tener un User.

Eso requiere:

```text
Generation Profile
Factory definition
explicit rule
```

---

# 100. Cardinality distribution

Ejemplo:

```text
Orders per User

0 orders    25%
1–3         50%
4–10        20%
>10          5%
```

---

# 101. Many-to-many

Ejemplo:

```text
Users ↔ Roles
```

podrá generarse mediante membership distributions.

---

# 102. Relationship ownership

La generación deberá respetar metadata ORM.

No adivinar ownership desde nombres de columnas cuando existe metadata.

---

# 103. Polymorphic relations

Deberán utilizar:

```text
stable polymorphic aliases
```

del Relationship System.

Nunca FQCN arbitrarios persistidos.

---

# 104. Cyclic object graphs

Ejemplo:

```text
Employee
 ↓ manager
Employee
```

requiere límites.

---

# 105. Graph Depth

```text
maxDepth
maxNodes
maxChildren
```

serán parte de resource governance.

---

# 106. Cycle policy

```text
REUSE_EXISTING
STOP
REFERENCE
REJECT
CUSTOM
```

---

# 107. Generated ID barriers

Si una relación requiere ID DB-generated:

```text
Parent
 ↓ persist/flush
Generated ID
 ↓
Child
```

la Factory/Persistence bridge podrá introducir barrera.

El Generator no ejecutará flush por sí mismo.

---

# 108. Generator no persiste

Regla crítica:

> **Generation y Persistence son fases diferentes.**

```text
Generate
≠
Persist
```

---

# 109. GeneratedDataset

El resultado podrá ser:

```php
final readonly class GeneratedDataset
{
    public function __construct(
        public DatasetId $id,
        public GenerationSeed $seed,
        public iterable $records,
        public DatasetStatistics $statistics,
        public GenerationDescriptor $descriptor,
    ) {}
}
```

---

# 110. Dataset materialization

Podrá existir:

```text
MATERIALIZED
LAZY
STREAMING
```

---

# 111. Materialized

```text
generate all
→
memory
```

adecuado para datasets pequeños.

---

# 112. Lazy

```text
generate when iterated
```

---

# 113. Streaming

```text
generate
→
consume
→
discard
```

para grandes datasets.

---

# 114. Streaming determinism

La secuencia deberá conservar determinismo incluso sin materialización cuando la policy lo exija.

---

# 115. Single-pass stream

Un stream podrá ser:

```text
single pass
```

y deberá declararlo.

---

# 116. Rewindable generation

Si se necesita iterar nuevamente, el sistema puede reconstruir desde:

```text
seed
+
generation descriptor
```

en vez de almacenar todos los registros.

---

# 117. Memory model

Para:

```text
10,000,000 rows
```

no deberá requerirse necesariamente:

```text
10,000,000 PHP objects simultaneously
```

---

# 118. Block 19 integration

Esto conecta directamente con:

```text
199 Pagination
200 Cursor Pagination
201 Chunk Processing
202 Lazy Collection
203 Bulk Insert
...
208 Large Dataset Processing
```

---

# 119. Large dataset pipeline

```text
DatasetDefinition
       ↓
GenerationPlan
       ↓
Lazy Generator
       ↓
Chunk
       ↓
Bulk Insert
       ↓
Database
```

---

# 120. Batch size

Ejemplo:

```text
generation batch = 1000
```

No deberá confundirse con:

```text
transaction batch
```

---

# 121. Generation batch ≠ transaction boundary

Siempre:

```text
GenerationChunk
≠
Transaction
```

---

# 122. Backpressure

El Generator deberá poder producir datos al ritmo del consumidor.

```text
Producer
 ↓
bounded buffer
 ↓
Consumer
```

---

# 123. Cancellation

Generación grande deberá aceptar:

```text
CancellationToken
```

---

# 124. ResourceBudget

Propuesta:

```php
final readonly class GenerationResourceBudget
{
    public function __construct(
        public ?int $maxRecords,
        public ?int $maxGraphNodes,
        public ?int $maxMemoryBytes,
        public ?int $maxDurationMs,
        public ?int $maxUniqueAttempts,
        public ?int $maxDepth,
    ) {}
}
```

---

# 125. Budget enforcement

Debe evitar:

```text
runaway graph generation
infinite uniqueness loops
unbounded JSON depth
unbounded strings
unbounded dataset materialization
```

---

# 126. Performance Profile

Para benchmarks:

```text
PERFORMANCE
```

puede generar millones de registros con distribuciones representativas.

---

# 127. Benchmark reproducibility

Un benchmark debe registrar:

```text
generator version
seed
dataset profile
record count
schema generation
platform
```

para comparación válida.

---

# 128. Generator performance ≠ DB performance

Al medir DB:

```text
data generation time
```

debe separarse de:

```text
database insertion/query time
```

---

# 129. Pre-generation

Benchmarks podrán:

```text
generate dataset
freeze descriptor/snapshot
benchmark database
```

para evitar medir generator accidentalmente.

---

# 130. Privacy

El sistema deberá favorecer:

```text
synthetic data
```

en testing.

---

# 131. Production PII

Nunca deberá utilizar producción como fuente implícita.

---

# 132. Production data synthesis

Un sistema futuro podría aprender distribuciones de producción de forma:

```text
aggregated
anonymized
privacy-preserving
```

pero será una integración especializada.

---

# 133. No copiar datos reales por defecto

Nunca:

```text
Generator
→
SELECT * FROM production.users
```

como mecanismo estándar.

---

# 134. Synthetic email domains

Preferir:

```text
example.com
example.net
example.org
```

o dominios de prueba configurados.

---

# 135. Synthetic phones

No deberán intentar representar números reales sin necesidad.

---

# 136. Synthetic payment data

No deberán generarse accidentalmente credenciales financieras reales.

---

# 137. Sensitive fields

Metadata podrá marcar:

```text
SENSITIVE
SECRET
PII
FINANCIAL
TOKEN
```

para seleccionar providers seguros.

---

# 138. Sensitive provider

Ejemplo:

```text
password
→
synthetic test password/hash

API token
→
synthetic non-production token
```

---

# 139. Secret telemetry

Valores generados sensibles no deberán aparecer en:

```text
logs
metrics
traces
exceptions
```

por defecto.

---

# 140. Validation integration

VoltStack podrá integrar:

```text
Validation System
```

para generar valores que satisfagan constraints.

---

# 141. Pero

```text
Generator
≠
Validator
```

---

# 142. Generation-time validation

Podrá existir:

```text
GENERATE_ONLY
GENERATE_AND_VALIDATE
VALIDATE_SAMPLE
```

---

# 143. GENERATE_AND_VALIDATE

Útil para verificar que providers realmente satisfacen reglas.

---

# 144. Performance tradeoff

Validar millones de registros puede ser costoso.

Por ello:

```text
VALIDATE_SAMPLE
```

puede utilizarse para performance datasets.

---

# 145. Invalid profile validation

Para perfil INVALID, el sistema deberá validar que se violó la restricción objetivo cuando sea posible.

---

# 146. Schema awareness

El Generator podrá consultar metadata compilada:

```text
column length
nullable
precision
scale
type
```

pero no deberá depender de introspection DB en cada valor generado.

---

# 147. Preferred flow

```text
Schema/ORM metadata
        ↓
compiled Generation Metadata
        ↓
Generation Plan
```

---

# 148. Metadata cache

La metadata compilada podrá utilizar:

```text
189_DATABASE_METADATA_CACHE_SYSTEM
```

---

# 149. Generation cache

Se podrán cachear:

```text
compiled providers
constraint plans
dataset profiles
locale datasets
relationship generation plans
```

---

# 150. Nunca cachear como metadata

```text
current random state
current unique registry
generated object instances
current test references
```

---

# 151. Generation Registry

Propuesta:

```text
TestDataProviderRegistry
```

---

# 152. Registry lifecycle

Durante bootstrap:

```text
register
validate
compile
freeze
```

---

# 153. Runtime mutation

Después de freeze:

```text
register provider dynamically
```

deberá ser rechazado o requerir un nuevo registry generation.

---

# 154. Custom provider

```php
final class MexicanRfcProvider implements ValueProvider
{
    public function generate(
        GenerationContext $context
    ): string {
        // ...
    }
}
```

---

# 155. Registration

Conceptualmente:

```php
$registry->register(
    'mx.rfc',
    MexicanRfcProvider::class
);
```

---

# 156. ProviderId

Preferir identidad estable:

```text
mx.rfc
commerce.sku
internet.email
person.name
```

---

# 157. ProviderId ≠ FQCN

Permite cambiar implementación sin alterar dataset definition.

---

# 158. Provider version

Un provider podrá declarar:

```text
ProviderVersion
```

para reproducibilidad.

---

# 159. Extension conflicts

Dos paquetes intentando registrar:

```text
internet.email
```

deberán producir conflicto determinista salvo override policy explícita.

---

# 160. Third-party Faker integration

VoltStack podrá proporcionar bridges hacia bibliotecas Faker.

Pero:

> El core de Database Test Data Generation no dependerá obligatoriamente de una biblioteca Faker externa.

---

# 161. Adapter

Conceptualmente:

```text
FakerAdapter
implements
TestDataProvider
```

---

# 162. External provider reproducibility

VoltStack no podrá prometer reproducción exacta cross-version si la biblioteca externa cambia sus datasets/algoritmos.

Debe registrar:

```text
provider
version
locale
```

---

# 163. Provider capability

Un provider podrá declarar:

```text
DETERMINISTIC
SECURE_RANDOM
LOCALIZED
UNIQUE_CAPABLE
STREAM_SAFE
PARALLEL_SAFE
```

---

# 164. Capability validation

Un perfil STRICT no deberá utilizar un provider:

```text
NON_DETERMINISTIC
```

sin override explícito.

---

# 165. Thread/coroutine safety

Providers compartidos deberán ser:

```text
stateless
```

o recibir todo estado desde `GenerationContext`.

---

# 166. Persistent runtime

Con FrankenPHP:

```text
Request/Test A
seed=100

Request/Test B
seed=200
```

no deberá existir contaminación.

---

# 167. RoadRunner/OpenSwoole

Misma regla:

```text
worker reuse
≠
generation state reuse
```

---

# 168. UniqueRegistry persistent worker

Debe destruirse/resetearse al terminar el scope.

---

# 169. Factory integration architecture

```text
FactoryDefinition
       │
       ├── constants
       ├── states
       ├── sequences
       └── TestDataProviders
                │
                ▼
        GenerationContext
                │
                ▼
          Factory Engine
```

---

# 170. Ejemplo

```php
final class UserFactory extends ModelFactory
{
    protected function definition(
        FactoryContext $context
    ): array {
        return [
            'name' => TestData::person()->name(),
            'email' => TestData::internet()->uniqueEmail(),
            'age' => TestData::integer(18, 80),
        ];
    }
}
```

---

# 171. Lazy provider

Las expresiones anteriores deberán representar providers/deferred values cuando sea necesario.

No generar todos los valores al definir la Factory.

---

# 172. Per-instance evaluation

```text
Factory Definition
       ↓
Instance 0 → providers evaluate
Instance 1 → providers evaluate
Instance 2 → providers evaluate
```

---

# 173. Seeder integration

```text
Seeder
  ↓
Factory
  ↓
TestDataGenerator
```

o:

```text
Seeder
  ↓
DatasetGenerator
  ↓
Bulk Insert
```

---

# 174. Fixture integration

```text
Fixture
  ↓
Fixed scenario fields
+
Generated irrelevant fields
```

---

# 175. Ejemplo

```php
$order = OrderEntityFactory::new()
    ->state([
        'number' => $context->data()->unique()->orderNumber(),
        'status' => OrderStatus::PENDING,
        'expiresAt' => $context->clock()->now()->minusHour(),
    ])
    ->make();
```

Aquí:

```text
number
=
synthetic

status/expiresAt
=
scenario-defined
```

---

# 176. Property-based integration

Conceptualmente:

```php
property(
    generator: TestData::integer(-1000, 1000),
    test: function (int $value) {
        // property
    }
);
```

El futuro Testing System podrá utilizarlo.

---

# 177. Shrinking

Property-based frameworks suelen reducir un failing input.

```text
100000
 ↓
1000
 ↓
100
 ↓
1
```

`Shrinker` será una extensión posible.

---

# 178. Generator ≠ Shrinker

Separación:

```text
Generator
produces candidates

Shrinker
minimizes failing candidate
```

---

# 179. Fuzz integration

Igualmente:

```text
Fuzzer
 ↓
Generator providers
 ↓
mutations
 ↓
test target
```

sin fusionar subsistemas.

---

# 180. Mutation testing

No pertenece a Database Generator.

---

# 181. Referential integrity

Dataset generation podrá planificar relaciones válidas.

---

# 182. Pero DB sigue siendo autoridad

```text
Generator believes valid
```

no equivale a:

```text
Database accepted
```

---

# 183. Unique constraints

El Generator puede evitar colisiones conocidas.

La DB verifica finalmente.

---

# 184. Foreign keys

El Relationship Generation Planner puede generar referencias coherentes.

La DB verifica finalmente.

---

# 185. Check constraints

Podrán formar parte de metadata cuando sean introspectables/representables.

Pero:

```text
NotUnderstood
≠
Satisfied
```

---

# 186. Unknown constraints

Si una DB constraint no puede representarse:

```text
ConstraintCoverage = UNKNOWN/PARTIAL
```

no deberá fingirse generación garantizada.

---

# 187. Generation Confidence

Podrá existir:

```text
GUARANTEED_BY_GENERATOR
VALIDATED
BEST_EFFORT
UNKNOWN
```

---

# 188. `GUARANTEED_BY_GENERATOR` alcance

Significa únicamente respecto a las reglas que el Generator conoce y controla.

No sustituye DB validation.

---

# 189. Diagnostics

API:

```php
TestData::explain(User::class);
```

podrá producir:

```text
TEST DATA GENERATION PLAN

Target:
    User

Profile:
    realistic

Seed:
    581992

Providers:
    name:
        person.name
    email:
        internet.email.unique
    age:
        integer.range

Constraints:
    name.maxLength = 120
    email.maxLength = 150
    age.min = 18

Relationships:
    addresses: 0..3

Estimated graph:
    1–4 nodes

Reproducibility:
    deterministic
```

---

# 190. Dataset explain

```text
DATASET PLAN

Profile:
    ecommerce-large

Users:
    100,000

Products:
    25,000

Orders:
    500,000

OrderLines:
    estimated 1,900,000

Generation:
    streaming

Chunk:
    2,000

Estimated memory:
    bounded

Seed:
    182771

Parallel safe:
    yes
```

---

# 191. Telemetry

Eventos conceptuales:

```text
GenerationPlanCreated
GenerationStarted
GenerationBatchCompleted
GenerationUniquenessRetry
GenerationBudgetWarning
GenerationCompleted
GenerationFailed
GenerationCancelled
```

---

# 192. Métricas

```text
db.testdata.generated_records
db.testdata.generation_duration
db.testdata.failures
db.testdata.uniqueness_retries
db.testdata.budget_exceeded
db.testdata.graph_nodes
```

---

# 193. No high-cardinality labels

Nunca:

```text
generated email
entity ID
raw seed
tenant ID
random value
```

como labels no controlados.

---

# 194. Seed en diagnostics

El seed sí puede ser necesario para reproducción, pero debe almacenarse como diagnostic/tracing field apropiado, no necesariamente como metric label.

---

# 195. Errors

Jerarquía propuesta:

```text
DatabaseException
└── TestDataGenerationException
    ├── GenerationConfigurationException
    ├── GenerationPlanningException
    ├── GenerationProviderException
    ├── GenerationConstraintException
    ├── GenerationConstraintConflictException
    ├── GenerationDependencyCycleException
    ├── GenerationUniquenessException
    │   └── GenerationUniquenessExhaustedException
    ├── GenerationResourceBudgetException
    ├── GenerationReproducibilityException
    ├── GenerationLocaleException
    ├── GenerationTypeException
    ├── GenerationRelationshipException
    └── GenerationCancelledException
```

---

# 196. Directory structure

Propuesta:

```text
src/Quantum/Database/TestData/
│
├── TestDataGenerator.php
├── GenerationContext.php
├── GenerationSeed.php
├── GenerationIndex.php
├── GenerationProfile.php
├── GenerationDescriptor.php
│
├── Random/
│   ├── RandomSource.php
│   ├── DeterministicRandomSource.php
│   ├── SecureRandomSource.php
│   └── SeedDeriver.php
│
├── Provider/
│   ├── ValueProvider.php
│   ├── ProviderId.php
│   ├── ProviderRegistry.php
│   ├── ConstantProvider.php
│   ├── SequenceProvider.php
│   ├── DerivedProvider.php
│   ├── ChoiceProvider.php
│   ├── WeightedChoiceProvider.php
│   ├── UniqueProvider.php
│   └── BoundaryProvider.php
│
├── Constraint/
│   ├── GenerationConstraint.php
│   ├── ConstraintSet.php
│   ├── ConstraintResolver.php
│   ├── ConstraintProvenance.php
│   └── ConstraintValidator.php
│
├── Distribution/
│   ├── Distribution.php
│   ├── UniformDistribution.php
│   ├── NormalDistribution.php
│   ├── WeightedDistribution.php
│   └── ZipfDistribution.php
│
├── Type/
│   ├── IntegerGenerator.php
│   ├── DecimalGenerator.php
│   ├── StringGenerator.php
│   ├── BooleanGenerator.php
│   ├── EnumGenerator.php
│   ├── JsonGenerator.php
│   ├── TemporalGenerator.php
│   └── ValueObjectGenerator.php
│
├── Unique/
│   ├── UniqueRegistry.php
│   ├── UniqueScope.php
│   └── UniquePolicy.php
│
├── Dataset/
│   ├── DatasetDefinition.php
│   ├── DatasetId.php
│   ├── DatasetProfile.php
│   ├── GeneratedDataset.php
│   ├── DatasetGenerator.php
│   └── DatasetStatistics.php
│
├── Relationship/
│   ├── RelationshipGenerationPlanner.php
│   ├── RelationshipGenerationPlan.php
│   ├── CardinalityDistribution.php
│   └── GraphGenerationPolicy.php
│
├── Planning/
│   ├── GenerationPlanner.php
│   └── GenerationPlan.php
│
├── Locale/
│   ├── GenerationLocale.php
│   ├── LocaleDataset.php
│   └── LocaleProviderRegistry.php
│
├── Resource/
│   └── GenerationResourceBudget.php
│
├── Diagnostics/
│   ├── GenerationInspector.php
│   └── GenerationExplainer.php
│
├── Telemetry/
│   └── GenerationTelemetry.php
│
└── Exception/
    └── ...
```

---

# 197. Dependencias internas

```text
TestData
   │
   ├── Support
   ├── Type metadata
   ├── ORM metadata (optional)
   ├── Relationship metadata (optional)
   ├── Validation metadata (optional)
   └── Schema metadata (optional)
```

Nunca:

```text
TestData
→
Driver internals
```

ni:

```text
TestData
→
SQL Compiler
```

---

# 198. Dependency direction

```text
Factory
   ↓
TestData

Fixture
   ↓
Factory / TestData

Seeder
   ↓
Factory / TestData
```

El Generator no deberá depender de Seeder/Fixture.

---

# 199. Arquitectura de extensiones

```text
ProviderRegistry
├── Core Providers
├── Locale Providers
├── Application Providers
└── Package Providers
```

---

# 200. Custom domain generator

Ejemplo:

```php
final class MoneyGenerator implements ValueProvider
{
    public function generate(
        GenerationContext $context
    ): Money {
        $amount = $context->random()
            ->nextInt(100, 100_000);

        return Money::mxn($amount);
    }
}
```

---

# 201. IDE support

Cuando sea posible:

```php
/** @return ValueProvider<Money> */
```

o mediante generics compatibles con analizadores estáticos.

---

# 202. Testing del propio Generator

VoltStack deberá probar:

```text
determinism
distribution
constraints
uniqueness
locale
boundary generation
type correctness
parallel isolation
resource budgets
streaming
provider extensions
```

---

# 203. Determinism test

```text
Seed = 100
Provider = integer(1,100)

Run A:
42, 17, 81...

Run B:
42, 17, 81...
```

bajo misma versión/configuración.

---

# 204. Different seed

```text
Seed 100
≠
Seed 101
```

deberá normalmente producir secuencias distintas.

No es requisito matemático que cada valor individual sea distinto.

---

# 205. Distribution test

Para:

```text
70% A
30% B
```

no se deberá afirmar:

```text
exactly 700 A
exactly 300 B
```

en una muestra aleatoria de 1000 salvo distribución determinística por cuotas.

---

# 206. Statistical tolerance

Tests de distribuciones aleatorias utilizarán tolerancias estadísticas apropiadas.

---

# 207. Boundary profile test

Para:

```text
integer 1..100
```

deberá priorizar casos como:

```text
1
2
99
100
```

según algoritmo.

---

# 208. Invalid profile test

Si se solicita:

```text
violate maxLength
```

el valor generado deberá exceder `maxLength` sin violar otras reglas innecesariamente cuando sea posible.

---

# 209. Uniqueness exhaustion test

```text
domain = {A,B}
requested unique = 3
```

deberá terminar con error controlado.

Nunca loop infinito.

---

# 210. Parallel test

Dos contexts:

```text
Context A
seed=100

Context B
seed=200
```

no compartirán:

```text
UniqueRegistry
RandomSource mutable state
GenerationIndex
```

---

# 211. Persistent worker test

```text
Execution A
→ generate

reset

Execution B
→ generate
```

B no deberá depender del state final de A.

---

# 212. Streaming test

Generar:

```text
1,000,000 records
```

deberá mantener memoria acotada bajo modo streaming.

---

# 213. Cancellation test

Cancelar después de:

```text
50,000
```

deberá detener generación sin afirmar que:

```text
dataset complete
```

---

# 214. Architectural invariants

## DB-TDG-001
TestDataGenerator generará datos sintéticos.

## DB-TDG-002
TestDataGenerator no será Factory.

## DB-TDG-003
TestDataGenerator no será Seeder.

## DB-TDG-004
TestDataGenerator no será Fixture.

## DB-TDG-005
TestDataGenerator no será Fuzzer.

## DB-TDG-006
TestDataGenerator no será Property-Based Test Runner.

## DB-TDG-007
TestDataGenerator no será Benchmark Runner.

## DB-TDG-008
TestDataGenerator no será Database Dump loader.

## DB-TDG-009
Generation no implicará persistence.

## DB-TDG-010
Generator no ejecutará SQL.

## DB-TDG-011
Generator no accederá directamente a Driver.

## DB-TDG-012
Generator no administrará Connection.

## DB-TDG-013
Generator no administrará Transaction.

## DB-TDG-014
Factories podrán consumir Generator.

## DB-TDG-015
Fixtures podrán consumir Generator.

## DB-TDG-016
Seeders podrán consumir Generator.

## DB-TDG-017
Generator no dependerá de Seeder.

## DB-TDG-018
Generator no dependerá de Fixture.

## DB-TDG-019
GenerationContext será scoped.

## DB-TDG-020
RandomSource será scoped.

## DB-TDG-021
UniqueRegistry será scoped.

## DB-TDG-022
GenerationIndex será scoped.

## DB-TDG-023
No se modificará random state global de PHP.

## DB-TDG-024
Deterministic randomness será distinta de secure randomness.

## DB-TDG-025
Secure randomness no prometerá reproducibilidad.

## DB-TDG-026
Seed derivation será determinista bajo policy STRICT.

## DB-TDG-027
Scheduling no deberá alterar generación cuando se solicite scheduling-independent determinism.

## DB-TDG-028
Worker identity no será la única fuente de seed.

## DB-TDG-029
Locale será distinto de timezone.

## DB-TDG-030
Locale datasets serán versionables.

## DB-TDG-031
Provider versions podrán participar en reproducción.

## DB-TDG-032
Default profile generará datos válidos.

## DB-TDG-033
INVALID profile será explícito.

## DB-TDG-034
Invalid generation intentará violar la constraint objetivo.

## DB-TDG-035
Boundary generation será explícita.

## DB-TDG-036
Edge-case generation será explícita.

## DB-TDG-037
Constraint sources conservarán provenance.

## DB-TDG-038
Schema constraint no será igual a domain constraint.

## DB-TDG-039
ORM nullable no será igual a domain optional.

## DB-TDG-040
Constraint conflicts no se resolverán silenciosamente.

## DB-TDG-041
Type System será fuente canónica de tipos DB/ORM relevantes.

## DB-TDG-042
Decimal generation respetará precision/scale.

## DB-TDG-043
String generation respetará encoding/length.

## DB-TDG-044
JSON null será distinto de missing.

## DB-TDG-045
Temporal types permanecerán distintos.

## DB-TDG-046
Generator no predecirá IDs DB-generated por defecto.

## DB-TDG-047
Generator uniqueness no sustituirá DB unique constraints.

## DB-TDG-048
Uniqueness tendrá scope explícito.

## DB-TDG-049
Uniqueness no será global al proceso por defecto.

## DB-TDG-050
Uniqueness tendrá attempt budget.

## DB-TDG-051
Uniqueness exhaustion terminará con error.

## DB-TDG-052
ChoiceProvider podrá ser uniforme.

## DB-TDG-053
WeightedChoice podrá ser explícito.

## DB-TDG-054
Distribution será distinta de RandomSource.

## DB-TDG-055
Distribution algorithms serán documentables.

## DB-TDG-056
Normal distribution tendrá out-of-range policy.

## DB-TDG-057
Dataset Profile será distinto de Fixture.

## DB-TDG-058
Dataset Profile describirá población sintética.

## DB-TDG-059
Relationship generation utilizará metadata cuando esté disponible.

## DB-TDG-060
FK no definirá cardinalidad de negocio por sí sola.

## DB-TDG-061
Relationship ownership no se adivinará si existe metadata.

## DB-TDG-062
Polymorphic generation utilizará stable aliases.

## DB-TDG-063
Graph generation tendrá depth limit.

## DB-TDG-064
Graph generation tendrá node budget.

## DB-TDG-065
Cycles tendrán policy explícita.

## DB-TDG-066
Generated ID barriers pertenecerán a persistence integration.

## DB-TDG-067
Generator no ejecutará flush.

## DB-TDG-068
GeneratedDataset podrá ser materialized.

## DB-TDG-069
GeneratedDataset podrá ser lazy.

## DB-TDG-070
GeneratedDataset podrá ser streaming.

## DB-TDG-071
Streaming no requerirá materialización completa.

## DB-TDG-072
Streaming podrá ser single-pass.

## DB-TDG-073
Rewindability será explícita.

## DB-TDG-074
Large generation deberá soportar memoria acotada.

## DB-TDG-075
Generation chunk no será transaction boundary.

## DB-TDG-076
Backpressure será soportable.

## DB-TDG-077
Cancellation será cooperativa.

## DB-TDG-078
Cancellation no marcará dataset como completo.

## DB-TDG-079
ResourceBudget será explícito.

## DB-TDG-080
ResourceBudget limitará graph explosion.

## DB-TDG-081
ResourceBudget limitará uniqueness attempts.

## DB-TDG-082
ResourceBudget limitará materialization cuando corresponda.

## DB-TDG-083
Performance profile será reproducible cuando se configure.

## DB-TDG-084
Generator performance será separable de DB performance.

## DB-TDG-085
Production data no será fuente implícita.

## DB-TDG-086
Synthetic data será default para testing.

## DB-TDG-087
Sensitive metadata podrá seleccionar providers seguros.

## DB-TDG-088
Generated secrets no aparecerán en logs por defecto.

## DB-TDG-089
Generated PII-like data será sintética.

## DB-TDG-090
Validation integration será opcional.

## DB-TDG-091
Generator no será Validator.

## DB-TDG-092
Generation validation policy será explícita.

## DB-TDG-093
Large datasets podrán usar sample validation.

## DB-TDG-094
Schema introspection no ocurrirá por valor generado.

## DB-TDG-095
Generation metadata podrá compilarse.

## DB-TDG-096
Compiled generation metadata podrá cachearse.

## DB-TDG-097
Current random state no será metadata cacheable.

## DB-TDG-098
Current unique state no será metadata cacheable.

## DB-TDG-099
Generated objects no serán metadata cacheable.

## DB-TDG-100
Provider Registry podrá congelarse.

## DB-TDG-101
Provider conflicts serán deterministas.

## DB-TDG-102
ProviderId será estable.

## DB-TDG-103
ProviderId no requerirá ser FQCN.

## DB-TDG-104
Custom providers serán soportados.

## DB-TDG-105
Core no dependerá obligatoriamente de Faker externo.

## DB-TDG-106
External provider versions participarán en reproducibility descriptor.

## DB-TDG-107
Provider capabilities serán explícitas.

## DB-TDG-108
STRICT profile no aceptará provider no determinista silenciosamente.

## DB-TDG-109
Shared providers serán stateless o context-driven.

## DB-TDG-110
Persistent workers no compartirán mutable generation state.

## DB-TDG-111
FrankenPHP será soportado.

## DB-TDG-112
RoadRunner será soportable.

## DB-TDG-113
OpenSwoole será soportable.

## DB-TDG-114
Coroutine generation contexts estarán aislados.

## DB-TDG-115
Factory providers serán lazy cuando corresponda.

## DB-TDG-116
Factory provider se evaluará por instancia.

## DB-TDG-117
Seeder podrá generar datasets sin ORM.

## DB-TDG-118
Fixture podrá mezclar fixed y generated values.

## DB-TDG-119
Property-based integration será posible.

## DB-TDG-120
Generator no será Shrinker.

## DB-TDG-121
Fuzzer podrá consumir providers sin convertirse en Generator.

## DB-TDG-122
Referential integrity podrá modelarse.

## DB-TDG-123
DB seguirá siendo autoridad final de constraints.

## DB-TDG-124
Unknown DB constraint no será considerada satisfecha.

## DB-TDG-125
Generation confidence será representable.

## DB-TDG-126
Explainability será soportada.

## DB-TDG-127
Seed será reportable para reproducción.

## DB-TDG-128
Seed no será metric label de alta cardinalidad.

## DB-TDG-129
Telemetry tendrá bounded cardinality.

## DB-TDG-130
Telemetry no expondrá valores generados sensibles.

## DB-TDG-131
Errors conservarán original cause.

## DB-TDG-132
Generation dependency cycles serán rechazados.

## DB-TDG-133
Invalid configuration fallará durante planning cuando sea posible.

## DB-TDG-134
Invalid constraint intersection fallará temprano.

## DB-TDG-135
Unsupported provider capability fallará temprano.

## DB-TDG-136
Generator será platform-independent cuando sea posible.

## DB-TDG-137
MySQL-specific data assumptions no serán default.

## DB-TDG-138
MariaDB-specific data assumptions no serán default.

## DB-TDG-139
PostgreSQL-specific data assumptions no serán default.

## DB-TDG-140
SQLite-specific data assumptions no serán default.

## DB-TDG-141
Platform capabilities podrán limitar datasets.

## DB-TDG-142
Version no será Capability.

## DB-TDG-143
Tenant context será scoped cuando participe.

## DB-TDG-144
Tenant no será Shard.

## DB-TDG-145
Shard context será scoped cuando participe.

## DB-TDG-146
UNKNOWN shard no será ALL shards.

## DB-TDG-147
Generation no realizará cross-shard writes.

## DB-TDG-148
Persistence layer decidirá routing.

## DB-TDG-149
Generator no manipulará cache de datos.

## DB-TDG-150
Generator no invalidará Entity Cache.

## DB-TDG-151
Generator no invalidará Result Cache.

## DB-TDG-152
Persistence de generated data utilizará cache invalidation normal.

## DB-TDG-153
GenerationPlan será immutable después de validación.

## DB-TDG-154
Generation descriptors serán serializables de forma segura cuando se requiera.

## DB-TDG-155
Descriptor no incluirá closures arbitrarias para persistencia externa.

## DB-TDG-156
Generation registry será extensible.

## DB-TDG-157
Extensions respetarán Type System.

## DB-TDG-158
Extensions respetarán Resource Governance.

## DB-TDG-159
Extensions respetarán Security Policy.

## DB-TDG-160
TestDataGenerator será una fuente de datos, no un segundo Database Engine.

---

# 215. Modelo formal de generación

Sea:

```text
G = Generator
S = Seed
P = Profile
C = Constraints
L = Locale
I = GenerationIndex
V = ProviderVersions
```

Entonces:

```text
Value =
G(S, P, C, L, I, V)
```

Bajo una política determinista:

```text
same(S,P,C,L,I,V)
⇒
same(Value)
```

siempre que el algoritmo registrado permanezca compatible.

---

# 216. Dataset formal

Sea:

```text
D = DatasetDefinition
N = requested population
R = relationship rules
B = resource budget
```

Entonces:

```text
GenerationPlan =
Plan(D, N, R, B, Context)
```

y:

```text
GeneratedDataset =
Execute(GenerationPlan)
```

---

# 217. Validez de generación

```text
ValidGeneration
=
TypeCompatible
∧ ConstraintCompatible
∧ ProviderCompatible
∧ ResourceBudgetSatisfied
∧ ReproducibilityRequirementsSatisfied
```

---

# 218. Validez de persistencia

No deberá confundirse con:

```text
ValidPersistence
```

porque:

```text
ValidGeneration
≠
DatabaseAccepted
```

La DB conserva autoridad final.

---

# 219. Pipeline Factory

```text
Factory Definition
       │
       ▼
Value Providers
       │
       ▼
Generation Context
       │
       ▼
Test Data Generator
       │
       ▼
Generated Attributes
       │
       ▼
Factory Construction
       │
       ▼
Model / Entity
```

---

# 220. Pipeline de grandes datasets

```text
Dataset Profile
       │
       ▼
Generation Planner
       │
       ▼
Lazy / Streaming Generator
       │
       ▼
Chunk
       │
       ▼
Bulk Persistence API
       │
       ▼
Transaction Policy
       │
       ▼
Query Engine
       │
       ▼
Database
```

---

# 221. Pipeline completo del Bloque 18

```text
                         TestData
                            │
                    synthetic values
                            │
                            ▼
                       Factories
                 ┌──────────┴──────────┐
                 ▼                     ▼
            ModelFactory          EntityFactory
                 │                     │
                 └──────────┬──────────┘
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
            Seeder                    Fixture
       dataset population        scenario construction
               │                         │
               └────────────┬────────────┘
                            ▼
                   Persistence APIs
                            │
                            ▼
                       Database
```

---

# 222. Resultado arquitectónico del Bloque 18

VoltStack dispondrá ahora de cuatro conceptos claramente separados:

```text
Factory
    "¿Cómo construyo este objeto?"

Seeder
    "¿Qué datos debo poblar?"

Fixture
    "¿Qué estado conocido necesita este escenario?"

TestDataGenerator
    "¿Cómo genero valores y poblaciones sintéticas?"
```

Esto permite:

```php
User::factory()
    ->count(100)
    ->create();
```

sin convertir la Factory en un Seeder.

Permite:

```php
$this->fixtures->load(
    UserWithExpiredSubscriptionFixture::class
);
```

sin convertir Fixture en Factory.

Y permite:

```php
TestData::integer(1, 100);
```

sin involucrar ORM ni Database.

---

# 223. Regla maestra final

> **VoltStack tratará la generación de datos de prueba como una infraestructura determinista y gobernada, no como llamadas aleatorias dispersas dentro de Factories y tests.**

La separación final será:

```text
TestDataGenerator
    generates values

Factory
    constructs objects

Seeder
    orchestrates population

Fixture
    constructs scenarios

Persistence Engine
    persists state

Database
    enforces persistent constraints
```

Y siempre:

```text
Generated
≠
Persisted
≠
Committed
```

así como:

```text
GeneratorValid
≠
DatabaseAccepted
```

y:

```text
PseudoRandom
≠
Unreproducible
```

cuando se utilice un contexto determinista.

---

# 224. Bloque 18 completado

```text
✓ 193_DATABASE_FACTORY_SYSTEM.md
✓ 194_DATABASE_MODEL_FACTORY_SYSTEM.md
✓ 195_DATABASE_ENTITY_FACTORY_SYSTEM.md
✓ 196_DATABASE_SEEDER_SYSTEM.md
✓ 197_DATABASE_FIXTURE_SYSTEM.md
✓ 198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md
```

Arquitectónicamente queda establecido:

```text
Construction
    ↓
Factories

Population
    ↓
Seeders

Scenario
    ↓
Fixtures

Synthetic Data
    ↓
Test Data Generation
```

con una sola infraestructura Database/ORM subyacente y sin crear motores de persistencia paralelos.

---

# 225. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 19
PAGINATION, BATCH & LARGE DATA
```

Secuencia:

```text
199_DATABASE_PAGINATION_SYSTEM.md
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
202_DATABASE_LAZY_COLLECTION_SYSTEM.md
203_DATABASE_BULK_INSERT_SYSTEM.md
204_DATABASE_BULK_UPDATE_SYSTEM.md
205_DATABASE_BULK_DELETE_SYSTEM.md
206_DATABASE_IMPORT_SYSTEM.md
207_DATABASE_EXPORT_SYSTEM.md
208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

El bloque resolverá un problema diferente:

```text
Database Dataset
      │
      ├── Navigate efficiently
      ├── Process incrementally
      ├── Stream lazily
      ├── Insert in bulk
      ├── Update in bulk
      ├── Delete in bulk
      ├── Import
      ├── Export
      └── Process massive datasets
```

sin convertir:

```text
Pagination
```

en:

```text
Large Dataset Processing
```

ni:

```text
Chunking
```

en:

```text
Transaction Boundary
```

ni:

```text
Bulk Operation
```

en un escape que ignore:

```text
Query semantics
Transactions
Cache invalidation
Distribution
Security
Telemetry
Resource governance
```

---

# 226. Siguiente documento

```text
199_DATABASE_PAGINATION_SYSTEM.md
```

Este documento iniciará el **Bloque 19 — Pagination, Batch & Large Data** definiendo la arquitectura general de paginación de VoltStack:

```text
Pagination
├── offset pagination
├── page-number pagination
├── total-count strategies
├── pagination metadata
├── stable ordering requirements
├── deterministic ordering
├── query integration
├── ORM integration
├── relationship interaction
├── distributed-query implications
├── consistency semantics
├── resource governance
└── developer API
```

y establecerá la separación fundamental:

```text
Pagination
≠
Cursor Pagination
≠
Chunk Processing
≠
Lazy Collection
≠
Streaming Result
```

antes de especializar `Cursor Pagination` en el documento 200.