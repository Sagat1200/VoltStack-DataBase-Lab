# 197_DATABASE_FIXTURE_SYSTEM.md

# VoltStack Quantum Database
## Database Fixture System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 197 — Database Fixture System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `196_DATABASE_SEEDER_SYSTEM.md`  
**Siguiente documento:** `198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md`

---

# 1. Propósito

`Database Fixture System` define la arquitectura mediante la cual VoltStack podrá crear, cargar, identificar, reutilizar y limpiar **escenarios de datos conocidos y reproducibles** utilizados principalmente durante pruebas automatizadas.

Una Fixture no responde simplemente:

> ¿Cómo genero un usuario?

Eso corresponde a Factory.

Tampoco responde:

> ¿Qué datos iniciales necesita esta aplicación?

Eso corresponde a Seeder.

Fixture responde:

> **¿Qué estado de datos conocido necesita existir para ejecutar este escenario de prueba de forma reproducible?**

Ejemplo:

```php
final class OrderWithExpiredPaymentFixture extends Fixture
{
    public function load(FixtureContext $context): FixtureResult
    {
        // Construye el escenario completo.
    }
}
```

Uso:

```php
$fixture = $fixtures->load(
    OrderWithExpiredPaymentFixture::class
);

$order = $fixture->get('order');
$customer = $fixture->get('customer');
```

La regla central será:

> **Fixture representa un escenario de datos reproducible con identidad, dependencias, referencias, alcance y ciclo de vida explícitos; puede utilizar Factories, Seeders y APIs de Database, pero no se convierte en ninguno de ellos.**

Formalmente:

```text
Fixture
=
Scenario Definition
+
Dataset Composition
+
Reference Registry
+
Lifecycle Policy
+
Isolation Policy
+
Reproducibility Contract
```

Nunca:

```text
Fixture
=
Factory
```

ni:

```text
Fixture
=
Seeder
```

ni:

```text
Fixture
=
Database Dump
```

---

# 2. Posición arquitectónica

El bloque queda:

```text
193_DATABASE_FACTORY_SYSTEM
        │
        ├── 194_DATABASE_MODEL_FACTORY_SYSTEM
        │
        └── 195_DATABASE_ENTITY_FACTORY_SYSTEM
        │
        ▼
196_DATABASE_SEEDER_SYSTEM
        │
        ▼
197_DATABASE_FIXTURE_SYSTEM
        │
        ▼
198_DATABASE_TEST_DATA_GENERATION_SYSTEM
```

Responsabilidades:

| Sistema | Responsabilidad |
|---|---|
| Factory | Construir instancias |
| ModelFactory | Construir Models |
| EntityFactory | Construir Entities |
| Seeder | Orquestar datasets |
| Fixture | Representar escenarios reproducibles |
| Test Data Generator | Generar datasets especializados para testing |

---

# 3. Distinciones fundamentales

```text
Fixture
≠
Factory
≠
Seeder
≠
Migration
≠
Snapshot
≠
Database Dump
≠
Test Case
≠
Transaction
```

Estas diferencias deberán mantenerse en toda la implementación.

---

# 4. Fixture vs Factory

Factory:

```php
$user = UserEntityFactory::new()->make();
```

describe:

```text
How to construct User
```

Fixture:

```php
$scenario = $fixtures->load(
    UserWithThreePendingOrdersFixture::class
);
```

describe:

```text
User
├── 3 Pending Orders
├── Billing Profile
├── Address
└── Payment Method
```

con valores, referencias y relaciones coherentes para un escenario concreto.

---

# 5. Fixture vs Seeder

Seeder:

```text
Application Dataset
```

Fixture:

```text
Test Scenario Dataset
```

Ejemplo:

```text
RoleSeeder

admin
manager
customer
```

frente a:

```text
CustomerWithoutPermissionFixture

Customer
Role: customer
Permission: intentionally missing
```

---

# 6. Fixture vs Migration

Migration:

```text
Schema(t0)
→
Schema(t1)
```

Fixture:

```text
Valid Schema
+
Known Test Scenario
→
Known Data State
```

Fixture no deberá modificar estructura DB como responsabilidad normal.

---

# 7. Fixture vs Database Dump

Un dump representa principalmente:

```text
physical database contents
```

Una Fixture representa:

```text
semantic test scenario
```

El dump puede contener:

```text
auto-increment IDs
engine-specific SQL
irrelevant rows
historical state
platform-specific syntax
```

La Fixture debe describir el escenario que interesa al test.

---

# 8. Fixture vs Snapshot

Snapshot puede representar:

```text
captured state at time T
```

Fixture representa:

```text
reconstructable scenario definition
```

Por tanto:

```text
Snapshot
≠
Fixture
```

aunque un Snapshot pueda ser una fuente de Fixture especializada.

---

# 9. Objetivos

El sistema deberá proporcionar:

1. escenarios reproducibles;
2. referencias simbólicas;
3. dependency graph;
4. Factory integration;
5. Seeder integration controlada;
6. ORM integration;
7. Query API integration;
8. transaction isolation;
9. cleanup;
10. deterministic generation;
11. fixture composition;
12. reusable fixture modules;
13. test isolation;
14. tenant isolation;
15. shard awareness;
16. platform portability;
17. diagnostics;
18. telemetry;
19. persistent-runtime safety;
20. parallel-test safety.

---

# 10. API conceptual

```php
$result = $fixtures->load(
    UserWithOrdersFixture::class
);
```

Después:

```php
$user = $result->get('user');
$order = $result->get('order.pending');
```

---

# 11. Fixture base

```php
abstract class Fixture
{
    abstract public function load(
        FixtureContext $context
    ): FixtureResult;
}
```

---

# 12. Ejemplo básico

```php
final class ActiveCustomerFixture extends Fixture
{
    public function load(
        FixtureContext $context
    ): FixtureResult {
        $customer = CustomerEntityFactory::new()
            ->active()
            ->make();

        $context->entityManager()->persist($customer);
        $context->entityManager()->flush();

        return FixtureResult::make()
            ->reference('customer', $customer);
    }
}
```

---

# 13. Referencias

Una característica esencial será:

```text
Fixture Reference
```

Ejemplo:

```php
$result->reference(
    'customer',
    $customer
);
```

---

# 14. FixtureReferenceId

Las referencias no deberán depender de variables internas del método.

Conceptualmente:

```text
FixtureReferenceId
=
fixture.customer
```

---

# 15. Uso

```php
$customer = $fixtureResult->get('customer');
```

---

# 16. Referencias tipadas

Idealmente:

```php
$customer = $fixtureResult->get(
    'customer',
    Customer::class
);
```

---

# 17. Type mismatch

Si:

```text
reference = customer
actual = Customer
requested = Order
```

deberá fallar explícitamente.

---

# 18. FixtureResult

Propuesta:

```php
final readonly class FixtureResult
{
    public function __construct(
        public FixtureId $fixtureId,
        public FixtureExecutionId $executionId,
        public FixtureReferenceRegistry $references,
        public FixtureExecutionStatus $status,
    ) {}
}
```

---

# 19. Reference Registry

```text
FixtureReferenceRegistry
├── customer
├── pendingOrder
├── expiredPayment
└── product
```

---

# 20. Referencias ≠ IdentityMap

Muy importante:

```text
FixtureReferenceRegistry
≠
IdentityMap
```

Fixture Registry permite localizar objetos del escenario.

IdentityMap garantiza canonical managed identity dentro del EntityManager scope.

---

# 21. Referencias ≠ Cache

Tampoco:

```text
FixtureReferenceRegistry
≠
Entity Cache
≠
Result Cache
```

---

# 22. FixtureId

Cada Fixture tendrá identidad estable:

```text
tests.orders.expired-payment
```

---

# 23. No depender exclusivamente del FQCN

El FQCN podrá ser implementación:

```text
Tests\Fixtures\OrderWithExpiredPaymentFixture
```

pero la identidad lógica podrá ser estable.

---

# 24. Fixture Metadata

Propuesta:

```php
final readonly class FixtureMetadata
{
    public function __construct(
        public FixtureId $id,
        public string $class,
        public array $dependencies,
        public array $tags,
        public FixtureIsolationPolicy $isolation,
        public FixtureCleanupPolicy $cleanup,
        public FixtureReproducibilityPolicy $reproducibility,
    ) {}
}
```

---

# 25. Fixture Context

Cada carga recibirá:

```text
FixtureContext
├── FixtureExecutionId
├── DatabaseContext
├── EntityManager
├── FactoryRuntime
├── RandomSource
├── Clock
├── ReferenceRegistry
├── TransactionContext
├── TenantContext?
├── ShardContext?
├── TestExecutionContext?
├── CancellationToken
└── TelemetryContext
```

---

# 26. Context scoped

Nunca:

```php
static $fixtureContext;
```

---

# 27. Reproducibilidad

La reproducibilidad será una propiedad explícita.

Propuesta:

```php
enum FixtureReproducibilityPolicy
{
    case STRICT;
    case SEEDED;
    case BEST_EFFORT;
    case EXTERNAL_DEPENDENCY;
}
```

---

# 28. STRICT

Mismo:

```text
Fixture Definition
Database Schema
Configuration
Input
```

deberá producir un escenario lógicamente equivalente.

---

# 29. SEEDED

Permite datos pseudoaleatorios, pero con seed conocida.

```php
$fixtures->load(
    CustomerFixture::class,
    seed: 12345,
);
```

---

# 30. BEST_EFFORT

Puede contener valores como timestamps dinámicos cuando el escenario no requiere igualdad exacta.

---

# 31. EXTERNAL_DEPENDENCY

Fixture depende explícitamente de recursos externos.

Debe ser poco común.

---

# 32. Clock

Fixture deberá poder recibir:

```php
FixedClock::at(
    '2026-01-01T12:00:00Z'
);
```

---

# 33. Razón

Un escenario:

```text
subscription expired yesterday
```

debe poder reproducirse.

---

# 34. Ejemplo temporal

```php
$now = $context->clock()->now();

$subscription = SubscriptionEntityFactory::new()
    ->expiredAt(
        $now->minusDays(1)
    )
    ->make();
```

---

# 35. No wall clock oculto

Evitar:

```php
new DateTimeImmutable();
```

dentro de Fixtures reproducibles.

---

# 36. Random source

Igualmente:

```text
Fixture RandomSource
```

deberá ser injectable/seedable.

---

# 37. Factory integration

Fixture podrá utilizar:

```text
ModelFactory
EntityFactory
```

---

# 38. Ejemplo ModelFactory

```php
$user = User::factory()
    ->verified()
    ->create();
```

---

# 39. Ejemplo EntityFactory

```php
$user = UserEntityFactory::new()
    ->verified()
    ->persist(
        $context->entityManager()
    );
```

---

# 40. Factory seed propagation

El `FixtureContext` deberá poder propagar una fuente determinista a Factories hijas.

---

# 41. No random collision global

Dos Fixtures paralelas no deberán compartir mutable random state.

---

# 42. Fixture composition

Una Fixture podrá depender de otras.

Ejemplo:

```text
OrderPaymentFailureFixture
├── CustomerFixture
├── ProductCatalogFixture
└── PaymentProviderFixture
```

---

# 43. Declaración

```php
public function dependencies(): array
{
    return [
        CustomerFixture::class,
        ProductCatalogFixture::class,
    ];
}
```

---

# 44. Dependency graph

```text
CustomerFixture ───────────┐
                           ▼
ProductCatalogFixture ─► OrderFixture
                           │
                           ▼
                   PaymentFailureFixture
```

---

# 45. Fixture DAG

El sistema deberá construir:

```text
FixtureDependencyGraph
```

y detectar ciclos.

---

# 46. Cycle

```text
Fixture A
 ↓
Fixture B
 ↓
Fixture C
 ↓
Fixture A
```

deberá ser rechazado antes de mutation cuando sea detectable.

---

# 47. Shared dependencies

Caso:

```text
        CustomerFixture
        /             \
       ▼               ▼
OrderFixture      ProfileFixture
       \               /
        ▼             ▼
        CheckoutFixture
```

`CustomerFixture` no deberá cargarse dos veces accidentalmente dentro del mismo Fixture Graph si su metadata declara reuse compatible.

---

# 48. Fixture reuse policy

Propuesta:

```php
enum FixtureReusePolicy
{
    case REUSE_WITHIN_GRAPH;
    case ALWAYS_NEW;
    case EXPLICIT;
}
```

---

# 49. REUSE_WITHIN_GRAPH

Una dependencia compartida se carga una vez por graph execution.

---

# 50. ALWAYS_NEW

Cada edge produce un escenario independiente.

---

# 51. EXPLICIT

El caller decide.

---

# 52. Fixture dependency ≠ Seeder dependency

Aunque ambos utilicen DAGs:

```text
Fixture Graph
≠
Seeder Graph
```

porque tienen ciclos de vida y objetivos diferentes.

---

# 53. References entre Fixtures

Una Fixture hija podrá exportar:

```text
customer
```

y la padre podrá importarla como:

```text
base.customer
```

---

# 54. Namespaces

Evitar colisiones:

```text
customer
customer
customer
```

entre dependencias.

---

# 55. Reference Namespace

Ejemplo:

```text
customerFixture.customer
catalog.product.standard
payment.card.expired
```

---

# 56. Alias local

La Fixture padre podrá definir:

```text
customerFixture.customer
→
customer
```

---

# 57. Reference immutability

Después de publicar una referencia:

```text
customer
```

no deberá reemplazarse silenciosamente por otro objeto.

---

# 58. Duplicate reference

Debe producir:

```text
FixtureDuplicateReferenceException
```

salvo policy explícita.

---

# 59. Fixture scenario graph

Ejemplo:

```text
OrderWithExpiredPaymentFixture
│
├── customer
│
├── order
│   ├── line[0]
│   │   └── product
│   └── payment
│       └── card
│
└── expected:
    paymentStatus = EXPIRED
```

---

# 60. Fixture ≠ Assertion

La Fixture prepara datos.

El test decide qué afirmar.

---

# 61. Anti-pattern

No:

```php
final class FailedPaymentFixture extends Fixture
{
    public function load(...): FixtureResult
    {
        // setup

        assert($payment->failed());

        // ...
    }
}
```

---

# 62. Excepción

Fixture podrá validar sus propias invariantes estructurales.

Ejemplo:

```text
required reference missing
relationship malformed
fixture setup incomplete
```

Eso no sustituye assertions del test.

---

# 63. Isolation

El sistema deberá definir cómo aislar Fixtures.

---

# 64. FixtureIsolationPolicy

Propuesta:

```php
enum FixtureIsolationPolicy
{
    case NONE;
    case TRANSACTION;
    case DATABASE_RESET;
    case SCHEMA_RESET;
    case TENANT;
    case DATABASE;
    case CUSTOM;
}
```

---

# 65. NONE

Fixture modifica el contexto actual sin aislamiento adicional.

Útil solo en escenarios controlados.

---

# 66. TRANSACTION

```text
BEGIN
  load fixture
  execute test
ROLLBACK
```

es uno de los mecanismos preferidos para integration tests cuando la plataforma/test lo permite.

---

# 67. Transaction Fixture Lifecycle

```text
Test starts
   ↓
BEGIN
   ↓
Fixture load
   ↓
Test execution
   ↓
ROLLBACK
   ↓
Test ends
```

---

# 68. Transaction ≠ universal solution

No funciona completamente cuando el test:

```text
opens independent connections
commits intentionally
uses async workers
uses multiple databases
uses multiple shards
tests transaction boundaries
```

---

# 69. Database reset

Alternativa:

```text
known database baseline
 ↓
load fixture
 ↓
test
 ↓
reset database
```

---

# 70. Schema reset

Más costoso:

```text
drop/recreate schema
migrate
load fixture
```

pero ofrece mayor aislamiento.

---

# 71. Tenant isolation

En sistemas multitenant:

```text
Test
 ↓
temporary tenant
 ↓
fixture
 ↓
test
 ↓
destroy tenant data
```

---

# 72. Database isolation

Puede utilizarse:

```text
temporary database
```

por test worker/suite.

---

# 73. Isolation choice

La selección dependerá de:

```text
test type
platform capabilities
parallelism
transaction semantics
distribution
performance
```

---

# 74. Fixture Isolation ≠ Transaction Isolation

Muy importante:

```text
FixtureIsolationPolicy
≠
SQL Transaction Isolation Level
```

---

# 75. Cleanup

Fixture deberá declarar una estrategia.

---

# 76. FixtureCleanupPolicy

```php
enum FixtureCleanupPolicy
{
    case NONE;
    case ROLLBACK;
    case DELETE_CREATED;
    case RESET_DATABASE;
    case RESET_SCHEMA;
    case DROP_TENANT;
    case DROP_DATABASE;
    case CUSTOM;
}
```

---

# 77. ROLLBACK

Preferido cuando Fixture y test están completamente contenidos en una transacción compatible.

---

# 78. DELETE_CREATED

Requiere conocer exactamente qué creó la Fixture.

---

# 79. Riesgo de DELETE_CREATED

Eliminar en orden incorrecto puede violar:

```text
foreign keys
relationship constraints
domain invariants
```

---

# 80. Cleanup Planner

Si se utiliza cleanup explícito deberá existir:

```text
FixtureCleanupPlanner
```

---

# 81. Reverse dependency order

Frecuentemente:

```text
Creation:
Customer → Order → Payment

Cleanup:
Payment → Order → Customer
```

---

# 82. Cleanup ≠ rollback

Nunca:

```text
DELETE_CREATED
=
ROLLBACK
```

---

# 83. Cleanup failure

Puede dejar:

```text
DIRTY_TEST_ENVIRONMENT
```

---

# 84. Fixture environment health

Podrá existir:

```text
FixtureEnvironmentState
```

con:

```text
CLEAN
DIRTY
UNKNOWN
RESETTING
BROKEN
```

---

# 85. UNKNOWN

Si cleanup no puede confirmarse:

```text
UNKNOWN
```

deberá impedir reutilización insegura del entorno según policy.

---

# 86. Test database reset

Para un entorno marcado:

```text
DIRTY
UNKNOWN
```

el runner podrá ejecutar un reset fuerte antes del siguiente test.

---

# 87. Fixture transaction ownership

Si FixtureRunner inicia transaction:

```text
owner = FixtureRunner
```

puede hacer rollback.

---

# 88. Existing transaction

Si el caller proporciona transaction:

```text
owner = caller
```

Fixture no deberá hacer commit/rollback arbitrario.

---

# 89. Savepoints

Para composición podrán utilizarse savepoints si la plataforma/policy lo permite.

---

# 90. Savepoint ≠ independent transaction

Se mantiene:

```text
Savepoint
≠
Transaction
```

---

# 91. Test lifecycle integration

Arquitectura:

```text
Test Runner
   ↓
Database Test Environment
   ↓
Fixture Manager
   ↓
Fixture Planner
   ↓
Fixture Executor
   ↓
Test
   ↓
Fixture Cleanup
```

---

# 92. FixtureManager

Responsabilidades:

```text
discover
resolve
plan
load
reference
cleanup
diagnose
```

No:

```text
generate SQL
manage drivers
compile queries
```

---

# 93. FixturePlanner

Pipeline:

```text
Fixture Request
      ↓
Resolve Fixture Metadata
      ↓
Dependency Expansion
      ↓
Cycle Detection
      ↓
Isolation Validation
      ↓
Database/Tenant/Shard Resolution
      ↓
Reproducibility Setup
      ↓
FixtureExecutionPlan
```

---

# 94. FixtureExecutionPlan

Conceptualmente:

```text
FixtureExecutionPlan
├── FixtureGraph
├── ExecutionOrder
├── DatabaseDomains
├── IsolationStrategy
├── CleanupStrategy
├── RandomSeed
├── Clock
├── ResourceBudget
└── ReferencePlan
```

---

# 95. Plan immutable

Una vez validado:

```text
FixtureExecutionPlan
```

será immutable.

---

# 96. Fixture states

```text
PLANNED
LOADING
LOADED
TEST_ACTIVE
CLEANING
CLEAN
FAILED
DIRTY
UNKNOWN
```

---

# 97. Execution status

Separado:

```text
PENDING
RUNNING
SUCCEEDED
FAILED
CANCELLED
PARTIAL
UNKNOWN
```

---

# 98. LOADED ≠ COMMITTED

Una Fixture puede estar cargada dentro de una transaction no committed.

---

# 99. Importante para tests

Eso es deseable:

```text
BEGIN
Fixture loaded
Test observes fixture
ROLLBACK
```

---

# 100. IdentityMap

Después de cargar entidades mediante ORM, el mismo EntityManager puede conservarlas en IdentityMap.

---

# 101. Riesgo

El test puede querer verificar una lectura real desde DB.

Entonces:

```php
$entityManager->clear();
```

puede ser necesario.

---

# 102. Fixture no deberá clear() ocultamente

Nunca:

```text
load fixture
→
EntityManager::clear()
```

sin policy explícita.

---

# 103. FixtureEntityManagerPolicy

Podrá existir:

```php
enum FixtureEntityManagerPolicy
{
    case PRESERVE;
    case CLEAR_AFTER_LOAD;
    case NEW_SCOPE;
    case CUSTOM;
}
```

---

# 104. PRESERVE

Útil cuando el test necesita referencias managed.

---

# 105. CLEAR_AFTER_LOAD

Útil para probar:

```text
repository
hydration
database reload
```

---

# 106. Reference handling after clear

Si las referencias eran objetos managed y se ejecuta `clear()`:

```text
references become detached objects
```

No deberán fingirse managed.

---

# 107. Reference descriptor

Fixture podrá exportar no solo objeto:

```text
customer
```

sino también:

```text
customer.id
customer.reference
```

para re-fetch posterior.

---

# 108. EntityReference

Conceptualmente:

```php
final readonly class FixtureEntityReference
{
    public function __construct(
        public string $entityType,
        public mixed $identifier,
        public mixed $object = null,
    ) {}
}
```

---

# 109. Object ≠ persistent identity

La Fixture deberá distinguir:

```text
Fixture object reference
```

de:

```text
persistent entity reference
```

---

# 110. ORM lifecycle events

Fixture loading mediante ORM puede disparar eventos normales:

```text
prePersist
postPersist
flush
```

según ORM.

---

# 111. Suppression

No deberán suprimirse automáticamente.

---

# 112. Event policy

Podrá existir un test-specific event policy cuando se requiera, pero debe ser explícito.

---

# 113. Domain events

Una Fixture que utiliza domain methods puede producir Domain Events.

---

# 114. Fixture setup vs business workflow

Para algunos tests se desea ejecutar el workflow real.

Para otros se desea establecer directamente el estado inicial.

Son casos distintos.

---

# 115. FixtureConstructionMode

Propuesta:

```php
enum FixtureConstructionMode
{
    case DOMAIN_REALISTIC;
    case PERSISTENCE_DIRECT;
    case HYBRID;
}
```

---

# 116. DOMAIN_REALISTIC

Usa:

```text
domain services
commands
entities
```

para alcanzar el estado.

---

# 117. PERSISTENCE_DIRECT

Establece directamente datos persistentes válidos para reducir setup.

---

# 118. HYBRID

Combina ambos.

---

# 119. No mode universal

Tests de dominio y tests de persistence tienen necesidades diferentes.

---

# 120. Fixture data declarativa

VoltStack podrá permitir:

```php
final class CurrencyFixture extends Fixture
{
    public function data(): array
    {
        return [
            ['code' => 'MXN'],
            ['code' => 'USD'],
        ];
    }
}
```

pero esta API será una convenience layer.

---

# 121. Declarative Fixture Compiler

Podrá convertir:

```text
Declarative Fixture
→
Fixture Data Model
→
Persistence Operations
```

---

# 122. YAML/JSON

Podrían soportarse mediante extensiones:

```text
fixtures/users.yaml
fixtures/orders.json
```

---

# 123. No formato obligatorio

El core no deberá depender de YAML.

---

# 124. Typed PHP preferred

Para dominio complejo, PHP tipado será preferible.

---

# 125. External fixture files

Si se usan archivos:

```text
file checksum
format version
schema
encoding
```

deberán validarse.

---

# 126. Untrusted fixture files

Nunca deberán permitir:

```text
arbitrary PHP object deserialization
dynamic class execution
```

---

# 127. Fixture version

Cada Fixture podrá tener:

```text
FixtureVersion
```

---

# 128. Fingerprint

```text
FixtureFingerprint =
FixtureId
+ FixtureVersion
+ DefinitionFingerprint
+ DependencyFingerprints
+ RelevantConfiguration
```

---

# 129. Schema generation

Podrá participar:

```text
SchemaGeneration
```

cuando el escenario depende de una estructura concreta.

---

# 130. Metadata generation

Igualmente:

```text
ORMMetadataGeneration
TypeRegistryGeneration
RelationshipMetadataGeneration
```

cuando corresponda.

---

# 131. Fixture cache

Podrá cachearse:

```text
compiled Fixture metadata
dependency graph
parsed declarative fixture definitions
```

---

# 132. No cachear objetos de escenario globalmente

Nunca:

```text
Fixture Cache
→
managed Customer object
```

entre tests.

---

# 133. Snapshot optimization

VoltStack podrá soportar optimización futura:

```text
Known Fixture State
→
Database Snapshot
→
Fast Restore
```

---

# 134. Pero

```text
Snapshot Optimization
≠
Fixture Definition
```

---

# 135. Portabilidad

Una Fixture semántica debería poder ejecutarse en:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando el escenario no dependa de una capacidad específica.

---

# 136. Platform requirement

Una Fixture podrá declarar:

```text
requires:
    JSON
    RETURNING
    advisory locks
```

según caso.

---

# 137. Capability ≠ vendor

Preferir:

```text
supportsFeature(X)
```

sobre:

```php
if ($db === 'postgresql') {
}
```

---

# 138. Fixture portability status

Podrá evaluarse como:

```text
PORTABLE
CAPABILITY_DEPENDENT
PLATFORM_SPECIFIC
UNSUPPORTED
UNKNOWN
```

---

# 139. Schema precondition

Fixture no deberá intentar cargar datos si la schema requerida no está disponible.

---

# 140. Error temprano

Preferible:

```text
Fixture schema requirement unsatisfied
```

antes de cientos de INSERT fallidos.

---

# 141. Sharding

Fixture podrá declarar:

```text
SINGLE_SHARD
SHARD_SET
ALL_SHARDS
ROUTED
```

---

# 142. Default

Preferencia:

```text
SINGLE / ROUTED TEST DOMAIN
```

---

# 143. UNKNOWN ≠ ALL

Siempre:

```text
Unknown shard target
≠
fan-out to all shards
```

---

# 144. Distributed Fixture

Una Fixture cross-shard podrá existir, pero deberá declarar que no posee una transacción ACID global salvo infraestructura específica.

---

# 145. Per-shard state

```text
FixtureExecution
├── Shard A: SUCCEEDED
├── Shard B: SUCCEEDED
└── Shard C: FAILED
```

resultado agregado:

```text
PARTIAL
```

si no existe rollback global.

---

# 146. Fan-out Fixture result

No deberá convertirse en:

```text
SUCCESS
```

si un shard no fue verificado.

---

# 147. Multitenancy

Fixture podrá crear un tenant temporal:

```text
TestTenantFixture
```

---

# 148. Ejemplo

```text
TenantFixture
├── Tenant
├── TenantDatabase
├── TenantAdmin
└── TenantSettings
```

---

# 149. Tenant isolation

El `FixtureContext` deberá propagar:

```text
TenantContext
```

a:

```text
Factories
EntityManager
Query Engine
Cache
Routing
```

---

# 150. Cross-tenant leakage

Será un error crítico.

---

# 151. Parallel testing

Supongamos:

```text
Worker 1
Worker 2
Worker 3
Worker 4
```

Cada worker deberá disponer de un dominio aislado.

---

# 152. Estrategias

```text
database per worker
schema per worker
tenant per worker
transaction per worker
namespace/prefix
```

según plataforma.

---

# 153. Worker identity

Podrá existir:

```text
TestWorkerId
```

---

# 154. Test execution domain

```text
TestExecutionDomain
=
WorkerId
+ LogicalDatabase
+ Tenant?
+ Shard?
+ Schema?
```

---

# 155. Fixture keys

Cualquier recurso temporal deberá incluir suficiente contexto para evitar colisiones.

---

# 156. Auto-increment IDs

Los tests no deberían asumir:

```text
first User ID = 1
```

salvo que el escenario lo garantice explícitamente.

---

# 157. Referencias simbólicas

Preferir:

```php
$user = $fixture->get('customer');
```

sobre:

```php
$user = User::find(1);
```

---

# 158. Razón

Reduce dependencia en:

```text
sequence state
parallel execution
platform differences
test ordering
```

---

# 159. Test ordering

Cada Fixture deberá asumir:

```text
no previous test state
```

salvo suite policy explícita.

---

# 160. Anti-pattern

```text
Test B requires Test A to execute first
```

deberá evitarse.

---

# 161. Shared suite fixtures

Puede haber escenarios costosos compartidos, pero deberán tener:

```text
immutable baseline
explicit isolation
reset strategy
```

---

# 162. Suite Fixture

Conceptualmente:

```text
SUITE
CLASS
TEST
```

scopes.

---

# 163. FixtureScope

```php
enum FixtureScope
{
    case TEST;
    case TEST_CLASS;
    case TEST_SUITE;
    case WORKER;
    case CUSTOM;
}
```

---

# 164. Default

```text
TEST
```

es el más seguro.

---

# 165. Shared scope risk

A mayor scope:

```text
performance ↑
isolation risk ↑
```

---

# 166. Mutation protection

Una Fixture compartida podrá requerir:

```text
READ_ONLY
```

para ciertos tests.

---

# 167. Read-only Fixture

El sistema podrá detectar/impedir writes cuando la infraestructura lo permita.

---

# 168. Read-only ≠ immutable PHP objects

Son conceptos distintos.

---

# 169. Cache consistency

Fixture setup puede interactuar con Entity/Result Cache.

---

# 170. Problema

Test A:

```text
cache User(10)
```

Cleanup:

```text
DB rollback/reset
```

Test B podría encontrar:

```text
stale User(10)
```

si cache no fue aislada/reset.

---

# 171. Fixture Cache Isolation

El test environment deberá incluir cache domain isolation.

---

# 172. Estrategias

```text
cache namespace per test
cache namespace per worker
generation bump
cache disabled
isolated cache backend
```

---

# 173. No FLUSHALL normal

No se deberá depender de:

```text
Redis FLUSHALL
```

como estrategia universal.

---

# 174. Cache namespace

Ejemplo:

```text
test-worker-3.execution-492
```

como persistence/cache domain lógico.

---

# 175. Cache cleanup

Debe respetar documentos:

```text
186 Cache Architecture
191 Cache Invalidation
192 Cache Consistency
```

---

# 176. IdentityMap reset

Entre tests deberá resetearse:

```text
EntityManager
IdentityMap
UnitOfWork
TransactionContext
```

según test environment policy.

---

# 177. Connection state

También:

```text
session variables
temporary tables
transaction state
read/write stickiness
tenant context
```

---

# 178. Persistent runtime

En FrankenPHP/RoadRunner/OpenSwoole:

```text
Test A state
```

no deberá sobrevivir a:

```text
Test B
```

---

# 179. Fixture runtime state

Siempre scoped:

```text
ReferenceRegistry
RandomSource
Clock override
FixtureGraphRegistry
Cleanup registry
Tenant context
Transaction references
Execution state
```

---

# 180. No static reference registry

Nunca:

```php
Fixture::$references
```

como estado mutable global.

---

# 181. Concurrency

Fixtures concurrentes deberán mantener:

```text
separate FixtureExecutionId
separate references
separate transaction context
separate random state
separate cleanup registry
```

---

# 182. Resource budgets

Fixture podrá declarar:

```text
maxEntities
maxRows
maxQueries
maxDuration
maxMemory
maxGraphNodes
```

---

# 183. Razón

Una Factory mal configurada:

```text
100 users
× 100 orders
× 100 lines
```

puede generar:

```text
1,010,100+
```

objetos/rows.

---

# 184. Budget failure

Deberá fallar antes o durante ejecución controladamente:

```text
FixtureResourceBudgetExceededException
```

---

# 185. Cancellation

Fixtures grandes podrán soportar cancelación cooperativa.

---

# 186. Cancellation ≠ clean state

Después de cancelar:

```text
environment may be DIRTY
```

hasta cleanup/reset confirmado.

---

# 187. Telemetry

Eventos:

```text
FixturePlanCreated
FixtureExecutionStarted
FixtureStarted
FixtureLoaded
FixtureReferencePublished
FixtureCleanupStarted
FixtureCleanupCompleted
FixtureFailed
FixtureEnvironmentMarkedDirty
FixtureExecutionCompleted
```

---

# 188. Métricas

```text
db.fixture.executions
db.fixture.duration
db.fixture.failures
db.fixture.references
db.fixture.graph_nodes
db.fixture.cleanup.duration
db.fixture.environment_dirty
```

---

# 189. Bounded cardinality

No usar labels:

```text
raw entity IDs
emails
tenant IDs
fixture-generated values
SQL text
```

---

# 190. Test telemetry correlation

Queries podrán llevar:

```text
FixtureExecutionId
TestExecutionId
TestWorkerId
```

como tracing context, no necesariamente metric labels.

---

# 191. Diagnostics

API conceptual:

```php
DB::fixtures()->explain(
    OrderWithExpiredPaymentFixture::class
);
```

---

# 192. Ejemplo

```text
FIXTURE PLAN

Fixture:
    tests.orders.expired-payment

Dependencies:
    customer.active
    catalog.standard
    payment.expired-card

Isolation:
    TRANSACTION

Cleanup:
    ROLLBACK

Reproducibility:
    STRICT

Database:
    test

Tenant:
    none

Shard:
    routed

References:
    customer
    order
    payment

Estimated entities:
    8

External I/O:
    none
```

---

# 193. Explain graph

```text
tests.orders.expired-payment
│
├── tests.customer.active
├── tests.catalog.standard
└── tests.payment.expired-card
```

---

# 194. Security

Fixtures son tooling poderoso.

No deberán exponerse automáticamente mediante:

```text
HTTP
RPC
public admin endpoints
```

---

# 195. Production

Por defecto:

```text
Fixture execution in production
=
FORBIDDEN
```

---

# 196. Razón

Fixture puede contener:

```text
fake credentials
fake users
test payment data
deliberately unusual states
destructive cleanup
```

---

# 197. Explicit production Fixture

Casos excepcionales deberán requerir una policy especial y autorización explícita.

Normalmente un Seeder será la herramienta correcta.

---

# 198. Secrets

Fixtures no deberán contener secretos reales.

---

# 199. PII

Los datasets de pruebas deberán preferir información sintética.

---

# 200. Production database copies

Copiar datos productivos para Fixtures deberá ser una capacidad separada con:

```text
masking
anonymization
authorization
audit
retention
```

No comportamiento base de Fixture.

---

# 201. Error hierarchy

```text
DatabaseException
└── FixtureException
    ├── FixtureDiscoveryException
    ├── FixtureRegistrationException
    ├── FixturePlanningException
    │   ├── FixtureDependencyException
    │   ├── FixtureDependencyCycleException
    │   └── FixtureIsolationException
    ├── FixtureLoadingException
    ├── FixtureReferenceException
    │   ├── FixtureReferenceNotFoundException
    │   ├── FixtureDuplicateReferenceException
    │   └── FixtureReferenceTypeException
    ├── FixtureCleanupException
    ├── FixtureResourceBudgetException
    ├── FixtureEnvironmentDirtyException
    └── FixtureUnknownOutcomeException
```

---

# 202. Directory structure

Propuesta:

```text
src/Quantum/Database/Fixture/
│
├── Fixture.php
├── FixtureId.php
├── FixtureContext.php
├── FixtureMetadata.php
├── FixtureManager.php
├── FixtureRegistry.php
├── FixtureResult.php
│
├── Discovery/
│   ├── FixtureDiscovery.php
│   └── FixtureRegistrar.php
│
├── Planning/
│   ├── FixturePlanner.php
│   ├── FixtureExecutionPlan.php
│   ├── FixtureExecutionStep.php
│   ├── FixtureDependencyGraph.php
│   └── FixtureCycleDetector.php
│
├── Reference/
│   ├── FixtureReferenceId.php
│   ├── FixtureReference.php
│   ├── FixtureEntityReference.php
│   ├── FixtureReferenceRegistry.php
│   └── FixtureReferenceNamespace.php
│
├── Execution/
│   ├── FixtureExecutor.php
│   ├── FixtureExecutionId.php
│   ├── FixtureExecutionStatus.php
│   └── FixtureExecutionState.php
│
├── Isolation/
│   ├── FixtureIsolationPolicy.php
│   ├── FixtureIsolationManager.php
│   ├── TransactionFixtureIsolation.php
│   ├── DatabaseFixtureIsolation.php
│   └── TenantFixtureIsolation.php
│
├── Cleanup/
│   ├── FixtureCleanupPolicy.php
│   ├── FixtureCleanupPlanner.php
│   ├── FixtureCleanupExecutor.php
│   └── FixtureEnvironmentState.php
│
├── Reproducibility/
│   ├── FixtureReproducibilityPolicy.php
│   ├── FixtureRandomSource.php
│   └── FixtureClock.php
│
├── Distribution/
│   ├── FixtureShardScope.php
│   ├── FixtureTenantScope.php
│   └── FixtureExecutionDomain.php
│
├── Resource/
│   └── FixtureResourceBudget.php
│
├── Diagnostics/
│   ├── FixtureInspector.php
│   └── FixtureExplainer.php
│
├── Telemetry/
│   └── FixtureTelemetry.php
│
└── Exception/
    └── ...
```

---

# 203. Application structure

```text
database/
├── factories/
├── seeders/
└── fixtures/
    ├── Customer/
    │   ├── ActiveCustomerFixture.php
    │   └── SuspendedCustomerFixture.php
    │
    ├── Orders/
    │   ├── PendingOrderFixture.php
    │   └── ExpiredPaymentFixture.php
    │
    └── Security/
        └── UserWithoutPermissionFixture.php
```

---

# 204. Testing structure alternativa

En proyectos que prefieran mantener test infrastructure fuera de `database/`:

```text
tests/
├── Fixtures/
├── Integration/
└── Unit/
```

VoltStack no deberá forzar una única ubicación mientras el Registry pueda resolverla.

---

# 205. Test matrix

| Área | Caso |
|---|---|
| Basic load | escenario correcto |
| References | retrieval |
| Typed refs | type mismatch |
| Dependencies | ordering |
| Cycles | rejection |
| Shared deps | reuse |
| Random | deterministic seed |
| Clock | deterministic time |
| Factory | ModelFactory |
| Factory | EntityFactory |
| Transaction | rollback cleanup |
| Database reset | clean state |
| EntityManager | preserve |
| EntityManager | clear |
| Cache | isolation |
| Tenant | isolation |
| Shard | routing |
| Parallel | worker isolation |
| Failure | dirty environment |
| Unknown | preserved |
| Security | production blocked |
| Telemetry | bounded |
| Runtime | no state leakage |

---

# 206. Test de reproducibilidad

```php
$a = $fixtures->load(
    CustomerFixture::class,
    seed: 100
);

$fixtures->reset();

$b = $fixtures->load(
    CustomerFixture::class,
    seed: 100
);
```

Deberán ser lógicamente equivalentes bajo `STRICT/SEEDED`.

No necesariamente:

```php
$a->get('customer') === $b->get('customer');
```

porque son objetos de ejecuciones distintas.

---

# 207. Test references

```php
$customer = $result->get(
    'customer',
    Customer::class
);
```

deberá devolver el objeto/reference correcto.

---

# 208. Test duplicate reference

```php
$result->reference('customer', $a);
$result->reference('customer', $b);
```

deberá fallar por defecto.

---

# 209. Test transaction cleanup

```text
BEGIN
load fixture
assert rows exist
ROLLBACK
assert rows absent
```

---

# 210. Test IdentityMap reset

Después del cleanup:

```text
IdentityMap
UnitOfWork
EntityManager state
```

deberán estar en el estado definido por la Test Environment Policy.

---

# 211. Test parallel workers

```text
Worker A
Fixture Customer
ID generated = 10

Worker B
Fixture Customer
ID generated = 10
```

puede ser válido si utilizan bases/schema/tenant aislados.

No debe producir colisión cross-worker.

---

# 212. Test dirty environment

Si cleanup falla:

```text
FixtureEnvironmentState = DIRTY
```

y el siguiente test no deberá reutilizar el entorno como si estuviera limpio.

---

# 213. Test UNKNOWN

Si no puede determinarse si una operación cross-domain fue aplicada:

```text
FixtureExecutionStatus = UNKNOWN
```

y deberá aplicarse reset conservador.

---

# 214. Architectural invariants

## DB-FIX-001
Fixture representará un escenario reproducible.

## DB-FIX-002
Fixture no será Factory.

## DB-FIX-003
Fixture no será Seeder.

## DB-FIX-004
Fixture no será Migration.

## DB-FIX-005
Fixture no será Database Dump.

## DB-FIX-006
Fixture no será Snapshot.

## DB-FIX-007
Fixture no será Test Case.

## DB-FIX-008
Fixture podrá utilizar Factories.

## DB-FIX-009
Fixture podrá utilizar Seeders solo mediante integración explícita.

## DB-FIX-010
Fixture podrá utilizar ORM.

## DB-FIX-011
Fixture podrá utilizar Query APIs.

## DB-FIX-012
Fixture no generará SQL directamente.

## DB-FIX-013
Fixture no accederá directamente al Driver por defecto.

## DB-FIX-014
Fixture tendrá identidad lógica estable.

## DB-FIX-015
Fixture podrá tener versión.

## DB-FIX-016
Fixture podrá tener fingerprint.

## DB-FIX-017
FixtureContext será scoped.

## DB-FIX-018
FixtureReferenceRegistry será scoped.

## DB-FIX-019
Fixture RandomSource será scoped.

## DB-FIX-020
Fixture Clock será scoped.

## DB-FIX-021
Fixture Graph Registry será scoped.

## DB-FIX-022
Cleanup Registry será scoped.

## DB-FIX-023
Fixture references no serán IdentityMap.

## DB-FIX-024
Fixture references no serán Entity Cache.

## DB-FIX-025
Fixture references no serán Result Cache.

## DB-FIX-026
Reference names serán deterministas.

## DB-FIX-027
Duplicate references serán rechazadas por defecto.

## DB-FIX-028
Typed reference mismatch será rechazado.

## DB-FIX-029
Object reference no implicará managed state.

## DB-FIX-030
Persistent reference no implicará objeto currently managed.

## DB-FIX-031
Fixture dependencies serán explícitas.

## DB-FIX-032
Fixture dependency graph será validado.

## DB-FIX-033
Dependency cycles serán rechazados.

## DB-FIX-034
Shared dependencies tendrán reuse policy explícita.

## DB-FIX-035
Fixture DAG será distinto de Seeder DAG.

## DB-FIX-036
Fixture composition preservará reference namespaces.

## DB-FIX-037
Fixture podrá exportar aliases controlados.

## DB-FIX-038
Fixture no ejecutará assertions de negocio del test.

## DB-FIX-039
Fixture podrá validar su propia integridad estructural.

## DB-FIX-040
Reproducibility policy será explícita.

## DB-FIX-041
STRICT no dependerá de wall-clock oculto.

## DB-FIX-042
SEEDED utilizará random source determinista.

## DB-FIX-043
External dependencies serán declaradas.

## DB-FIX-044
Factory random context podrá derivarse del Fixture context.

## DB-FIX-045
Parallel Fixtures no compartirán random mutable state.

## DB-FIX-046
Fixture isolation será explícita.

## DB-FIX-047
Fixture isolation no será SQL transaction isolation.

## DB-FIX-048
Transaction isolation strategy no será universal.

## DB-FIX-049
Transaction-owned Fixture podrá rollback su transaction.

## DB-FIX-050
Fixture no hará rollback de transaction ajena.

## DB-FIX-051
Fixture no hará commit de transaction ajena.

## DB-FIX-052
Savepoint no será transaction independiente.

## DB-FIX-053
Cleanup policy será explícita.

## DB-FIX-054
Cleanup no será rollback.

## DB-FIX-055
Delete cleanup respetará dependency ordering.

## DB-FIX-056
Cleanup failure marcará environment dirty/unknown.

## DB-FIX-057
DIRTY environment no será reutilizado silenciosamente.

## DB-FIX-058
UNKNOWN environment no será considerado CLEAN.

## DB-FIX-059
Database rollback no rebobinará object graph.

## DB-FIX-060
EntityManager state será gestionado explícitamente.

## DB-FIX-061
Fixture no ejecutará EntityManager::clear() ocultamente.

## DB-FIX-062
CLEAR_AFTER_LOAD será policy explícita.

## DB-FIX-063
Detached fixture references no serán presentadas como managed.

## DB-FIX-064
Fixture construction mode será explícito cuando importe.

## DB-FIX-065
Domain-realistic setup será soportado.

## DB-FIX-066
Persistence-direct setup será soportado.

## DB-FIX-067
Fixture no obligará a ejecutar workflows completos.

## DB-FIX-068
Declarative fixtures serán una convenience layer.

## DB-FIX-069
Core no dependerá obligatoriamente de YAML.

## DB-FIX-070
External fixture files serán validados.

## DB-FIX-071
Untrusted fixture files no podrán instanciar clases arbitrarias.

## DB-FIX-072
Compiled fixture metadata podrá cachearse.

## DB-FIX-073
Generated fixture objects no serán cache global.

## DB-FIX-074
Snapshot optimization será distinta de Fixture definition.

## DB-FIX-075
Fixture portability será capability-driven.

## DB-FIX-076
Version no será capability.

## DB-FIX-077
Schema requirements serán verificables.

## DB-FIX-078
Invalid schema precondition deberá fallar temprano.

## DB-FIX-079
Fixture podrá ejecutarse en MySQL cuando compatible.

## DB-FIX-080
Fixture podrá ejecutarse en MariaDB cuando compatible.

## DB-FIX-081
Fixture podrá ejecutarse en PostgreSQL cuando compatible.

## DB-FIX-082
Fixture podrá ejecutarse en SQLite cuando compatible.

## DB-FIX-083
Platform-specific fixtures serán explícitas.

## DB-FIX-084
Shard scope será explícito.

## DB-FIX-085
UNKNOWN shard no significará ALL.

## DB-FIX-086
Cross-shard Fixture no fingirá atomicidad global.

## DB-FIX-087
Partial shard success será PARTIAL.

## DB-FIX-088
Unverified shard result no será SUCCESS.

## DB-FIX-089
Multitenancy será integración opcional.

## DB-FIX-090
Tenant no será shard.

## DB-FIX-091
Tenant context será scoped.

## DB-FIX-092
Cross-tenant leakage será error crítico.

## DB-FIX-093
Parallel tests tendrán execution domains aislables.

## DB-FIX-094
TestWorkerId podrá formar parte del execution domain.

## DB-FIX-095
Fixtures no dependerán de IDs autoincrementales por defecto.

## DB-FIX-096
Symbolic references serán preferidas sobre hard-coded IDs.

## DB-FIX-097
Tests no dependerán del orden de otros tests por defecto.

## DB-FIX-098
Shared fixtures requerirán explicit scope.

## DB-FIX-099
TEST scope será el default más seguro.

## DB-FIX-100
Shared scope no implicará mutable shared state seguro.

## DB-FIX-101
Fixture cache domain estará aislado entre tests según policy.

## DB-FIX-102
Fixture cleanup no dependerá de global cache flush.

## DB-FIX-103
Cache namespace podrá ser per-test/per-worker.

## DB-FIX-104
Entity Cache no compartirá synthetic test state entre domains.

## DB-FIX-105
Result Cache no compartirá stale fixture state entre domains.

## DB-FIX-106
IdentityMap será reset según test lifecycle.

## DB-FIX-107
UnitOfWork será reset según test lifecycle.

## DB-FIX-108
TransactionContext será reset según test lifecycle.

## DB-FIX-109
Connection state será reset según test lifecycle.

## DB-FIX-110
Read/write sticky state será reset según test lifecycle.

## DB-FIX-111
Tenant context será reset según test lifecycle.

## DB-FIX-112
Persistent worker no compartirá Fixture execution state.

## DB-FIX-113
FrankenPHP será seguro para Fixture scopes.

## DB-FIX-114
RoadRunner será soportable.

## DB-FIX-115
OpenSwoole será soportable.

## DB-FIX-116
Coroutine Fixture executions estarán aisladas.

## DB-FIX-117
Resource budgets serán configurables.

## DB-FIX-118
Graph expansion tendrá límites.

## DB-FIX-119
Cancellation será cooperativa.

## DB-FIX-120
Cancellation no implicará clean environment.

## DB-FIX-121
Telemetry estará correlacionada con FixtureExecutionId.

## DB-FIX-122
Telemetry tendrá bounded cardinality.

## DB-FIX-123
Telemetry no expondrá PII por defecto.

## DB-FIX-124
Diagnostics no expondrán secrets.

## DB-FIX-125
Fixture execution estará bloqueada en producción por defecto.

## DB-FIX-126
Fixture tooling no tendrá endpoint HTTP público por defecto.

## DB-FIX-127
Production data no será convertido en Fixture sin protección explícita.

## DB-FIX-128
Production-derived data requerirá anonymization/masking.

## DB-FIX-129
Fixture errors conservarán original cause.

## DB-FIX-130
UNKNOWN permanecerá UNKNOWN.

## DB-FIX-131
PARTIAL permanecerá distinto de FAILED.

## DB-FIX-132
DIRTY permanecerá distinto de UNKNOWN.

## DB-FIX-133
LOADED no significará COMMITTED.

## DB-FIX-134
SUCCEEDED load no garantizará successful cleanup.

## DB-FIX-135
Fixture Plan será immutable después de validación.

## DB-FIX-136
Fixture planning será determinista bajo inputs equivalentes.

## DB-FIX-137
Fixture dependency validation ocurrirá antes de mutation cuando sea posible.

## DB-FIX-138
Fixture isolation validation ocurrirá antes de mutation cuando sea posible.

## DB-FIX-139
Fixture schema validation ocurrirá antes de mutation cuando sea posible.

## DB-FIX-140
Fixture distribution validation ocurrirá antes de mutation cuando sea posible.

## DB-FIX-141
Fixture no creará un segundo ORM.

## DB-FIX-142
Fixture no creará un segundo UnitOfWork.

## DB-FIX-143
Fixture no creará un segundo IdentityMap.

## DB-FIX-144
Fixture no creará un segundo Transaction Manager.

## DB-FIX-145
Fixture no creará un segundo Cache System.

## DB-FIX-146
Fixture no creará un segundo Relationship System.

## DB-FIX-147
Fixture no creará un segundo Type System.

## DB-FIX-148
Fixture no creará un segundo Query Engine.

## DB-FIX-149
Fixture deberá integrarse con el canonical Database stack.

## DB-FIX-150
Fixture podrá utilizarse desde integration tests.

## DB-FIX-151
Fixture podrá utilizarse desde repository tests.

## DB-FIX-152
Fixture podrá utilizarse desde ORM tests.

## DB-FIX-153
Fixture podrá utilizarse desde migration compatibility tests cuando sea apropiado.

## DB-FIX-154
Fixture podrá utilizarse desde acceptance tests.

## DB-FIX-155
Fixture podrá utilizarse desde regression tests.

## DB-FIX-156
Fixture no será necesaria para pure unit tests.

## DB-FIX-157
Fixture design favorecerá escenarios pequeños y semánticos.

## DB-FIX-158
Large random datasets corresponderán principalmente al Test Data Generation System.

## DB-FIX-159
Fixture deberá ser explainable.

## DB-FIX-160
Fixture deberá ser reproducible según su contrato declarado.

---

# 215. Modelo formal

Sea:

```text
F = Fixture
D = Dependencies
C = FixtureContext
R = Reproducibility Policy
I = Isolation Policy
L = Lifecycle/Cleanup Policy
```

Entonces:

```text
FixturePlan =
Plan(F, D, C, R, I, L)
```

y:

```text
FixtureResult =
Execute(FixturePlan)
```

---

# 216. Validez

```text
ValidFixturePlan
=
DependenciesAcyclic
∧ SchemaCompatible
∧ IsolationCompatible
∧ DistributionCompatible
∧ EnvironmentAllowed
∧ ResourceBudgetValid
```

---

# 217. Reproducibilidad

Para una Fixture estricta:

```text
Load(F, Context₁)
≈
Load(F, Context₂)
```

si:

```text
Relevant(Context₁)
=
Relevant(Context₂)
```

donde `≈` representa equivalencia semántica del escenario.

---

# 218. Consistencia del escenario

Una Fixture cargada correctamente deberá cumplir:

```text
ScenarioConsistent
=
RequiredReferencesPresent
∧ RelationshipGraphValid
∧ PersistenceOutcomeSufficient
∧ IsolationStateValid
```

Si alguno es:

```text
UNKNOWN
```

bajo una política estricta:

```text
ScenarioConsistent = UNKNOWN
```

no:

```text
TRUE
```

---

# 219. Pipeline completo

```text
Test Runner
    │
    ▼
Fixture Request
    │
    ▼
Fixture Registry
    │
    ▼
Fixture Metadata
    │
    ▼
Dependency Graph
    │
    ▼
Fixture Planner
    │
    ├── Reproducibility
    ├── Isolation
    ├── Cleanup
    ├── Distribution
    ├── Resource Budget
    └── Schema Requirements
    │
    ▼
FixtureExecutionPlan
    │
    ▼
Isolation Manager
    │
    ▼
Fixture Executor
    │
    ├── Factory Engine
    ├── Seeder integration
    ├── EntityManager
    ├── Query Engine
    └── Bulk APIs
    │
    ▼
Database
    │
    ▼
Reference Registry
    │
    ▼
Test Execution
    │
    ▼
Cleanup Planner
    │
    ▼
Cleanup / Rollback / Reset
    │
    ▼
Environment Verification
```

---

# 220. Arquitectura completa del bloque

```text
                    DATABASE FACTORY SYSTEM
                             193
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
           ModelFactory              EntityFactory
               194                       195
                 │                         │
                 └────────────┬────────────┘
                              │
                              ▼
                           Seeder
                             196
                              │
                              ▼
                           Fixture
                             197
                              │
                              ▼
                   Test Data Generation
                             198
```

No representa una cadena de dependencia obligatoria.

Más correctamente:

```text
                 Shared Data Construction
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
        ModelFactory            EntityFactory
             │                       │
             └──────────┬────────────┘
                        │
          ┌─────────────┴──────────────┐
          ▼                            ▼
       Seeder                       Fixture
  Dataset Population          Scenario Construction
                                       │
                                       ▼
                            Test Data Generation
                         specialized/large datasets
```

---

# 221. Regla maestra final

> **Una Fixture no es simplemente un conjunto de INSERTs: es la definición ejecutable de un estado de prueba conocido, identificable, reproducible, aislable y limpiable.**

La arquitectura deberá conservar:

```text
Factory
=
Instance Construction

Seeder
=
Dataset Orchestration

Fixture
=
Scenario Construction

Test Data Generator
=
Specialized Test Dataset Generation
```

y:

```text
Fixture Scenario
       ↓
Factories / Explicit Data
       ↓
ORM / Query / Bulk APIs
       ↓
Database
       ↓
Known References
       ↓
Test
       ↓
Cleanup / Rollback / Reset
```

sin permitir que Fixture invada responsabilidades de:

```text
Schema
Migration
Transaction
ORM
Query Engine
Cache
Driver
```

---

# 222. Resultado arquitectónico

Con `197_DATABASE_FIXTURE_SYSTEM.md`, VoltStack obtiene una capa formal para construir escenarios como:

```text
CustomerWithNoOrders
CustomerWithOverdueInvoice
OrderWithExpiredPayment
UserWithoutPermission
TenantWithStorageQuotaExceeded
ProductWithoutInventory
AccountWithConcurrentVersionConflict
```

sin convertir el test en una colección de:

```php
DB::insert(...);
DB::insert(...);
DB::insert(...);
```

difícil de mantener.

La Fixture pasa a ser una unidad semántica:

```text
Fixture
├── identity
├── dependencies
├── data construction
├── references
├── isolation
├── reproducibility
├── cleanup
├── distribution
├── diagnostics
└── telemetry
```

y queda integrada con:

```text
Factory Engine
Seeder System
ORM
UnitOfWork
Transaction System
Cache System
Distribution
Multitenancy
Testing
Persistent Runtime
```

sin duplicarlos.

---

# 223. Siguiente documento

```text
198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md
```

Este documento cerrará el **Bloque 18 — Factories, Seeders & Fixtures** y definirá la infraestructura especializada para generar datos de prueba:

```text
Test Data Generation
├── deterministic random generation
├── realistic synthetic data
├── edge-case generation
├── boundary-value generation
├── invalid-data generation
├── relationship-aware generation
├── property-based datasets
├── large datasets
├── reproducible seeds
├── distributions
├── uniqueness
├── privacy-safe synthetic data
├── dataset profiles
└── resource budgets
```

manteniendo especialmente:

```text
TestDataGenerator
≠
Factory
≠
Seeder
≠
Fixture
≠
Fuzzer
```

aunque pueda integrarse con todos ellos.

Después de `198`, el siguiente bloque será:

```text
Block 19 — Pagination, Batch & Large Data
```

comenzando con:

```text
199_DATABASE_PAGINATION_SYSTEM.md
```