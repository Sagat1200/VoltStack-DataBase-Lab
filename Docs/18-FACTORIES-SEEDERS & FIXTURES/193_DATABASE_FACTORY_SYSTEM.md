# 193_DATABASE_FACTORY_SYSTEM.md

# VoltStack Quantum Database
## Database Factory System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 193 — Database Factory System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md`  
**Siguiente documento:** `194_DATABASE_MODEL_FACTORY_SYSTEM.md`

---

# 1. Propósito

`Database Factory System` define la arquitectura general mediante la cual VoltStack podrá construir de forma declarativa, reproducible y extensible:

- Models;
- Entities;
- Value Objects;
- aggregate graphs;
- relationship graphs;
- DTOs auxiliares;
- estados de dominio;
- datasets sintéticos;
- datos para testing;
- datos para desarrollo;
- datos destinados posteriormente a persistencia.

La regla central será:

> **Una Factory describe y construye objetos o valores; no representa una operación de persistencia, no ejecuta SQL y no convierte automáticamente la construcción de un objeto en estado persistente.**

Por tanto:

```text
Factory
≠
Persistence Engine
≠
Seeder
≠
Fixture
≠
Hydrator
≠
Repository
≠
EntityManager
≠
Random Generator
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. API expresiva.
2. Tipado fuerte.
3. reproducibilidad.
4. composición.
5. estados reutilizables.
6. relaciones.
7. secuencias.
8. datos deterministas.
9. generación aleatoria controlada.
10. extensibilidad.
11. integración ORM.
12. integración con testing.
13. soporte Model API.
14. soporte Entity/Data Mapper.
15. aislamiento de runtime.
16. ejecución segura bajo FrankenPHP.
17. soporte futuro RoadRunner/OpenSwoole.
18. generación eficiente de grandes datasets.
19. separación construcción/persistencia.
20. debugging y telemetry.

---

# 3. Posición arquitectónica

```text
Application / Tests / Seeders
             │
             ▼
        Factory API
             │
             ▼
      Factory Definition
             │
             ▼
        Factory State
             │
             ▼
      Attribute Resolver
             │
             ▼
    Relationship Resolver
             │
             ▼
      Object Constructor
             │
             ▼
     Model / Entity / VO
             │
       optional boundary
             ▼
     Persistence Adapter
             │
             ▼
      EntityManager / ORM
             │
             ▼
     Persistence Engine
```

La Factory termina conceptualmente en:

```text
Constructed Object Graph
```

La persistencia constituye una fase adicional.

---

# 4. Separación fundamental

VoltStack deberá mantener:

```text
Factory Definition
        ↓
Object Construction
        ↓
optional
        ↓
Persistence
```

y no:

```text
Factory
=
INSERT generator
```

---

# 5. Factory ≠ Seeder

Una Factory responde:

> ¿Cómo construyo una instancia válida o útil de este tipo?

Un Seeder responde:

> ¿Qué datos debe contener este entorno o base de datos?

Ejemplo:

```text
UserFactory
```

puede saber generar usuarios.

Pero:

```text
DevelopmentSeeder
```

decide que se crearán:

```text
1 admin
20 customers
100 orders
```

---

# 6. Factory ≠ Fixture

Una Fixture representa normalmente un escenario o dataset conocido.

Ejemplo:

```text
CustomerWithOverdueInvoiceFixture
```

La Factory puede ser utilizada para construirlo.

---

# 7. Factory ≠ Faker

Un generador de datos responde:

```text
give me a random-looking email
```

Una Factory responde:

```text
construct a valid User according to this definition
```

Por tanto:

```text
FakeDataProvider
⊂ possible Factory dependency
```

pero:

```text
FakeDataProvider
≠ Factory
```

---

# 8. Factory ≠ Hydrator

Hydration parte de:

```text
Database Result
```

y reconstruye estado persistente observado.

Factory parte de:

```text
Factory Definition
```

y genera nuevo estado.

```text
Hydration
=
observed persistent state → application object

Factory
=
generation specification → application object
```

---

# 9. Factory ≠ EntityManager

Factory no deberá:

- mantener IdentityMap;
- hacer dirty tracking;
- administrar UnitOfWork;
- controlar transactions;
- ejecutar flush.

---

# 10. Factory ≠ Repository

Repository recupera o consulta entidades persistentes.

Factory crea nuevas representaciones.

---

# 11. Factory ≠ Persistence Engine

La Factory no deberá conocer:

```text
SQL
INSERT
UPDATE
DELETE
PreparedStatement
Connection
Driver
Dialect
```

---

# 12. Dos familias principales

VoltStack soportará:

```text
DatabaseFactory
├── ModelFactory
└── EntityFactory
```

---

# 13. ModelFactory

Será la API optimizada para el estilo Laravel-like:

```php
User::factory()
    ->count(10)
    ->create();
```

Se desarrollará en:

```text
194_DATABASE_MODEL_FACTORY_SYSTEM.md
```

---

# 14. EntityFactory

Será la API orientada al modelo Data Mapper:

```php
$users = $userFactory
    ->count(10)
    ->make();

foreach ($users as $user) {
    $entityManager->persist($user);
}

$entityManager->flush();
```

Se desarrollará en:

```text
195_DATABASE_ENTITY_FACTORY_SYSTEM.md
```

---

# 15. Un solo motor conceptual

Aunque existan dos APIs:

```text
Model Factory
Entity Factory
```

no deberán convertirse en dos sistemas incompatibles.

Ambas utilizarán:

```text
Factory Definition Engine
Factory State Engine
Attribute Resolution
Relationship Generation
Sequence Engine
Random Data Engine
Construction Pipeline
```

---

# 16. Arquitectura general

```text
                 Factory API
                     │
          ┌──────────┴──────────┐
          │                     │
     ModelFactory          EntityFactory
          │                     │
          └──────────┬──────────┘
                     ▼
              FactoryDefinition
                     │
                     ▼
              FactoryBlueprint
                     │
                     ▼
               StatePipeline
                     │
                     ▼
             SequenceEngine
                     │
                     ▼
            AttributeResolver
                     │
                     ▼
          RelationshipPlanner
                     │
                     ▼
            FactoryConstructor
                     │
                     ▼
            Constructed Graph
                     │
            optional │
                     ▼
           PersistenceBridge
```

---

# 17. FactoryDefinition

Representará la definición reusable de una Factory.

```php
interface FactoryDefinition
{
    public function target(): FactoryTarget;

    public function attributes(
        FactoryContext $context
    ): iterable;
}
```

---

# 18. Factory target

```php
final readonly class FactoryTarget
{
    public function __construct(
        public string $type,
        public FactoryTargetKind $kind,
    ) {}
}
```

---

# 19. FactoryTargetKind

```php
enum FactoryTargetKind
{
    case MODEL;
    case ENTITY;
    case VALUE_OBJECT;
    case PROJECTION;
    case CUSTOM;
}
```

---

# 20. Definition ≠ runtime instance

Una Factory Definition deberá poder ser:

```text
immutable
shareable
compiled
cached
```

mientras la ejecución mantiene su propio contexto.

---

# 21. FactoryBlueprint

La definición podrá compilarse a:

```php
final readonly class FactoryBlueprint
{
    public function __construct(
        public FactoryTarget $target,
        public FactoryAttributePlan $attributes,
        public FactoryRelationshipPlan $relationships,
        public FactoryConstructionPlan $construction,
        public FactoryStateRegistry $states,
    ) {}
}
```

---

# 22. Blueprint

El Blueprint representa:

```text
compiled factory structure
```

No representa:

```text
generated object
```

---

# 23. Factory instance

La expresión:

```php
UserFactory::new()
```

deberá producir un descriptor/configuración de ejecución independiente.

---

# 24. Inmutabilidad preferida

Operaciones como:

```php
$factory->count(10);
$factory->state(...);
$factory->with(...);
```

preferentemente devolverán una nueva configuración.

Ejemplo:

```php
$base = UserFactory::new();

$admins = $base->state('admin');

$customers = $base->state('customer');
```

`$base` no deberá mutar accidentalmente.

---

# 25. FactoryRequest

Cada ejecución se normalizará a:

```php
final readonly class FactoryRequest
{
    public function __construct(
        public FactoryBlueprint $blueprint,
        public int $count,
        public FactoryStateSet $states,
        public FactoryOverrideSet $overrides,
        public FactoryRelationshipRequestSet $relationships,
        public FactoryGenerationPolicy $generation,
    ) {}
}
```

---

# 26. Count

```php
UserFactory::new()->count(100);
```

deberá representar intención de cardinalidad.

No deberá construir 100 objetos inmediatamente.

---

# 27. Lazy factory configuration

La configuración de una Factory deberá ser barata.

La construcción ocurre al llamar:

```text
make
create
build
generate
```

según API.

---

# 28. make()

Semántica base:

```text
make()
=
construct objects
without persistence
```

---

# 29. create()

En APIs que la soporten:

```text
create()
=
construct
+
explicit persistence bridge
```

No significa que la Factory misma sea el Persistence Engine.

---

# 30. build()

Podrá utilizarse internamente como operación neutral:

```text
FactoryRequest
→ FactoryBuildResult
```

---

# 31. FactoryBuildResult

```php
final readonly class FactoryBuildResult
{
    public function __construct(
        public array $objects,
        public FactoryExecutionReport $report,
    ) {}
}
```

---

# 32. FactoryContext

Toda generación tendrá un contexto scoped.

```php
final readonly class FactoryContext
{
    public function __construct(
        public FactoryExecutionId $executionId,
        public FactoryRandomSource $random,
        public FactorySequenceContext $sequence,
        public FactoryClock $clock,
        public FactoryGenerationPolicy $policy,
    ) {}
}
```

---

# 33. Context scoped

`FactoryContext` deberá pertenecer a una ejecución.

Nunca:

```php
static FactoryContext $current;
```

---

# 34. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request A
│   └── FactoryContext A
│
└── Request B
    └── FactoryContext B
```

Nunca:

```text
FactoryContext A
→ Request B
```

---

# 35. Deterministic generation

Factories deberán soportar:

```php
UserFactory::new()
    ->seed(12345)
    ->count(10)
    ->make();
```

---

# 36. Seed

Con:

```text
same Factory definition
same seed
same states
same inputs
same compatible generation version
```

deberá poder obtenerse un resultado reproducible cuando todos los providers involucrados sean deterministas.

---

# 37. Determinism contract

Formalmente:

```text
F(definition, seed, state, input)
=
same output
```

bajo una misma versión semántica del generador.

---

# 38. Determinism ≠ forever identical

Cambios en:

```text
factory definition
faker provider
algorithm
locale data
generation version
```

pueden cambiar resultados.

---

# 39. GenerationVersion

Para reproducibilidad estricta podrá existir:

```php
final readonly class FactoryGenerationVersion
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 40. Random source

No utilizar directamente:

```php
rand()
mt_rand()
random_int()
```

dentro de factories si se requiere determinismo.

---

# 41. FactoryRandomSource

```php
interface FactoryRandomSource
{
    public function integer(int $min, int $max): int;

    public function float(): float;

    public function boolean(float $probability = 0.5): bool;

    public function pick(array $values): mixed;
}
```

---

# 42. Cryptographic randomness

Datos de prueba normalmente no requieren CSPRNG.

Pero factories de security testing podrán solicitar explícitamente un provider criptográfico.

---

# 43. Randomness policy

```php
enum FactoryRandomnessPolicy
{
    case DETERMINISTIC;
    case NON_DETERMINISTIC;
    case CRYPTOGRAPHIC;
}
```

---

# 44. Fake data provider

Contrato:

```php
interface FactoryDataProvider
{
    public function supports(string $capability): bool;

    public function generate(
        string $capability,
        FactoryContext $context,
    ): mixed;
}
```

---

# 45. Providers

Podrán existir:

```text
PersonDataProvider
InternetDataProvider
AddressDataProvider
CompanyDataProvider
LoremDataProvider
FinanceDataProvider
DateDataProvider
UUIDDataProvider
CustomDomainProvider
```

---

# 46. Provider registry

```php
interface FactoryDataProviderRegistry
{
    public function resolve(string $capability): FactoryDataProvider;
}
```

---

# 47. Locale

La generación podrá declarar:

```php
->locale('es_MX')
```

---

# 48. Locale ≠ database collation

Son conceptos diferentes.

```text
Factory Locale
≠
Database Locale
≠
Database Collation
```

---

# 49. Attribute definition

Ejemplo:

```php
final class UserFactory extends ModelFactory
{
    protected function definition(): array
    {
        return [
            'name' => fn (FactoryContext $ctx) =>
                $ctx->data()->person()->name(),

            'email' => fn (FactoryContext $ctx) =>
                $ctx->data()->internet()->email(),

            'active' => true,
        ];
    }
}
```

---

# 50. Attribute values

Un atributo podrá ser:

```text
literal
closure/resolver
sequence
derived value
value object factory
reference
relationship reference
custom generator
```

---

# 51. Lazy resolution

Closures no deberán evaluarse al definir la Factory.

Se evaluarán por instancia.

---

# 52. Per-instance evaluation

```php
'uuid' => fn ($ctx) => $ctx->uuid()
```

deberá generar un valor diferente por objeto, salvo policy determinista que indique lo contrario.

---

# 53. AttributeResolver

```php
interface FactoryAttributeResolver
{
    public function resolve(
        FactoryAttributePlan $plan,
        FactoryInstanceContext $context,
    ): ResolvedFactoryAttributes;
}
```

---

# 54. FactoryInstanceContext

```php
final readonly class FactoryInstanceContext
{
    public function __construct(
        public FactoryContext $factory,
        public int $index,
        public FactoryStateSet $states,
        public FactoryOverrideSet $overrides,
    ) {}
}
```

---

# 55. Attribute resolution order

Orden conceptual:

```text
Base Definition
      ↓
Sequences
      ↓
Named States
      ↓
Inline States
      ↓
Relationship-derived Values
      ↓
Explicit Overrides
      ↓
Validation
      ↓
Construction
```

---

# 56. Explicit override precedence

Ejemplo:

```php
UserFactory::new()
    ->state('inactive')
    ->make([
        'active' => true,
    ]);
```

El override explícito deberá tener precedencia por defecto.

---

# 57. Precedence configurable

Factories especializadas podrán cambiarlo, pero nunca ambiguamente.

---

# 58. Factory state

Un state representa una transformación declarativa de la definición.

Ejemplo:

```php
public function admin(): static
{
    return $this->state([
        'role' => 'admin',
    ]);
}
```

---

# 59. Named states

```php
UserFactory::new()
    ->state('verified')
    ->state('admin');
```

---

# 60. State composition

Estados deberán poder componerse.

---

# 61. State order

El orden deberá ser determinista.

```text
state A
then state B
```

puede diferir de:

```text
state B
then state A
```

si ambos modifican el mismo atributo.

---

# 62. State conflict

Podrá detectarse:

```text
StateConflict
```

cuando la policy lo requiera.

---

# 63. StateTransformation

```php
interface FactoryStateTransformation
{
    public function apply(
        FactoryAttributeSet $attributes,
        FactoryInstanceContext $context,
    ): FactoryAttributeSet;
}
```

---

# 64. Immutable transformation

Preferentemente:

```text
Input AttributeSet
→ New AttributeSet
```

---

# 65. Inline state

```php
UserFactory::new()
    ->state([
        'status' => 'pending',
    ]);
```

---

# 66. Conditional state

```php
->when(
    $condition,
    fn ($factory) => $factory->state('verified')
)
```

será API de composición, no lógica escondida en Persistence Engine.

---

# 67. Sequence system

Factories deberán soportar secuencias.

Ejemplo:

```php
UserFactory::new()
    ->sequence(
        ['role' => 'admin'],
        ['role' => 'editor'],
        ['role' => 'customer'],
    );
```

---

# 68. Cyclic sequence

Para 10 usuarios:

```text
admin
editor
customer
admin
editor
customer
...
```

si la sequence policy es cíclica.

---

# 69. Indexed sequence

```php
->sequence(
    fn (FactorySequence $sequence) => [
        'email' => "user{$sequence->index}@example.test",
    ]
)
```

---

# 70. FactorySequence

```php
final readonly class FactorySequence
{
    public function __construct(
        public int $index,
        public int $count,
        public int $iteration,
    ) {}
}
```

---

# 71. Sequence scope

Sequence state será local a la ejecución.

---

# 72. No global sequence counters

Prohibido:

```php
static int $factoryCounter = 0;
```

---

# 73. Relationships

Factories deberán construir grafos.

Ejemplo:

```php
UserFactory::new()
    ->has(
        PostFactory::new()->count(3)
    );
```

---

# 74. Relationship factory

La relación deberá utilizar metadata ORM cuando exista.

---

# 75. Relationship name

Preferible:

```php
->has(
    'posts',
    PostFactory::new()->count(3)
)
```

o API tipada equivalente.

---

# 76. Relationship validation

El sistema deberá validar:

```text
User.posts exists
relationship cardinality compatible
target factory compatible
```

---

# 77. has()

Representa:

```text
parent
→ children
```

---

# 78. for()

Ejemplo:

```php
PostFactory::new()
    ->for(
        UserFactory::new()
    );
```

Representa un parent requerido.

---

# 79. belongsTo

La semántica deberá basarse en metadata de relación, no simplemente en nombres como:

```text
user_id
```

---

# 80. Many-to-many

Ejemplo conceptual:

```php
UserFactory::new()
    ->hasAttached(
        RoleFactory::new()->count(2)
    );
```

---

# 81. Association metadata

Para relaciones ricas:

```text
User
↔
ProjectMembership
↔
Project
```

deberá utilizarse Factory de la association entity.

---

# 82. Factory graph

```text
UserFactory
│
├── ProfileFactory
│
├── PostFactory × 5
│   └── CommentFactory × 3
│
└── RoleFactory × 2
```

---

# 83. FactoryGraphPlan

```php
final readonly class FactoryGraphPlan
{
    public function __construct(
        public FactoryNode $root,
        public array $edges,
    ) {}
}
```

---

# 84. FactoryGraphPlanner

```php
interface FactoryGraphPlanner
{
    public function plan(
        FactoryRequest $request
    ): FactoryGraphPlan;
}
```

---

# 85. Graph construction ≠ persistence planning

Muy importante:

```text
FactoryGraphPlan
≠
PersistencePlan
```

Factory Graph determina qué objetos construir.

Persistence Planner determina cómo sincronizarlos con la DB.

---

# 86. Cyclic relationships

Ejemplo:

```text
User
↔
Profile
```

podrá necesitar construcción en múltiples fases.

---

# 87. Construction phases

```text
Allocate
   ↓
Resolve independent attributes
   ↓
Construct references
   ↓
Link relationships
   ↓
Finalize graph
```

---

# 88. Generated database IDs

Factory no deberá asumir que el ID generado por DB existe durante `make()`.

---

# 89. Temporary identity

Para grafos pre-persistencia podrán utilizarse:

```text
object identity
temporary factory node identity
application-generated UUID
```

---

# 90. Database-generated identity

Solo estará disponible después de persistencia.

---

# 91. Relationship foreign keys

En Entity/Data Mapper mode:

```text
relationship object reference
```

deberá ser preferible a manipular manualmente FK cuando el modelo de dominio lo permita.

---

# 92. Model API

En Active Record-like models, la API podrá ofrecer comodidad adicional para FKs.

Pero deberá converger al mismo relationship metadata model.

---

# 93. Value Object factories

Ejemplo:

```php
AddressFactory::new()->make();
```

---

# 94. Value Object factory

No deberá intentar:

```text
persist(Address)
```

si Address es un Value Object embebido.

---

# 95. Nested value objects

```text
Customer
└── Address
    ├── Street
    └── PostalCode
```

podrán componerse mediante factories.

---

# 96. Constructor strategies

Una Factory podrá construir mediante:

```text
constructor
named constructor
property assignment
builder
custom constructor adapter
```

---

# 97. FactoryConstructionStrategy

```php
interface FactoryConstructionStrategy
{
    public function construct(
        FactoryTarget $target,
        ResolvedFactoryAttributes $attributes,
        FactoryConstructionContext $context,
    ): object;
}
```

---

# 98. Constructor metadata

Cuando ORM metadata exista, podrá reutilizarse información estructural.

Pero:

```text
Factory Construction
≠
Hydration Construction
```

---

# 99. Constructor side effects

Factories deberán favorecer targets sin efectos externos durante construcción.

---

# 100. External I/O

Por defecto, una Factory Definition no deberá realizar:

```text
HTTP calls
database reads
filesystem writes
queue dispatch
email sending
```

---

# 101. Pure generation

El modo preferido será:

```text
inputs
+
seed
+
definition
→ objects
```

---

# 102. I/O-dependent factories

Si una aplicación necesita I/O:

```text
ExternalFactoryResolver
```

deberá ser explícito y marcado como non-deterministic/side-effecting.

---

# 103. Database lookup in factory

No deberá ser comportamiento implícito.

Ejemplo problemático:

```php
'country_id' => Country::first()->id
```

---

# 104. Preferred alternative

```php
->for(
    CountryFactory::existing(...)
)
```

o:

```text
FactoryReference
```

explícita.

---

# 105. Existing references

Factories podrán referenciar entidades existentes.

```php
PostFactory::new()
    ->for($existingUser);
```

---

# 106. Existing entity ≠ generated entity

El graph planner deberá distinguir:

```text
GENERATED
EXISTING
REFERENCE_ONLY
```

---

# 107. FactoryNodeKind

```php
enum FactoryNodeKind
{
    case GENERATED;
    case EXISTING;
    case REFERENCE;
}
```

---

# 108. Persistence ownership

Una Factory no deberá asumir que una entidad existente debe volver a persistirse.

---

# 109. Callbacks

Podrán existir:

```text
afterMaking
beforePersisting
afterPersisting
```

---

# 110. Callback boundaries

Deben distinguir claramente fases.

---

# 111. afterMaking

Se ejecuta después de construir el objeto.

No implica persistencia.

---

# 112. beforePersisting

Solo se ejecuta si se solicita persistencia.

---

# 113. afterPersisting

Solo después de que la operación de persistencia correspondiente haya tenido éxito según el contrato definido.

---

# 114. afterPersisting ≠ afterCommit

Muy importante:

```text
Persisted/Flushed
≠
Transaction Committed
```

---

# 115. afterCommit factory hook

Si se necesita:

```text
after commit
```

deberá integrarse con Transaction Event System.

---

# 116. No external side effects before commit

Callbacks no deberán utilizarse para asumir atomicidad con:

```text
email
HTTP
external API
message broker
```

---

# 117. PersistenceBridge

```php
interface FactoryPersistenceBridge
{
    public function persist(
        FactoryBuildResult $result,
        FactoryPersistenceContext $context,
    ): FactoryPersistenceResult;
}
```

---

# 118. Bridge responsibility

El bridge adapta:

```text
constructed objects
→ ORM persistence API
```

---

# 119. Bridge does not generate SQL

Seguirá usando:

```text
EntityManager
UnitOfWork
Persistence Engine
Query Engine
```

---

# 120. Model bridge

Podrá ofrecer semántica Laravel-like:

```php
User::factory()->create();
```

---

# 121. Entity bridge

Podrá integrarse con:

```php
$entityManager->persist(...)
```

---

# 122. Flush policy

`create()` podrá tener una policy:

```php
enum FactoryFlushPolicy
{
    case IMMEDIATE;
    case DEFERRED;
    case MANUAL;
}
```

---

# 123. Default

El default deberá ser explícito por API.

---

# 124. Batch creation

Para:

```php
User::factory()
    ->count(10000)
    ->create();
```

no deberán ejecutarse necesariamente 10,000 flush individuales.

---

# 125. Persistence batching

El bridge podrá utilizar:

`133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md`

---

# 126. Batch factory ≠ Bulk DML

Factory crea semánticamente objetos.

Bulk DML puede evitar objetos completamente.

Por tanto:

```text
Factory Batch Creation
≠
Bulk Insert
```

---

# 127. Large dataset warning

Para millones de registros:

```text
Factory + ORM
```

puede no ser la herramienta adecuada.

---

# 128. Test Data Generation System

El documento 198 definirá mecanismos optimizados para grandes datasets.

---

# 129. Factory memory policy

Para counts elevados:

```text
count(1_000_000)
```

el sistema deberá evitar retener todos los objetos innecesariamente.

---

# 130. Streaming generation

Podrá soportarse:

```php
foreach (
    UserFactory::new()->stream(1_000_000)
    as $user
) {
    // ...
}
```

---

# 131. stream()

No deberá significar persistencia automática.

---

# 132. Chunk generation

```php
UserFactory::new()
    ->count(100_000)
    ->chunks(1000);
```

---

# 133. Chunk lifecycle

Cada chunk podrá liberar:

```text
temporary objects
relationship planning state
generation buffers
```

cuando ya no sea necesario.

---

# 134. ORM cleanup

En persistencia masiva podrá integrarse con:

```text
EntityManager clear/detach policy
```

de forma explícita.

---

# 135. No hidden clear()

Nunca limpiar IdentityMap silenciosamente si el caller depende de objetos managed.

---

# 136. Factory Registry

VoltStack podrá mantener un registro:

```php
interface FactoryRegistry
{
    public function register(
        FactoryTarget $target,
        FactoryDefinition $factory,
    ): void;

    public function resolve(
        FactoryTarget $target
    ): FactoryDefinition;
}
```

---

# 137. Registry freeze

En runtime persistente:

```text
bootstrap
→ register factories
→ compile
→ freeze
```

---

# 138. Runtime mutation

Modificar globalmente el registry durante requests estará prohibido por defecto.

---

# 139. Factory discovery

Podrá soportarse:

```text
convention
attributes
explicit registration
package registration
```

---

# 140. Explicit registration

Será el mecanismo más determinista.

---

# 141. Auto-discovery

Si existe, deberá ocurrir en bootstrap/compile phase.

---

# 142. No per-request filesystem scanning

En producción no deberá escanearse el filesystem para descubrir factories por request.

---

# 143. Factory naming

Convención:

```text
User
→ UserFactory

Order
→ OrderFactory
```

---

# 144. Namespace

Ejemplo:

```php
App\Database\Factories\UserFactory
```

---

# 145. Package factories

Paquetes podrán registrar:

```text
Package\Database\Factories
```

sin contaminar namespaces de aplicación.

---

# 146. Factory resolution

Para:

```php
User::factory()
```

se resolverá:

```text
Entity/Model Type
       ↓
FactoryResolver
       ↓
FactoryRegistry
       ↓
Factory Definition
```

---

# 147. FactoryResolver

```php
interface FactoryResolver
{
    public function resolve(
        string $targetType
    ): FactoryDefinition;
}
```

---

# 148. Missing factory

Deberá producir error explícito:

```text
FactoryNotFoundException
```

---

# 149. No magical fallback

VoltStack no generará automáticamente una Factory arbitraria para cualquier entidad sin contrato explícito.

---

# 150. Automatic factory inference

Podrá existir como tooling:

```text
php voltstack make:factory User
```

pero no como runtime guess inseguro.

---

# 151. Factory code generation

CLI podrá inspeccionar metadata y generar un esqueleto.

---

# 152. Example CLI

```bash
php voltstack make:factory User
```

---

# 153. Generated output

```php
final class UserFactory extends ModelFactory
{
    protected function definition(): array
    {
        return [
            'name' => $this->data()->person()->name(),
            'email' => $this->data()->internet()->email(),
        ];
    }
}
```

---

# 154. Metadata-assisted generation

CLI podrá inferir:

```text
field names
logical types
nullability
enum mappings
relationships
```

---

# 155. Metadata inference ≠ business semantics

Una columna:

```text
status VARCHAR
```

no permite inferir correctamente:

```text
business-valid status
```

sin metadata adicional.

---

# 156. Generated factories require review

El tooling deberá marcar campos ambiguos.

---

# 157. Type-aware generation

Factory Engine podrá aprovechar:

`155_DATABASE_TYPE_SYSTEM.md`

---

# 158. Example

Logical type:

```text
UUID
```

puede sugerir:

```text
UUID generator
```

---

# 159. Decimal

```text
DECIMAL(12,2)
```

no deberá generarse usando float arbitrariamente si el type system exige decimal exacto.

---

# 160. Enum

Podrá seleccionar entre:

```text
valid enum cases
```

---

# 161. DateTime

Utilizará semantics del:

`162_DATABASE_DATE_TIME_TYPE_SYSTEM.md`

---

# 162. JSON

Podrá generar:

```text
valid canonical JSON-compatible structures
```

pero no objetos PHP arbitrarios.

---

# 163. Value Objects

Podrá resolver:

```text
Factory<ValueObject>
```

cuando exista registro.

---

# 164. Validation

Factory generation podrá validar el resultado.

---

# 165. FactoryValidationPolicy

```php
enum FactoryValidationPolicy
{
    case NONE;
    case STRUCTURAL;
    case DOMAIN;
    case FULL;
}
```

---

# 166. Validation ≠ Factory

La Factory podrá invocar Validation System.

Pero:

```text
Factory
≠
Validator
```

---

# 167. Invalid generated state

Deberá producir:

```text
FactoryGenerationException
```

o subtipo especializado.

---

# 168. Intentionally invalid factories

Testing requiere generar estados inválidos.

Por tanto deberá existir modo explícito:

```php
UserFactory::new()
    ->withoutValidation()
    ->state([
        'email' => 'invalid',
    ]);
```

---

# 169. Invalid test data ≠ framework bug

Si la invalidación fue solicitada explícitamente, el sistema deberá permitirla.

---

# 170. Factory constraints

Podrán definirse restricciones de generación.

Ejemplo:

```text
age >= 18
startDate <= endDate
currency matches amount
```

---

# 171. Cross-field generation

El resolver deberá soportar dependencias entre atributos.

---

# 172. Derived attribute

Ejemplo:

```php
'first_name' => ...,
'last_name'  => ...,

'display_name' => fn (FactoryAttributes $a) =>
    "{$a->first_name} {$a->last_name}",
```

---

# 173. Attribute dependency graph

```text
first_name ─┐
            ├── display_name
last_name ──┘
```

---

# 174. Dependency cycles

Ejemplo inválido:

```text
A depends B
B depends A
```

deberá detectarse.

---

# 175. AttributeDependencyCycleException

```php
final class AttributeDependencyCycleException
    extends FactoryDefinitionException
{
}
```

---

# 176. Resolution graph

Factory Blueprint podrá compilar dependencias de atributos.

---

# 177. Topological resolution

Cuando sea posible:

```text
Attribute DAG
→ topological ordering
```

---

# 178. Lazy dependency resolution

También podrá utilizarse resolución lazy con cycle detection.

---

# 179. Uniqueness

Factories suelen necesitar:

```text
unique email
unique username
```

---

# 180. Factory uniqueness ≠ DB uniqueness

Muy importante:

```text
GeneratedUnique
≠
DatabaseConstraintGuaranteed
```

---

# 181. Local uniqueness

Un provider podrá garantizar unicidad dentro de:

```text
FactoryExecution
```

---

# 182. Global database uniqueness

Requeriría conocimiento de DB y no deberá asumirse.

---

# 183. Existing DB collision

Puede ocurrir:

```text
factory generated email
=
existing database email
```

---

# 184. Persistence error

La DB UniqueConstraint seguirá siendo la autoridad final.

---

# 185. UniqueValuePool

```php
interface FactoryUniqueValuePool
{
    public function reserve(
        FactoryValueKey $key,
        mixed $value,
    ): bool;
}
```

---

# 186. Scope

Pool deberá ser scoped.

---

# 187. Memory bound

Unique pools deberán tener límites.

---

# 188. Exhaustion

Ejemplo:

```text
boolean unique values
count = 3
```

es imposible.

Debe producir:

```text
FactoryUniqueValueExhaustedException
```

---

# 189. Retry generation

Unique providers podrán reintentar dentro de un presupuesto.

---

# 190. Retry budget

Nunca:

```text
while (true)
```

---

# 191. Factory clock

Factories deberán poder controlar tiempo.

---

# 192. Default test clock

Tests reproducibles podrán usar:

```php
FactoryClock::frozen(
    '2026-01-01T00:00:00Z'
);
```

---

# 193. Clock ≠ system time

No depender directamente de:

```php
new DateTimeImmutable();
```

en definitions deterministas.

---

# 194. Relative dates

Ejemplo:

```text
createdAt = now - random 30 days
```

deberá utilizar FactoryClock.

---

# 195. Timezone

Aplicarán reglas del DateTime Type System.

---

# 196. Callbacks and determinism

Un callback que consulta:

```text
system time
network
database
```

rompe determinismo salvo que se declare.

---

# 197. Determinism report

Telemetry/debugging podrá indicar:

```text
DETERMINISTIC
PARTIALLY_DETERMINISTIC
NON_DETERMINISTIC
```

---

# 198. FactoryExecutionReport

```php
final readonly class FactoryExecutionReport
{
    public function __construct(
        public FactoryExecutionId $executionId,
        public int $requested,
        public int $constructed,
        public Duration $duration,
        public FactoryDeterminismState $determinism,
        public FactoryGenerationStatistics $statistics,
    ) {}
}
```

---

# 199. Factory lifecycle

```text
NEW
 ↓
CONFIGURED
 ↓
COMPILED
 ↓
GENERATING
 ↓
GENERATED
 ↓
optional PERSISTING
 ↓
COMPLETED
```

Errores:

```text
FAILED
CANCELLED
```

---

# 200. Factory execution state

```php
enum FactoryExecutionState
{
    case NEW;
    case CONFIGURED;
    case COMPILED;
    case GENERATING;
    case GENERATED;
    case PERSISTING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
}
```

---

# 201. Cancellation

Generación grande deberá aceptar:

```text
CancellationToken
```

del sistema de runtime/concurrency.

---

# 202. Cancellation ≠ rollback

Si todavía no hay persistencia:

```text
cancel
→ discard generated state
```

Si ya comenzó persistencia:

```text
transaction semantics
```

determinan rollback.

---

# 203. Factory transaction policy

Persistencia mediante `create()` podrá solicitar:

```text
NONE
USE_EXISTING
WRAP_BATCH
WRAP_ALL
```

según API y capabilities.

---

# 204. Transaction ≠ Factory

Factory no administrará directamente transacciones.

El PersistenceBridge coordinará con Transaction Manager.

---

# 205. Cross-shard factory graph

Un Factory Graph puede producir objetos destinados a distintos shards.

---

# 206. Construction allowed

Construirlos puede ser válido.

---

# 207. Persistence warning

Persistirlos como una sola unidad atómica puede no ser posible.

---

# 208. Distribution rule

```text
FactoryGraph
≠
DistributedTransaction
```

---

# 209. Shard key generation

Factories deberán poder generar shard keys válidos.

---

# 210. Partition routing

Al persistir:

```text
Persistence Planner
→ Partition Routing
```

determinará execution domains.

Factory no elegirá endpoints.

---

# 211. Tenant-aware factories

Con Multitenancy instalado, una Factory podrá recibir:

```text
TenantFactoryContext
```

---

# 212. Tenant context

No deberá depender de:

```text
static CurrentTenant
```

---

# 213. Tenant assignment

Ejemplo:

```php
UserFactory::new()
    ->forTenant($tenant);
```

---

# 214. Cross-tenant graphs

Deberán rechazarse si las relaciones ORM no los permiten.

---

# 215. Tenant Factory ≠ Multitenancy core

Database Factory System no dependerá obligatoriamente de Multitenancy.

---

# 216. Extension integration

Multitenancy podrá aportar:

```text
FactoryContextContributor
FactoryConstraint
FactoryPersistencePolicy
```

---

# 217. Security

Factories suelen ejecutarse en:

```text
tests
development
CLI
```

pero pueden existir en producción.

---

# 218. Production safety

Seed/test factories no deberán exponerse accidentalmente por HTTP.

---

# 219. ProductionFactoryPolicy

```php
enum ProductionFactoryPolicy
{
    case ALLOW;
    case WARN;
    case FORBID;
}
```

---

# 220. Sensitive data

Factories no deberán utilizar datos personales reales por defecto.

---

# 221. Synthetic data

Default:

```text
synthetic
non-real
non-secret
```

---

# 222. Secrets

No generar ni imprimir:

```text
production API keys
real passwords
real access tokens
```

---

# 223. Password factories

Podrán generar:

```text
known test password
```

pero su hash deberá pasar por el Hashing System cuando el target lo requiera.

---

# 224. Password callback

No almacenar password real en telemetry.

---

# 225. PII

FakeDataProvider deberá favorecer datos sintéticos claramente no reales cuando sea posible.

---

# 226. Factory code execution

Definitions son código de aplicación confiable.

Pero valores externos utilizados como overrides deberán pasar por los límites normales de seguridad/validation.

---

# 227. Serialization

Factory definitions no deberán serializar closures arbitrariamente para ejecución remota.

---

# 228. Queue integration

Si se desea generar datos mediante jobs:

```text
serialize FactoryRequest descriptor
```

no el closure graph completo.

---

# 229. FactoryRequest portability

Solo será portable si todos sus componentes poseen representación serializable estable.

---

# 230. Closure state

Una request con closure dinámica deberá marcarse:

```text
LOCAL_ONLY
```

---

# 231. Factory compilation

Factories podrán compilarse durante bootstrap.

---

# 232. Compiled factory cache

Podrá almacenar:

```text
attribute plans
state plans
relationship plans
constructor accessors
type mappings
```

---

# 233. Compiled cache ≠ generated data cache

Nunca almacenar objetos generados como parte del compiled factory cache.

---

# 234. Generation-aware compilation

Compiled Factory Blueprint deberá depender de:

```text
Factory definition generation
ORM metadata generation
Type registry generation
Relationship metadata generation
```

---

# 235. Schema dependency

No toda Factory necesita Schema metadata.

Preferir ORM/domain metadata.

---

# 236. Factory ≠ Schema

Una Factory genera objetos.

Schema describe estructura de DB.

---

# 237. Schema-aware tooling

CLI sí podrá consultar Schema Metadata para ayudar a generar factories.

---

# 238. Hot reload

En desarrollo:

```text
factory source change
→ factory generation bump
→ recompile
```

---

# 239. Production

Definitions compiladas podrán mantenerse worker-local si son:

```text
immutable
generation-compatible
request-independent
```

---

# 240. Telemetry

Eventos:

```text
FactoryExecutionStarted
FactoryObjectConstructed
FactoryGraphPlanned
FactoryPersistenceStarted
FactoryPersistenceCompleted
FactoryExecutionCompleted
FactoryExecutionFailed
FactoryGenerationRetried
FactoryUniqueValueExhausted
```

---

# 241. Telemetry cardinality

No incluir como labels:

```text
generated email
generated name
entity ID
tenant ID raw
random values
```

---

# 242. Metrics

```text
db.factory.executions
db.factory.objects.generated
db.factory.graph.nodes
db.factory.graph.edges
db.factory.duration
db.factory.failures
db.factory.persistence.duration
db.factory.unique.retries
```

---

# 243. Tracing

Span principal:

```text
database.factory.generate
```

Child spans opcionales:

```text
factory.attributes
factory.relationships
factory.construct
factory.persist
```

---

# 244. Debug mode

Podrá producir:

```text
FactoryExecutionPlan
```

sin construir objetos.

---

# 245. Explain API

Conceptualmente:

```php
DB::factories()
    ->explain(UserFactory::new()->state('admin'));
```

---

# 246. Explain output

```text
FACTORY PLAN

Target:
    App\Models\User

Kind:
    MODEL

Count:
    1

States:
    admin

Attributes:
    name      PersonDataProvider
    email     InternetDataProvider
    role      state:admin
    active    literal:true

Relationships:
    profile   HAS_ONE ProfileFactory

Randomness:
    DETERMINISTIC

Seed:
    4201

Persistence:
    NONE

Validation:
    STRUCTURAL
```

---

# 247. Error hierarchy

```text
DatabaseFactoryException
│
├── FactoryDefinitionException
│   ├── InvalidFactoryDefinitionException
│   ├── FactoryTargetException
│   ├── FactoryStateDefinitionException
│   └── AttributeDependencyCycleException
│
├── FactoryResolutionException
│   ├── FactoryNotFoundException
│   └── FactoryProviderNotFoundException
│
├── FactoryGenerationException
│   ├── FactoryAttributeGenerationException
│   ├── FactoryConstructionException
│   ├── FactoryValidationException
│   ├── FactoryUniqueValueExhaustedException
│   └── FactoryGenerationLimitException
│
├── FactoryRelationshipException
│   ├── FactoryRelationshipNotFoundException
│   ├── FactoryRelationshipTypeException
│   ├── FactoryRelationshipCycleException
│   └── FactoryCrossDomainRelationshipException
│
├── FactoryPersistenceException
│   ├── FactoryPersistenceFailedException
│   └── FactoryPersistencePolicyException
│
└── FactoryRuntimeException
    ├── FactoryCancelledException
    └── FactoryContextLeakException
```

---

# 248. Error context

Errores deberán poder incluir:

```text
factory target
state
attribute
relationship path
instance index
execution ID
```

sin exponer datos sensibles.

---

# 249. Relationship error example

```text
Factory relationship generation failed.

Factory:
    UserFactory

Path:
    users[3].posts[2].author

Reason:
    Relationship target is incompatible with configured factory.

Expected:
    App\Entity\User

Received:
    App\Entity\Organization
```

---

# 250. Deterministic error reproduction

Cuando exista seed:

```text
FactoryExecutionId
Seed
GenerationVersion
```

deberán facilitar reproducir el fallo.

---

# 251. Testing the Factory System

La implementación deberá incluir pruebas de:

- definitions;
- states;
- sequences;
- overrides;
- deterministic generation;
- random providers;
- locales;
- relationship graphs;
- cyclic graph handling;
- constructor strategies;
- value objects;
- Model API;
- Entity API;
- persistence bridge;
- transaction behavior;
- batching;
- streaming;
- cancellation;
- multitenancy extension;
- sharding;
- persistent workers;
- concurrency;
- security;
- telemetry.

---

# 252. Determinism test

```text
Factory F
Seed 123
Count 100

Run A
=
Run B
```

para providers deterministas compatibles.

---

# 253. Different seed test

```text
Seed 123
≠
Seed 456
```

deberá producir diversidad suficiente.

No es obligatorio que cada atributo difiera.

---

# 254. State precedence test

```text
base
→ state A
→ state B
→ explicit override
```

deberá producir resultado determinista.

---

# 255. Sequence isolation test

Dos ejecuciones independientes no compartirán índices.

---

# 256. Worker isolation test

```text
Request A factory seed
≠ leaked into Request B
```

---

# 257. Concurrent factory test

Dos coroutines podrán generar datos simultáneamente sin corromper:

```text
sequence
random source
unique pools
execution reports
```

---

# 258. Graph test

Verificar:

```text
User × 10
└── Posts × 5
    └── Comments × 3
```

Cardinalidad:

```text
Users = 10
Posts = 50
Comments = 150
```

---

# 259. Graph identity test

Un parent compartido no deberá duplicarse accidentalmente cuando el plan declare reutilización.

---

# 260. Existing reference test

Factory deberá mantener la referencia existente sin reconstruirla.

---

# 261. No-persistence test

```php
$users = UserFactory::new()
    ->count(100)
    ->make();
```

deberá ejecutar:

```text
0 INSERT
0 UPDATE
0 DELETE
```

---

# 262. create test

`create()` deberá pasar por Persistence Engine.

Nunca SQL directo desde Factory.

---

# 263. Transaction test

Factory persistence deberá respetar:

```text
flush ≠ commit
```

---

# 264. Failure test

Si persistencia falla parcialmente:

```text
FactoryExecutionReport
```

no deberá declarar éxito completo.

---

# 265. UNKNOWN persistence outcome

Si transaction outcome es UNKNOWN:

```text
FactoryPersistenceResult
```

deberá conservar UNKNOWN.

---

# 266. FactoryPersistenceResult

```php
final readonly class FactoryPersistenceResult
{
    public function __construct(
        public FactoryPersistenceOutcome $outcome,
        public int $scheduled,
        public int $synchronized,
        public int $confirmed,
    ) {}
}
```

---

# 267. Outcome

```php
enum FactoryPersistenceOutcome
{
    case SUCCEEDED;
    case FAILED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 268. No fabricated success

Regla:

```text
UNKNOWN
≠
SUCCESS
```

---

# 269. Performance model

Costo aproximado:

```text
FactoryCost
=
AttributeGeneration
+
StateResolution
+
RelationshipPlanning
+
ObjectConstruction
+
OptionalValidation
+
OptionalPersistence
```

---

# 270. Graph cost

Para un árbol:

```text
TotalObjects
=
Σ generated nodes
```

---

# 271. Example

```text
100 users
× 10 posts
× 5 comments
```

produce:

```text
100 users
1,000 posts
5,000 comments
=
6,100 objects
```

---

# 272. Hidden explosion protection

El planner deberá poder detectar grafos excesivos.

---

# 273. FactoryGenerationBudget

```php
final readonly class FactoryGenerationBudget
{
    public function __construct(
        public int $maxObjects,
        public int $maxDepth,
        public int $maxRelationships,
        public int $maxUniqueRetries,
        public ?Duration $maxDuration,
    ) {}
}
```

---

# 274. Budget exceeded

Deberá producir:

```text
FactoryGenerationLimitException
```

---

# 275. Default limits

Desarrollo y testing podrán tener límites razonables.

CLI podrá ampliarlos explícitamente.

---

# 276. Recursive factories

Ejemplo:

```text
Category
└── children
    └── children
        └── ...
```

requerirá depth limit.

---

# 277. No infinite recursion

Obligatorio detectar/proteger.

---

# 278. Relationship recursion policy

```php
enum FactoryRecursionPolicy
{
    case FORBID;
    case BOUNDED;
    case EXPLICIT;
}
```

---

# 279. Factory graph reuse

Podrá reutilizar nodos.

Ejemplo:

```text
100 posts
→ same User
```

sin generar 100 usuarios.

---

# 280. Shared node

Debe declararse explícitamente.

---

# 281. Factory references

```php
$author = UserFactory::new();

PostFactory::new()
    ->count(10)
    ->forShared($author);
```

API final podrá variar.

---

# 282. Deferred references

El planner podrá representar:

```text
FactoryNodeReference
```

antes de construir el objeto.

---

# 283. Directory structure

Propuesta:

```text
src/Quantum/Database/Factory/
│
├── Factory.php
├── FactoryDefinition.php
├── FactoryBlueprint.php
├── FactoryRequest.php
├── FactoryContext.php
├── FactoryInstanceContext.php
├── FactoryBuildResult.php
├── FactoryExecutionId.php
├── FactoryExecutionState.php
├── FactoryExecutionReport.php
│
├── Target/
│   ├── FactoryTarget.php
│   └── FactoryTargetKind.php
│
├── Attribute/
│   ├── FactoryAttribute.php
│   ├── FactoryAttributeSet.php
│   ├── FactoryAttributePlan.php
│   ├── FactoryAttributeResolver.php
│   ├── FactoryAttributeDependencyGraph.php
│   └── ResolvedFactoryAttributes.php
│
├── State/
│   ├── FactoryState.php
│   ├── FactoryStateSet.php
│   ├── FactoryStateRegistry.php
│   ├── FactoryStateTransformation.php
│   └── FactoryStateResolver.php
│
├── Sequence/
│   ├── FactorySequence.php
│   ├── FactorySequenceContext.php
│   ├── FactorySequencePlan.php
│   └── FactorySequenceResolver.php
│
├── Random/
│   ├── FactoryRandomSource.php
│   ├── FactoryRandomnessPolicy.php
│   ├── DeterministicRandomSource.php
│   └── SecureRandomSource.php
│
├── Data/
│   ├── FactoryDataProvider.php
│   ├── FactoryDataProviderRegistry.php
│   ├── PersonDataProvider.php
│   ├── InternetDataProvider.php
│   ├── AddressDataProvider.php
│   ├── CompanyDataProvider.php
│   └── DateDataProvider.php
│
├── Relationship/
│   ├── FactoryGraphPlan.php
│   ├── FactoryGraphPlanner.php
│   ├── FactoryNode.php
│   ├── FactoryNodeKind.php
│   ├── FactoryNodeReference.php
│   ├── FactoryRelationshipPlan.php
│   └── FactoryRecursionPolicy.php
│
├── Construction/
│   ├── FactoryConstructor.php
│   ├── FactoryConstructionPlan.php
│   ├── FactoryConstructionContext.php
│   ├── FactoryConstructionStrategy.php
│   └── FactoryConstructorRegistry.php
│
├── Persistence/
│   ├── FactoryPersistenceBridge.php
│   ├── FactoryPersistenceContext.php
│   ├── FactoryPersistenceResult.php
│   ├── FactoryPersistenceOutcome.php
│   └── FactoryFlushPolicy.php
│
├── Validation/
│   ├── FactoryValidator.php
│   └── FactoryValidationPolicy.php
│
├── Unique/
│   ├── FactoryUniqueValuePool.php
│   ├── FactoryValueKey.php
│   └── FactoryUniquePolicy.php
│
├── Clock/
│   └── FactoryClock.php
│
├── Registry/
│   ├── FactoryRegistry.php
│   └── FactoryResolver.php
│
├── Compilation/
│   ├── FactoryCompiler.php
│   ├── CompiledFactoryRegistry.php
│   └── FactoryGenerationVersion.php
│
├── Runtime/
│   ├── FactoryGenerationBudget.php
│   ├── FactoryGenerationPolicy.php
│   └── FactoryCancellationHandler.php
│
├── Telemetry/
│   ├── FactoryTelemetry.php
│   └── FactoryExecutionTracer.php
│
├── Diagnostics/
│   ├── FactoryInspector.php
│   ├── FactoryExplainer.php
│   └── FactoryExecutionPlan.php
│
└── Exception/
    ├── DatabaseFactoryException.php
    ├── FactoryDefinitionException.php
    ├── FactoryResolutionException.php
    ├── FactoryGenerationException.php
    ├── FactoryRelationshipException.php
    ├── FactoryPersistenceException.php
    └── FactoryRuntimeException.php
```

---

# 284. Public API conceptual

## Construcción simple

```php
$user = UserFactory::new()->make();
```

---

## Múltiples objetos

```php
$users = UserFactory::new()
    ->count(10)
    ->make();
```

---

## State

```php
$admin = UserFactory::new()
    ->state('admin')
    ->make();
```

---

## Fluent state

```php
$admin = UserFactory::new()
    ->admin()
    ->verified()
    ->make();
```

---

## Override

```php
$user = UserFactory::new()
    ->make([
        'email' => 'john@example.test',
    ]);
```

---

## Deterministic

```php
$users = UserFactory::new()
    ->seed(100)
    ->count(10)
    ->make();
```

---

## Relationship

```php
$user = UserFactory::new()
    ->has(
        PostFactory::new()->count(5),
        'posts'
    )
    ->make();
```

---

## Existing parent

```php
$post = PostFactory::new()
    ->for($user, 'author')
    ->make();
```

---

## Persistence

```php
$users = UserFactory::new()
    ->count(10)
    ->create();
```

---

## Deferred persistence

```php
$users = UserFactory::new()
    ->count(10)
    ->make();

foreach ($users as $user) {
    $entityManager->persist($user);
}

$entityManager->flush();
```

---

# 285. Architectural invariants

## DB-FAC-001
Factory construirá objetos o valores.

## DB-FAC-002
Factory no será Persistence Engine.

## DB-FAC-003
Factory no generará SQL.

## DB-FAC-004
Factory no accederá directamente a Driver.

## DB-FAC-005
Factory no accederá directamente a Connection.

## DB-FAC-006
Factory será distinta de Seeder.

## DB-FAC-007
Factory será distinta de Fixture.

## DB-FAC-008
Factory será distinta de Faker/DataProvider.

## DB-FAC-009
Factory será distinta de Hydrator.

## DB-FAC-010
Factory será distinta de Repository.

## DB-FAC-011
Factory será distinta de EntityManager.

## DB-FAC-012
ModelFactory y EntityFactory compartirán motor conceptual.

## DB-FAC-013
Factory Definition será distinta de Factory execution.

## DB-FAC-014
Compiled Blueprint será distinto de generated object.

## DB-FAC-015
Factory configuration será preferentemente immutable.

## DB-FAC-016
count() no construirá inmediatamente.

## DB-FAC-017
make() no persistirá.

## DB-FAC-018
create() utilizará PersistenceBridge.

## DB-FAC-019
PersistenceBridge utilizará ORM/Persistence Engine.

## DB-FAC-020
PersistenceBridge no generará SQL directamente.

## DB-FAC-021
FactoryContext será scoped.

## DB-FAC-022
No existirá static current FactoryContext.

## DB-FAC-023
Factory state no filtrará entre requests.

## DB-FAC-024
Factory state no filtrará entre coroutines.

## DB-FAC-025
Factory state no filtrará entre fibers.

## DB-FAC-026
Deterministic generation utilizará explicit seed.

## DB-FAC-027
Randomness será abstraída.

## DB-FAC-028
Definitions deterministas no usarán global rand directamente.

## DB-FAC-029
Generation version podrá formar parte del determinism contract.

## DB-FAC-030
FactoryDataProvider será distinto de Factory.

## DB-FAC-031
Locale será explícito.

## DB-FAC-032
Factory locale será distinto de DB collation.

## DB-FAC-033
Attributes podrán resolverse lazily.

## DB-FAC-034
Attribute resolvers se evaluarán por instancia cuando corresponda.

## DB-FAC-035
Explicit overrides tendrán precedencia definida.

## DB-FAC-036
State composition será determinista.

## DB-FAC-037
State order tendrá semántica definida.

## DB-FAC-038
Sequences serán scoped.

## DB-FAC-039
No habrá global mutable sequence counters.

## DB-FAC-040
Relationships utilizarán metadata cuando esté disponible.

## DB-FAC-041
Relationship factory target será validado.

## DB-FAC-042
FactoryGraphPlan será distinto de PersistencePlan.

## DB-FAC-043
Factory graph podrá existir sin DB.

## DB-FAC-044
Cyclic construction será explícitamente manejada.

## DB-FAC-045
Database-generated IDs no serán asumidos durante make().

## DB-FAC-046
Temporary object identity será distinta de DB identity.

## DB-FAC-047
Value Object Factory no persistirá VO independiente automáticamente.

## DB-FAC-048
Factory Construction será distinta de Hydration Construction.

## DB-FAC-049
External I/O estará prohibido por defecto en pure generation.

## DB-FAC-050
I/O-dependent generation será explícita.

## DB-FAC-051
Existing references serán distintas de generated nodes.

## DB-FAC-052
Factory no asumirá persistence ownership de existing objects.

## DB-FAC-053
Lifecycle callbacks tendrán fases explícitas.

## DB-FAC-054
afterPersisting será distinto de afterCommit.

## DB-FAC-055
Factory callbacks no fabricarán atomicidad externa.

## DB-FAC-056
Factory flush será distinto de transaction commit.

## DB-FAC-057
Batch factory creation será distinto de Bulk DML.

## DB-FAC-058
Large generation tendrá resource limits.

## DB-FAC-059
Streaming generation no persistirá automáticamente.

## DB-FAC-060
Chunk generation liberará temporary state cuando sea seguro.

## DB-FAC-061
Factory no ejecutará hidden EntityManager clear.

## DB-FAC-062
Factory Registry será congelable.

## DB-FAC-063
Factory Registry no mutará globalmente por request.

## DB-FAC-064
Production no realizará filesystem discovery por request.

## DB-FAC-065
Factory resolution será determinista.

## DB-FAC-066
Missing Factory producirá error explícito.

## DB-FAC-067
Runtime no inventará factories arbitrarias.

## DB-FAC-068
CLI podrá generar factory skeletons.

## DB-FAC-069
Schema inference no equivaldrá a business semantics.

## DB-FAC-070
Generated factories ambiguas requerirán revisión.

## DB-FAC-071
Factory generation será type-aware cuando corresponda.

## DB-FAC-072
Decimal generation respetará exactitud.

## DB-FAC-073
Enum generation utilizará cases válidos salvo invalid testing explícito.

## DB-FAC-074
DateTime generation respetará semantic type.

## DB-FAC-075
JSON generation producirá estructuras compatibles.

## DB-FAC-076
Validation será distinta de Factory.

## DB-FAC-077
Invalid testing será posible explícitamente.

## DB-FAC-078
Cross-field dependencies serán soportadas.

## DB-FAC-079
Attribute cycles serán detectados.

## DB-FAC-080
Uniqueness de Factory será distinta de DB uniqueness.

## DB-FAC-081
Database constraint seguirá siendo autoridad de uniqueness persistente.

## DB-FAC-082
Unique pools serán scoped.

## DB-FAC-083
Unique pools serán bounded.

## DB-FAC-084
Unique exhaustion producirá error.

## DB-FAC-085
Unique generation no tendrá retries infinitos.

## DB-FAC-086
FactoryClock será inyectable.

## DB-FAC-087
Deterministic factories no dependerán implícitamente de system time.

## DB-FAC-088
Relative dates utilizarán FactoryClock.

## DB-FAC-089
Non-deterministic callbacks serán observables.

## DB-FAC-090
Factory lifecycle será explícito.

## DB-FAC-091
Cancellation será soportada en generación larga.

## DB-FAC-092
Cancellation será distinta de rollback.

## DB-FAC-093
Transactions serán coordinadas fuera del Factory Engine.

## DB-FAC-094
FactoryGraph no implicará distributed transaction.

## DB-FAC-095
Factory no elegirá shards físicos arbitrariamente.

## DB-FAC-096
Partition Routing decidirá execution domain durante persistence.

## DB-FAC-097
Multitenancy será integración opcional.

## DB-FAC-098
No habrá static current tenant en Factory core.

## DB-FAC-099
Cross-tenant relationships respetarán ORM/domain rules.

## DB-FAC-100
Factories de testing no se expondrán accidentalmente en producción.

## DB-FAC-101
Synthetic data será default.

## DB-FAC-102
Factories no utilizarán secretos reales.

## DB-FAC-103
Sensitive generated values no aparecerán en telemetry.

## DB-FAC-104
Factory closures no serán serializadas arbitrariamente.

## DB-FAC-105
Remote generation requerirá portable descriptors.

## DB-FAC-106
Compiled Factory Cache no almacenará generated objects.

## DB-FAC-107
Compiled Blueprint será generation-aware.

## DB-FAC-108
Factory será distinta de Schema.

## DB-FAC-109
Schema-aware generation pertenecerá principalmente a tooling.

## DB-FAC-110
Worker-local compiled definitions deberán ser immutable.

## DB-FAC-111
Telemetry será bounded.

## DB-FAC-112
Telemetry no incluirá fake PII como labels.

## DB-FAC-113
Factory execution será explainable.

## DB-FAC-114
Errors identificarán factory path sin exponer secrets.

## DB-FAC-115
Seeds permitirán reproducir fallos cuando sea posible.

## DB-FAC-116
make() ejecutará cero operaciones de escritura DB.

## DB-FAC-117
create() pasará por Persistence Engine.

## DB-FAC-118
UNKNOWN persistence outcome permanecerá UNKNOWN.

## DB-FAC-119
Factory no fabricará persistence success.

## DB-FAC-120
Graph expansion tendrá budget.

## DB-FAC-121
Recursive factories tendrán depth policy.

## DB-FAC-122
Infinite recursion estará prohibida.

## DB-FAC-123
Shared graph nodes serán explícitos.

## DB-FAC-124
Factory node references podrán resolverse diferidamente.

## DB-FAC-125
Object construction será independiente del SQL dialect.

## DB-FAC-126
Factory Engine será independiente de PDO.

## DB-FAC-127
Factory Engine será independiente de Redis.

## DB-FAC-128
Factory Engine será independiente del HTTP layer.

## DB-FAC-129
Factory Engine será utilizable desde CLI.

## DB-FAC-130
Factory Engine será utilizable desde tests.

## DB-FAC-131
Factory Engine será utilizable desde seeders.

## DB-FAC-132
Factory Engine será utilizable desde fixtures.

## DB-FAC-133
Factory Engine será utilizable sin Seeder System.

## DB-FAC-134
Factory Engine será utilizable sin Fixture System.

## DB-FAC-135
Model API no creará un segundo factory engine.

## DB-FAC-136
Entity API no creará un segundo factory engine.

## DB-FAC-137
Factory relationships respetarán canonical ORM metadata.

## DB-FAC-138
Rich many-to-many associations usarán association entities cuando corresponda.

## DB-FAC-139
Generated object graph no será automáticamente managed.

## DB-FAC-140
make() no registrará automáticamente objetos en UnitOfWork.

## DB-FAC-141
create() podrá registrar objetos mediante PersistenceBridge.

## DB-FAC-142
Factory no modificará IdentityMap directamente.

## DB-FAC-143
Factory no manipulará UnitOfWork internals.

## DB-FAC-144
Factory no llamará Compiler directamente.

## DB-FAC-145
Factory no llamará Driver directamente.

## DB-FAC-146
Factory no ejecutará prepared statements.

## DB-FAC-147
Factory no inferirá commit por flush.

## DB-FAC-148
Factory persistence respetará transaction ownership.

## DB-FAC-149
Factory persistence respetará shard affinity.

## DB-FAC-150
Factory persistence respetará tenant isolation.

## DB-FAC-151
Factory generation deberá ser testable sin database.

## DB-FAC-152
Factory compilation deberá ser testable independientemente.

## DB-FAC-153
Provider registries deberán ser testables independientemente.

## DB-FAC-154
Factory graph planning deberá ser testable sin persistence.

## DB-FAC-155
Factory construction deberá ser testable sin ORM execution.

## DB-FAC-156
Factory execution state será local a la operación.

## DB-FAC-157
Factory budgets serán enforceable.

## DB-FAC-158
Factories no producirán crecimiento de memoria sin límites.

## DB-FAC-159
Generated values deberán respetar logical database types cuando se solicite type-aware generation.

## DB-FAC-160
Factory behavior deberá ser predecible bajo persistent runtimes.

---

# 286. Modelo formal

Sea una Factory:

```text
F
```

con:

```text
D = Factory Definition
S = State Set
O = Overrides
R = Random Source
Q = Sequence Context
C = Factory Context
N = Requested Count
```

La construcción será:

```text
Objects
=
F(D, S, O, R, Q, C, N)
```

La persistencia no forma parte de esta función fundamental.

Es una transformación posterior:

```text
PersistenceResult
=
P(Objects, PersistenceContext)
```

Por tanto:

```text
F
≠
P
```

---

# 287. Modelo de reproducibilidad

Para una Factory determinista:

```text
F(D, S, O, Seed, Version)
```

deberá satisfacer:

```text
F(x) = F(x)
```

para ejecuciones compatibles.

Si cambia:

```text
D
Version
ProviderVersion
LocaleDataset
```

la igualdad ya no está garantizada.

---

# 288. Modelo de grafo

Sea:

```text
G = (V, E)
```

donde:

```text
V = generated/existing factory nodes
E = relationships
```

El Factory Graph Planner produce:

```text
FactoryGraphPlan(G)
```

pero no:

```text
SQLPlan(G)
```

Posteriormente:

```text
Construct(G)
       ↓
Objects
       ↓
Persistence Planner
       ↓
Database Operations
```

---

# 289. Modelo de cardinalidad

Si una Factory raíz genera:

```text
N
```

objetos y cada relación `r` genera:

```text
C(r)
```

objetos por parent, el tamaño puede crecer multiplicativamente.

Ejemplo:

```text
N(User) = 100
C(Post) = 10
C(Comment) = 5
```

entonces:

```text
Users = 100

Posts =
100 × 10
=
1,000

Comments =
100 × 10 × 5
=
5,000

Total =
6,100
```

Por ello `FactoryGenerationBudget` será una parte fundamental del diseño.

---

# 290. Integración con Database Architecture

```text
Factory System
     │
     ├───────────────► Type System
     │
     ├───────────────► ORM Metadata
     │
     ├───────────────► Relationship Metadata
     │
     ├───────────────► Validation
     │
     └───────────────► Persistence Bridge
                              │
                              ▼
                         EntityManager
                              │
                              ▼
                         UnitOfWork
                              │
                              ▼
                    Persistence Engine
                              │
                              ▼
                    Persistence Planner
                              │
                              ▼
                         Query Engine
                              │
                              ▼
                      Partition Routing
                              │
                              ▼
                         Execution
```

Factory System no saltará ninguna de estas capas para "optimizar" `create()`.

---

# 291. Relación con documentos anteriores

Factory System reutilizará conceptos de:

```text
112 DATABASE_ORM_ARCHITECTURE
113 DATABASE_ENTITY_MODEL
114 DATABASE_MODEL_API_SYSTEM
115 DATABASE_ENTITY_METADATA_SYSTEM
116 DATABASE_ENTITY_MAPPING_SYSTEM
118 DATABASE_ENTITY_MANAGER_SYSTEM
123 DATABASE_IDENTITY_MAP_SYSTEM
124 DATABASE_UNIT_OF_WORK_ARCHITECTURE
127 DATABASE_PERSISTENCE_ENGINE
128 DATABASE_PERSISTENCE_PLANNER_SYSTEM
133 DATABASE_BATCH_PERSISTENCE_SYSTEM
142 DATABASE_RELATIONSHIP_ARCHITECTURE
148 DATABASE_RELATIONSHIP_METADATA_SYSTEM
155 DATABASE_TYPE_SYSTEM
163 DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM
164 DATABASE_TRANSACTION_ARCHITECTURE
174 DATABASE_CONCURRENCY_CONTROL_SYSTEM
183 DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE
184 DATABASE_SHARDING_SYSTEM
185 DATABASE_PARTITION_ROUTING_SYSTEM
```

sin duplicar sus responsabilidades.

---

# 292. Arquitectura final

```text
                     FACTORY DEFINITION
                            │
                            ▼
                    FACTORY COMPILER
                            │
                            ▼
                  COMPILED BLUEPRINT
                            │
                            ▼
                     FACTORY REQUEST
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
          STATES        SEQUENCES       OVERRIDES
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                    ATTRIBUTE GRAPH
                            │
                            ▼
                   ATTRIBUTE RESOLVER
                            │
                            ▼
                 RELATIONSHIP PLANNER
                            │
                            ▼
                   CONSTRUCTION PLAN
                            │
                            ▼
                     OBJECT GRAPH
                            │
                    ┌───────┴────────┐
                    │                │
                  make()           create()
                    │                │
                    ▼                ▼
                 RETURN       PERSISTENCE BRIDGE
                                     │
                                     ▼
                                EntityManager
                                     │
                                     ▼
                                  UnitOfWork
                                     │
                                     ▼
                            Persistence Engine
                                     │
                                     ▼
                                Query Engine
                                     │
                                     ▼
                                  Database
```

---

# 293. Regla maestra final

El sistema seguirá:

```text
Factory
    ↓
Describe generation
    ↓
Resolve state
    ↓
Generate attributes
    ↓
Construct relationships
    ↓
Construct objects
```

y solo cuando el caller lo solicite:

```text
Constructed Objects
    ↓
Persistence Bridge
    ↓
ORM
    ↓
Persistence Engine
    ↓
Database
```

Nunca:

```text
Factory
    ↓
Generate SQL
    ↓
INSERT
```

---

# 294. Principio arquitectónico

> **VoltStack Factory System será un motor de construcción de objetos y grafos de dominio, no un generador disfrazado de INSERTs.**

Esto permite que una misma Factory pueda utilizarse para:

```text
Unit Tests
Integration Tests
Seeders
Fixtures
Development
Benchmarks
Data Generation
CLI Tools
ORM Testing
Package Testing
Demo Environments
```

sin acoplarla a una base de datos concreta.

---

# 295. Resultado del documento

Con `193_DATABASE_FACTORY_SYSTEM.md` queda establecida la arquitectura raíz del **Bloque 18**:

```text
                193 FACTORY SYSTEM
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
194 MODEL FACTORY            195 ENTITY FACTORY
          │                           │
          └─────────────┬─────────────┘
                        ▼
                 196 SEEDER SYSTEM
                        │
                        ▼
                197 FIXTURE SYSTEM
                        │
                        ▼
          198 TEST DATA GENERATION
```

---

# 296. Siguiente documento

```text
194_DATABASE_MODEL_FACTORY_SYSTEM.md
```

Este documento definirá específicamente la API Laravel-like de VoltStack:

```php
User::factory()->make();

User::factory()
    ->count(10)
    ->create();

User::factory()
    ->admin()
    ->verified()
    ->has(
        Post::factory()->count(5)
    )
    ->create();
```

manteniendo la regla:

```text
ModelFactory
=
Developer Convenience API
        ↓
Shared Factory Engine
        ↓
Model Persistence Bridge
        ↓
Canonical ORM/Persistence Engine
```

y nunca:

```text
ModelFactory
=
Second ORM
```