# 325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md

## 1. Propósito

Este documento define la arquitectura oficial de **VoltStack Database** para migrar aplicaciones que actualmente utilizan ORMs, capas de persistencia o abstracciones de base de datos externas hacia el subsistema:

```text
VoltStack/Quantum/Database
```

El objetivo no es construir una capa permanente de emulación de otros frameworks. El objetivo es ofrecer un proceso de transición **predecible, verificable, incremental y reversible** que permita adoptar progresivamente la arquitectura nativa de VoltStack.

Los principales escenarios contemplados son:

```text
Laravel Eloquent
Doctrine ORM / DBAL
PDO / SQL directo
Legacy Active Record
Legacy Data Mapper
Custom Repository Layers
Third-party PHP ORMs
```

La migración deberá preservar, en la medida técnicamente posible:

```text
Data Integrity
Behavioral Compatibility
Transaction Semantics
Query Semantics
Entity Identity
Relationship Semantics
Schema Integrity
Operational Continuity
```

---

## 2. Objetivos

El sistema deberá permitir:

- detectar automáticamente dependencias de un ORM externo;
- inventariar modelos, entidades, repositories y mappings;
- analizar esquemas existentes;
- convertir metadata cuando sea posible;
- migrar consultas gradualmente;
- preservar identificadores y relaciones;
- detectar diferencias semánticas;
- comparar resultados entre sistemas;
- ejecutar migraciones parciales;
- mantener temporalmente ambos sistemas;
- generar reportes de incompatibilidad;
- automatizar transformaciones seguras;
- impedir pérdida o corrupción silenciosa de datos;
- validar la aplicación antes de retirar el ORM anterior.

---

## 3. Principio arquitectónico

La migración se concibe como:

```text
External ORM
     │
     ▼
Discovery
     │
     ▼
Analysis
     │
     ▼
Intermediate Representation
     │
     ▼
Compatibility Assessment
     │
     ▼
Migration Plan
     │
     ▼
Code / Metadata Transformation
     │
     ▼
VoltStack Database
     │
     ▼
Verification
     │
     ▼
Legacy ORM Removal
```

La migración no debe ser una simple sustitución textual de APIs.

---

## 4. Regla fundamental

```text
Migration != API Renaming
```

Dos ORMs pueden representar conceptos similares con semánticas distintas.

Por ejemplo:

```text
Eloquent Model
Doctrine Entity
VoltStack Entity
```

pueden diferir en:

- lifecycle;
- identity management;
- lazy loading;
- dirty tracking;
- hydration;
- relationship ownership;
- cascading;
- transaction boundaries;
- serialization;
- event execution;
- persistence semantics.

Por ello, toda transformación deberá realizarse sobre una representación semántica intermedia.

---

## 5. Alcance

La arquitectura cubre:

```text
External Database Layer
│
├── Connection Configuration
├── Models / Entities
├── Metadata
├── Primary Keys
├── Relationships
├── Repositories
├── Query Builders
├── Raw SQL
├── Transactions
├── Migrations
├── Schema Definitions
├── Seeders
├── Factories
├── Events
├── Lifecycle Hooks
├── Casts / Types
├── Pagination
├── Soft Deletes
├── Timestamps
├── Optimistic Locking
├── Pessimistic Locking
├── Unit of Work
└── Custom Extensions
```

---

## 6. No objetivos

El sistema no pretende:

```text
VoltStack = Laravel compatibility clone
VoltStack = Doctrine compatibility clone
```

Tampoco se busca mantener indefinidamente:

```text
Eloquent APIs
Doctrine APIs
legacy annotations
external ORM runtime dependencies
```

Las capas de compatibilidad serán herramientas de transición.

---

## 7. Fuentes de migración

### 7.1 Laravel Eloquent

El sistema deberá reconocer conceptos como:

```text
Model
$table
$primaryKey
$fillable
$guarded
$casts
$hidden
$dates
timestamps
SoftDeletes
Scopes
Accessors
Mutators
hasOne
hasMany
belongsTo
belongsToMany
morphTo
morphMany
pivot tables
Eloquent Builder
```

---

## 8. Doctrine ORM

El sistema deberá reconocer:

```text
Entity
MappedSuperclass
Embeddable
Repository
EntityManager
UnitOfWork
IdentityMap
Attributes
Annotations
XML Mapping
YAML Mapping
OneToOne
OneToMany
ManyToOne
ManyToMany
Embeddables
LifecycleCallbacks
Value Generators
Inheritance Mapping
Discriminator Maps
```

---

## 9. Doctrine DBAL

Para aplicaciones que utilizan DBAL sin ORM:

```text
Connection
QueryBuilder
SchemaManager
Types
Platforms
Transactions
Prepared Statements
Result
```

pueden mapearse hacia componentes equivalentes de VoltStack Database.

---

## 10. PDO y SQL directo

También debe existir una ruta para:

```php
$pdo->prepare(...);
$statement->execute(...);
```

o:

```text
legacy database wrappers
custom query helpers
stored procedure clients
```

La migración podrá conservar SQL directo inicialmente y trasladar solamente:

```text
connection management
transactions
parameter binding
result handling
telemetry
```

---

## 11. Migration Manager

El componente principal será conceptualmente:

```text
ExternalOrmMigrationManager
```

Responsabilidades:

```text
discover
analyze
normalize
plan
transform
verify
report
```

Interfaz conceptual:

```php
interface ExternalOrmMigrationManagerInterface
{
    public function analyze(MigrationSource $source): MigrationAnalysis;

    public function plan(MigrationAnalysis $analysis): MigrationPlan;

    public function migrate(MigrationPlan $plan): MigrationResult;

    public function verify(MigrationResult $result): VerificationResult;
}
```

---

## 12. Arquitectura general

```text
┌──────────────────────────────────────────────────────┐
│             External ORM / Database Layer            │
├──────────────────────────────────────────────────────┤
│ Eloquent │ Doctrine │ DBAL │ PDO │ Custom ORM        │
└────┬─────────┬─────────┬──────┬───────────────┬──────┘
     │         │         │      │               │
     └─────────┴─────────┴──────┴───────────────┘
                         │
                         ▼
               Migration Discovery
                         │
                         ▼
                 Source Adapters
                         │
                         ▼
             Intermediate ORM Model
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Schema       Metadata     Queries
             │           │           │
             └───────────┼───────────┘
                         ▼
                Compatibility Engine
                         │
                         ▼
                   Migration Plan
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
            Code      Metadata    Config
              │          │          │
              └──────────┼──────────┘
                         ▼
                 VoltStack Database
                         │
                         ▼
                 Verification Engine
```

---

## 13. Source Adapter Architecture

Cada sistema externo deberá implementarse mediante un adaptador.

```php
interface OrmMigrationSourceAdapterInterface
{
    public function supports(MigrationSource $source): bool;

    public function inspect(MigrationSource $source): SourceModel;

    public function normalize(SourceModel $source): IntermediateDatabaseModel;
}
```

Implementaciones posibles:

```text
EloquentMigrationAdapter
DoctrineOrmMigrationAdapter
DoctrineDbalMigrationAdapter
PdoMigrationAdapter
CustomOrmMigrationAdapter
```

Esto evita introducir conocimiento específico de otros ORMs dentro del núcleo.

---

## 14. Intermediate Database Model

La pieza central será una representación neutral:

```text
IntermediateDatabaseModel
```

que puede contener:

```text
Connections
Entities
Fields
Types
Identifiers
Relationships
Indexes
Constraints
Repositories
Queries
Transactions
Lifecycle Hooks
Events
Migrations
Schema
Custom Behaviors
```

---

## 15. Modelo intermedio de entidad

Ejemplo conceptual:

```php
final class IntermediateEntity
{
    public string $name;

    public string $table;

    public array $fields;

    public array $identifiers;

    public array $relationships;

    public array $behaviors;
}
```

No debe contener clases específicas de Eloquent o Doctrine.

---

## 16. Intermediate Relationship Model

Las relaciones se normalizan:

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
POLYMORPHIC
EMBEDDED
```

Con metadata adicional:

```text
owner
inverse
foreign key
join table
cascade
nullable
fetch strategy
orphan removal
```

---

## 17. Intermediate Query Model

Las consultas que puedan analizarse se transformarán a un modelo neutral:

```text
Query
├── Select
├── Source
├── Join
├── Predicate
├── Group
├── Having
├── Order
├── Limit
└── Parameters
```

Este modelo puede integrarse con el Query AST nativo de VoltStack.

---

## 18. Discovery Engine

El sistema deberá detectar:

```text
composer dependencies
namespace usage
base classes
attributes
annotations
traits
configuration files
database migrations
repositories
query builders
raw SQL
facades
service container bindings
```

Ejemplo:

```text
composer.json
     │
     ▼
laravel/framework detected
     │
     ▼
Eloquent migration profile enabled
```

---

## 19. Static Analysis

La migración deberá preferir análisis estático antes que ejecución dinámica.

Podrá inspeccionar:

```text
PHP AST
attributes
docblocks
method calls
inheritance
traits
configuration
composer metadata
```

Esto permite identificar patrones sin ejecutar código de aplicación.

---

## 20. Runtime Analysis

Cuando el análisis estático no sea suficiente, podrá habilitarse instrumentación temporal.

Ejemplo:

```text
Application Runtime
       │
       ▼
Migration Observer
       │
       ├── executed queries
       ├── hydrated entities
       ├── transaction boundaries
       ├── lazy loads
       └── lifecycle events
```

La instrumentación no deberá modificar los resultados de negocio.

---

## 21. Inventario de migración

Antes de transformar código deberá generarse un inventario.

Ejemplo:

```text
ORM Migration Analysis

Models/Entities                148
Repositories                    37
Relationships                  426
Custom Types                    12
Scopes                          28
Lifecycle Hooks                 17
Raw SQL Queries                 63
Unsupported Constructs           4
Manual Review Required          11
```

---

## 22. Clasificación de compatibilidad

Cada elemento deberá clasificarse como:

```text
DIRECT
TRANSFORMABLE
ADAPTABLE
MANUAL
UNSUPPORTED
RISKY
```

### DIRECT

Existe equivalente semántico directo.

### TRANSFORMABLE

Puede convertirse automáticamente.

### ADAPTABLE

Requiere una capa temporal.

### MANUAL

Necesita intervención del desarrollador.

### UNSUPPORTED

No existe equivalente.

### RISKY

La conversión automática podría alterar comportamiento o datos.

---

## 23. Compatibility Matrix

Ejemplo conceptual:

| Source | VoltStack | Migration |
|---|---|---|
| Eloquent `belongsTo` | Many-to-One | Transformable |
| Eloquent `hasMany` | One-to-Many | Transformable |
| Doctrine `ManyToMany` | Many-to-Many | Transformable |
| Doctrine Embeddable | Embedded Value Object | Transformable |
| Raw SQL | Native SQL | Direct |
| Custom Eloquent Scope | Query Extension | Review |
| Custom Doctrine Type | Database Type | Adapter/Manual |
| Dynamic ORM Magic | Explicit API | Review |

La matriz real deberá ser versionada junto con Database.

---

## 24. Migración de configuración

Ejemplo Laravel:

```php
'default' => env('DB_CONNECTION', 'mysql')
```

podrá convertirse a la configuración de VoltStack:

```php
'database' => [
    'default' => env('DB_CONNECTION', 'mysql'),
]
```

Las credenciales no deberán copiarse a archivos generados si actualmente se obtienen del entorno.

---

## 25. Migración de conexiones

La migración deberá preservar:

```text
driver
host
port
database
username source
charset
collation
SSL settings
read/write topology
pool settings
```

Las diferencias deberán reportarse explícitamente.

---

## 26. Migración desde Eloquent

Ejemplo conceptual de origen:

```php
class User extends Model
{
    protected $table = 'users';

    protected $fillable = [
        'name',
        'email',
    ];

    protected $casts = [
        'active' => 'boolean',
    ];
}
```

El sistema deberá extraer:

```text
Entity: User
Table: users

Fields:
name
email
active:boolean
```

y generar la representación equivalente de VoltStack según la API final definida para entidades.

---

## 27. Relaciones Eloquent

Ejemplo:

```php
public function posts()
{
    return $this->hasMany(Post::class);
}
```

Normalización:

```text
Relationship
type: ONE_TO_MANY
source: User
target: Post
```

Posteriormente se genera metadata nativa de VoltStack.

---

## 28. Eloquent Scopes

Ejemplo:

```php
public function scopeActive($query)
{
    return $query->where('active', true);
}
```

Puede migrarse conceptualmente hacia:

```text
Repository Query Method
or
Reusable Query Specification
```

VoltStack no debe copiar obligatoriamente el mecanismo mágico `scopeX`.

---

## 29. Accessors y Mutators

Los accessors/mutators deberán clasificarse.

```text
database transformation
domain transformation
serialization transformation
presentation transformation
```

No todos pertenecen necesariamente al ORM.

La migración podrá recomendar trasladarlos a:

```text
Value Objects
Entity Methods
Casts
Transformers
Serializers
```

---

## 30. Eloquent Casts

Los casts estándar podrán mapearse:

```text
integer
boolean
float
decimal
string
array
json
date
datetime
immutable_datetime
enum
```

Los casts personalizados deberán analizarse individualmente.

---

## 31. Soft Deletes

Eloquent:

```php
use SoftDeletes;
```

deberá mapearse a la capacidad equivalente de VoltStack si está habilitada.

La migración debe conservar:

```text
deleted_at column
query filtering behavior
restore semantics
force delete semantics
```

---

## 32. Timestamps

La migración deberá detectar:

```text
created_at
updated_at
```

y configuración:

```php
public $timestamps = false;
```

No deberá asumir automáticamente que todas las tablas utilizan timestamps.

---

## 33. Polymorphic Relationships

Las relaciones polimórficas requieren tratamiento especial.

Ejemplo Eloquent:

```text
morphTo
morphOne
morphMany
morphToMany
morphedByMany
```

Deben marcarse al menos como:

```text
TRANSFORMABLE
or
MANUAL REVIEW
```

dependiendo de la estrategia polimórfica de VoltStack.

---

## 34. Migración desde Doctrine

Ejemplo:

```php
#[Entity]
#[Table(name: 'users')]
class User
{
    #[Id]
    #[Column(type: 'integer')]
    #[GeneratedValue]
    private int $id;
}
```

El adapter deberá convertir los atributos Doctrine a la representación intermedia antes de generar metadata VoltStack.

---

## 35. Doctrine Repositories

Ejemplo:

```php
class UserRepository extends ServiceEntityRepository
{
}
```

podrá convertirse hacia:

```text
VoltStack Repository
```

manteniendo métodos personalizados cuando no dependan directamente de APIs Doctrine.

---

## 36. Doctrine Unit of Work

Doctrine utiliza explícitamente conceptos como:

```text
EntityManager
UnitOfWork
IdentityMap
```

VoltStack deberá mapear estos conceptos hacia su propio:

```text
Entity Manager
Unit of Work
Identity Map
Persistence Context
```

sin reutilizar objetos internos de Doctrine.

---

## 37. Doctrine Lifecycle Callbacks

Ejemplo:

```text
PrePersist
PostPersist
PreUpdate
PostUpdate
PreRemove
PostRemove
```

deberá mapearse al sistema de lifecycle/events de VoltStack.

El orden de ejecución deberá verificarse.

---

## 38. Doctrine Custom Types

Los tipos personalizados deberán generar:

```text
Custom Type Migration Report
```

Ejemplo:

```text
money_type
uuid_binary
encrypted_string
json_document
```

El sistema podrá:

```text
map automatically
generate adapter
or require manual implementation
```

---

## 39. Doctrine Inheritance

Estrategias como:

```text
Single Table Inheritance
Joined Table Inheritance
Mapped Superclass
```

requieren validación específica.

No deberán transformarse automáticamente si VoltStack no puede garantizar semántica equivalente.

---

## 40. Schema Introspection

La migración no deberá confiar exclusivamente en código.

También deberá inspeccionar la base real:

```text
Database
   │
   ▼
Schema Introspector
   │
   ├── tables
   ├── columns
   ├── indexes
   ├── constraints
   ├── foreign keys
   ├── sequences
   ├── generated columns
   └── platform-specific metadata
```

---

## 41. Three-Way Comparison

Una capacidad importante será comparar:

```text
ORM Metadata
     +
Migration Files
     +
Actual Database Schema
```

Resultado:

```text
Expected
vs
Declared
vs
Actual
```

Esto permite detectar drift antes de migrar.

---

## 42. Schema Drift

Ejemplo:

```text
Doctrine metadata:
VARCHAR(255)

Migration:
VARCHAR(191)

Production database:
VARCHAR(128)
```

La herramienta deberá detener la automatización y solicitar revisión.

---

## 43. Migraciones existentes

Los archivos históricos de migración no necesitan convertirse siempre.

Estrategias:

```text
KEEP_HISTORY
BASELINE_CURRENT_SCHEMA
FULL_CONVERSION
```

### KEEP_HISTORY

Se conservan las migraciones antiguas y VoltStack administra únicamente las nuevas.

### BASELINE_CURRENT_SCHEMA

Se crea un baseline del esquema actual.

### FULL_CONVERSION

Se transforman migraciones antiguas cuando exista equivalencia segura.

---

## 44. Estrategia recomendada para sistemas grandes

Para aplicaciones maduras:

```text
Existing Migration History
        │
        ▼
Schema Baseline
        │
        ▼
VoltStack Migration Baseline
        │
        ▼
New migrations use VoltStack
```

Esto reduce riesgos innecesarios.

---

## 45. Migración de datos

Migrar ORM no significa necesariamente mover datos.

En el escenario común:

```text
Same Database
Same Tables
Same Data
Different Persistence Layer
```

Por lo tanto, la prioridad será cambiar la capa de acceso sin modificar datos.

---

## 46. Migración entre motores

Si además se cambia:

```text
MySQL → PostgreSQL
SQL Server → PostgreSQL
SQLite → MySQL
```

debe tratarse como un proyecto adicional:

```text
ORM Migration
+
Database Platform Migration
```

Los riesgos y verificaciones serán independientes.

---

## 47. Dual ORM Mode

VoltStack podrá permitir temporalmente:

```text
Application
│
├── Legacy ORM
│
└── VoltStack Database
```

Este modo permite migrar módulo por módulo.

---

## 48. Límites del Dual ORM

Dos ORMs no deberán administrar simultáneamente la misma entidad dentro de una misma unidad lógica de trabajo sin coordinación explícita.

Riesgos:

```text
stale entities
identity conflicts
double writes
lost updates
transaction mismatch
cache incoherence
```

---

## 49. Migration Boundary

La aplicación deberá definir fronteras claras.

Ejemplo:

```text
Users Module
    → VoltStack

Billing Module
    → Doctrine

Legacy Reports
    → PDO
```

Esto es preferible a mezclar ORMs dentro del mismo agregado.

---

## 50. Transaction Bridge

Durante transición puede ser necesario compartir una conexión o transacción.

Arquitectura conceptual:

```text
Transaction Coordinator
       │
       ├── Legacy ORM
       └── VoltStack Database
```

Esta capacidad debe considerarse avanzada y dependiente del driver.

No deberá prometer atomicidad cuando técnicamente no pueda garantizarse.

---

## 51. Read Migration

Una estrategia segura puede comenzar por lecturas:

```text
Legacy ORM
    │
    ├── writes
    └── reads
          ↓
       migrate
          ↓
VoltStack reads
```

Posteriormente:

```text
VoltStack reads + writes
```

---

## 52. Shadow Reads

Para validar consultas:

```text
Request
   │
   ├── Legacy ORM Query ──────► Result A
   │
   └── VoltStack Query ───────► Result B
                                  │
                                  ▼
                               Comparator
```

Solo el resultado primario se devuelve a la aplicación.

El resultado secundario se utiliza para validación.

---

## 53. Shadow Writes

Los shadow writes son considerablemente más peligrosos.

No deberán habilitarse por defecto.

Si se utilizan, deberán ejecutarse únicamente en:

```text
isolated environments
staging
replicas
disposable databases
```

salvo una estrategia explícita y validada de producción.

---

## 54. Result Comparator

El sistema deberá comparar:

```text
row count
identifiers
field values
types
nullability
ordering
pagination
relationship loading
aggregate results
```

considerando diferencias legítimas de representación.

---

## 55. Query Equivalence

Dos consultas no son equivalentes únicamente porque produzcan el mismo SQL.

La verificación debe considerar:

```text
parameters
binding types
transaction context
ordering
locking
pagination
hydration
result normalization
```

---

## 56. SQL Capture

Durante análisis podrá capturarse:

```text
Legacy ORM SQL
VoltStack SQL
```

para diagnóstico.

Ejemplo:

```text
Legacy:
SELECT * FROM users WHERE active = ?

VoltStack:
SELECT u.* FROM users AS u WHERE u.active = ?
```

Estas consultas pueden ser semánticamente equivalentes.

---

## 57. Query Fingerprinting

Para aplicaciones grandes se pueden generar fingerprints:

```text
SELECT users WHERE active = ?
→ QF-8A12C4
```

permitiendo comparar frecuencia y comportamiento entre runtimes.

---

## 58. Migración de repositories

El proceso recomendado:

```text
Legacy Repository
      │
      ▼
Method Inventory
      │
      ▼
Query Analysis
      │
      ▼
VoltStack Repository
      │
      ▼
Behavior Verification
```

Métodos de dominio no dependientes del ORM pueden preservarse.

---

## 59. Repository Compatibility Adapter

Temporalmente:

```php
final class LegacyUserRepositoryAdapter
{
    public function __construct(
        private UserRepository $repository
    ) {
    }
}
```

puede exponer la API antigua mientras internamente utiliza VoltStack.

Esta capa debe marcarse como temporal.

---

## 60. Migración de servicios

El sistema deberá localizar dependencias como:

```php
EntityManagerInterface
Model
DB facade
Doctrine Connection
RepositoryInterface
```

y proponer contratos nativos de VoltStack.

---

## 61. Container Integration

La migración puede actualizar bindings:

```text
Old ORM Service
      │
      ▼
Compatibility Binding
      │
      ▼
VoltStack Service
```

Esto se integra con:

```text
312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md
```

---

## 62. Framework Integration

Los adapters de migración deberán respetar:

```text
311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md
```

La lógica de migración no deberá acoplar Database al framework de origen.

---

## 63. Eventos

Los eventos deberán mapearse cuidadosamente.

Ejemplo:

```text
Eloquent creating
Eloquent created
Doctrine prePersist
Doctrine postPersist
```

no deberán asumirse equivalentes sin analizar el momento exacto en que ocurren.

---

## 64. Event Ordering

La migración deberá documentar diferencias:

```text
before validation
before SQL
after SQL
before commit
after commit
```

Una diferencia en orden puede modificar comportamiento de negocio.

---

## 65. Observers y Subscribers

Los observers/listeners externos deberán inventariarse.

Ejemplo:

```text
UserObserver
AuditSubscriber
DomainEventListener
```

La herramienta debe determinar si pertenecen a:

```text
ORM lifecycle
domain events
application events
infrastructure events
```

---

## 66. Caching

Debe analizarse:

```text
query cache
entity cache
second-level cache
repository cache
application cache
```

Durante Dual ORM Mode, compartir caches de entidades puede ser inseguro.

---

## 67. Cache Isolation

Recomendación durante migración:

```text
legacy.orm.*
voltstack.database.*
```

con namespaces separados hasta verificar coherencia.

---

## 68. Lazy Loading

El sistema deberá detectar dependencias implícitas de lazy loading.

Ejemplo:

```php
$user->posts
```

podría disparar SQL sin que el desarrollador lo perciba.

La migración debe identificar:

```text
N+1 patterns
lazy proxies
implicit relation access
serialization-triggered queries
```

---

## 69. Eager Loading

Las estrategias:

```text
with()
join fetch
explicit eager loading
batch loading
```

deberán traducirse hacia las capacidades nativas de VoltStack.

---

## 70. Pagination

La herramienta deberá verificar:

```text
offset pagination
cursor pagination
keyset pagination
total count semantics
ordering stability
```

antes de considerar una migración equivalente.

---

## 71. Locking

Deben detectarse:

```text
FOR UPDATE
shared locks
optimistic version fields
pessimistic locks
advisory locks
```

La migración no deberá degradar silenciosamente garantías de concurrencia.

---

## 72. Transactions

Se deberán comparar:

```text
begin
commit
rollback
nested transactions
savepoints
retry behavior
deadlock handling
isolation levels
```

Una diferencia deberá aparecer como riesgo de migración.

---

## 73. Type Conversion

Pipeline:

```text
External ORM Type
      │
      ▼
Intermediate Type
      │
      ▼
VoltStack Type
      │
      ▼
Platform SQL Type
```

Esto desacopla la migración del motor concreto.

---

## 74. Type Compatibility Matrix

Ejemplo:

```text
Doctrine integer
   ↓
INTEGER
   ↓
VoltStack IntegerType
```

```text
Eloquent datetime cast
   ↓
DATETIME
   ↓
VoltStack DateTimeType
```

Los tipos ambiguos deberán solicitar revisión.

---

## 75. Enums

La herramienta deberá diferenciar:

```text
PHP Enum
Database Native Enum
String-backed domain enum
Integer-backed enum
```

y evitar convertir automáticamente entre estrategias incompatibles.

---

## 76. UUID / ULID

Se deberán preservar:

```text
generation strategy
binary/string representation
database type
ordering semantics
```

especialmente cuando el ORM de origen utiliza extensiones.

---

## 77. Composite Keys

Las entidades con claves compuestas deberán marcarse para análisis especial.

```text
(id_a, id_b)
```

El sistema deberá comprobar que todas las operaciones nativas de VoltStack utilizadas soportan dicha estrategia.

---

## 78. Generated Values

Se deberán detectar:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE
UUID
ULID
database-generated
custom generator
```

y mapear la estrategia, no solamente el tipo de columna.

---

## 79. Naming Strategies

Doctrine o sistemas personalizados pueden utilizar estrategias automáticas:

```text
UserProfile → user_profile
```

VoltStack deberá resolver nombres físicos antes de generar metadata para evitar cambios accidentales de tabla o columna.

---

## 80. Naming Freeze

Durante migración se recomienda:

```text
freeze physical database names
```

Es decir, generar mappings explícitos aunque VoltStack pudiera inferirlos.

Esto reduce diferencias por convenciones.

---

## 81. Migración de migrations/seeds/factories

El sistema puede ofrecer transformadores independientes:

```text
MigrationConverter
SeederConverter
FactoryConverter
FixtureConverter
```

No deberán bloquear la migración principal del runtime si pueden mantenerse temporalmente.

---

## 82. Data Fixtures

Doctrine Fixtures y factories de Laravel pueden convertirse conceptualmente hacia:

```text
VoltStack Database Fixtures
VoltStack Factories
Seeders
```

La generación automática deberá preservar dependencias y orden.

---

## 83. CLI

Comando principal conceptual:

```bash
php volt database:migrate-orm
```

El primer paso deberá ser análisis, no modificación.

---

## 84. Analyze

```bash
php volt database:migrate-orm --analyze
```

Salida:

```text
Detected ORM: Doctrine ORM

Entities:              84
Repositories:          22
Relationships:        197
Custom Types:           7
Lifecycle Hooks:       13

Direct:               211
Transformable:         74
Manual Review:         19
Unsupported:            2
```

---

## 85. Source Selection

Ejemplo:

```bash
php volt database:migrate-orm --from=eloquent
```

o:

```bash
php volt database:migrate-orm --from=doctrine
```

También podrá existir autodetección.

---

## 86. Migration Plan

```bash
php volt database:migrate-orm --plan
```

Generará:

```text
database-migration-plan.json
```

o una representación equivalente.

El plan deberá ser inspeccionable antes de aplicarse.

---

## 87. Dry Run

```bash
php volt database:migrate-orm --dry-run
```

Nunca modificará código ni esquema.

---

## 88. Apply

```bash
php volt database:migrate-orm --apply
```

deberá requerir un plan previamente validado para migraciones complejas.

---

## 89. Scope

Podrá migrarse un módulo:

```bash
php volt database:migrate-orm \
    --from=eloquent \
    --module=Users
```

Esto favorece migraciones incrementales.

---

## 90. Entity Scope

También:

```bash
php volt database:migrate-orm \
    --entity=User
```

incluyendo sus dependencias cuando corresponda.

---

## 91. Report

El proceso deberá generar un reporte:

```text
ORM_MIGRATION_REPORT.md
```

con:

```text
source ORM
source version
target VoltStack version
analyzed files
transformed files
manual actions
risks
unsupported features
verification status
```

---

## 92. Manifest

Cada ejecución podrá generar:

```text
.orm-migration-manifest.json
```

para registrar:

```text
source hashes
generated files
migration rules
tool version
target version
timestamp
```

Esto facilita reproducibilidad.

---

## 93. Backup

Antes de transformaciones destructivas:

```text
Source Tree
   │
   ▼
Version Control Check
   │
   ▼
Backup / Snapshot Verification
   │
   ▼
Migration
```

La herramienta debe recomendar trabajar sobre una rama limpia.

---

## 94. No direct production mutation

El migrador de ORM no deberá modificar directamente una base de producción por defecto.

La migración principal es de código y metadata.

Los cambios de schema deberán pasar por el sistema normal de migrations.

---

## 95. Verification Engine

Después de la conversión:

```text
Static Verification
      │
      ▼
Schema Verification
      │
      ▼
Query Verification
      │
      ▼
Behavior Verification
      │
      ▼
Application Tests
```

---

## 96. Static Verification

Comprueba:

```text
unresolved imports
legacy ORM references
invalid metadata
missing repositories
unsupported types
broken relationships
```

---

## 97. Schema Verification

Comprueba que VoltStack espera el mismo esquema físico:

```text
tables
columns
types
indexes
foreign keys
constraints
defaults
```

No deberá ejecutar automáticamente cambios si el objetivo es únicamente sustituir ORM.

---

## 98. Data Verification

Podrán ejecutarse checks:

```text
row counts
primary key uniqueness
foreign key consistency
null constraints
sample checksums
aggregate comparisons
```

en entornos autorizados.

---

## 99. Behavioral Verification

Deberán probarse:

```text
CRUD
relationships
transactions
events
locking
pagination
soft delete
casts
custom types
```

---

## 100. Test Generation

La herramienta puede generar pruebas de caracterización antes de migrar.

Ejemplo:

```text
Legacy Behavior
      │
      ▼
Characterization Tests
      │
      ▼
Migration
      │
      ▼
Same Tests
```

Esto reduce dependencia de suposiciones sobre el ORM anterior.

---

## 101. Golden Master

Para consultas críticas puede utilizarse:

```text
Input Dataset
     │
     ▼
Legacy Result
     │
     ▼
Golden Snapshot
     │
     ▼
VoltStack Result Comparison
```

Los snapshots deben evitar datos sensibles.

---

## 102. Performance Baseline

Antes de migrar se recomienda capturar:

```text
query count
query latency
hydration time
memory usage
transaction duration
N+1 occurrences
```

Después se repite la medición.

---

## 103. Performance Regression Detection

Ejemplo:

```text
Legacy:
42 queries / request

VoltStack:
117 queries / request
```

Aunque los resultados sean correctos, la migración no debería considerarse completamente validada.

---

## 104. Telemetry Integration

El sistema se integra con:

```text
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
```

Métricas:

```text
database.orm_migration.legacy_queries
database.orm_migration.voltstack_queries
database.orm_migration.mismatch
database.orm_migration.manual_items
database.orm_migration.validation_failures
```

---

## 105. Logging

Canal recomendado:

```text
database.orm_migration
```

No deberán registrarse valores sensibles de parámetros sin sanitización.

---

## 106. Security

La migración debe proteger:

```text
credentials
connection strings
query parameters
PII
encrypted columns
secrets
production datasets
```

Los reportes deberán utilizar redacción automática.

---

## 107. Data Integrity First

Regla:

```text
Data Integrity
      >
Automatic Migration Coverage
```

Si una transformación no puede demostrarse segura:

```text
do not transform automatically
```

Debe producirse una acción manual.

---

## 108. Confidence Score interno

El motor puede manejar internamente un nivel de confianza para decidir si automatizar:

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

Este valor no sustituye la clasificación de compatibilidad.

Ejemplo:

```text
HIGH → automatic candidate
LOW  → manual review
```

---

## 109. Idempotencia

Ejecutar nuevamente el analizador no deberá producir modificaciones adicionales inesperadas.

Idealmente:

```text
Migration(Migration(Source))
=
Migration(Source)
```

para las transformaciones ya aplicadas.

---

## 110. Reversibilidad

Las transformaciones de código deberán poder revisarse mediante control de versiones.

Para operaciones generadas se recomienda registrar suficiente metadata para conocer:

```text
what changed
why
which rule
source location
target location
```

---

## 111. Incremental Migration

Ruta recomendada:

```text
Phase 1
Discovery

Phase 2
Infrastructure

Phase 3
Read paths

Phase 4
Repositories

Phase 5
Write paths

Phase 6
Transactions

Phase 7
Lifecycle/events

Phase 8
Legacy removal
```

---

## 112. Strangler Pattern

Para aplicaciones grandes puede utilizarse:

```text
Legacy Persistence
       │
       ├──────────────┐
       │              │
       ▼              ▼
Legacy Modules    Migrated Modules
                       │
                       ▼
                VoltStack Database
```

Con el tiempo:

```text
Legacy → 0
VoltStack → 100%
```

---

## 113. Migration Gateway

Una abstracción temporal puede aislar módulos:

```php
interface UserPersistenceGateway
{
    public function find(UserId $id): ?User;

    public function save(User $user): void;
}
```

Implementaciones:

```text
LegacyUserGateway
VoltStackUserGateway
```

Esto facilita el cambio progresivo.

---

## 114. Anti-Corruption Layer

Para sistemas con fuerte dependencia del ORM anterior:

```text
Domain
  │
  ▼
Persistence Contract
  │
  ▼
Anti-Corruption Layer
  │
  ├── Legacy ORM
  └── VoltStack
```

La capa debe eliminarse o simplificarse cuando finalice la transición.

---

## 115. Migración de Laravel Eloquent: estrategia recomendada

```text
Eloquent Application
       │
       ▼
Model Discovery
       │
       ▼
Table / Cast / Relation Extraction
       │
       ▼
Scope Analysis
       │
       ▼
Repository Boundary Introduction
       │
       ▼
VoltStack Metadata Generation
       │
       ▼
Query Migration
       │
       ▼
Behavior Verification
       │
       ▼
Remove Eloquent Dependency
```

---

## 116. Migración de Doctrine: estrategia recomendada

```text
Doctrine Application
       │
       ▼
Metadata Extraction
       │
       ▼
Entity / Repository Inventory
       │
       ▼
Type Analysis
       │
       ▼
UnitOfWork Semantics Analysis
       │
       ▼
VoltStack Metadata Generation
       │
       ▼
Repository Migration
       │
       ▼
Lifecycle Verification
       │
       ▼
Remove Doctrine Dependency
```

---

## 117. Diferencia clave Eloquent vs Doctrine

Conceptualmente:

```text
Eloquent
   → Active Record oriented

Doctrine
   → Data Mapper / Unit of Work oriented
```

VoltStack no debe aplicar exactamente la misma estrategia de migración a ambos.

---

## 118. Migración hacia el modelo VoltStack

La representación final deberá respetar la arquitectura definida por Database:

```text
Entity
   │
Metadata
   │
Repository
   │
Persistence Context
   │
Unit of Work
   │
Query AST
   │
SQL Compiler
   │
Driver
```

Los adapters externos terminan antes de esta arquitectura.

---

## 119. Boundary Rule

Regla crítica:

```text
External ORM concepts
        │
        ▼
Migration Layer
        │
        X
        │
VoltStack Database Core
```

El core no deberá importar:

```text
Illuminate\Database\*
Doctrine\ORM\*
Doctrine\DBAL\*
```

La dependencia siempre apunta desde el adapter hacia VoltStack.

---

## 120. Paquetes de migración

Los adapters pueden distribuirse separadamente:

```text
VoltStack/Database-Migration-Core

VoltStack/Database-Migration-Eloquent
VoltStack/Database-Migration-Doctrine
VoltStack/Database-Migration-DBAL
```

Esto evita aumentar el peso del runtime principal.

---

## 121. Dev-only Dependency

Los migradores deberían instalarse preferentemente como dependencias de desarrollo:

```bash
composer require --dev \
    voltstack/database-migration-eloquent
```

Una vez completada la transición:

```bash
composer remove \
    voltstack/database-migration-eloquent
```

---

## 122. Runtime Independence

Después de finalizar la migración:

```text
Application
     │
     ▼
VoltStack Database
```

no deberá requerir el ORM de origen ni el paquete migrador.

---

## 123. Version Compatibility

Cada adapter deberá declarar explícitamente las versiones soportadas.

Ejemplo conceptual:

```text
Eloquent Adapter
supports:
10.x
11.x
12.x
```

No deberán inferirse APIs de versiones no verificadas.

---

## 124. Migration Rule Registry

Las transformaciones estarán registradas:

```text
MigrationRuleRegistry
│
├── metadata rules
├── query rules
├── type rules
├── relationship rules
├── config rules
└── repository rules
```

---

## 125. Rule Versioning

Cada regla deberá tener:

```text
id
source platform
source version
target version
risk
transformer
validator
```

Ejemplo:

```text
VSDB-MIG-ELOQ-0042
```

---

## 126. Diagnostics

Los errores deberán ser accionables.

Ejemplo:

```text
[VSDB-MIG-ELOQ-0042]

Unable to automatically migrate relationship:

App\Models\Image::imageable()

Reason:
Polymorphic discriminator strategy cannot be inferred.

Action:
Define an explicit VoltStack polymorphic mapping.
```

---

## 127. Migration Documentation Links

Cada diagnóstico podrá enlazar a documentación correspondiente:

```text
docs/database/migration/eloquent/polymorphic-relations
```

sin depender de mensajes genéricos.

---

## 128. Manual Migration Queue

Los elementos no automatizados se agrupan:

```text
Manual Migration Queue

[1] Custom Doctrine Type: MoneyType
[2] Eloquent Morph Relation: imageable
[3] Dynamic table selection: TenantModel
[4] Raw SQL with vendor-specific syntax
```

El desarrollador podrá resolverlos uno por uno.

---

## 129. Progress Tracking

La herramienta podrá mostrar:

```text
Migration Progress

Analyzed          100%
Converted          81%
Verified           74%
Manual             17%
Blocked             2%
```

---

## 130. CI Integration

Durante transición:

```text
CI
│
├── Legacy Tests
├── VoltStack Tests
├── Migration Scanner
├── Schema Comparator
└── Deprecated ORM Reference Check
```

La política puede impedir introducir nuevas dependencias del ORM antiguo.

---

## 131. Legacy Dependency Gate

Ejemplo:

```text
Allowed Doctrine references:
127

Current:
125
```

El build falla si aumenta:

```text
128 → FAIL
```

Esto fuerza una migración monotónica.

---

## 132. Architecture Tests

Podrán definirse reglas:

```text
New modules MUST NOT depend on Doctrine.
New modules MUST NOT extend Eloquent Model.
```

Esto evita expandir deuda técnica durante la transición.

---

## 133. Migration Completion Criteria

La migración se considera terminada cuando:

```text
0 runtime dependencies on legacy ORM
0 legacy entity mappings
0 legacy query builders
0 legacy transaction managers
0 unresolved compatibility adapters
0 migration blockers
all verification suites pass
```

Puede mantenerse documentación histórica y código de migraciones antiguas si no forma parte del runtime.

---

## 134. Legacy Removal

Proceso:

```text
Verify
  │
  ▼
Remove Compatibility Adapters
  │
  ▼
Remove Source ORM Package
  │
  ▼
Regenerate Autoload
  │
  ▼
Run Static Analysis
  │
  ▼
Run Tests
  │
  ▼
Run Schema Verification
```

---

## 135. Post-Migration Audit

Después de retirar el ORM:

```text
dependency audit
query performance audit
transaction audit
schema audit
security audit
telemetry audit
```

---

## 136. Rollback Strategy

Si la migración falla:

```text
Code rollback
Configuration rollback
Dependency rollback
```

deberán ser posibles mediante control de versiones.

Si no hubo cambios de schema, el rollback resulta considerablemente más seguro.

---

## 137. Schema Change Separation

Principio recomendado:

```text
ORM Replacement
       ≠
Schema Redesign
```

Primero:

```text
same schema + new ORM
```

Después, en proyectos independientes:

```text
schema modernization
```

Esto reduce el número de variables durante la transición.

---

## 138. Domain Refactoring Separation

También:

```text
ORM Migration
       ≠
Domain Model Rewrite
```

Aunque la migración pueda revelar problemas arquitectónicos, no debe obligar a rediseñar simultáneamente todo el dominio.

---

## 139. Observability During Cutover

Durante el cambio final deberán observarse:

```text
query errors
deadlocks
timeouts
query count
latency
connection saturation
transaction rollback rate
data mismatch
```

---

## 140. FrankenPHP

VoltStack utiliza FrankenPHP como servidor de aplicación predeterminado.

La migración debe comprobar especialmente:

```text
persistent worker state
connection reuse
entity manager reset
unit of work reset
request context cleanup
transaction cleanup
```

Código proveniente de frameworks con ciclo request/process distinto podría contener supuestos incompatibles.

---

## 141. RoadRunner y OpenSwoole

Los mismos principios aplican a los paquetes oficiales de integración con:

```text
RoadRunner
OpenSwoole
```

El Migration Analyzer deberá detectar servicios ORM que retengan estado entre requests.

---

## 142. Persistent Runtime Audit

Se deberán detectar patrones como:

```text
static EntityManager
static Repository state
global connection state
unclosed transaction
persistent identity map
```

y marcarlos como riesgos.

---

## 143. Multi-Tenancy

Cuando el paquete opcional de Multitenancy esté instalado, la migración deberá considerar:

```text
tenant connection resolution
tenant schema
tenant database
shared database
tenant filters
tenant-aware repositories
```

La migración del ORM no deberá asumir una única conexión global.

---

## 144. SaaS Integration

El paquete oficial opcional SaaS podrá apoyarse en Database, pero el migrador deberá permanecer independiente de SaaS.

Si una aplicación externa implementa tenancy mediante el ORM anterior, esa lógica deberá identificarse explícitamente.

---

## 145. Security Model

La migración deberá respetar el modelo de seguridad de Database:

```text
parameter binding
SQL injection prevention
credential handling
least privilege
sensitive telemetry redaction
```

Una consulta legacy insegura no deberá considerarse automáticamente válida por el hecho de funcionar.

---

## 146. Validation Integration

Se integra con:

```text
317_DATABASE_VALIDATION_INTEGRATION_SYSTEM.md
```

pero no deberá confundir:

```text
database constraints
entity invariants
request validation
domain validation
```

---

## 147. Authentication / Authorization

La migración de entidades relacionadas con autenticación no deberá modificar por accidente:

```text
password hashes
remember tokens
MFA secrets
session identifiers
authorization relationships
```

La transformación de persistencia deberá preservar exactamente su almacenamiento salvo migración explícita separada.

---

## 148. Jobs y Queues

Se deberán detectar entidades serializadas dentro de jobs.

Riesgo:

```text
Job created with legacy entity
        │
        ▼
deployment
        │
        ▼
worker now expects VoltStack entity
```

Debe existir una estrategia de compatibilidad durante despliegues graduales.

---

## 149. Serialized ORM Objects

Como regla general, la migración deberá recomendar no serializar objetos internos del ORM.

Preferencia:

```text
Entity ID
Value Object
DTO
```

en lugar de proxies o entidades gestionadas.

---

## 150. Deployment Strategy

Para sistemas críticos:

```text
Analyze
   ↓
Prepare
   ↓
Deploy compatibility layer
   ↓
Migrate module
   ↓
Observe
   ↓
Expand
   ↓
Cut over
   ↓
Remove legacy
```

---

## 151. Canary Migration

Puede migrarse inicialmente un subconjunto controlado de tráfico cuando la arquitectura de la aplicación lo permita.

El sistema deberá comparar resultados sin comprometer consistencia.

---

## 152. Feature Flags

Los módulos pueden seleccionar runtime:

```php
'database.persistence.users' => 'voltstack',
'database.persistence.billing' => 'legacy',
```

Estas flags son temporales y deberán eliminarse al finalizar la migración.

---

## 153. Rollout State Machine

```text
LEGACY
  │
  ▼
SHADOW
  │
  ▼
PARTIAL
  │
  ▼
VOLTSTACK
  │
  ▼
LEGACY_REMOVED
```

El estado deberá poder auditarse por módulo.

---

## 154. Migration State Store

Para proyectos grandes puede existir:

```text
MigrationStateStore
```

con información sobre:

```text
modules
entities
rules
verification
blockers
cutover state
```

No deberá convertirse en dependencia de producción una vez terminada la migración.

---

## 155. Testing Strategy

La estrategia completa incluye:

```text
Unit Tests
Integration Tests
Repository Tests
Database Contract Tests
Schema Tests
Transaction Tests
Concurrency Tests
Performance Tests
Migration Tests
```

---

## 156. Contract Tests

Una misma suite podrá ejecutarse contra:

```text
Legacy Persistence Adapter
VoltStack Persistence Adapter
```

Ejemplo:

```text
UserRepositoryContract
```

Esto facilita comparar comportamiento.

---

## 157. Test Database

Las migraciones automáticas y shadow writes deberán utilizar preferentemente:

```text
ephemeral database
containerized database
snapshot
staging clone
```

No datos productivos reales salvo proceso expresamente autorizado.

---

## 158. Fixtures de equivalencia

Se recomienda un dataset determinista:

```text
MigrationFixtureSet
```

que cubra:

```text
nulls
boundaries
relationships
unicode
dates
decimals
large values
soft deletes
concurrency cases
```

---

## 159. Decimal Precision

Especial atención a:

```text
money
decimal
numeric
float
```

No deberá aceptarse equivalencia si cambia:

```text
precision
scale
rounding
```

---

## 160. Date and Time

La migración deberá verificar:

```text
timezone
mutable vs immutable
database timezone
application timezone
microseconds
date-only fields
```

---

## 161. Boolean Semantics

Motores distintos pueden representar booleanos como:

```text
BOOLEAN
TINYINT
INTEGER
CHAR
```

El modelo intermedio deberá preservar la semántica lógica.

---

## 162. JSON

Debe analizarse:

```text
JSON
JSONB
TEXT serialized JSON
custom serialized structures
```

y evitar cambiar formato de almacenamiento implícitamente.

---

## 163. Serialization

Campos serializados por PHP u otros mecanismos deberán identificarse como riesgo.

La migración del ORM no debe cambiar automáticamente el codec de datos persistidos.

---

## 164. Encryption

Campos cifrados requieren:

```text
same cipher
same key source
same encoding
same rotation semantics
```

o una migración criptográfica explícita separada.

---

## 165. Generated Columns

Las columnas calculadas por base de datos deberán preservarse como:

```text
read-only/generated
```

cuando corresponda.

---

## 166. Database Triggers

Los triggers deberán formar parte del inventario de esquema.

Un ORM puede depender indirectamente de ellos para:

```text
timestamps
audit
IDs
derived values
```

---

## 167. Stored Procedures

Las aplicaciones que dependan de:

```text
stored procedures
functions
packages
```

deberán conservar llamadas nativas mientras no exista abstracción equivalente.

VoltStack no deberá impedir SQL específico de plataforma cuando sea necesario.

---

## 168. Vendor-Specific SQL

Debe clasificarse:

```text
portable
platform-specific
unsupported
```

La migración de ORM no implica obligatoriamente portabilidad entre motores.

---

## 169. Error Mapping

Los errores externos deberán mapearse cuidadosamente.

Ejemplo:

```text
Doctrine UniqueConstraintViolationException
```

podría convertirse hacia una excepción equivalente de VoltStack.

La aplicación que capture clases concretas deberá actualizarse.

---

## 170. Exception Inventory

El analyzer deberá localizar:

```php
catch (DoctrineException $e)
```

o excepciones de Eloquent/DBAL.

Estas dependencias forman parte del alcance de migración.

---

## 171. Migration Exceptions

VoltStack puede definir:

```text
MigrationException
UnsupportedMappingException
AmbiguousRelationException
SchemaMismatchException
BehaviorMismatchException
MigrationVerificationException
```

---

## 172. Developer Experience

La herramienta deberá responder:

```text
What was detected?
What can be migrated automatically?
What cannot?
Why?
Where is it?
What should replace it?
How can it be verified?
```

---

## 173. Generated Code Policy

El código generado deberá:

- seguir las convenciones de VoltStack;
- ser legible;
- poder modificarse manualmente;
- evitar comentarios innecesarios;
- no ocultar lógica crítica;
- identificar únicamente los TODO realmente necesarios.

---

## 174. No Black Box Migration

Nunca deberá existir:

```text
"Migration completed successfully"
```

sin evidencia verificable.

El resultado deberá mostrar:

```text
converted
verified
unverified
manual
unsupported
```

por separado.

---

## 175. Migration Report Example

```text
VoltStack Database ORM Migration

Source:
Laravel Eloquent 12.x

Target:
VoltStack Database 1.x

Models analyzed:            126
Relations analyzed:         341
Queries analyzed:           518

Automatically converted:    842
Manual review:               37
Unsupported:                  3

Schema differences:           0
Behavior mismatches:           2

Status:
MIGRATION NOT YET VERIFIED
```

---

## 176. Definition of Safe Automation

Una regla podrá ejecutarse automáticamente cuando:

```text
source semantics known
target semantics known
mapping unambiguous
schema impact known
validation available
```

Si cualquiera de estos elementos falta:

```text
manual review
```

---

## 177. Extensibilidad

Terceros podrán implementar adapters:

```php
final class PropelMigrationAdapter
    implements OrmMigrationSourceAdapterInterface
{
}
```

sin modificar el core.

---

## 178. Adapter Certification

Los adapters oficiales podrán disponer de suites de conformidad:

```text
Migration Adapter Contract Tests
```

que validen:

```text
discovery
normalization
diagnostics
version handling
security
idempotence
```

---

## 179. API Stability

Las APIs del Migration Core deberán seguir:

```text
323_DATABASE_VERSIONING_SYSTEM.md
324_DATABASE_DEPRECATION_POLICY.md
```

Los adapters externos no deben depender de clases internas.

---

## 180. Deprecación del propio migrador

Una regla antigua también puede deprecarse.

Ejemplo:

```text
VSDB-MIG-DOC-0014
deprecated because Doctrine mapping semantics changed
```

Las ejecuciones deberán registrar la versión exacta del rule set.

---

## 181. Reproducibilidad

Un reporte debe permitir conocer:

```text
source ORM version
adapter version
VoltStack version
PHP version
database platform
migration rule set
configuration profile
```

---

## 182. Determinismo

Con el mismo:

```text
source
configuration
tool version
rule set
```

el migrador deberá producir resultados equivalentes.

---

## 183. Architecture Decision

VoltStack adoptará una arquitectura de migración basada en:

```text
Source Adapter
      │
      ▼
Intermediate Model
      │
      ▼
Compatibility Engine
      │
      ▼
Migration Rules
      │
      ▼
Native VoltStack Model
```

y no una colección de reemplazos específicos dispersos.

---

## 184. Decisiones arquitectónicas

### Decisión 1

La migración desde ORMs externos será una capacidad separada del runtime principal de Database.

### Decisión 2

VoltStack utilizará adapters específicos por ORM.

### Decisión 3

Los adapters convertirán primero hacia un modelo intermedio neutral.

### Decisión 4

El core de Database no dependerá de Eloquent, Doctrine ni otros ORMs.

### Decisión 5

Las migraciones deberán poder ejecutarse incrementalmente por módulo o entidad.

### Decisión 6

VoltStack permitirá temporalmente Dual ORM Mode cuando sea necesario.

### Decisión 7

La integridad de datos tendrá prioridad sobre el porcentaje de automatización.

### Decisión 8

Los cambios de ORM y los rediseños de schema deberán separarse por defecto.

### Decisión 9

Las transformaciones ambiguas deberán requerir revisión manual.

### Decisión 10

El sistema proporcionará análisis, planificación, dry-run, transformación y verificación como fases separadas.

### Decisión 11

Los migradores serán preferentemente dependencias de desarrollo removibles.

### Decisión 12

El sistema deberá soportar pruebas de equivalencia entre el runtime anterior y VoltStack.

### Decisión 13

Las aplicaciones con workers persistentes deberán pasar auditorías específicas de estado y lifecycle.

### Decisión 14

Los adapters y reglas de migración estarán versionados.

### Decisión 15

Una migración no se considerará completada hasta eliminar las dependencias runtime del ORM anterior y superar las verificaciones definidas.

---

## 185. Arquitectura final

```text
                    External Persistence Systems
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
    Eloquent               Doctrine              Custom ORM
       │                      │                      │
       ▼                      ▼                      ▼
 Eloquent Adapter       Doctrine Adapter        Custom Adapter
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              ▼
                 Intermediate Database Model
                              │
                              ▼
                    Compatibility Engine
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
          Direct Rules    Adapters       Manual Queue
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                       Migration Plan
                              │
                              ▼
                      Transformation
                              │
                              ▼
                    VoltStack Database
                              │
                              ▼
                     Verification Engine
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
            Schema          Behavior       Performance
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                         Cutover
                              │
                              ▼
                    Legacy ORM Removal
```

---

## 186. Flujo operativo recomendado

```text
1. Detect external ORM
            │
            ▼
2. Build inventory
            │
            ▼
3. Inspect actual schema
            │
            ▼
4. Normalize source metadata
            │
            ▼
5. Generate compatibility matrix
            │
            ▼
6. Produce migration plan
            │
            ▼
7. Create characterization tests
            │
            ▼
8. Transform one bounded module
            │
            ▼
9. Compare behavior
            │
            ▼
10. Measure performance
            │
            ▼
11. Expand migration
            │
            ▼
12. Cut over
            │
            ▼
13. Remove legacy ORM
            │
            ▼
14. Post-migration audit
```

---

## 187. Relación con la comparación Laravel / Symfony

La estrategia reconoce dos modelos particularmente relevantes para VoltStack.

Laravel/Eloquent aporta una experiencia de desarrollo altamente productiva y un modelo Active Record ampliamente utilizado en aplicaciones PHP.

Symfony suele integrarse con Doctrine, cuyo modelo Data Mapper, Unit of Work, metadata explícita e Identity Map ofrece una separación más fuerte entre dominio y persistencia.

VoltStack no debe exigir que una aplicación proveniente de uno de estos ecosistemas adopte instantáneamente la filosofía del otro.

El Migration System funcionará como puente:

```text
Laravel / Eloquent
        │
        ├───────────┐
        │           │
        ▼           ▼
Intermediate Migration Model
                    ▲
        │           │
        └───────────┤
                    │
             Symfony / Doctrine
                    │
                    ▼
            VoltStack Database
```

La arquitectura final seguirá siendo propia de VoltStack.

---

## 188. Resultado esperado

Una aplicación existente podrá pasar de:

```text
Application
    │
    ▼
External ORM
    │
    ▼
Database
```

a:

```text
Application
    │
    ├── Legacy ORM
    │
    └── VoltStack Database
```

durante transición, y finalmente:

```text
Application
    │
    ▼
VoltStack Database
    │
    ▼
Driver / SQL Compiler
    │
    ▼
Database Platform
```

sin requerir una migración monolítica.

---

## 189. Principio final

El sistema seguirá el principio:

```text
Discover
   ↓
Understand
   ↓
Normalize
   ↓
Compare
   ↓
Plan
   ↓
Transform
   ↓
Verify
   ↓
Cut Over
   ↓
Remove Legacy
```

VoltStack no considerará una migración correcta simplemente porque el código compile.

La migración será correcta cuando:

```text
Schema Integrity
+
Data Integrity
+
Behavioral Equivalence
+
Transaction Correctness
+
Operational Stability
+
Acceptable Performance
```

hayan sido verificadas para el alcance definido.

---

## 190. Conclusión

`DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM` convierte la adopción de VoltStack Database en un proceso arquitectónico controlado en lugar de una reescritura obligatoria.

La combinación de adapters específicos, representación intermedia, análisis estático, inspección de schema, migración incremental, Dual ORM Mode, pruebas de equivalencia, telemetry y eliminación progresiva de dependencias permitirá incorporar aplicaciones provenientes de Laravel/Eloquent, Symfony/Doctrine y otras capas de persistencia sin contaminar el núcleo de VoltStack con compatibilidad permanente.

El objetivo final permanece claro:

```text
External ORM knowledge exists in migration tooling.

VoltStack Database remains native, independent
and architecturally coherent.
```

---

**Documento:** `325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Estado:** Architectural Specification
