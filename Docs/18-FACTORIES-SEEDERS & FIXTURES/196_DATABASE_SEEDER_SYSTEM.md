# 196_DATABASE_SEEDER_SYSTEM.md

# VoltStack Quantum Database
## Database Seeder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 196 — Database Seeder System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `195_DATABASE_ENTITY_FACTORY_SYSTEM.md`  
**Siguiente documento:** `197_DATABASE_FIXTURE_SYSTEM.md`

---

# 1. Propósito

`Database Seeder System` define la arquitectura mediante la cual VoltStack podrá **poblar una base de datos con conjuntos de datos intencionales, ordenados, reproducibles y gobernados por políticas explícitas**.

Ejemplo:

```php
final class DatabaseSeeder extends Seeder
{
    public function run(SeederContext $context): void
    {
        $this->call([
            RoleSeeder::class,
            PermissionSeeder::class,
            AdminUserSeeder::class,
        ]);
    }
}
```

Ejecución:

```bash
php voltstack database:seed
```

o:

```bash
php voltstack database:seed --class=RoleSeeder
```

La regla central será:

> **Factory define cómo construir datos; Seeder define qué datos deben poblarse, en qué orden, bajo qué contexto y mediante qué política de persistencia.**

Por tanto:

```text
Factory
=
Data Construction

Seeder
=
Dataset Orchestration
```

Nunca:

```text
Seeder
=
Migration
```

ni:

```text
Seeder
=
Fixture
```

ni:

```text
Seeder
=
Persistence Engine
```

---

# 2. Posición dentro del bloque

La arquitectura del bloque queda:

```text
193 DATABASE_FACTORY_SYSTEM
        │
        ├── 194 MODEL_FACTORY_SYSTEM
        │
        └── 195 ENTITY_FACTORY_SYSTEM
                     │
                     ▼
             196 SEEDER_SYSTEM
                     │
                     ▼
             197 FIXTURE_SYSTEM
                     │
                     ▼
       198 TEST_DATA_GENERATION_SYSTEM
```

Las responsabilidades son diferentes:

```text
Factory
    construye objetos/datos

Seeder
    orquesta datasets

Fixture
    representa escenarios reproducibles

Test Data Generator
    genera datasets especializados
    para testing
```

---

# 3. Problema arquitectónico

Una aplicación necesita frecuentemente poblar datos para:

```text
development
testing
demonstration
initial installation
reference data
local environments
CI
staging
controlled bootstrap
```

Ejemplos:

```text
roles
permissions
countries
currencies
application settings
development users
demo products
test organizations
reference catalogs
```

Sin un sistema formal, los proyectos terminan utilizando scripts arbitrarios:

```php
DB::insert(...);
DB::insert(...);
DB::insert(...);
```

o:

```php
User::create(...);
Role::create(...);
```

sin:

```text
ordering
dependency control
environment policies
transaction policy
repeatability
diagnostics
telemetry
failure semantics
```

VoltStack deberá evitarlo.

---

# 4. Seeder como orquestador

El Seeder será principalmente un:

```text
Dataset Orchestrator
```

No un generador de datos.

Conceptualmente:

```text
Seeder
   │
   ├── coordinates
   ├── orders
   ├── invokes
   ├── scopes
   ├── persists
   └── reports
```

---

# 5. Separación fundamental

```text
Seeder
≠
Factory
≠
Fixture
≠
Migration
≠
Schema Builder
≠
EntityManager
≠
UnitOfWork
≠
Query Builder
≠
SQL Compiler
```

---

# 6. Seeder vs Factory

Factory:

```php
UserEntityFactory::new()
    ->verified()
    ->make();
```

responde:

> ¿Cómo construyo un User válido?

Seeder:

```php
final class AdminSeeder extends Seeder
{
    public function run(SeederContext $context): void
    {
        // ...
    }
}
```

responde:

> ¿Qué usuarios administrativos necesita esta instalación?

---

# 7. Seeder vs Migration

Migration modifica:

```text
database structure
```

Seeder modifica:

```text
database data
```

Formalmente:

```text
Migration
=
Schema Evolution

Seeder
=
Dataset Population
```

---

# 8. Regla crítica

Nunca se deberá asumir:

```text
Migration success
⇒
Seeder success
```

Son operaciones distintas.

---

# 9. Seeder vs Fixture

Seeder normalmente describe:

```text
application/environment dataset
```

Fixture describe:

```text
specific reproducible scenario
```

Ejemplo Seeder:

```text
RoleSeeder
├── admin
├── manager
└── customer
```

Ejemplo Fixture:

```text
OrderWithExpiredPaymentFixture
```

---

# 10. Seeder vs Test Data Generator

Seeder decide:

```text
qué poblar
```

Test Data Generator decide:

```text
cómo generar grandes datasets
especializados para pruebas
```

---

# 11. API base

Propuesta:

```php
abstract class Seeder
{
    abstract public function run(
        SeederContext $context
    ): void;
}
```

---

# 12. Seeder sencillo

```php
final class RoleSeeder extends Seeder
{
    public function run(
        SeederContext $context
    ): void {
        // dataset
    }
}
```

---

# 13. Root Seeder

VoltStack utilizará normalmente:

```php
final class DatabaseSeeder extends Seeder
{
    public function run(
        SeederContext $context
    ): void {
        $this->call([
            ReferenceDataSeeder::class,
            SecuritySeeder::class,
            DevelopmentSeeder::class,
        ]);
    }
}
```

---

# 14. Seeder hierarchy

```text
DatabaseSeeder
│
├── ReferenceDataSeeder
│   ├── CountrySeeder
│   ├── CurrencySeeder
│   └── LanguageSeeder
│
├── SecuritySeeder
│   ├── RoleSeeder
│   └── PermissionSeeder
│
└── DevelopmentSeeder
    ├── UserSeeder
    ├── ProductSeeder
    └── OrderSeeder
```

---

# 15. call()

API:

```php
$this->call([
    RoleSeeder::class,
    PermissionSeeder::class,
]);
```

deberá registrar/invocar seeders mediante el `SeederRunner`.

---

# 16. call() no es llamada PHP directa

Conceptualmente:

```text
Seeder::call()
       ↓
SeederExecutionRequest
       ↓
SeederPlanner
       ↓
SeederRunner
```

Esto permite aplicar:

```text
dependency resolution
cycle detection
telemetry
environment policies
transactions
diagnostics
failure handling
```

---

# 17. Seeder discovery

Los seeders podrán descubrirse mediante:

```text
explicit root seeder
configuration
package registration
module registration
CLI class selection
```

---

# 18. No global filesystem scan obligatorio

VoltStack no deberá depender de escanear todo el proyecto en cada ejecución.

---

# 19. Compiled Seeder Registry

Podrá existir:

```text
SeederRegistry
```

conteniendo:

```text
SeederId
SeederClass
Dependencies
Tags
EnvironmentPolicy
TransactionPolicy
DatabaseTarget
Priority
Metadata
```

---

# 20. SeederId

Cada Seeder tendrá una identidad estable.

Ejemplo:

```text
app.security.roles
```

preferible a depender únicamente de:

```text
App\Database\Seeders\RoleSeeder
```

---

# 21. Razón

Los namespaces PHP pueden cambiar.

Una identidad lógica estable facilita:

```text
diagnostics
history
dependencies
packages
telemetry
execution records
```

---

# 22. Seeder Metadata

Propuesta:

```php
final readonly class SeederMetadata
{
    public function __construct(
        public SeederId $id,
        public string $class,
        public array $dependencies,
        public array $tags,
        public SeederEnvironmentPolicy $environment,
        public SeederTransactionPolicy $transaction,
        public SeederRepeatabilityPolicy $repeatability,
    ) {}
}
```

---

# 23. Dependencias

Ejemplo:

```text
PermissionSeeder
      ↓
RoleSeeder
      ↓
AdminUserSeeder
```

---

# 24. Declaración

Podrá realizarse mediante:

```php
public function dependencies(): array
{
    return [
        RoleSeeder::class,
        PermissionSeeder::class,
    ];
}
```

---

# 25. O mediante metadata

Conceptualmente:

```php
#[DependsOn(
    RoleSeeder::class,
    PermissionSeeder::class
)]
final class AdminUserSeeder extends Seeder
{
}
```

Los atributos serán una posible fuente de metadata, no la arquitectura misma.

---

# 26. Dependency Graph

El sistema construirá:

```text
Seeder Dependency Graph
```

Ejemplo:

```text
CountrySeeder ───────────────┐
                             ▼
CurrencySeeder          CompanySeeder
                             │
RoleSeeder ───────┐           │
                  ▼           ▼
PermissionSeeder ──────► AdminUserSeeder
```

---

# 27. DAG

Idealmente:

```text
SeederGraph = Directed Acyclic Graph
```

---

# 28. Cycle detection

Caso:

```text
SeederA
 ↓
SeederB
 ↓
SeederC
 ↓
SeederA
```

deberá fallar antes de modificar la DB cuando sea detectable durante planificación.

---

# 29. Error

```text
Seeder dependency cycle detected.

app.seed.a
  → app.seed.b
  → app.seed.c
  → app.seed.a
```

---

# 30. Seeder Planner

Responsabilidad:

```text
Selected Seeders
      ↓
Dependency Expansion
      ↓
Policy Validation
      ↓
Database Domain Resolution
      ↓
Topological Ordering
      ↓
Execution Groups
      ↓
SeederExecutionPlan
```

---

# 31. Plan immutable

`SeederExecutionPlan` deberá ser immutable una vez validado.

---

# 32. Ejemplo

```text
SeederExecutionPlan

1. CountrySeeder
2. CurrencySeeder
3. RoleSeeder
4. PermissionSeeder
5. AdminUserSeeder
```

---

# 33. Determinismo

Dadas las mismas:

```text
Seeder definitions
dependencies
configuration
environment
```

el orden deberá ser determinista.

---

# 34. Equal priority

Si dos seeders no tienen dependencia entre sí, el sistema deberá aplicar una regla determinista.

Por ejemplo:

```text
priority
then SeederId
```

---

# 35. Priority ≠ dependency

Una prioridad:

```text
priority = 100
```

no reemplaza:

```text
DependsOn(RoleSeeder)
```

---

# 36. Dependencias son semánticas

Si B necesita A:

```text
B depends on A
```

debe declararse como dependencia.

No confiar únicamente en números.

---

# 37. SeederContext

Toda ejecución recibirá:

```text
SeederContext
```

---

# 38. Contenido conceptual

```text
SeederContext
├── ExecutionId
├── Environment
├── DatabaseContext
├── RandomSource
├── Clock
├── EntityManager resolver
├── Factory runtime
├── Transaction context
├── Tenant context optional
├── Shard context optional
├── Parameters
├── Cancellation token
└── Telemetry context
```

---

# 39. SeederContext scoped

Nunca:

```text
static $currentSeederContext
```

---

# 40. Environment

VoltStack distinguirá:

```text
LOCAL
DEVELOPMENT
TESTING
CI
STAGING
PRODUCTION
CUSTOM
```

sin obligar a que todos los proyectos usen exactamente esos nombres.

---

# 41. Environment Policy

Cada Seeder podrá declarar:

```text
ANY
NON_PRODUCTION
DEVELOPMENT_ONLY
TEST_ONLY
PRODUCTION_ALLOWED
EXPLICIT
CUSTOM
```

---

# 42. Ejemplo

`DemoUserSeeder`:

```text
NON_PRODUCTION
```

---

# 43. Production safety

Intentar ejecutar:

```bash
php voltstack database:seed --class=DemoUserSeeder
```

en producción deberá ser rechazado por defecto cuando la policy lo prohíba.

---

# 44. `--force`

Un flag CLI no deberá automáticamente anular cualquier política de seguridad.

---

# 45. Capability de override

Podrá existir una política explícita:

```text
OVERRIDE_ALLOWED
OVERRIDE_REQUIRES_CONFIRMATION
OVERRIDE_FORBIDDEN
```

---

# 46. Reference Data

Datos como:

```text
currencies
country codes
system roles
permission definitions
```

pueden ser válidos en producción.

---

# 47. Demo Data

Datos como:

```text
fake customers
fake orders
fake credit cards
demo companies
```

normalmente no.

---

# 48. Seeder classification

Propuesta:

```php
enum SeederCategory
{
    case REFERENCE;
    case BOOTSTRAP;
    case DEVELOPMENT;
    case DEMO;
    case TEST;
    case PACKAGE;
    case CUSTOM;
}
```

---

# 49. Category ≠ environment policy

Un Seeder `REFERENCE` puede tener diferentes políticas según proyecto.

---

# 50. Factories dentro de Seeders

Ejemplo:

```php
final class UserSeeder extends Seeder
{
    public function run(
        SeederContext $context
    ): void {
        User::factory()
            ->count(100)
            ->create();
    }
}
```

---

# 51. EntityFactory

También:

```php
final class UserSeeder extends Seeder
{
    public function run(
        SeederContext $context
    ): void {
        $entityManager = $context->entityManager();

        $users = UserEntityFactory::new()
            ->count(100)
            ->persist($entityManager);

        $entityManager->flush();
    }
}
```

---

# 52. Seeder no exige Factory

También podrá usar objetos explícitos:

```php
$role = new Role(
    RoleId::from('admin'),
    RoleName::from('Administrator'),
);
```

---

# 53. Query API

Podrá utilizar APIs Database cuando sea apropiado:

```php
DB::table('countries')->insert([...]);
```

pero esta decisión pertenece al Seeder.

---

# 54. Regla de capas

Seeder puede consumir APIs públicas de Database.

No deberá acceder directamente a:

```text
PDO
Driver internals
Compiler internals
Connection pool internals
```

---

# 55. Seeder persistence styles

VoltStack permitirá:

```text
Model API
EntityManager
Query Builder
Bulk Insert
custom application service
```

según el dataset.

---

# 56. Dataset pequeño

Para:

```text
10 roles
50 permissions
```

Entity/Model APIs son razonables.

---

# 57. Dataset masivo

Para:

```text
10,000,000 synthetic telemetry rows
```

crear diez millones de entidades puede ser ineficiente.

El futuro:

```text
DATABASE_BULK_INSERT_SYSTEM
```

será preferible.

---

# 58. Seeder ≠ ORM-only

Seeder es un orquestador de datos.

No debe obligar a usar ORM.

---

# 59. Idempotencia

Uno de los problemas principales es:

```text
¿qué ocurre si ejecuto el Seeder dos veces?
```

VoltStack deberá representar esta decisión explícitamente.

---

# 60. Repeatability Policy

Propuesta:

```php
enum SeederRepeatabilityPolicy
{
    case REPEATABLE;
    case IDEMPOTENT;
    case ONCE;
    case REPLACE;
    case CUSTOM;
}
```

---

# 61. REPEATABLE

Puede ejecutarse varias veces y generar datos adicionales.

Ejemplo:

```text
DevelopmentRandomUserSeeder
```

---

# 62. IDEMPOTENT

Múltiples ejecuciones convergen al mismo estado lógico.

Ejemplo:

```text
RoleSeeder
```

---

# 63. ONCE

Debe ejecutarse una vez por contexto definido.

---

# 64. REPLACE

Puede reemplazar/reconstruir su dataset administrado.

Debe utilizarse con cautela.

---

# 65. Seeder ONCE ≠ Migration

Aunque un Seeder sea `ONCE`, no se convierte en Migration.

---

# 66. Seeder Repository

Para `ONCE` podrá existir:

```text
SeederExecutionRepository
```

---

# 67. Registro

Ejemplo:

```text
SeederId
Version
Checksum
ExecutionId
StartedAt
CompletedAt
Status
Environment
DatabaseDomain
```

---

# 68. No obligatorio para todos

Seeders repetibles de desarrollo no necesitan necesariamente registro permanente.

---

# 69. Seeder version

Un Seeder podrá declarar:

```text
version = 2
```

---

# 70. Version ≠ Migration version

Sirve para identificar la definición/dataset del Seeder.

---

# 71. Checksum

Podrá existir un fingerprint:

```text
SeederFingerprint =
SeederId
+ Version
+ DefinitionFingerprint
+ RelevantConfiguration
```

---

# 72. Code checksum limitations

No deberá fingirse que un hash de archivo PHP captura toda la semántica si el Seeder depende de:

```text
external files
configuration
packages
runtime parameters
```

---

# 73. Dataset fingerprint

Podrá incorporar dependencias explícitas.

---

# 74. Upsert

Para reference data, un patrón común será:

```text
UPSERT
```

pero no deberá ser comportamiento universal.

---

# 75. Ejemplo

```php
DB::table('currencies')->upsert(
    values: [
        ['code' => 'MXN', 'name' => 'Mexican Peso'],
        ['code' => 'USD', 'name' => 'US Dollar'],
    ],
    uniqueBy: ['code'],
);
```

---

# 76. Upsert ≠ idempotencia automática

Un Seeder puede tener efectos adicionales.

Por tanto:

```text
uses upsert
≠
entire Seeder is idempotent
```

---

# 77. Transactions

Seeder deberá declarar política transaccional.

---

# 78. SeederTransactionPolicy

Propuesta:

```php
enum SeederTransactionPolicy
{
    case NONE;
    case PER_SEEDER;
    case WHOLE_PLAN;
    case USE_EXISTING;
    case REQUIRE_EXISTING;
    case CUSTOM;
}
```

---

# 79. PER_SEEDER

```text
BEGIN
 Seeder A
COMMIT

BEGIN
 Seeder B
COMMIT
```

---

# 80. WHOLE_PLAN

```text
BEGIN

Seeder A
Seeder B
Seeder C

COMMIT
```

solo cuando todas las operaciones pertenezcan al mismo dominio transaccional y las capacidades lo permitan.

---

# 81. Cross-shard

Si Seeders afectan múltiples shards:

```text
WHOLE_PLAN
```

no deberá fingir una transacción distribuida.

---

# 82. Multidatabase

Igualmente:

```text
DB A + DB B
```

no implica atomicidad global.

---

# 83. Transaction compatibility

El `SeederPlanner` deberá validar:

```text
Seeder transaction requirements
×
Database execution domains
×
Platform capabilities
```

---

# 84. External side effects

Un Seeder puede ejecutar:

```text
filesystem
object storage
API
queue
```

solo si su diseño lo requiere explícitamente.

Estas operaciones no se vuelven ACID por estar dentro de:

```text
DB::transaction()
```

---

# 85. Side Effect Policy

Podrá declararse:

```text
DATABASE_ONLY
EXTERNAL_READ
EXTERNAL_WRITE
CUSTOM
```

---

# 86. Recomendación

Seeders de base de datos deberían preferir:

```text
DATABASE_ONLY
```

cuando sea posible.

---

# 87. Failure semantics

Estados:

```text
PENDING
RUNNING
SUCCEEDED
FAILED
CANCELLED
PARTIAL
UNKNOWN
SKIPPED
```

---

# 88. UNKNOWN

Debe existir para escenarios donde el sistema no puede determinar con seguridad el resultado.

---

# 89. Ejemplo

```text
Seeder
 ↓
Transaction COMMIT sent
 ↓
Connection lost
 ↓
UNKNOWN
```

Nunca:

```text
assume success
```

---

# 90. PARTIAL

Puede ocurrir cuando:

```text
no encompassing transaction
+
some operations succeeded
+
later operation failed
```

---

# 91. Failure ≠ automatic cleanup

Seeder no deberá intentar borrar arbitrariamente datos previos para "revertir".

---

# 92. Seeder rollback

Solo podrá afirmarse rollback cuando:

```text
Transaction Manager
```

lo haya confirmado.

---

# 93. Object graph

Como en ORM:

```text
DatabaseRollback
≠
ObjectGraphRewind
```

---

# 94. Compensation

Para seeders complejos podrá existir una estrategia explícita de:

```text
compensation
```

pero:

```text
Compensation
≠
Rollback
```

---

# 95. Retry

Retry de un Seeder completo solo será seguro cuando su política lo permita.

---

# 96. RetryPolicy

```php
enum SeederRetryPolicy
{
    case NEVER;
    case TRANSACTION_SAFE;
    case IDEMPOTENT_ONLY;
    case CUSTOM;
}
```

---

# 97. Deadlock

Si una transacción completa del Seeder falla por deadlock, podrá reutilizarse el `Transaction Retry System`.

---

# 98. No statement replay arbitrario

Nunca:

```text
retry only failed INSERT
```

dentro de una transacción ya abortada.

---

# 99. Seeder execution scope

Cada ejecución tendrá:

```text
SeederExecutionId
```

---

# 100. Child executions

Cada Seeder individual tendrá:

```text
SeederRunId
```

---

# 101. Modelo

```text
SeederExecutionId
│
├── SeederRun A
├── SeederRun B
└── SeederRun C
```

---

# 102. Cancellation

CLI o runtime podrán solicitar cancelación.

---

# 103. Cancellation token

```text
SeederContext
└── CancellationToken
```

---

# 104. Safe points

Los Seeders de larga duración deberán poder verificar:

```php
$context->cancellation()->throwIfRequested();
```

---

# 105. Cancellation ≠ rollback

Si no existe transaction que abarque las operaciones anteriores:

```text
cancelled
≠
nothing persisted
```

---

# 106. Progress

Seeders grandes podrán reportar:

```text
current item
processed count
total estimate
phase
```

---

# 107. CLI progress

Ejemplo:

```text
ProductSeeder

[████████████████░░░░] 82%
820,000 / 1,000,000
```

---

# 108. Progress bounded

No deberá generar un evento de telemetry costoso por cada fila salvo modo explícito.

---

# 109. Chunking

Seeder podrá procesar:

```text
chunks
```

---

# 110. Chunk ≠ transaction automáticamente

`chunkSize=1000` no significa:

```text
transaction size = 1000
```

salvo policy explícita.

---

# 111. Resource Budget

Propuesta:

```text
SeederResourceBudget
├── maxRows
├── maxDuration
├── maxMemory
├── maxQueries
├── maxBatchSize
└── maxExternalOperations
```

---

# 112. Production budget

Los budgets podrán ser más estrictos en producción.

---

# 113. Dry Run

VoltStack deberá considerar:

```bash
php voltstack database:seed --dry-run
```

---

# 114. Dry Run semantics

Debe mostrar:

```text
selected seeders
dependency order
environment decisions
database targets
transaction policy
estimated operations where possible
warnings
```

---

# 115. Dry Run ≠ guaranteed simulation

Si el código del Seeder es arbitrario, no siempre puede conocerse todo sin ejecutarlo.

---

# 116. Plan mode

Por ello distinguiremos:

```text
PLAN
```

de:

```text
EXECUTION
```

---

# 117. `--plan`

Puede ser semánticamente más correcto:

```bash
php voltstack database:seed --plan
```

---

# 118. Explain

```bash
php voltstack database:seed --explain
```

podrá mostrar:

```text
why selected
dependencies
policies
ordering
database domain
risks
```

---

# 119. Example explain

```text
Seeder: app.security.admin-user

Selected because:
    dependency of DatabaseSeeder

Depends on:
    app.security.roles
    app.security.permissions

Environment:
    production allowed

Repeatability:
    idempotent

Transaction:
    per-seeder

Database:
    primary

Risk:
    low
```

---

# 120. Seeder parameters

CLI:

```bash
php voltstack database:seed \
    --class=DemoTenantSeeder \
    --param=users:100
```

---

# 121. Typed parameters

Internamente no deberá depender de strings arbitrarios.

---

# 122. Parameter schema

```php
public function parameters(): SeederParameterSchema
{
    return SeederParameterSchema::make()
        ->integer('users')
        ->min(1)
        ->max(10_000)
        ->default(100);
}
```

---

# 123. Untrusted input

CLI parameters deberán validarse.

---

# 124. No SQL injection

Seeder parameters no deberán interpolarse en SQL.

Se mantienen las reglas generales del Query/Binding System.

---

# 125. Secrets

No deberán imprimirse:

```text
passwords
API keys
database credentials
tokens
PII
```

en diagnostics.

---

# 126. Password seeding

Para usuarios bootstrap podrá usarse:

```text
secure generated credential
explicit environment secret
one-time activation flow
```

según aplicación.

---

# 127. No default production password

Nunca:

```text
admin@example.com
password123
```

como credenciales predeterminadas de producción.

---

# 128. Seeder discovery security

Una clase arbitraria suministrada desde CLI no deberá poder ejecutarse solo porque exista.

---

# 129. Registry validation

`--class` deberá resolverse contra:

```text
registered/allowed Seeder types
```

o una política equivalente.

---

# 130. Package Seeders

Los paquetes VoltStack podrán registrar Seeders.

Ejemplo:

```text
VoltStack/SaaS
└── PlanReferenceSeeder
```

---

# 131. Package isolation

Un paquete no deberá ejecutar su Seeder automáticamente durante boot.

---

# 132. Explicit execution

Seeders deben ejecutarse mediante:

```text
installation workflow
explicit command
deployment process
application request
```

según política.

---

# 133. Seeder boot prohibition

Nunca:

```text
Framework boot
  ↓
Seeder executes
  ↓
DB mutation
```

como comportamiento predeterminado.

---

# 134. Deployment

Un deployment podría ejecutar:

```text
database:migrate
database:seed --tag=reference
```

como pasos separados.

---

# 135. Migration first

Normalmente:

```text
Migration
  ↓
Seeder
```

pero la arquitectura no los fusionará.

---

# 136. Schema compatibility

Antes de ejecutar un Seeder, podrá verificarse:

```text
required schema generation
required table/column capability
```

---

# 137. SeederRequirements

```php
final readonly class SeederRequirements
{
    public function __construct(
        public ?SchemaGeneration $schema,
        public array $capabilities,
        public array $packages,
    ) {}
}
```

---

# 138. Capability check

Ejemplo:

```text
requires JSON column capability
```

---

# 139. No vendor conditionals

Evitar:

```php
if ($database === 'mysql') {
}
```

cuando pueda expresarse como:

```text
Platform Capability
```

---

# 140. Multiple platforms

Seeders deberán funcionar cuando sea razonable en:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 141. Platform-specific Seeder

Cuando realmente sea necesario, deberá declararse explícitamente.

---

# 142. Database target

Un Seeder puede dirigirse a:

```text
logical database
```

no necesariamente a un connection name físico.

---

# 143. Ejemplo

```text
database = analytics
```

---

# 144. Connection resolution

El `ConnectionManager` resolverá posteriormente el endpoint.

---

# 145. Seeder ≠ ConnectionManager

Seeder no deberá administrar:

```text
pool
socket
PDO
reconnect
```

---

# 146. Read/write routing

Los Seeders que mutan datos deberán utilizar writer semantics.

---

# 147. Replica

No deberán enviarse escrituras a replicas.

---

# 148. Read-after-write

Las lecturas internas posteriores deberán respetar:

```text
transaction affinity
sticky semantics
consistency requirements
```

---

# 149. Sharding

Un Seeder podrá ser:

```text
SINGLE_SHARD
ALL_SHARDS
SHARD_SET
ROUTED_BY_DATA
CUSTOM
```

---

# 150. Shard scope

Propuesta:

```php
enum SeederShardScope
{
    case ROUTED;
    case SINGLE;
    case ALL;
    case EXPLICIT_SET;
}
```

---

# 151. ALL shards

Debe ser explícito.

---

# 152. No accidental fan-out

Un Seeder no deberá convertirse en global solo porque no pudo resolver shard.

Siempre:

```text
UNKNOWN
≠
ALL
```

---

# 153. Per-shard execution

```text
SeederPlan
   │
   ├── Shard A → Seeder
   ├── Shard B → Seeder
   └── Shard C → Seeder
```

---

# 154. Cross-shard transaction

No se fingirá atomicidad global.

---

# 155. Resharding

Seeder execution deberá utilizar:

```text
current validated ShardMap generation
```

cuando aplique.

---

# 156. Tenant seeding

Multitenancy será integración opcional.

---

# 157. Tenant scopes

Podrán existir:

```text
GLOBAL
SINGLE_TENANT
TENANT_SET
ALL_TENANTS
ROUTED
```

---

# 158. Ejemplo

```bash
php voltstack database:seed \
    --class=TenantDefaultsSeeder \
    --tenant=acme
```

---

# 159. ALL tenants

Debe ser explícito y gobernado por budgets.

---

# 160. Tenant ≠ shard

Incluso si:

```text
one tenant per shard
```

en una instalación concreta:

```text
Tenant
≠
Shard
```

arquitectónicamente.

---

# 161. Seeder Cache interaction

Seeders mutan DB.

Por ello deberán participar en:

```text
Cache Invalidation System
```

a través de las rutas normales de persistence/query mutation.

---

# 162. No direct cache mutation

Seeder no deberá mantener manualmente:

```text
Entity Cache
Result Cache
```

salvo API especializada explícita.

---

# 163. Commit ordering

Siempre:

```text
DB mutation
 ↓
confirmed commit
 ↓
cache invalidation/publication
```

cuando exista transaction.

---

# 164. UNKNOWN outcome

Debe aplicar:

```text
conservative cache invalidation
```

según documentos 191–192.

---

# 165. Events

Eventos conceptuales:

```text
SeederPlanCreated
SeederExecutionStarted
SeederStarted
SeederProgressed
SeederSucceeded
SeederFailed
SeederSkipped
SeederCancelled
SeederExecutionCompleted
```

---

# 166. Events ≠ transaction outcome

Un evento:

```text
SeederSucceeded
```

solo deberá emitirse cuando el nivel correspondiente realmente haya terminado con éxito según su contrato.

---

# 167. Telemetry

Métricas:

```text
db.seeder.executions
db.seeder.duration
db.seeder.seeders
db.seeder.failures
db.seeder.skipped
db.seeder.rows_estimated
db.seeder.rows_processed
```

---

# 168. Tracing

```text
DatabaseSeedExecution
│
├── Seeder:Country
├── Seeder:Currency
├── Seeder:Role
└── Seeder:AdminUser
```

---

# 169. Query tracing

Las queries generadas podrán correlacionarse con:

```text
SeederExecutionId
SeederRunId
```

---

# 170. Bounded cardinality

No usar como metric labels:

```text
raw tenant ID
entity ID
email
random data
SQL text
```

---

# 171. Logging

Ejemplo:

```text
[database.seed] Seeder completed
seeder=app.reference.currencies
duration_ms=31
status=success
```

---

# 172. Error reporting

Ejemplo:

```text
Seeder execution failed.

Seeder:
    app.security.admin-user

Dependency path:
    database
    → security
    → admin-user

Status:
    FAILED

Transaction:
    rolled_back

Cause:
    UniqueConstraintViolationException
```

---

# 173. UNKNOWN report

```text
Seeder outcome is uncertain.

Seeder:
    app.reference.products

Reason:
    connection lost while commit acknowledgement
    was pending

Database outcome:
    UNKNOWN

Automatic retry:
    DISABLED
```

---

# 174. Seeder Result

Propuesta:

```php
final readonly class SeederResult
{
    public function __construct(
        public SeederExecutionId $executionId,
        public SeederExecutionStatus $status,
        public array $runs,
        public SeederStatistics $statistics,
        public array $warnings,
    ) {}
}
```

---

# 175. SeederRunResult

```text
SeederId
Status
StartedAt
FinishedAt
Duration
RowsAffectedEstimate
TransactionOutcome
Warnings
Failure
```

---

# 176. Rows affected

No siempre será posible obtener un conteo exacto.

Por tanto:

```text
exact
estimated
unknown
```

deberá distinguirse.

---

# 177. Seeder Repository

Opcionalmente:

```text
database_seed_history
```

podrá almacenar ejecuciones persistentes.

---

# 178. Bootstrapping problem

El Seeder Repository no deberá ser obligatorio si la tabla todavía no existe.

---

# 179. Repository strategies

```text
NULL
IN_MEMORY
DATABASE
FILE
CUSTOM
```

según entorno.

---

# 180. History ≠ source of truth

La DB sigue siendo autoridad sobre los datos.

Seeder history describe ejecuciones, no garantiza que los datos sigan presentes.

---

# 181. Example

Un administrador puede borrar manualmente un Role.

El historial:

```text
RoleSeeder = succeeded
```

no prueba:

```text
Role currently exists
```

---

# 182. Idempotent reference Seeder

Debe verificar/reconciliar estado mediante su estrategia, no únicamente confiar en history.

---

# 183. Concurrency

Dos procesos podrían ejecutar el mismo Seeder simultáneamente.

---

# 184. Seeder execution lock

Podrá existir:

```text
SeederExecutionLock
```

para evitar concurrencia incompatible.

---

# 185. Lock ≠ DB data lock

Siempre:

```text
Seeder Execution Lock
≠
Pessimistic Database Lock
```

---

# 186. Distributed lock

Si existe provider compartido podrá utilizarse.

Pero debe tener:

```text
owner token
lease
expiration
renewal
fencing where needed
```

---

# 187. DB advisory locks

Podrán utilizarse cuando Platform Capability lo soporte.

---

# 188. Lock failure

No deberá implicar automáticamente:

```text
Seeder failed
```

puede resultar:

```text
SKIPPED
ALREADY_RUNNING
WAITING
```

según policy.

---

# 189. Concurrent idempotent seeders

Aunque sean idempotentes, DB constraints siguen siendo necesarias para proteger invariantes.

---

# 190. Persistent runtimes

Los Seeders pueden ejecutarse dentro de procesos persistentes.

Nunca:

```text
static currentSeeder
static currentTenant
static currentTransaction
static currentRandomSource
```

---

# 191. Shared state

Solo podrán compartirse:

```text
immutable Seeder metadata
compiled dependency graph
immutable definitions
```

---

# 192. Scoped state

Siempre:

```text
SeederContext
ExecutionId
TransactionContext
RandomSource
FactoryContext
Progress
Cancellation
EntityManager
```

---

# 193. FrankenPHP

Aunque los Seeders normalmente se ejecutarán desde CLI, la arquitectura deberá ser segura si son invocados desde un proceso FrankenPHP controlado.

---

# 194. RoadRunner/OpenSwoole

Misma regla:

```text
worker reuse
≠
execution state reuse
```

---

# 195. No HTTP Seeder endpoint por defecto

VoltStack no deberá registrar automáticamente:

```text
POST /seed
```

---

# 196. Administration integration

Si en el futuro existe un panel administrativo, deberá aplicar:

```text
authentication
authorization
environment policy
audit
confirmation
rate/resource limits
```

---

# 197. Audit

Seeders sensibles podrán generar audit records:

```text
who
what
environment
database domain
when
result
```

sin almacenar secretos.

---

# 198. CLI commands

Propuesta:

```bash
php voltstack database:seed
php voltstack database:seed --class=RoleSeeder
php voltstack database:seed --tag=reference
php voltstack database:seed --environment=development
php voltstack database:seed --plan
php voltstack database:seed --explain
php voltstack database:seed --status
```

---

# 199. `--reset`

No deberá existir como operación destructiva ambigua.

---

# 200. Operaciones destructivas

Deberán utilizar nombres explícitos como:

```text
database:seed:replace
database:seed:clear-managed-data
```

si alguna vez se soportan.

---

# 201. Seeder tags

Ejemplo:

```text
reference
security
demo
development
tenant
package
```

---

# 202. Selección por tag

```bash
php voltstack database:seed --tag=reference
```

---

# 203. Dependencies remain

Si `CurrencySeeder` seleccionado depende de `CountrySeeder`, el planner podrá incluir la dependencia aunque no tenga el tag, según policy.

---

# 204. Tag ≠ dependency boundary

Nunca deberá romperse un DAG válido solo para respetar selección superficial por tag.

---

# 205. Directory structure

Propuesta:

```text
src/Quantum/Database/Seeder/
│
├── Seeder.php
├── SeederId.php
├── SeederContext.php
├── SeederMetadata.php
├── SeederRegistry.php
├── SeederRunner.php
├── SeederResult.php
├── SeederRunResult.php
│
├── Discovery/
│   ├── SeederDiscovery.php
│   └── SeederRegistrar.php
│
├── Planning/
│   ├── SeederPlanner.php
│   ├── SeederExecutionPlan.php
│   ├── SeederExecutionStep.php
│   ├── SeederDependencyGraph.php
│   └── SeederCycleDetector.php
│
├── Policy/
│   ├── SeederEnvironmentPolicy.php
│   ├── SeederRepeatabilityPolicy.php
│   ├── SeederTransactionPolicy.php
│   ├── SeederRetryPolicy.php
│   ├── SeederSideEffectPolicy.php
│   └── SeederExecutionPolicy.php
│
├── Execution/
│   ├── SeederExecutor.php
│   ├── SeederExecutionId.php
│   ├── SeederRunId.php
│   ├── SeederExecutionStatus.php
│   ├── SeederExecutionLock.php
│   └── SeederCancellation.php
│
├── Parameter/
│   ├── SeederParameterSchema.php
│   ├── SeederParameters.php
│   └── SeederParameterValidator.php
│
├── Persistence/
│   ├── SeederExecutionRepository.php
│   ├── NullSeederExecutionRepository.php
│   └── DatabaseSeederExecutionRepository.php
│
├── Distribution/
│   ├── SeederShardScope.php
│   ├── SeederTenantScope.php
│   └── SeederDatabaseTarget.php
│
├── Resource/
│   ├── SeederResourceBudget.php
│   └── SeederProgress.php
│
├── Diagnostics/
│   ├── SeederInspector.php
│   └── SeederExplainer.php
│
├── Telemetry/
│   └── SeederTelemetry.php
│
└── Exception/
    ├── SeederException.php
    ├── SeederPlanningException.php
    ├── SeederDependencyException.php
    ├── SeederExecutionException.php
    └── SeederPolicyException.php
```

---

# 206. Application structure

Aplicaciones VoltStack podrán utilizar:

```text
database/
├── factories/
│   ├── UserFactory.php
│   └── OrderFactory.php
│
├── seeders/
│   ├── DatabaseSeeder.php
│   ├── ReferenceDataSeeder.php
│   ├── RoleSeeder.php
│   └── DevelopmentSeeder.php
│
└── fixtures/
```

La estructura definitiva se consolidará en:

```text
330_DATABASE_DIRECTORY_STRUCTURE.md
```

---

# 207. Error hierarchy

```text
DatabaseException
└── SeederException
    ├── SeederDiscoveryException
    ├── SeederRegistrationException
    ├── SeederPlanningException
    │   ├── SeederDependencyException
    │   ├── SeederDependencyCycleException
    │   └── SeederPolicyConflictException
    ├── SeederEnvironmentException
    ├── SeederParameterException
    ├── SeederLockException
    ├── SeederExecutionException
    ├── SeederCancellationException
    └── SeederUnknownOutcomeException
```

---

# 208. Test matrix

| Área | Prueba |
|---|---|
| Discovery | registro correcto |
| Dependencies | orden topológico |
| Cycles | rechazo |
| Priority | determinismo |
| Environment | bloqueo producción |
| Factory integration | ModelFactory |
| Factory integration | EntityFactory |
| ORM | UnitOfWork |
| Query Builder | direct dataset |
| Idempotence | múltiples runs |
| ONCE | history |
| Transaction | per Seeder |
| Transaction | whole plan |
| Rollback | confirmed |
| Unknown | preserved |
| Retry | safe boundary |
| Cancellation | partial state |
| Sharding | explicit scope |
| Tenant | isolation |
| Cache | commit ordering |
| Concurrency | execution lock |
| Worker | state reset |
| Telemetry | bounded labels |
| Security | secrets redacted |
| CLI | plan/explain |

---

# 209. Test dependency ordering

Dado:

```text
AdminUserSeeder
depends:
    RoleSeeder
    PermissionSeeder

PermissionSeeder
depends:
    RoleSeeder
```

resultado:

```text
RoleSeeder
PermissionSeeder
AdminUserSeeder
```

---

# 210. Test cycle

```text
A → B → C → A
```

debe fallar durante planning.

Idealmente:

```text
queries executed = 0
```

---

# 211. Test production safety

```text
environment = production
Seeder = DemoUserSeeder
policy = NON_PRODUCTION
```

resultado:

```text
REJECTED
```

antes de mutation.

---

# 212. Test transaction ownership

Si existe transaction externa:

```text
SeederRunner
```

no deberá hacer commit de ella salvo contrato explícito.

---

# 213. Test unknown commit

```text
COMMIT sent
connection lost
```

resultado:

```text
SeederStatus = UNKNOWN
automatic retry = false
```

por defecto.

---

# 214. Test tenant isolation

```text
Tenant A Seeder
```

no deberá insertar datos bajo:

```text
Tenant B
```

por leakage de worker state.

---

# 215. Test persistent worker

```text
Execution A
tenant=A
seed=100

Execution B
tenant=B
seed=200
```

debe conservar aislamiento completo.

---

# 216. Architectural invariants

## DB-SEED-001
Seeder será un orquestador de datasets.

## DB-SEED-002
Seeder no será Factory.

## DB-SEED-003
Seeder no será Fixture.

## DB-SEED-004
Seeder no será Migration.

## DB-SEED-005
Seeder no será Schema Builder.

## DB-SEED-006
Seeder no será EntityManager.

## DB-SEED-007
Seeder no será UnitOfWork.

## DB-SEED-008
Seeder no será Persistence Engine.

## DB-SEED-009
Seeder no será SQL Compiler.

## DB-SEED-010
Seeder no será Driver.

## DB-SEED-011
Seeder podrá consumir Factories.

## DB-SEED-012
Seeder no requerirá Factories.

## DB-SEED-013
Seeder podrá consumir Model API.

## DB-SEED-014
Seeder podrá consumir EntityManager.

## DB-SEED-015
Seeder podrá consumir Query APIs.

## DB-SEED-016
Seeder podrá utilizar Bulk APIs.

## DB-SEED-017
Seeder no accederá directamente a PDO por defecto.

## DB-SEED-018
Seeder no administrará Connection Pool.

## DB-SEED-019
Seeder dependencies serán explícitas.

## DB-SEED-020
Priority no sustituirá dependencies.

## DB-SEED-021
Dependency graph deberá ser validado.

## DB-SEED-022
Dependency cycles serán rechazados.

## DB-SEED-023
Planning será determinista.

## DB-SEED-024
Execution plan será immutable después de validación.

## DB-SEED-025
SeederContext será scoped.

## DB-SEED-026
No existirá current Seeder global mutable.

## DB-SEED-027
Environment será explícito.

## DB-SEED-028
Production safety será policy-driven.

## DB-SEED-029
Demo data estará bloqueada en producción por defecto.

## DB-SEED-030
Reference data podrá habilitarse en producción.

## DB-SEED-031
`--force` no anulará políticas no-overridable.

## DB-SEED-032
Seeder category no será igual a environment policy.

## DB-SEED-033
Repeatability será explícita.

## DB-SEED-034
REPEATABLE no significará idempotent.

## DB-SEED-035
IDEMPOTENT no dependerá únicamente de execution history.

## DB-SEED-036
ONCE no convertirá Seeder en Migration.

## DB-SEED-037
REPLACE será explícito.

## DB-SEED-038
Seeder history no será DB truth.

## DB-SEED-039
Seeder history no probará existencia actual de datos.

## DB-SEED-040
Seeder version será distinta de migration version.

## DB-SEED-041
Fingerprint incluirá únicamente dependencias semánticas conocidas.

## DB-SEED-042
Upsert no garantizará idempotencia completa.

## DB-SEED-043
Transaction policy será explícita.

## DB-SEED-044
PER_SEEDER será distinto de WHOLE_PLAN.

## DB-SEED-045
WHOLE_PLAN requerirá dominio transaccional compatible.

## DB-SEED-046
Cross-database execution no implicará atomicidad global.

## DB-SEED-047
Cross-shard execution no implicará atomicidad global.

## DB-SEED-048
Seeder no implementará distributed 2PC implícitamente.

## DB-SEED-049
Seeder no hará commit de transaction ajena.

## DB-SEED-050
Rollback solo se afirmará cuando sea confirmado.

## DB-SEED-051
Rollback DB no rebobinará object graph.

## DB-SEED-052
Compensation no será rollback.

## DB-SEED-053
External side effects no serán ACID por asociación.

## DB-SEED-054
Retry será policy-driven.

## DB-SEED-055
Retry no ocurrirá después de UNKNOWN commit por defecto.

## DB-SEED-056
Deadlock retry reutilizará Transaction Retry semantics.

## DB-SEED-057
No se repetirá arbitrariamente un statement de transaction abortada.

## DB-SEED-058
Execution tendrá identidad propia.

## DB-SEED-059
Cada Seeder run tendrá identidad propia.

## DB-SEED-060
Cancellation será cooperativa.

## DB-SEED-061
Cancellation no significará rollback.

## DB-SEED-062
Progress será bounded.

## DB-SEED-063
Resource budgets serán configurables.

## DB-SEED-064
Plan mode no fingirá conocer runtime behavior imposible de analizar.

## DB-SEED-065
Parameters serán validados.

## DB-SEED-066
Parameters CLI serán tratados como input no confiable.

## DB-SEED-067
Parameters no se interpolarán directamente en SQL.

## DB-SEED-068
Secrets no aparecerán en diagnostics.

## DB-SEED-069
No existirán passwords de producción inseguros por defecto.

## DB-SEED-070
Seeder classes deberán resolverse mediante registro/policy válida.

## DB-SEED-071
Package Seeders no se ejecutarán durante boot.

## DB-SEED-072
Seeder execution será explícita.

## DB-SEED-073
Migration y Seeder serán pasos independientes.

## DB-SEED-074
Schema requirements podrán validarse antes de execution.

## DB-SEED-075
Capabilities serán preferidas sobre vendor conditionals.

## DB-SEED-076
MySQL será soportable.

## DB-SEED-077
MariaDB será soportable.

## DB-SEED-078
PostgreSQL será soportable.

## DB-SEED-079
SQLite será soportable.

## DB-SEED-080
Platform-specific behavior será explícito.

## DB-SEED-081
Seeder target será logical database cuando sea posible.

## DB-SEED-082
ConnectionManager resolverá physical connection.

## DB-SEED-083
Seeder no elegirá Connection Pool internals.

## DB-SEED-084
Writes utilizarán writer semantics.

## DB-SEED-085
Seeder no enviará writes a replicas.

## DB-SEED-086
Read-after-write respetará consistency policy.

## DB-SEED-087
Shard scope será explícito.

## DB-SEED-088
UNKNOWN shard no significará ALL shards.

## DB-SEED-089
Global shard fan-out será explícito.

## DB-SEED-090
Global shard execution tendrá resource budget.

## DB-SEED-091
Shard map generation será respetada.

## DB-SEED-092
Tenant integration será opcional.

## DB-SEED-093
Tenant no será shard.

## DB-SEED-094
ALL tenants será explícito.

## DB-SEED-095
Tenant execution tendrá resource budget.

## DB-SEED-096
Tenant context será scoped.

## DB-SEED-097
Seeder mutations participarán en cache invalidation normal.

## DB-SEED-098
Seeder no publicará cache state antes de commit confirmado.

## DB-SEED-099
UNKNOWN outcome producirá tratamiento conservador de cache.

## DB-SEED-100
Seeder no administrará Entity Cache manualmente por defecto.

## DB-SEED-101
Events no redefinirán transaction outcome.

## DB-SEED-102
Telemetry estará correlacionada con ExecutionId.

## DB-SEED-103
Telemetry tendrá bounded cardinality.

## DB-SEED-104
Telemetry no incluirá PII por defecto.

## DB-SEED-105
Logs no expondrán secrets.

## DB-SEED-106
Query telemetry podrá correlacionarse con SeederRunId.

## DB-SEED-107
Rows affected podrá ser exact/estimated/unknown.

## DB-SEED-108
Unknown evidence no será convertida en exact count.

## DB-SEED-109
Seeder Repository será opcional.

## DB-SEED-110
Seeder Repository no será requisito de bootstrap.

## DB-SEED-111
Concurrent Seeder execution será gobernable.

## DB-SEED-112
Seeder lock no será DB pessimistic lock.

## DB-SEED-113
Distributed execution locks usarán ownership seguro.

## DB-SEED-114
DB constraints seguirán protegiendo invariantes.

## DB-SEED-115
Immutable Seeder metadata podrá compartirse entre workers.

## DB-SEED-116
Mutable Seeder state será scoped.

## DB-SEED-117
RandomSource será scoped.

## DB-SEED-118
FactoryContext será scoped.

## DB-SEED-119
EntityManager será scoped.

## DB-SEED-120
TransactionContext será scoped.

## DB-SEED-121
Progress state será scoped.

## DB-SEED-122
Cancellation state será scoped.

## DB-SEED-123
Worker reuse no implicará Seeder state reuse.

## DB-SEED-124
FrankenPHP será soportado.

## DB-SEED-125
RoadRunner será soportable.

## DB-SEED-126
OpenSwoole será soportable.

## DB-SEED-127
Coroutine executions estarán aisladas.

## DB-SEED-128
Seeder HTTP endpoint no existirá por defecto.

## DB-SEED-129
Remote execution requerirá Authentication.

## DB-SEED-130
Remote execution requerirá Authorization.

## DB-SEED-131
Sensitive Seeders podrán requerir Audit.

## DB-SEED-132
CLI plan será side-effect free en la medida definida por planner.

## DB-SEED-133
CLI explain será side-effect free.

## DB-SEED-134
Destructive operations tendrán nombres explícitos.

## DB-SEED-135
Tags no sustituirán dependencies.

## DB-SEED-136
Dependency expansion podrá incluir Seeders fuera del tag seleccionado.

## DB-SEED-137
Factories conservarán su propia semántica dentro de Seeder.

## DB-SEED-138
ModelFactory seguirá usando canonical ORM engine.

## DB-SEED-139
EntityFactory seguirá usando canonical ORM engine.

## DB-SEED-140
Seeder no creará un segundo Persistence Engine.

## DB-SEED-141
Seeder no creará un segundo Transaction Manager.

## DB-SEED-142
Seeder no creará un segundo Cache subsystem.

## DB-SEED-143
Seeder no creará un segundo Event subsystem.

## DB-SEED-144
Seeder failures conservarán original cause.

## DB-SEED-145
UNKNOWN permanecerá UNKNOWN.

## DB-SEED-146
PARTIAL permanecerá distinguible de FAILED.

## DB-SEED-147
SKIPPED permanecerá distinguible de SUCCEEDED.

## DB-SEED-148
Environment rejection ocurrirá antes de mutation cuando sea posible.

## DB-SEED-149
Dependency cycle rejection ocurrirá antes de mutation.

## DB-SEED-150
Invalid parameter rejection ocurrirá antes de mutation.

## DB-SEED-151
Transaction compatibility deberá validarse antes de execution cuando sea posible.

## DB-SEED-152
Distribution compatibility deberá validarse antes de execution cuando sea posible.

## DB-SEED-153
Seeder design favorecerá reproducibilidad.

## DB-SEED-154
Reproducibilidad no implicará igualdad cuando el Seeder declare randomness no determinista.

## DB-SEED-155
Clock podrá ser injectable.

## DB-SEED-156
RandomSource podrá ser seedable.

## DB-SEED-157
Seeder execution será observable.

## DB-SEED-158
Seeder execution será diagnosable.

## DB-SEED-159
Seeder execution será cancelable donde sea seguro.

## DB-SEED-160
Seeder seguirá siendo orchestration, no persistence infrastructure.

---

# 217. Modelo formal

Sea:

```text
S = conjunto de Seeders
D = relación de dependencia
E = environment
P = policies
C = database context
```

El planner produce:

```text
Plan = Plan(S, D, E, P, C)
```

tal que para:

```text
A depends on B
```

se cumple:

```text
Position(B) < Position(A)
```

---

# 218. Validez del plan

Formalmente:

```text
ValidPlan(P)
=
AcyclicDependencies
∧ EnvironmentCompatible
∧ TransactionCompatible
∧ DatabaseDomainCompatible
∧ DistributionCompatible
∧ ParametersValid
∧ SecurityPolicySatisfied
```

---

# 219. Execution outcome

Para un Seeder `S`:

```text
Outcome(S)
∈
{
    SUCCEEDED,
    FAILED,
    PARTIAL,
    UNKNOWN,
    CANCELLED,
    SKIPPED
}
```

La arquitectura nunca deberá aplicar:

```text
UNKNOWN → SUCCEEDED
```

por conveniencia.

---

# 220. Idempotencia formal

Para un Seeder idempotente ideal:

```text
Seed(Seed(DB))
≈
Seed(DB)
```

donde `≈` significa equivalencia lógica del dataset administrado por el Seeder, no necesariamente igualdad física de:

```text
timestamps
internal IDs
audit rows
storage layout
```

salvo que el contrato lo exija.

---

# 221. Pipeline completo

```text
CLI / Application / Deployment
             │
             ▼
      Seeder Selection
             │
             ▼
      Seeder Discovery
             │
             ▼
       Seeder Registry
             │
             ▼
   Dependency Expansion
             │
             ▼
      Seeder Planner
             │
      ┌──────┼────────┐
      │      │        │
      ▼      ▼        ▼
Environment Transaction Distribution
 Policy       Policy      Policy
      │      │        │
      └──────┼────────┘
             ▼
     Execution Plan
             │
             ▼
       Seeder Runner
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
    Factory     Direct APIs
       │           │
       └─────┬─────┘
             ▼
          ORM /
       Query Engine
             │
             ▼
       Transactions
             │
             ▼
     Connection Manager
             │
             ▼
          Driver
             │
             ▼
         Database
```

---

# 222. Arquitectura final del bloque hasta ahora

```text
                    FACTORY ENGINE
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
                Dataset Orchestration
                          │
                          ▼
                   Database APIs
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
      Model API      EntityManager    Query/Bulk
          │               │               │
          └───────────────┼───────────────┘
                          ▼
                  Persistence Layer
                          │
                          ▼
                       Database
```

---

# 223. Regla maestra final

> **Seeder describe y orquesta el conjunto de datos que debe poblarse; Factory construye instancias; ORM y Query Engine expresan las mutaciones; Transaction System gobierna atomicidad; Distribution determina el dominio físico; Cache responde únicamente después de resultados suficientemente confirmados; y Database sigue siendo la autoridad sobre el estado persistente.**

En forma compacta:

```text
Seeder
   ↓
Dataset Intent
   ↓
Factories / Explicit Data
   ↓
ORM / Query APIs
   ↓
Transaction
   ↓
Routing
   ↓
Execution
   ↓
Database
```

Nunca:

```text
Seeder
   ↓
invented SQL internals
```

ni:

```text
Seeder
   ↓
automatic schema evolution
```

---

# 224. Siguiente documento

```text
197_DATABASE_FIXTURE_SYSTEM.md
```

El siguiente documento definirá el sistema de **Fixtures reproducibles** para representar escenarios de datos conocidos, especialmente útiles en:

```text
integration tests
acceptance tests
ORM tests
repository tests
database tests
regression tests
```

La separación principal será:

```text
Factory
=
cómo construir datos

Seeder
=
qué dataset poblar

Fixture
=
qué estado reproducible necesita un escenario

Test Data Generator
=
cómo generar datasets especializados o masivos
```

y deberá establecerse especialmente:

```text
Fixture
≠
Seeder
≠
Factory
≠
Snapshot
≠
Database Dump
```

aunque VoltStack podrá permitir que una Fixture utilice Factories, Seeders y datasets declarativos como mecanismos internos controlados.